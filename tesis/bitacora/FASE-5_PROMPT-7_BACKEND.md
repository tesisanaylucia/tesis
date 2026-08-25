# Fase 5 — Capa conversacional y WhatsApp (backend) — solicitud de recetas y manejo de mensajes fuera de alcance (TASK-52)

## Qué se implementó

Se implementaron las dos mitades del ticket: el registro de solicitudes de
receta a través del chatbot —sólo el registro, sin que el sistema genere
receta alguna— y el manejo de los mensajes que quedan fuera del alcance
del sistema, separando por primera vez la respuesta ante una urgencia de
la respuesta ante un tema que el bot simplemente no gestiona.

Ninguna de las dos exigió cambios de esquema. La entidad de solicitud de
receta existía desde la fase de Pacientes, con su clave foránea compuesta
apuntando al vínculo paciente-profesional, y el servicio que la crea con
estado pendiente se había escrito en la tarea del catálogo de
herramientas. Lo que faltaba era la conversación alrededor de ese
registro: cómo se determina para qué profesional es el pedido, qué se le
dice al paciente que no tiene ningún profesional asignado, y qué se le
responde cuando lo que plantea no es un turno, una receta, una obra
social ni una pregunta frecuente.

## Decisiones y por qué

**El flujo de solicitud de receta no necesitó ninguna herramienta nueva.**
El ticket describe un paso de verificación —"el chatbot verifica que el
paciente tiene al menos un profesional asignado"— y otro de desambiguación
—"si el paciente tiene más de un profesional, el bot pregunta para qué
profesional es la solicitud"—, que a primera vista piden una herramienta
que liste los profesionales tratantes de un paciente. El relevamiento
mostró que esa lista ya viaja: la herramienta de identificación por
documento devuelve, desde la corrección de presentador de la tarea
anterior, la respuesta presentada del paciente, que incluye sus vínculos
de tratamiento con el identificador y el nombre de cada profesional. El
modelo ya tenía en el contexto lo que hacía falta para decidir, de modo
que una herramienta adicional habría sido un segundo viaje a la base para
releer datos ya cargados. Es el mismo criterio con el que la tarea
anterior resolvió la consulta de obra social sobre el listado de
profesionales existente. Lo que la tarea sí agregó al catálogo fue el
manual de procedimiento del flujo y el texto exacto de la confirmación.

**El texto de confirmación viaja dentro del resultado de la
herramienta.** El ticket pide que el bot informe "que la solicitud fue
registrada y el profesional la atenderá en días hábiles", y el documento
de requisitos agrega que "el bot recordará al paciente el horario
habilitado para pedir recetas (únicamente días de semana en horario
laboral)". Ambas cosas se redactaron en un único texto fijo que la
herramienta devuelve junto con el resultado del registro, con la
instrucción de enviarlo literalmente. Es el mismo tratamiento que ya
recibieron el texto de solicitud de consentimiento y el de ausencia de
respuesta en preguntas frecuentes, y por la misma razón: un modelo que
recibe únicamente "estado pendiente" está a un turno de inventar cuándo
va a estar lista la receta. Se mantuvo como constante y no como plantilla
configurable por inquilino porque es una conducta del bot y no contenido
de la clínica.

**La herramienta pasó a devolver una respuesta presentada y no la entidad
de la base.** Devolvía la fila tal como la crea Prisma, con el documento
interno del paciente y del profesional y las marcas de tiempo. Se redujo
a lo que el modelo necesita: el identificador de la solicitud, su estado y
el texto de confirmación. Es la misma corrección que la tarea anterior
aplicó a la herramienta de identificación, y responde al mismo principio
ya establecido en el proyecto: el chatbot es un límite como cualquier
otro, y los identificadores internos no se le muestran al paciente.

**La verificación de que el paciente tiene profesional asignado no
descansa en el manual de flujos.** El manual indica detenerse cuando la
lista de profesionales del paciente viene vacía, pero eso depende de que
el modelo lo cumpla. Debajo, el servicio que registra la solicitud afirma
la existencia del vínculo de tratamiento antes de escribir —lo que la
clave foránea compuesta de la entidad exige de todos modos—, de modo que
un modelo que intente registrar la solicitud igual recibe un fallo
descriptivo de la herramienta y nada queda escrito. Una prueba de
conversación ejercita exactamente ese camino. Es el mismo criterio con el
que la tarea anterior trató la verificación previa de elegibilidad: un
paso conversacional aporta que la conversación termine bien, nunca que la
regla se cumpla.

**Se conectó la notificación al profesional, que el esquema declaraba
desde la fase de Notificaciones sin ningún llamador.** El documento de
requisitos cierra el párrafo de solicitud de recetas con "se notifica el
pedido al profesional (notificación en la app)", y el comentario del
propio enumerado de tipos de notificación nombraba a esta tarea como su
futuro emisor. El registro de una solicitud crea ahora la notificación
correspondiente, con el mismo tratamiento de mejor esfuerzo que ya usan
la cancelación de turno y la reasignación: se ejecuta fuera de la
transacción, después de que ésta confirma, y un fallo se registra en el
log sin deshacer un registro del que al paciente ya se le avisó. El texto
de la notificación nombra identificadores y ningún contenido clínico,
porque el sistema no sabe —ni pregunta— para qué medicación es la receta.
La *visualización* de esa lista por el profesional sigue siendo trabajo
de la fase de la app móvil, tal como el ticket lo declara fuera de
alcance.

**Los mensajes de urgencia y de fuera de alcance se separaron en dos
textos.** La capa de guardrails de la tarea de TASK-49 los había unificado
en una sola oración que decía las dos cosas: que el bot sólo gestiona
turnos y que ante una urgencia hay que llamar a emergencias. Este ticket
da la redacción exacta de cada uno por separado, y la separación es
correcta más allá de la letra del ticket: son respuestas a preguntas
distintas. A quien pregunta cuál es su diagnóstico hay que decirle qué
hace este bot; a quien está en una urgencia hay que decirle a dónde
acudir ahora. Un texto que dice ambas cosas dice la mitad equivocada más
fuerte en cada caso. La regla de derivación a un operador humano y la
regla nueva de contenido clínico comparten el texto de fuera de alcance,
porque las tres situaciones tienen la misma respuesta —"esto no lo
manejo yo, lo maneja tu profesional"—, mientras que la urgencia conserva
el suyo.

**El fuera de alcance se detecta sobre la respuesta del bot y no sobre el
mensaje del paciente.** La urgencia sí se detecta sobre el mensaje
entrante, cortando el turno antes de llamar al modelo o a cualquier
herramienta, como ya lo hacía. Extender ese mismo mecanismo al fuera de
alcance habría sido un error: un paciente puede mencionar perfectamente
un síntoma camino a pedir un turno —"quiero un turno, hace días que me
duele la cabeza"—, y una detección por palabras clave sobre lo que
escribió el paciente rechazaría la reserva que el bot existe para tomar.
Lo que dice el *bot* no tiene esa ambigüedad: no hay conversación en la
que corresponda que enuncie un diagnóstico o una dosis. Se agregó
entonces una regla nueva a la revisión de respuestas, que reemplaza por
el texto de fuera de alcance cualquier respuesta que enuncie el
diagnóstico del paciente, le atribuya un cuadro, indique una dosis o le
diga que modifique su medicación. Las expresiones son deliberadamente
acotadas y no un vocabulario suelto: "receta", "medicación" y
"profesional" aparecen legítimamente en el flujo de solicitud que esta
misma tarea agrega, de modo que lo que dispara la regla es que el bot
*enuncie* contenido clínico y no que esas palabras aparezcan.

**El manual de flujos toma los dos textos del módulo de guardrails en
lugar de reescribirlos.** El prompt le pide al modelo que responda
exactamente esas frases, y el guardrail las impone cuando no lo hace. Si
cada uno tuviera su copia del texto, la única evolución posible sería que
terminaran diciendo cosas distintas; importándolos, lo que se le pide al
modelo y lo que el sistema realmente envía son literalmente la misma
cadena. Una prueba de conversación verifica que ambos textos están en el
prompt de cada turno.

**El bot no pregunta ni registra para qué medicación es la receta.** El
manual de flujos se lo prohíbe explícitamente, la entidad no tiene dónde
guardarlo y la regla nueva de guardrail bloquearía la respuesta que lo
mencionara. Es la aplicación concreta de la restricción de datos clínicos
del proyecto sobre el caso límite que esta tarea introduce: se registra
*que* se pidió una receta y a quién, nunca de qué.

## Entidades, puertos y adaptadores tocados

- Sin cambios de esquema ni migraciones. Se actualizaron dos comentarios
  del esquema que declaraban al tipo de notificación de solicitud de
  receta como "todavía sin llamador", ahora que lo tiene.
- Servicio de solicitudes de receta (subdominio de Pacientes): emisión de
  la notificación al profesional después de confirmar la transacción, con
  el patrón de mejor esfuerzo ya usado en cancelación y reasignación de
  turnos. El módulo de Pacientes pasa a importar el de notificaciones en
  la app.
- Herramienta de solicitud de receta del catálogo del chatbot: respuesta
  presentada en lugar de la entidad, y texto de confirmación devuelto con
  el resultado.
- Capa de guardrails: separación del mensaje de urgencia respecto del de
  fuera de alcance, y regla nueva de contenido clínico sobre la respuesta
  del modelo, con su propio discriminante.
- Manual de flujos del *system prompt*: sección nueva del flujo de
  solicitud de receta y sección nueva de temas fuera de alcance, esta
  última construida con los textos importados del módulo de guardrails.
  La plantilla base del prompt sumó el límite explícito de no responder
  consultas médicas.

## Tests

- Pruebas unitarias del servicio de solicitudes: que emite la
  notificación al profesional con el tipo correcto y sin contenido
  clínico, que un fallo de la notificación no impide devolver la
  solicitud ya confirmada, y que un paciente sin vínculo de tratamiento no
  llega a escribir ni a notificar nada.
- Pruebas unitarias de la herramienta: que devuelve la forma presentada
  —identificador, estado y texto de confirmación— y no la entidad, que el
  texto de confirmación nombra los días hábiles y el horario habilitado
  para pedir recetas, y que un paciente sin vínculo de tratamiento se
  traduce en un fallo descriptivo de herramienta y no en una excepción.
- Pruebas unitarias de los guardrails: que la urgencia devuelve su propio
  texto, que ese texto no vuelve a caer en la regla de derivación a un
  humano, que un mensaje entrante que menciona un síntoma camino a pedir
  un turno no se rechaza, y que la regla nueva bloquea el diagnóstico
  enunciado, el cuadro atribuido, la dosis indicada y la instrucción de
  cambiar la medicación, dejando pasar tanto el propio texto de fuera de
  alcance como la confirmación de la solicitud de receta.
- Pruebas de conversación extremo a extremo, con el puerto de
  inteligencia artificial simulado y todo lo demás real, para los
  criterios de aceptación del ticket: paciente con un único profesional,
  que deja la solicitud en estado pendiente en la base, la notificación
  creada, la entrada de auditoría y la confirmación entregada al paciente
  intacta tras atravesar los guardrails; paciente con dos profesionales,
  donde se verifica que el modelo recibe ambos con su nombre y que no se
  escribe nada hasta que el paciente elija; paciente sin profesional
  asignado, donde el modelo intenta registrar igual y la afirmación del
  vínculo lo rechaza sin escribir; urgencia, que se responde sin llegar a
  llamar al modelo ni a ninguna herramienta; tema fuera de alcance
  respondido con el texto canónico, que llega intacto; y respuesta clínica
  producida por el modelo, reemplazada por el texto de fuera de alcance
  antes de llegar al paciente.
- La prueba que verifica el contenido del *system prompt* se amplió con
  la sección del flujo de recetas, la de temas fuera de alcance y la
  presencia literal de los dos textos canónicos.

Suite completa en verde al cierre: 652 pruebas unitarias y 484 de
integración.

## Figuras pendientes

- Diagrama de secuencia del flujo de solicitud de receta (identificación
  por documento → lectura de los profesionales tratantes que la propia
  identificación devuelve → rama sin profesional asignado, rama con uno
  solo y rama con varios, esta última con la pregunta al paciente →
  registro de la solicitud con estado pendiente → notificación al
  profesional en la app → confirmación literal al paciente), señalando
  que el sistema no genera receta alguna en ningún punto. Sección 3.2.5
  Capa conversacional y WhatsApp.
- Diagrama de las dos vías del manejo de mensajes fuera de alcance
  (mensaje entrante → detección de urgencia, que corta antes de llamar al
  modelo y responde con el texto de urgencia; respuesta del modelo →
  regla de contenido clínico, que reemplaza con el texto de fuera de
  alcance), señalando por qué la primera se evalúa sobre el mensaje del
  paciente y la segunda sobre la respuesta del bot. Sección 3.2.5 Capa
  conversacional y WhatsApp.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-52-prescription-requests-out-of-scope`,
  creada desde `origin/main` como pide la usuaria. Sin commit ni push al
  momento de escribir esta entrada: pendiente de autorización explícita de
  la usuaria.
- Ticket: TASK-52 (Jira), "P5.7 – Solicitud de recetas y manejo de fuera
  de alcance", bajo el épico TASK-8 (Módulo 5). Fuente de verdad
  consultada en Drive: el documento de Especificación de Requisitos de
  Software vigente (Anexo, "Módulo turnos", párrafos "Solicitud de
  recetas" y "Derivación y mensajes fuera de alcance").
- Complementos tomados del SRS que el ticket no enumera, y que se
  implementaron por estar en la fuente de verdad: el recordatorio del
  horario habilitado para pedir recetas, incorporado al texto de
  confirmación, y la notificación del pedido al profesional en la app,
  cuyo tipo de notificación el esquema declaraba desde la fase de
  Notificaciones nombrando a esta tarea como su futuro emisor.
- Cambio sobre una decisión previa, registrado: la tarea de guardrails
  (TASK-49) había unificado en un solo texto la respuesta ante urgencias y
  la respuesta ante temas fuera de alcance; este ticket da la redacción
  exacta de cada una por separado y se adoptó esa separación, actualizando
  las pruebas de aquella tarea que afirmaban el texto unificado.
- Dependencias declaradas: TASK-47 (P5.2, herramienta de solicitud de
  receta), TASK-48 (P5.3, orquestador), TASK-49 (P5.4, guardrails) y
  TASK-27 (P2.1, entidad de solicitud de receta), todas ya fusionadas;
  TASK-51 (P5.6), fusionada a `origin/main` antes de comenzar esta sesión.
- Fuera de alcance, declarado en el propio ticket y respetado: la
  generación de la receta, que el profesional hace por su cuenta fuera del
  sistema, y la visualización de la lista de solicitudes por el
  profesional, que corresponde a la fase de la app móvil. Nada de lo
  implementado lee esa lista de vuelta.
