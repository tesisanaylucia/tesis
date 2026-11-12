# Fase 4 — Notificaciones y Scheduler (backend) — un paciente nuevo recibía dos pedidos de confirmación, dos recordatorios y dos avisos de cancelación por su única primera sesión (TASK-221, corrección a P4.2/P4.3/P4.4 y continuación de TASK-214)

## Contexto

Hallazgo de uso real, reportado por la usuaria probando el sistema como
paciente nuevo. La primera sesión de un paciente nuevo se guarda como **dos
turnos contiguos** (el "doble turno" de la regla de franja extra, P3.4/TASK-37,
con cada mitad de una `duracion_consulta` según TASK-86). TASK-214 ya había
colapsado ese par en todo lo que el chatbot le *responde* al paciente (agenda,
reserva, confirmar y cancelar desde la conversación), pero los mensajes que el
sistema le *inicia* al paciente desde los jobs programados seguían
recorriendo las filas una por una:

- el **pedido de confirmación** a 24 h (P4.2) salía dos veces: "tu turno es a
  las 9:00 y dura 30 minutos" y "tu turno es a las 9:30 y dura 30 minutos";
- el **recordatorio** configurable (P4.4) también salía dos veces, uno por
  cada horario;
- y cuando el paciente respondió "sí" la confirmación no se registró en
  ninguna de las dos filas, de modo que la **autocancelación** a las 4 h sin
  respuesta (P4.3) canceló ambas y le avisó al paciente dos veces que "su
  turno" había sido cancelado.

Para el paciente la división en dos filas es transparente: reservó un solo
turno de una hora y espera un solo mensaje de cada tipo.

## Qué se implementó

- **Una consulta compartida para "las otras mitades de esta sesión"**
  (`firstSessionCompanionsWhere`, en `src/appointments/first-session-pair.rule.ts`).
  Devuelve el filtro de Prisma que encuentra las demás mitades de la sesión
  de un turno dado, en los estados pedidos, o `null` si el turno es
  ordinario. Reconoce el par con exactamente los mismos tres hechos que el
  colapso en memoria de TASK-214 (mismo paciente y profesional, mismo día
  calendario, ambas filas marcadas como primera sesión). El método privado
  `AppointmentsService.loadFirstSessionCompanions`, que ya usaban confirmar y
  cancelar, pasó a construir su consulta con esta misma función.
- **Pedido de confirmación** (`AppointmentConfirmationService.requestConfirmation`,
  compartido por el cron de 24 h y por la reserva cuando el turno ya está
  dentro de la ventana): carga la sesión completa, envía **un único mensaje**
  con la hora de inicio de la mitad más temprana y la duración total de la
  sesión (el lapso de inicio a fin, la misma cifra que ya informa el chatbot),
  y marca `confirmationRequestedAt` en **todas las mitades con una misma
  escritura y un mismo instante**. Cuando el cron llega a la otra mitad en la
  misma corrida, ya no queda nada que reclamar y no se envía nada. Si el envío
  falla, la reversión de la marca (TUR-10) también abarca a todas las mitades.
- **Recordatorio** (`AppointmentReminderCron`): misma forma — una sola
  escritura reclama `reminderSentAt` en todas las mitades CONFIRMADO de la
  sesión, el mensaje nombra la hora de inicio de la sesión, y la otra mitad
  ya no dispara un segundo envío.
- **Autocancelación por falta de respuesta** (`AppointmentAutoCancellationCron`):
  cancela la **sesión** y no la fila. Cada mitad conserva su propia
  cancelación, asiento de auditoría, disparo de reasignación y aviso in-app al
  profesional —su agenda efectivamente liberó dos bloques—, pero el paciente
  recibe **un solo** "tu turno fue cancelado", anclado en el inicio de la
  sesión. Para eso `AppointmentsService.notifyCancellationToProfessionalAndPatient`
  se partió en dos métodos públicos, `notifyProfessionalOfCancellation` y
  `notifyPatientOfCancellation`, que el helper original sigue invocando en
  secuencia, de modo que los demás llamadores no cambian. Además, si una de
  las mitades quedó RESERVADO sin pedido de confirmación propio (datos
  anteriores a esta corrección), se cancela junto con la otra: es la misma
  regla de "una sesión no se cancela por mitades" que ya aplicaba
  `AppointmentsService.cancel` desde TASK-214.

## Decisiones y por qué

**Reclamar todas las mitades en una única escritura, en lugar de deduplicar
en memoria dentro de cada corrida.** La alternativa más corta habría sido
agrupar los candidatos de cada corrida y enviar un mensaje por grupo. Pero las
dos mitades no siempre entran juntas en la ventana de detección: con la banda
de 23 a 25 h, la mitad de las 9:00 puede quedar seleccionada en una corrida y
la de las 9:30 recién en la siguiente, y una deduplicación limitada a una
corrida habría vuelto a mandar dos mensajes. Reclamar todas las mitades con el
mismo `updateMany` guardado hace que la idempotencia la resuelva la propia
columna, sin importar en qué corrida —o en qué orden dentro de ella— aparezca
cada mitad, y deja además un único `confirmationRequestedAt` compartido, que
es lo que permite que la autocancelación trate a la sesión como una unidad.

**El mensaje se ancla en la mitad más temprana aunque el job haya
seleccionado primero la tardía.** El orden de los candidatos no está
garantizado; lo que el paciente necesita saber es cuándo tiene que llegar, y
eso es siempre el inicio de la sesión. Es el mismo criterio que TASK-214 fijó
para el código de acceso.

**El aviso al profesional sigue siendo por bloque.** Las dos filas son dos
bloques reales de su grilla, y la reasignación desde la lista de espera
también opera por bloque; la transparencia del par es una propiedad de lo que
lee el paciente, no del modelo de datos (misma decisión de fondo que
TASK-214, que descartó guardar la primera sesión como una sola fila de
duración doble).

**Un asiento de auditoría por fila efectivamente reclamada.** Cada mitad es
su propio `Appointment` en la traza, y cada una queda con un pedido (o un
recordatorio) pendiente. Después del reclamo se vuelve a leer qué filas
llevan exactamente el instante que esa misma llamada escribió, de modo que el
asiento no se registra para una fila que una corrida concurrente ya había
reclamado antes.

**Sobre la confirmación que "no se tomó".** El código del chatbot para
confirmar ya arrastraba la segunda mitad desde TASK-214; lo que el paciente
recibía, en cambio, eran dos pedidos que hablaban de dos turnos distintos de
30 minutos, mientras que el bot le mostraba uno solo de una hora, de modo que
la respuesta "sí" quedaba ambigua frente a lo que el bot veía. Con un único
pedido que habla de la misma sesión que el bot muestra, esa ambigüedad
desaparece. No se pudo reproducir con los registros de producción por qué esa
conversación puntual no llegó a invocar la confirmación, así que esa causa
exacta queda sin confirmar; se deja asentado como observación a revisar en la
próxima prueba manual. Además, el texto de los pedidos que salen por plantilla
de Meta no queda en el historial de la conversación del bot: un "sí" aislado
le llega sin el mensaje al que responde.

## Entidades, puertos y adaptadores tocados

- Sin cambios de esquema ni migraciones.
- `src/appointments/first-session-pair.rule.ts` — `FirstSessionAnchor`,
  `firstSessionCompanionsWhere`.
- `src/appointments/appointment-confirmation.service.ts` — el candidato
  incorpora `isFirstSession`; reclamo, reversión y auditoría por sesión.
- `src/appointments/appointment-confirmation.cron.ts` — selecciona
  `isFirstSession`.
- `src/appointments/appointment-reminder.cron.ts` — reclamo y mensaje por
  sesión.
- `src/appointments/appointment-auto-cancellation.cron.ts` — cancelación por
  sesión, un solo aviso al paciente.
- `src/appointments/appointments.service.ts` — `loadFirstSessionCompanions`
  reutiliza la consulta compartida; nuevos `notifyProfessionalOfCancellation`
  y `notifyPatientOfCancellation`; la reserva pasa `isFirstSession` al pedido
  inmediato.
- Ningún puerto nuevo: los mensajes siguen saliendo por
  `NotificationDeliveryService` (TASK-219) y `MessagingPort`.

## Tests agregados o modificados

- `appointment-confirmation.service.spec.ts`: el doble de Prisma pasó a
  modelar varias filas (reclamo con `id: { in: [...] }` y la consulta de
  mitades). Casos nuevos: un único pedido con la hora de inicio y la
  duración total (09:00 y 90 minutos para dos mitades de 45 minutos
  separadas por una cadencia de 45); la segunda mitad procesada después no
  envía nada y ambas quedan con el mismo instante y un asiento cada una; el
  mensaje se ancla en la mitad más temprana aunque el candidato sea la
  tardía; un envío fallido revierte la marca en las dos mitades.
- `appointment-reminder.cron.spec.ts`: un único recordatorio con la hora de
  inicio y ambas mitades marcadas; el mismo resultado si la mitad tardía se
  lee primero; dos turnos de primera sesión de pacientes distintos no se
  emparejan.
- `appointment-auto-cancellation.cron.spec.ts`: ambas mitades canceladas,
  con reasignación y aviso al profesional por mitad pero un solo aviso al
  paciente, anclado en la mitad más temprana; una mitad sin pedido propio se
  cancela igual junto con la otra.
- `appointments.service.spec.ts`: el pedido inmediato desde la reserva lleva
  `isFirstSession`.
- Suite unitaria completa: 103 suites / 1209 tests en verde. Suite e2e
  completa (`--runInBand`): 51 de 54 suites y 670 de 674 tests en verde. Los
  4 tests que fallan (`appointment-engine-integration`, `patients-import`,
  `patients-entities`) fallan exactamente igual sobre `origin/development`
  sin los cambios de esta tarea, de modo que son preexistentes y ajenos a
  ella; uno de ellos muestra filas sobrantes de corridas anteriores en la
  base local compartida.

## Figuras pendientes

Ninguna nueva. Si se dibuja la figura de secuencia del ciclo
confirmación → autocancelación de 4.5, conviene mostrar el doble turno como
una sola sesión con un único mensaje por etapa.

## Referencia

- Componente: backend.
- Rama: `feature/TASK-221-one-message-per-first-session` (desde
  `origin/development`).
