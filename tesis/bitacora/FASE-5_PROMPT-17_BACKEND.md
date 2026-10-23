# Fase 5 — Capa conversacional y WhatsApp (backend) — la primera sesión de un paciente nuevo se le informa como un único turno de una hora (TASK-214, corrección a P3.4/P5.5)

## Contexto

Hallazgo de uso real, reportado por la usuaria tras conversar con el bot:
cuando el paciente es nuevo para el profesional elegido, el sistema reserva
su primera sesión como **dos turnos contiguos** —el "doble turno" que crea
la regla de franja extra (P3.4, TASK-37, sobre la decisión de TASK-86 de
que cada mitad dure una `duracion_consulta` completa)—, y esa partición
estaba llegando tal cual al paciente. El bot le confirmaba un turno "de 30
minutos" para una sesión que en realidad ocupa una hora, y cuando después
le preguntaba qué turnos tenía agendados le respondía que tenía *dos*, uno
a las 9:00 y otro a las 9:30.

La partición en dos filas es un mecanismo de agenda: son dos bloques reales
de la grilla del profesional, y así los necesita ver la aplicación del
profesional. Para el paciente, en cambio, es un único compromiso: eligió un
horario y le debe a la clínica una sesión más larga. El ticket fija
exactamente ese criterio ("no le tiene que avisar que hay dos turnos, tiene
que decirle que es de 1 hora y chau").

## Qué se implementó

El par se colapsa en el borde de salida hacia la capa conversacional, y
solamente ahí.

- **Regla pura nueva** (`src/appointments/first-session-pair.rule.ts`):
  agrupa los turnos de un paciente en las sesiones que ese paciente
  efectivamente vivió. Reconoce las dos mitades por lo que dicen las filas
  mismas —igual profesional, igual día calendario, ambas marcadas como
  primera sesión—, sin ningún campo de esquema que las vincule, porque no
  existe: la cota de "un máximo de 1 paciente nuevo por jornada" que ya
  aplica la reserva es la que vuelve esa combinación inequívoca. La
  duración que reporta es el **lapso** desde el comienzo de la primera
  mitad hasta el final de la última (cadencia + duración de sesión), no la
  suma de las dos: con una cadencia configurada los dos números difieren, y
  lo que el paciente tiene que reservar en su día es el lapso.
- **Agenda del paciente** (`listActiveForPatient`, que alimenta
  `list_patient_appointments`): devuelve una entrada por sesión en lugar de
  una por fila.
- **Reserva** (`book_appointment`): devuelve un único turno con la duración
  de la sesión completa, en lugar del arreglo de dos filas que sigue
  devolviendo `POST /turnos` para la agenda del profesional.
- **Pre-chequeo de elegibilidad** (`check_booking_eligibility`): se le
  agregó `firstSessionDuration`, los minutos que durará esa primera sesión,
  para que el bot pueda decir la hora *antes* de que se reserve nada —la
  pregunta "¿cuánto dura?" aparece mientras el paciente todavía está
  eligiendo.
- **Manual de flujos** (`CONVERSATION_FLOWS_PROMPT`): una sección nueva,
  "Duración del turno y primera sesión", que fija la regla que las
  herramientas ya hacen cumplir —un horario de inicio, una duración, nunca
  dos turnos, y siempre el número que informó la herramienta y no el de los
  horarios disponibles—, más los dos pasos afectados de los flujos de
  reserva y de reprogramación.
- **Acciones del paciente sobre esa sesión** (motor de turnos, 4.4):
  confirmar y cancelar arrastran a la otra mitad, con un único código de
  acceso y un único mensaje de PIN por sesión, anclados en la mitad más
  temprana; reprogramar media sesión se rechaza con un error descriptivo.

## Decisiones y por qué

**El colapso vive en la capa de presentación hacia el paciente, no en el
modelo de datos.** La alternativa de fondo —guardar la primera sesión como
una sola fila de duración doble— haría innecesaria toda esta regla, pero
revierte una decisión ya tomada y documentada (TASK-86: dos turnos, cada
uno de una `duracion_consulta`) y, sobre todo, le quita al profesional la
vista que necesita: su grilla sí tiene dos bloques ocupados. Se mantuvo
entonces el esquema y se cambió únicamente lo que el paciente lee.

**Las dos mitades se reconocen por los datos, no por un campo nuevo.** El
código ya describía el vínculo entre ambas como implícito ("turnos
adyacentes del mismo paciente y profesional"), y la cota de un paciente
nuevo por jornada y por profesional vuelve imposible que dos turnos de
primera sesión del mismo par caigan el mismo día sin ser las dos mitades de
la misma sesión. Agregar una columna de vínculo habría sido una migración
de esquema para expresar algo que las propias filas ya determinan.

**Confirmar y cancelar tenían que arrastrar la sesión entera, o el colapso
habría sido peor que el problema.** Al mostrarle un solo turno, el único
identificador que el paciente puede entregarle al bot es el de la mitad más
temprana. Confirmar sólo esa mitad dejaba la otra reservada y a merced del
proceso automático de cancelación por falta de confirmación: media hora de
una sesión que el paciente acababa de confirmar, desaparecida en silencio.
Cancelar sólo esa mitad dejaba en la agenda un turno que el paciente cree
cancelado y al que, por el colapso mismo, ya no podría ni asomarse. Por eso
la propagación es parte de esta corrección y no de un ticket posterior.

**Un solo código de acceso y un solo mensaje por sesión, anclados en la
mitad más temprana.** El código se genera para el turno que marca el
comienzo de la sesión, que es cuando el paciente efectivamente llega,
cualquiera de las dos mitades que el llamador haya nombrado. La
reasignación del lugar liberado, en cambio, se dispara por mitad: dos
medias horas liberadas son dos lugares reales de la grilla y pueden
ofrecerse a dos pacientes distintos de la lista de espera —lo único que era
una sola cosa era la vista del paciente.

**La reprogramación se rechaza en lugar de aplicarse a medias.** El código
ya documentaba que la reprogramación mueve un turno a la vez y que no
existe mecanismo alguno para mantener emparejada la otra mitad del doble
turno. Esa limitación, hasta esta corrección, se resolvía silenciosamente
moviendo una sola mitad y partiendo la sesión en dos. Ahora el intento se
rechaza por completo y el manual de flujos le ofrece al paciente el camino
que sí funciona de punta a punta: cancelar la primera sesión —que ahora se
cancela entera— y reservarla de nuevo, que la vuelve a colocar como par.

**El prompt es el control primario y las herramientas el que no depende de
que el modelo obedezca**, la misma división de trabajo que el resto de esta
fase mantiene entre el manual de flujos y las verificaciones de código: el
par ya le llega al modelo colapsado en un único turno, así que no tiene de
dónde sacar el segundo horario aunque la instrucción se ignore.

## Alternativas descartadas

- **Corregir sólo el manual de flujos**, instruyendo al modelo a sumar las
  duraciones y a no nombrar el segundo horario. Se descartó porque el
  modelo seguiría recibiendo dos turnos con dos horarios distintos: la
  instrucción sería la única barrera entre el dato y el paciente, y el
  propio ticket nace de que el modelo ya había dicho lo que no debía.
- **Guardar la primera sesión como un único turno de duración doble.**
  Descartada por lo dicho más arriba: revierte una decisión ya tomada y le
  oculta al profesional la ocupación real de su grilla.
- **Mover el doble turno completo en la reprogramación**, validando el
  nuevo comienzo contra la grilla de pacientes nuevos y reubicando las dos
  mitades. Es la solución completa y quedó registrada como trabajo
  pendiente, no como parte de esta corrección: exige una variante
  emparejada de la escritura de reprogramación —con su reemisión de código
  de acceso, su aviso al paciente y el camino diferido de la confirmación
  del paciente—, un alcance propio de un ticket aparte frente a un hallazgo
  cuyo objeto es lo que el bot dice.

## Entidades / puertos / adaptadores tocados

- `src/appointments/first-session-pair.rule.ts` (nuevo): regla pura, sin
  estado ni entrada/salida, en la línea de las otras reglas puras del mismo
  directorio (transición de estados, elegibilidad de paciente nuevo,
  inactividad del paciente).
- `src/appointments/appointment.presenter.ts`: la agenda del paciente pasa
  a presentarse por sesión y se agregó la forma que devuelve la reserva a
  la conversación, deliberadamente más angosta que la respuesta HTTP
  (sin identificadores de paciente/profesional ni marcas de auditoría).
- `src/appointments/appointments.service.ts`: agenda del paciente,
  propagación en confirmar y cancelar, rechazo de la reprogramación de
  media sesión, y el nuevo dato de duración en el pre-chequeo de
  elegibilidad (leído de la misma consulta que ya traía las dos compuertas
  de paciente nuevo, sin un viaje extra a la base).
- `src/chatbot/tools/appointment.tools.ts` y
  `src/chatbot/conversation-flows.constants.ts`: herramienta de reserva,
  descripciones de las herramientas y manual de flujos.
- Ningún cambio de esquema ni migración. Ninguna ruta HTTP cambió de
  contrato: `POST /turnos`, `GET /turnos` y la agenda del profesional
  siguen reportando los dos bloques.

## Tests y qué validan

- `src/appointments/first-session-pair.rule.spec.ts` (nuevo): el colapso en
  sí —par reconocido en cualquier orden de llegada, lapso y no suma cuando
  hay cadencia, turnos ordinarios intactos, mitad suelta reportada con su
  propia duración, sesiones de días o profesionales distintos no fusionadas.
- `src/appointments/appointments.service.spec.ts`: la agenda del paciente
  informa una sola sesión de una hora para el par y deja separados los
  turnos de un paciente recurrente; confirmar y cancelar arrastran la otra
  mitad; un único código de acceso, anclado en la mitad más temprana; la
  confirmación del paciente se sostiene cuando la otra mitad no puede
  seguirla; y los tres valores posibles del nuevo dato de duración en el
  pre-chequeo (lapso del doble turno, sesión simple sin modalidad de franja
  configurada, y sin dato cuando el profesional no tiene duración de
  consulta cargada).
- `src/appointments/appointments-rescheduling.service.spec.ts`: el rechazo
  de mover media sesión —antes incluso de mirar el horario destino— y que
  una primera sesión sin otra mitad se mueva como cualquier otro turno.
- `src/chatbot/tools/appointment.tools.spec.ts`: la herramienta de reserva
  responde un único turno de una hora para el doble turno, y un turno
  ordinario con su propia duración.
- `test/chatbot-flows.e2e-spec.ts`: tres casos nuevos de extremo a extremo
  con Postgres real y un profesional con modalidad de franja configurada —
  la reserva escribe las dos filas de agenda mientras la conversación ve un
  solo turno de una hora en el pre-chequeo, en la reserva y en la consulta
  de turnos; confirmar y cancelar desde ese único turno dejan las dos filas
  confirmadas y después canceladas, con la agenda del paciente vacía al
  final; y la reprogramación se rechaza sin mover ninguna de las dos
  mitades.
- Suite unitaria completa en verde salvo una falla preexistente y ajena a
  esta tarea, heredada de la rama de integración (la prueba de selección de
  adaptador de cerradura, 1 de 1111 pruebas). Suite de extremo a extremo
  completa con `--runInBand` contra Postgres local: 664 de 666 pruebas en
  verde, con las dos fallas restantes también preexistentes y verificadas
  como tales sobre la rama base sin cambios (residuo de datos de corridas
  anteriores en la base compartida, no relacionado con esta corrección).
  Análisis estático y formato sin advertencias sobre los archivos tocados.
  Datos ficticios, sin contenido clínico ni datos personales reales.

## Figuras pendientes

Ninguna figura nueva: la corrección no cambia el diagrama de reserva ni el
catálogo de herramientas, sino qué devuelven dos de ellas.

## Trabajo pendiente que esta tarea deja registrado

Los avisos automáticos siguen enviándose **por fila** y no por sesión: el
pedido de confirmación de 24 horas antes y el recordatorio configurable
(fase 4) le mandan al paciente un mensaje por cada mitad, de modo que una
primera sesión todavía produce dos mensajes de WhatsApp nombrando las 9:00
y las 9:30. Es exactamente el mismo defecto que esta corrección arregla en
la conversación, en otra capa —los procesos programados y las plantillas de
notificación—, y queda para un ticket aparte junto con la reprogramación
emparejada del doble turno.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-214-first-session-single-turno`
  (creada a partir de `development`).
- Ticket: TASK-214 ("cuando hay un paciente nuevo, no le tiene que avisar
  que hay dos turnos, tiene que decirle que es de 1 hora y chau"), hallazgo
  de uso real sobre el doble turno de P3.4 (TASK-37) tal como lo presentan
  las herramientas conversacionales de P5.5 (TASK-50).
