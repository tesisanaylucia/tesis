# Fase 5 — Capa conversacional y WhatsApp (backend) — flujos conversacionales principales (TASK-50)

## Qué se implementó

Se implementaron los cinco flujos conversacionales principales que el
ticket enumera —reservar, confirmar, reprogramar, cancelar y consultar
turnos—, junto con los dos pasos previos comunes a todos ellos: la
identificación del paciente por número de documento y la verificación del
consentimiento antes de reservar.

En una arquitectura de *function calling* como la que TASK-47 y TASK-48
dejaron construida, "implementar un flujo" no significa escribir una
máquina de estados por flujo dentro del orquestador. El flujo lo conduce
el propio modelo, y lo que el sistema aporta son tres cosas: el manual de
procedimiento que el modelo lee en cada turno, las herramientas sin las
cuales ese procedimiento no puede ejecutarse, y la aplicación
determinística de aquellos pasos que el SRS atribuye al sistema y no al
bot. Esta tarea aportó las tres.

El manual quedó en `CONVERSATION_FLOWS_PROMPT`, un texto en español que
`ChatbotConfigService` anexa al system prompt de cada turno, con la
secuencia de pasos y herramientas de cada uno de los cinco flujos, las
dos etapas previas y un conjunto de reglas generales de conversación. Del
lado de las herramientas se agregó `list_professionals` y se reformularon
`check_consent` y `list_patient_appointments`. Del lado determinístico se
agregó `applyFlowPostConditions`, que garantiza el ofrecimiento de
reprogramación posterior a una cancelación.

## Decisiones y por qué

**El manual de flujos se anexa al system prompt en lugar de incorporarse
a la plantilla base que el inquilino puede sobrescribir.** TASK-48 dejó
el system prompt como una fila configurable de `OrganizationConfig`, con
un texto base por defecto. Incorporar allí los cinco flujos habría
significado que una clínica que personaliza el tono de su bot —el caso
que esa configuración existe para servir— borre sin advertencia el
procedimiento completo de reserva. La división que se adoptó separa por
naturaleza y no por conveniencia: el texto base describe rol, tono y
límites del bot, que es exactamente la parte que una marca blanca tiene
razones legítimas para reescribir; el manual de flujos describe el
procedimiento que imponen las herramientas del propio sistema, y un
inquilino que lo reescribiera no estaría personalizando la reserva sino
rompiéndola. Es el mismo criterio que TASK-49 aplicó a los guardrails
—reglas globales, no desactivables por configuración— trasladado del
plano de la seguridad al del procedimiento.

**El system prompt informa la fecha del día en cada turno.** No estaba
pedido en el ticket y se detectó al escribir el manual: la herramienta de
disponibilidad recibe un rango de fechas en formato AAAA-MM-DD, y un
modelo de lenguaje no tiene reloj. Sin ese dato, ninguna expresión como
"la semana que viene" o "el lunes" —que es como un paciente pide un turno
por WhatsApp— puede traducirse a un rango concreto, y el flujo de reserva
resulta inejecutable en la práctica aunque todas sus piezas existan. La
fecha se resuelve por turno y no al construir la constante, porque un
prompt armado una sola vez al arrancar el proceso le informaría a una
conversación de mañana la fecha de hoy. Se reutilizó `todayCalendarDate`,
la misma función con la que el resto del sistema escribe un día
calendario, en lugar de formatear la fecha aquí por segunda vez.

**Se agregó la herramienta `list_professionals`, que el catálogo de
TASK-47 no incluía.** Es la primera pieza que faltaba para que el flujo
de reserva pudiera ejecutarse: tanto la consulta de disponibilidad como
la reserva reciben el identificador del profesional, y hasta esta tarea
el modelo no tenía forma de obtener uno que no fuera inventándolo. La
herramienta delega en `ProfessionalsService.findAllActive`, la misma
lectura que ya servía `GET /profesionales` y cuyo propio comentario, de
la fase de Profesionales, anticipaba este consumidor. Al leer a través
del cliente de Prisma acotado por inquilino, el listado queda confinado a
la organización de la sesión por construcción, que es la respuesta
concreta al requisito de marca blanca del ticket.

**La herramienta de profesionales no recibe parámetros y no devuelve la
especialidad.** El SRS es explícito en que la especialidad no modifica el
flujo de reserva y que el paciente elige al profesional con independencia
de ella, y el ticket lo repite. Una alternativa era devolver la
especialidad y pedirle al manual de flujos que no filtrara por ella; se
descartó porque un filtro que la herramienta no puede expresar es un
filtro que el modelo no puede aplicar, garantía más fuerte que una
instrucción en el prompt. Lo que devuelve es exactamente lo que el SRS
dice que el chatbot muestra —nombre y matrículas— más el identificador
que las herramientas de reserva necesitan como asa. Que la clínica tenga
un puñado de profesionales (cuatro psiquiatras y una psicóloga, según el
propio SRS) hace que devolverlos todos no tenga costo y que el modelo
pueda emparejar el nombre tal como el paciente lo escribió.

**La verificación de consentimiento devuelve, en el mismo resultado, el
texto que hay que enviarle al paciente cuando no hay aceptación
registrada.** El ticket describe el flujo como "verificar con
`verificarConsentimiento`; si no tiene consentimiento aceptado, enviar el
texto de la plantilla SOLICITUD_CONSENTIMIENTO". Se consideró resolverlo
con una herramienta separada que devolviera esa plantilla, y se descartó:
son un solo paso, y un modelo que tuviera que acordarse de pedir la
redacción por separado tarde o temprano la improvisaría. Tratándose del
consentimiento de la Ley 25.326, la redacción es precisamente lo que no
puede improvisarse. Devolverlo junto con la verificación hace imposible
obtener una cosa sin la otra. El texto se renderiza con
`NotificationTemplateService` —la misma plantilla, y por lo tanto la
misma personalización por inquilino, que usan los avisos de WhatsApp de
la fase anterior—, y sólo en la rama que lo necesita: un paciente que ya
consintió, que es el caso frecuente porque el consentimiento se pide una
única vez por persona, no paga ni la lectura del paciente ni la búsqueda
de la plantilla.

**El celular de un paciente nuevo no se le pide: se toma del contacto de
WhatsApp desde el que escribe.** El enunciado del ticket lo listaba entre
los datos a solicitar, pero el número ya está en la conversación —la
propia clave de sesión que TASK-48 definió es organización más celular—,
de modo que pedírselo obliga al paciente a retipear un dato que ya dio
implícitamente al escribir, con la posibilidad de equivocarse al hacerlo
y de que la clínica termine con un número por el que no puede
contactarlo. El parámetro se quitó del esquema de la herramienta de alta,
no sólo de las instrucciones del manual: un dato que la herramienta no
admite es un dato que el modelo no puede inventar ni malinterpretar,
mismo criterio estructural aplicado a la especialidad en el listado de
profesionales. Para que la herramienta pueda leerlo se agregó
`ConversationContextService`, el equivalente por turno de
`TenantContextService` —el mismo `AsyncLocalStorage`, abierto por el
orquestador justo dentro del contexto de inquilino—, y
`mobilePhoneFromSessionId` en el archivo que ya era dueño del formato de
la clave de sesión. Se descartó agregar un cuarto parámetro a `procesar`
con el celular: duplicaría un dato que la clave de sesión ya transporta,
con la posibilidad de que ambos discrepen sobre quién está escribiendo.
Se descartó también interpolar el número en el system prompt para que el
modelo lo pasara como argumento, por la misma razón por la que el texto
del consentimiento viaja en el resultado de la herramienta y no en una
llamada aparte: lo que puede resolverse de forma determinística no se
delega en el modelo. Una sesión cuya clave no tenga la forma canónica
—una prueba unitaria, o un canal futuro que no sea WhatsApp— registra al
paciente sin celular en lugar de hacer fallar el turno entero: la columna
es opcional y un alta hecha por un administrativo que no tiene el número
ya es un caso legítimo. El número se escribe únicamente en el alta y
nunca sobre un paciente ya registrado: reescribir el dato de contacto de
una ficha existente con cualquier número que escriba sería una
actualización destructiva silenciosa que el SRS no pide —lo que pide es
una actualización de datos recién después de un año sin concurrir—.

**La consulta de turnos del paciente pasó a listar sólo los turnos
activos y futuros, con el nombre del profesional de cada uno.** Antes
delegaba en el listado general de turnos, que devuelve todos los estados
y todo el historial. Los cuatro flujos que parten de ella —consultar,
confirmar, reprogramar y cancelar— necesitan lo mismo y sólo eso: los
turnos sobre los que el paciente todavía puede actuar. Un turno cancelado
o ya transcurrido no es ninguna de esas cosas, y ofrecérselo al modelo
sólo abre la posibilidad de que el bot le proponga al paciente cancelar
el turno de la semana pasada. Se agregó `listActiveForPatient` al
servicio de turnos en lugar de ampliar el DTO de consulta del listado
general, porque las dos preguntas difieren en algo que ese DTO no puede
expresar sin desnaturalizarse: "activo" es un *conjunto* de estados,
mientras que el filtro existente admite exactamente uno, y el piso
temporal es el instante actual y no un día calendario.

**El conjunto de estados activos se deriva de la máquina de estados en
lugar de volver a enumerarse.** `ACTIVE_APPOINTMENT_STATUSES` se calcula
como "los estados desde los que todavía es alcanzable la cancelación",
sobre la misma tabla de transiciones que TASK-38 definió. Hoy eso da
reservado y confirmado, igual que una lista escrita a mano, pero una
lista escrita a mano puede quedar desfasada de la máquina de estados sin
que nada lo advierta, y el sistema ya tenía tres lugares distintos
enumerando ese mismo par de estados por su cuenta.

**El ofrecimiento de reprogramación posterior a una cancelación se aplica
de forma determinística, además de pedirse en el manual de flujos.** El
SRS lo redacta como una acción del sistema —"ante una cancelación, el
sistema le preguntará al paciente si desea reprogramar"—, no como una
sugerencia de estilo para el bot, y el criterio de aceptación del ticket
pide una prueba de que el bot efectivamente pregunta. Una prueba con el
puerto de IA simulado no puede demostrar nada sobre un texto que la
propia prueba redacta, de modo que dejar el ofrecimiento enteramente en
manos del modelo habría hecho que ese criterio no fuera verificable. La
función `applyFlowPostConditions` se aplica al texto final del turno, en
el mismo punto y por la misma razón que los guardrails de TASK-49: una
regla que debe cumplirse para todos los pacientes no puede depender de
que el modelo haya seguido el prompt esta vez. Si la respuesta ya ofrece
la reprogramación con palabras propias del modelo —lo cual es preferible
a una frase agregada al final— se la deja intacta.

**Esa post-condición se decide por las herramientas que tuvieron éxito en
el turno, no por el texto de la respuesta.** Una cancelación rechazada
por falta de anticipación —la ventana de cuatro horas del SRS, que ya
vivía en el servicio de turnos desde TASK-38— deja el turno en pie: no
hay nada que reprogramar y ofrecerlo sería incoherente. El orquestador
acumula los nombres de las herramientas que devolvieron éxito durante el
turno y esa es la entrada de la regla. Por la misma lógica, una respuesta
bloqueada por un guardrail nunca recibe el agregado: el mensaje canónico
es todo lo que el paciente puede leer, y anexarle una pregunta de flujo
reabriría la conversación que la regla acaba de cerrar.

**Los nombres de las herramientas se centralizaron en un enum
(`ChatbotToolName`).** Hasta esta tarea cada nombre era un literal dentro
de la definición de su herramienta, lo cual bastaba mientras nadie lo
leyera desde afuera. La post-condición de flujo necesita preguntar si se
canceló un turno en este turno de conversación, y hacerlo contra un
literal repetido significa que un error de tipeo no falla al compilar
sino que, simplemente, nunca coincide. Se siguió la forma que el proyecto
ya usa para las claves de plantilla de notificación.

**La ventana de cuatro horas no se reimplementó ni se duplicó en la capa
del chatbot.** El ticket pide "verificar 4h de anticipación (si < 4h,
informar que no puede cancelarse online)", lo que admitiría leerse como
un chequeo previo en el flujo. Se optó por no agregarlo: la regla ya está
en `AppointmentsService.cancel`, ya es configurable por inquilino, y su
mensaje de error llega al modelo como el resultado fallido de la
herramienta gracias al mecanismo de conversión de excepciones de
TASK-47. El manual de flujos le indica al bot qué decirle al paciente
ante ese fallo. Un chequeo previo en la capa conversacional habría sido
una segunda copia de la misma regla, con la posibilidad de discrepar de
la primera y sin ninguna capacidad de impedir nada que la primera no
impidiera ya.

**Se agregó `defineInputlessTool` para las herramientas sin parámetros.**
`list_professionals` es la primera del catálogo que no recibe ninguno, y
no podía definirse con un DTO vacío: la opción `forbidUnknownValues` de
class-validator —justamente la que impide que un campo alucinado por el
modelo llegue a un servicio— rechaza todo objeto cuya clase no registre
metadatos de validación, de modo que un DTO vacío habría hecho fallar
cada llamada. La verificación que queda es que el modelo no haya mandado
nada, y un argumento inventado se le informa por nombre en lugar de
descartarse en silencio, con el mismo criterio que la validación
existente. Ambas variantes comparten el resto del andamiaje —actor
SYSTEM, conversión de errores— en una función privada común.

## Alternativas descartadas

- **Escribir una máquina de estados por flujo dentro del orquestador**:
  descartada porque duplicaría, en código imperativo, la conducción que
  el modelo ya hace con el catálogo de herramientas, y anularía la razón
  por la que se eligió *function calling* en primer lugar.
- **Incorporar los cinco flujos a la plantilla base del system prompt
  configurable por inquilino**: descartada porque una clínica que
  personalizara el tono de su bot borraría el procedimiento de reserva
  sin advertencia.
- **Devolver la especialidad en el listado de profesionales y pedirle al
  prompt que no filtre por ella**: descartada en favor de no exponer el
  dato, garantía estructural en lugar de una instrucción que el modelo
  puede desoír.
- **Una herramienta separada que devolviera el texto de la plantilla de
  consentimiento**: descartada por permitir que el modelo verifique el
  consentimiento sin obtener la redacción y termine improvisándola.
- **Ampliar el DTO del listado general de turnos para admitir un conjunto
  de estados y un piso temporal**: descartada por cargar el contrato
  público de `GET /turnos` con parámetros que existen para un único
  consumidor interno.
- **Verificar la ventana de cuatro horas también en la capa del
  chatbot**: descartada por constituir una segunda copia de una regla que
  ya se aplica, sin capacidad de impedir nada adicional.
- **Dejar el ofrecimiento de reprogramación enteramente a cargo del
  modelo**: descartada porque el SRS lo atribuye al sistema y porque el
  criterio de aceptación quedaría sin forma de verificarse con el puerto
  de IA simulado.
- **Reescribir la respuesta del modelo cuando ya menciona la
  reprogramación**: descartada; si el modelo hizo el ofrecimiento con sus
  propias palabras, esa redacción es mejor que una frase anexada.
- **Pedirle el celular al paciente, como sugería el enunciado del
  ticket**: descartada; el número es el del contacto de WhatsApp desde el
  que escribe y ya está en la clave de sesión.
- **Agregar un cuarto parámetro con el celular a la operación del
  orquestador**: descartada por duplicar un dato que la clave de sesión ya
  transporta, con posibilidad de discrepancia entre ambos.
- **Interpolar el celular en el system prompt para que el modelo lo pasara
  como argumento de la herramienta de alta**: descartada por delegar en el
  modelo algo que el sistema puede resolver de forma determinística.
- **Actualizar el celular de un paciente ya registrado con el número desde
  el que escribe**: descartada por constituir una escritura destructiva
  silenciosa sobre la ficha de contacto de la clínica.

## Entidades / puertos / adaptadores tocados

- `src/chatbot/conversation-flows.constants.ts` (nuevo):
  `CONVERSATION_FLOWS_PROMPT` —el manual de los cinco flujos, la
  identificación por documento y la verificación de consentimiento—,
  `RESCHEDULE_OFFER_MESSAGE` y el patrón que detecta un ofrecimiento ya
  hecho.
- `src/chatbot/conversation-flows.ts` (nuevo):
  `applyFlowPostConditions`, función pura.
- `src/chatbot/chatbot-tool-names.ts` (nuevo): enum `ChatbotToolName` con
  los doce nombres del catálogo.
- `src/chatbot/conversation-context.service.ts` (nuevo):
  `ConversationContextService`, el contexto por turno que transporta el
  celular de WhatsApp del paciente que escribe.
- `src/chatbot/session-id.ts` (modificado): `mobilePhoneFromSessionId`,
  inversa de `buildSessionId`.
- `src/chatbot/tools/patient.tools.ts` (modificado):
  `find_or_create_patient` deja de aceptar `mobilePhone` y lo toma del
  contexto de conversación al dar de alta.
- `src/chatbot/tools/professional.tools.ts` (nuevo): `ProfessionalTools`
  y la herramienta `list_professionals`.
- `src/chatbot/tools/consent.tools.ts` (modificado): `check_consent`
  devuelve `{accepted}` o `{accepted, consentRequestMessage}`; incorpora
  `PatientsService` y `NotificationTemplateService`.
- `src/chatbot/tools/appointment.tools.ts` (modificado):
  `list_patient_appointments` delega en `listActiveForPatient`.
- `src/chatbot/define-tool.ts` (modificado): `defineInputlessTool` junto
  a `defineTool`, sobre un constructor privado compartido.
- `src/chatbot/validate-tool-input.ts` (modificado): `rejectAnyToolInput`
  y el tipo `ToolInputValidation` extraído.
- `src/chatbot/chatbot-config.service.ts` (modificado): el system prompt
  se compone como plantilla del inquilino más manual de flujos, con la
  fecha del día interpolada.
- `src/chatbot/orquestador.service.ts` (modificado): abre el contexto de
  conversación dentro del de inquilino, acumula las herramientas exitosas
  del turno y aplica las post-condiciones de flujo sobre el texto final ya
  validado por los guardrails.
- `src/chatbot/chatbot-tools.service.ts` y `chatbot.module.ts`
  (modificados): registro de `ProfessionalTools`; importación de
  `ProfessionalsModule` y `NotificationsModule`.
- `src/appointments/appointments.service.ts` (modificado):
  `listActiveForPatient`.
- `src/appointments/appointment.presenter.ts` (modificado):
  `patientAgendaSelect` y su presentador.
- `src/appointments/appointment-transition.rule.ts` (modificado):
  `ACTIVE_APPOINTMENT_STATUSES`, derivado de la tabla de transiciones.
- Ningún modelo de Prisma ni migración nueva: esta tarea no agrega
  entidades ni columnas, y tampoco endpoints HTTP —el orquestador sigue
  sin controlador propio hasta el webhook de WhatsApp (TASK-53)—.

## Tests y qué validan

- `test/chatbot-flows.e2e-spec.ts` (nuevo): nueve pruebas de conversación
  contra la base de datos real, con el puerto de IA reemplazado por un
  guion de llamadas a herramientas y todo lo demás real (orquestador,
  catálogo de herramientas, servicios de dominio, Postgres). Cubren la
  reserva completa de un paciente nunca visto —identificación, alta con
  los datos faltantes, consentimiento y reserva, verificando en la base
  el paciente, el consentimiento y el turno resultantes—; el
  consentimiento ya registrado, que no se vuelve a pedir; la confirmación
  del turno; la reprogramación a una franja libre; la cancelación con su
  ofrecimiento de reprogramación; la cancelación rechazada por
  anticipación menor a cuatro horas, con el turno intacto y sin
  ofrecimiento; el listado de turnos activos, que excluye el cancelado y
  el ya transcurrido; el aislamiento por inquilino del listado de
  profesionales; y la presencia del manual de flujos y de la fecha del
  día en el prompt de cada turno. El doble del puerto de IA verifica
  además que ninguna herramienta haya fallado de forma inadvertida: un
  guion cuyas herramientas fallaran todas, de otro modo, seguiría
  pasando.
- `conversation-flows.spec.ts` (nuevo): el ofrecimiento se agrega tras
  una cancelación exitosa; se omite si la respuesta ya lo hizo, con o sin
  tildes; se emite solo si el modelo no respondió nada; y no se emite si
  ninguna cancelación tuvo éxito en el turno.
- `tools/professional.tools.spec.ts` (nuevo): la forma reducida de la
  respuesta —identificador, nombre y matrículas, sin especialidad ni
  configuración interna— y el rechazo nominado de un parámetro inventado.
- `orquestador.service.spec.ts` (ampliado): el ofrecimiento llega tanto
  al valor devuelto como al historial de la sesión; no se emite cuando la
  cancelación fue rechazada; y nunca se anexa a una respuesta bloqueada
  por un guardrail.
- `tools/consent.tools.spec.ts` (reescrito): el caso ya aceptado no
  renderiza plantilla ni lee el paciente; el caso sin aceptación devuelve
  el texto renderizado con el nombre del paciente.
- `chatbot-config.service.spec.ts` (ampliado): el manual de flujos se
  anexa incluso cuando el inquilino sobrescribió el texto base, y el
  prompt informa la fecha del día sin dejar marcadores sin resolver.
- `appointment-transition.rule.spec.ts` (ampliado): el conjunto de
  estados activos coincide, estado por estado, con aquellos desde los que
  la cancelación es alcanzable.
- `tools/patient.tools.spec.ts` (ampliado): el alta registra el celular
  del turno; sin número en la sesión, el paciente se crea sin celular en
  lugar de fallar; un `mobilePhone` que el modelo intente pasar se rechaza
  por nombre.
- `session-id.spec.ts` (ampliado): constructor y lector de la clave de
  sesión son inversas exactas; una clave sin mitad de teléfono devuelve
  indefinido.
- `orquestador.service.spec.ts` (ampliado, además de lo anterior): el
  celular de la sesión queda abierto como contexto durante toda la
  ejecución de las herramientas y cerrado al terminar el turno.
- `tools/appointment.tools.spec.ts` y `validate-tool-input.spec.ts`
  (ampliados): delegación en `listActiveForPatient`; rechazo de
  parámetros en una herramienta sin parámetros.
- Suite unitaria completa en verde (63 suites, 614 pruebas) y suite e2e
  completa en verde ejecutada en serie (40 suites, 469 pruebas), más
  `tsc --noEmit` sin errores y ESLint sin advertencias. Nota de proceso:
  la suite e2e ejecutada en paralelo arroja fallas por interferencia
  entre suites sobre la base de datos compartida, condición preexistente
  y ajena a esta tarea —verificada ejecutando la suite completa en
  paralelo sobre la rama base, donde las fallas son más numerosas que con
  estos cambios aplicados—.

## Figuras pendientes

- Diagrama de secuencia del flujo de reserva completo, desde la
  identificación por documento hasta la confirmación del resumen del
  turno, señalando en qué punto se intercala la verificación de
  consentimiento y qué ocurre en cada una de sus dos ramas. Sección 3.2.5
  Capa conversacional y WhatsApp.
- Diagrama comparativo de los cuatro flujos que parten de la consulta de
  turnos activos del paciente (consultar, confirmar, reprogramar,
  cancelar), señalando el tramo común y el punto en que cada uno se
  bifurca, y las dos salidas de la cancelación (rechazo por anticipación
  insuficiente frente a cancelación efectiva seguida del ofrecimiento de
  reprogramación). Sección 3.2.5 Capa conversacional y WhatsApp.
- Diagrama de la composición del system prompt de un turno (texto base
  del inquilino o plantilla por defecto, con el nombre de la organización
  interpolado, más el manual de flujos no sobrescribible, con la fecha
  del día interpolada), señalando qué mitad es configurable y cuál no.
  Sección 3.2.5 Capa conversacional y WhatsApp.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-50-conversational-flows`, creada
  desde `origin/main` como pide la usuaria. Nota de proceso: `main` local
  estaba un merge detrás de `origin/main` —le faltaba el de TASK-49, la
  capa de guardrails sobre la que esta tarea se apoya— al comenzar la
  sesión, de modo que la rama se creó explícitamente desde `origin/main`.
  Sin commit ni push al momento de escribir esta entrada: pendiente de
  autorización explícita de la usuaria.
- Ticket: TASK-50 (Jira), "P5.5 – Flujos conversacionales principales",
  bajo el épico TASK-8 (Módulo 5). Fuente de verdad consultada en Drive:
  el documento de Especificación de Requisitos de Software vigente
  (Anexo, "Módulo turnos" y "Módulo pacientes"), del que se tomaron
  literalmente cuatro puntos: que la especialidad no modifica el flujo de
  reserva y el paciente elige al profesional con independencia de ella;
  que el chatbot muestra el nombre del profesional y sus dos matrículas;
  que ante una cancelación el sistema le pregunta al paciente si desea
  reprogramar; y que el consentimiento sólo se solicita si no se registró
  previamente su aceptación.
- Discrepancia registrada entre el ticket y la fuente de verdad, resuelta
  a favor de esta última: el ticket dice que, ante un paciente no
  registrado, "el bot solicita nombre, apellido y celular para crear el
  registro", mientras que el SRS establece como datos obligatorios para
  la reserva el documento, la fecha de nacimiento y el contacto de
  emergencia, y el alta de paciente implementada en la fase de Pacientes
  exige nombre, apellido, fecha de nacimiento y contacto de emergencia.
  El manual de flujos pide esos cuatro datos y no se relajó el alta para
  ajustarla a la enumeración del ticket, porque un paciente creado con
  menos datos no podría reservar, que es justamente lo que el flujo
  intenta hacer a continuación. El celular, en cambio, dejó de pedirse por
  completo: la usuaria precisó durante la sesión que se guarda del
  contacto de WhatsApp que pide el turno, dato que la clave de sesión ya
  transporta.
- Dependencias declaradas: TASK-48 (P5.3, orquestador) y TASK-47 (P5.2,
  herramientas), ambas ya fusionadas; TASK-49 (P5.4, guardrails),
  fusionada a `origin/main` durante la sesión anterior. Fuera de alcance:
  la integración real con WhatsApp (P5.8, TASK-53), que será el primer
  llamador real de `procesar`.
