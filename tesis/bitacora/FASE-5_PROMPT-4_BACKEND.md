# Fase 5 — Capa conversacional y WhatsApp (backend) — capa de validación de guardrails (TASK-49)

## Qué se implementó

Se implementó `GuardrailService`, la capa determinística que revisa cada
turno de la conversación en los dos puntos que el ticket pide: el mensaje
entrante del paciente, antes de que el orquestador llame al modelo o a
cualquier herramienta (`checkIncomingMessage`), y la respuesta final en
texto plano del modelo, antes de que se guarde en el historial o se
devuelva (`checkResponse`). Cubre las cinco reglas del ticket: sin
observaciones del profesional, sin monto de copago, sin derivación a un
operador humano, mensaje fijo ante urgencias o pedidos fuera de alcance, y
sin datos de pacientes de otros profesionales. `OrquestadorService` quedó
modificado para invocar ambos métodos en los puntos exactos que el propio
comentario de su clase, escrito durante TASK-48, ya dejaba anotados como
pendientes para esta tarea.

## Decisiones y por qué

**`GuardrailService` no depende de `ConfigTenantService` ni de ninguna
fila de `OrganizationConfig`.** A diferencia de `ChatbotConfigService`
(TASK-48), cuyos dos valores son explícitamente configurables por
inquilino, el ticket de esta tarea es igual de explícito en el sentido
contrario: "los guardrails son globales... no pueden ser desactivados por
la configuración del tenant". Es una excepción deliberada, y documentada
como tal en `guardrail.constants.ts`, a la instrucción general de
CLAUDE.md de que las reglas de negocio viven en configuración de
inquilino y no en constantes de código — la misma clase de excepción que
ya tienen los topes anti-abuso: esto es un piso de seguridad, no una
preferencia de la clínica.

**El chequeo del mensaje entrante corta el turno antes de llamar a
`AIPort` o a cualquier herramienta, no sólo antes de ejecutar las
herramientas que el modelo pida.** El criterio de aceptación del ticket
lo dice de forma literal: "el paciente escribe 'tengo una emergencia' →
el guardrail retorna el mensaje de urgencias sin llamar tools". Se
interpretó eso de la forma más estricta posible — ni siquiera se llama al
modelo — por dos razones: una urgencia detectada merece la respuesta más
rápida y predecible que el sistema pueda dar, y no hay ninguna razón para
gastar una llamada al modelo cuando la respuesta ya está decidida de
antemano por una regla determinística. `runTurn` arma primero el
historial de la sesión (igual que antes), corta ahí mismo si
`checkIncomingMessage` devuelve texto, y sólo si no hay corte arma la
lista de herramientas y entra al ciclo original de TASK-48.

**La respuesta que efectivamente queda guardada en el historial de la
sesión es la ya filtrada por `checkResponse`, nunca la original.** Si una
respuesta bloqueada volviera a entrar al historial tal cual la generó el
modelo, el próximo turno de esa sesión arrancaría desde una transcripción
donde el propio modelo "cree" haber dicho ya el monto de copago o haber
derivado a la secretaria — exactamente lo que el guardrail existe para
evitar. Por eso `runTurn` reemplaza `response.text` por
`checkResponse(...).text` antes de empujarlo al arreglo `messages` y de
llamar a `ConversationSessionStore.save`, no después.

**Una regla que dispara reemplaza la respuesta completa, no sólo el
fragmento que la violó.** El ticket describe la regla de copago como
"bloquear esa parte y responder con..." lo cual admitiría una lectura de
redacción parcial (dejar el resto de la oración intacta y sólo tapar el
monto), pero el propio criterio de aceptación usa el singular sobre el
todo — "el modelo retorna 'el copago es $500' → el guardrail **la**
reemplaza con el mensaje estándar" —, y redactar un fragmento dentro de
una oración generada por un modelo de lenguaje arriesga dejar una
respuesta gramaticalmente incoherente, peor para un paciente que una
respuesta corta y clara. Se optó por el reemplazo completo de la
respuesta para las cuatro reglas que aplican sobre la respuesta saliente.

**Las reglas 1 (observaciones) y 5 (datos de otro paciente) comparten un
mismo texto de reemplazo; las reglas 3 (derivar a humano) y 4 (urgencias
y fuera de alcance) comparten otro.** El ticket no da una redacción
exacta para el mensaje "genérico" de la regla 1 ni para la regla 5, pero
sí la da, textual, para las reglas 2 y 4, y pide explícitamente para la
regla 3 "reemplazar con la respuesta de fuera de alcance" — es decir, el
mismo texto que la regla 4. Se decidió no inventar una redacción distinta
para cada una de las dos reglas sin texto propio: 1 y 5 son, en el fondo,
la misma clase de falla (una respuesta a punto de revelar un dato que
pertenece al registro interno de la clínica, no a este paciente), y un
único mensaje de "no puedo compartir eso" es más fácil de interpretar
para un paciente real que dos negativas sutilmente distintas para lo que
en la práctica es el mismo problema.

**La regla 5 (sin datos de pacientes de otros profesionales) se implementó
como dos sub-chequeos de texto deterministas — un número con forma de DNI,
y una referencia en tercera persona a un paciente nombrado —, no como una
comprobación semántica de identidad.** El ticket describe la regla en
términos de "el profesional de la sesión activa", pero ese concepto —
saber a qué profesional pertenece la conversación en curso, o qué
paciente es el que está escribiendo — no existe todavía en ningún lugar
del sistema al que este guardrail tenga acceso: `GuardrailService`, por
diseño del propio ticket ("analiza el texto generado por el modelo"),
sólo ve el texto de la respuesta, no el `patientId` ni el `professionalId`
que las herramientas usaron para producirla. Introducir esa noción aquí
habría significado tomar, dentro de esta tarea, una decisión de diseño
que le corresponde a los flujos de negocio de una fase posterior (P5.5,
TASK-50 — la que efectivamente resuelve "de qué paciente y de qué
profesional es esta conversación"). Se optó entonces por un resguardo de
defensa en profundidad, coherente con el resto de las reglas: un número
de 7 u 8 dígitos sueltos (la forma de un DNI argentino, ver
`DNI_PATTERN` en `patient-fields.decorators.ts`) nunca tiene una razón
legítima de aparecer en una respuesta del chatbot, y una respuesta que
habla en tercera persona de "el turno de \<Nombre\>" en lugar de en
segunda persona ("tu turno") es, en sí misma, una señal de que se está
describiendo el registro de alguien más.

**No se modificó el comportamiento existente de `ToolCallLimitExceededError`
frente al agotamiento de las diez vueltas del ciclo de herramientas.** El
ticket menciona, como parte de la regla 4, que el bot debe responder el
mensaje de fuera de alcance "ante urgencias... o el modelo no puede
resolverlo con las tools disponibles", lo que podría leerse como un
pedido de atrapar `ToolCallLimitExceededError` y devolver el texto
canónico en lugar de dejar que la excepción se propague. Se decidió no
tocar ese camino: TASK-48 ya cubre exactamente ese escenario — un modelo
que no puede resolver el pedido con las herramientas disponibles termina,
por construcción, en el tope de iteraciones —, con una prueba de
aceptación propia que hoy espera la excepción y con una razón ya
documentada en esa misma tarea (el historial de un turno atascado
deliberadamente no se guarda, para que el próximo mensaje no repita el
ciclo trabado). Decidir qué llega a leer el paciente ante esa excepción
sigue siendo, como ya anotaba la entrada de TASK-48, una decisión del
futuro webhook de WhatsApp (TASK-53), que todavía no existe como
llamador real de `procesar`.

**Se extrajo `stripDiacritics` a `src/common/text/normalize-text.ts`,
compartida con `FaqService`.** El chequeo de palabras clave de
`GuardrailService` (urgencias, copago, derivación a humano, observaciones)
necesita exactamente la misma normalización — minúsculas, sin tildes —
que `faq-similarity.ts` ya tenía escrita puertas adentro para su propio
puntaje de Jaccard, sólo que esta última además fragmenta el texto en
palabras, lo cual habría destruido la puntuación y el espaciado que las
expresiones regulares de frase de `GuardrailService` necesitan conservar
(por ejemplo, distinguir "$500" o la posición de un nombre propio). En
lugar de duplicar la expresión regular de tildes combinantes en un
segundo módulo, se separó el primer paso (`stripDiacritics`) a una
ubicación compartida bajo `common/`, y `normalizeToWords` pasó a
construirse sobre esa función en lugar de repetirla — mismo criterio ya
aplicado en TASK-48 al trasladar la sustitución de `{placeholder}` a
`common/templates/`.

**Un error real de la primera versión de los patrones de la regla 3 se
encontró recién al escribir el test con el ejemplo del propio ticket.**
Los tres patrones de "derivar a humano" se escribieron primero admitiendo
sólo un artículo indefinido antes del sustantivo ("hablá con **un**
operador"), calcado del primer ejemplo del ticket. El test con el
segundo ejemplo del propio ticket — "hablá con **la** secretaria" — no
matcheaba: el artículo definido nunca se había contemplado. Corregido
ampliando el grupo de artículo opcional a `un|una|el|la` en los tres
patrones que lo usan. Se registra explícitamente porque es exactamente el
tipo de error que una prueba basada en los propios ejemplos del ticket
existe para atrapar, y no se habría encontrado con un ejemplo inventado
que casualmente sólo usara "un".

## Alternativas descartadas

- **Redactar sólo el fragmento ofensor de la respuesta, dejando el resto
  intacto**: descartada por las razones dadas arriba — el criterio de
  aceptación usa el singular sobre toda la respuesta, y una redacción
  parcial arriesga una oración incoherente.
- **Atrapar `ToolCallLimitExceededError` en el orquestador y devolver el
  mensaje de fuera de alcance en su lugar**: descartada — ese camino ya
  tiene un contrato probado desde TASK-48 (excepción, historial no
  guardado) y decidir qué llega a leer el paciente ante él es una decisión
  reservada al futuro webhook de WhatsApp, no de esta tarea.
- **Pasarle a `GuardrailService` un contexto explícito de paciente/
  profesional activo para la regla 5, en lugar de los dos sub-chequeos de
  texto**: descartada porque ese concepto no existe todavía en el sistema
  — resolver "de qué paciente y de qué profesional es esta conversación"
  es, por diseño del propio ticket, alcance de P5.5 (TASK-50).
- **Leer las cinco reglas o sus textos de reemplazo desde
  `ConfigTenantService`, con el mismo mecanismo que `ChatbotConfigService`
  usa para el system prompt**: descartada porque el ticket pide
  explícitamente que los guardrails sean globales y no desactivables por
  inquilino.
- **Textos de reemplazo distintos para las reglas 1 y 5, y para las reglas
  3 y 4**: descartada en favor de reusar un único texto por par, dado que
  el ticket no da una redacción propia para 1 y 5, y sí pide
  explícitamente el mismo texto para 3 que para 4.

## Entidades / puertos / adaptadores tocados

- `src/chatbot/guardrail.constants.ts` (nuevo): los cinco textos/patrones
  de regla como datos — `URGENCY_KEYWORDS`, `COPAY_TRIGGER_WORDS` +
  `DOLLAR_AMOUNT_PATTERN` + `BARE_AMOUNT_PATTERN`,
  `HUMAN_HANDOFF_PATTERNS`, `OBSERVATIONS_LEAK_PATTERN`,
  `DNI_LEAK_PATTERN` + `OTHER_PATIENT_REFERENCE_PATTERN` — y los tres
  textos canónicos de reemplazo (`OUT_OF_SCOPE_MESSAGE`,
  `COPAY_BLOCKED_MESSAGE`, `GENERIC_BLOCKED_MESSAGE`).
- `src/chatbot/guardrail.service.ts` (nuevo): `GuardrailService`,
  `checkIncomingMessage` y `checkResponse`.
- `src/chatbot/orquestador.service.ts` (modificado): `runTurn` invoca
  ambos métodos de `GuardrailService` en los dos puntos descriptos
  arriba.
- `src/chatbot/chatbot.module.ts` (modificado): agrega y exporta
  `GuardrailService`.
- `src/common/text/normalize-text.ts` (nuevo): `stripDiacritics`,
  extraída de `faq-similarity.ts`.
- `src/faq/faq-similarity.ts` (modificado): `normalizeToWords` pasa a
  construirse sobre `stripDiacritics` compartida, sin cambio de
  comportamiento.
- Ningún modelo de Prisma ni migración nueva — esta tarea no toca la base
  de datos: `GuardrailService` es puro y sin estado, coherente con la
  decisión de que las reglas son globales y no configurables.

## Tests y qué validan

- `guardrail.service.spec.ts` (nuevo): `checkIncomingMessage` — urgencia
  médica explícita, lenguaje de crisis de salud mental, insensibilidad a
  tildes/mayúsculas, y un mensaje de agenda ordinario que no dispara
  nada. `checkResponse` — cada una de las cuatro reglas con al menos dos
  variantes (observaciones: mención directa y paráfrasis de nota interna;
  copago: monto con signo `$` y monto sin signo junto a "costo", más un
  caso negativo de "copago" sin monto adjunto; derivar a humano: los dos
  ejemplos textuales del propio ticket, más un caso negativo explícito de
  que el propio mensaje de fuera de alcance del sistema no se autodispara;
  otros pacientes: DNI suelto y referencia nombrada en tercera persona,
  más un caso negativo de fecha/hora de turno que no debe confundirse con
  un DNI) — y un caso de respuesta limpia que pasa intacta.
- `orquestador.service.spec.ts` (ampliado): un mensaje entrante urgente
  corta el turno sin llamar ni a `AIPort.processMessage` ni a
  `tools.execute`, y el mensaje de fuera de alcance queda guardado en el
  historial de la sesión; una respuesta violatoria generada por el modelo
  mockeado llega reemplazada tanto en el valor devuelto por `procesar`
  como en lo que efectivamente queda guardado en
  `ConversationSessionStore` para el próximo turno.
- `normalize-text.spec.ts` (nuevo): minúsculas y sin tildes; texto ya en
  minúsculas queda igual salvo el casing; puntuación y espaciado se
  conservan (a diferencia de `normalizeToWords`); cadena vacía.
- Suite unitaria completa del repo en verde (61 suites, 591 tests), más
  `tsc --noEmit` sin errores y ESLint sin advertencias sobre los archivos
  tocados. No se corrió la suite e2e: esta tarea no agrega ni modifica
  ningún controlador ni ninguna ruta HTTP (`GuardrailService` no está
  detrás de ningún endpoint, igual que `OrquestadorService`), así que no
  hay superficie nueva que esa suite ejercite.

## Figuras pendientes

- Diagrama de los dos puntos de intercepción del guardrail dentro de un
  turno de conversación: mensaje entrante → `checkIncomingMessage` →
  ¿coincide con una palabra de urgencia? → [corta: mensaje fijo, sin
  llamar a `AIPort` ni a ninguna herramienta] / [continúa el ciclo normal
  de `OrquestadorService`, TASK-48] → respuesta final en texto plano →
  `checkResponse` → texto (reemplazado o intacto) guardado en el
  historial y devuelto. Sección 4.6 Capa conversacional y WhatsApp.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-49-guardrails-validation`, creada
  desde `main` como pide la usuaria. Nota de proceso: `main` local estaba
  dos commits detrás de `origin/main` (le faltaba el merge de TASK-48, el
  orquestador del que esta tarea depende) al empezar la sesión; se
  actualizó con `git fetch` + fast-forward antes de ramificar, para que
  la rama de esta tarea parta del mismo commit que efectivamente contiene
  el orquestador que integra. Sin commit ni push al momento de escribir
  esta entrada: pendiente de autorización explícita de la usuaria.
- Ticket: TASK-49 (Jira), "P5.4 – Capa de validación de guardarraíles",
  bajo el épico TASK-8 (Módulo 5). Fuente de verdad consultada en Drive:
  el documento de Especificación de Requisitos de Software (Anexo,
  "Módulo turnos" — la sección de reserva/confirmación por chatbot, que
  ya trae, con la misma redacción de fondo que el propio ticket, las
  reglas de "el campo observaciones... no se revela nunca al paciente",
  "el importe del copago no se informa por el bot", y "el bot no deriva a
  un operador humano; ante mensajes de urgencia... el bot responderá que
  su función se limita a la gestión de turnos"); no agregó ningún
  requisito que no estuviera ya en el propio texto del ticket, sólo lo
  corroboró. Dependencias: TASK-48 (P5.3, orquestador — ya mergeada a
  `main`). Fuera de alcance y reservado a tareas futuras: los flujos de
  negocio concretos por sobre el ciclo genérico de herramientas y los
  guardrails (P5.5, TASK-50, que es también donde correspondería resolver
  de forma no heurística "el profesional de la sesión activa" que la
  regla 5 menciona); la integración real con WhatsApp (P5.8, TASK-53).
