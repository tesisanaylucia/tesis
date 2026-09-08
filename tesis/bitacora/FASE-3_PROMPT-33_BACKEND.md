# Fase 3 — Motor de Turnos (backend) — la cancelación en cascada por ausencia u feriado podía perderse sin dejar rastro (TASK-135, corrección a TASK-39/TASK-127)

## Contexto

TASK-135 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, validada por la usuaria
antes de implementarse, con dos hallazgos combinados (PROF-9 y PROF-10). El
detalle completo de la auditoría está en el artefacto
`https://claude.ai/code/artifact/5c2ee112-c426-4d6e-a8e2-499ea7bbbc74`
("auditoria-psique-back.md").

PROF-9 apuntaba a `AbsencesService.create`: la cancelación en cascada de los
turnos que la ausencia bloquea se dispara con `AbsenceEventsPort` después de
que la propia transacción de la ausencia ya confirmó, en la misma petición.
Si esa cascada fallaba antes de leer siquiera qué turnos están afectados —un
timeout de pool de conexiones, un error transitorio— la excepción se
propagaba y la petición terminaba en un 500, aunque la ausencia ya estuviera
persistida; y no había ninguna forma de volver a pedir la cascada, porque
reintentar el mismo `POST` fallaba por el propio chequeo de solapamiento que
prueba que la ausencia ya existe.

PROF-10 apuntaba a `HolidaysService.create`: a diferencia de `remove`, que sí
cuenta y reporta los turnos afectados por eliminar un feriado, `create` no
hacía nada con los turnos ya reservados sobre la fecha que se está
declarando feriado. Al revisar el código vigente para implementar la
corrección, se confirmó que esta segunda mitad del hallazgo ya estaba resuelta
por TASK-127 ([[FASE-3_PROMPT-32]]), fusionada a `main` el 2026-09-07 —
posterior a la fecha de la auditoría (2026-08-20) que originó TASK-135. Desde
esa tarea, `HolidaysService.create` dispara la misma cascada de cancelación
que la ausencia, a través de `AppointmentsService.cancelForHoliday`. Esta
tarea documenta ese hallazgo como ya subsanado y, en cambio, extiende PROF-9 a
las dos cascadas por igual: la misma falta de durabilidad que afecta a
`cancelForAbsence` afecta, desde TASK-127, también a `cancelForHoliday`.

## Qué se implementó

Un patrón de **outbox transaccional**, nuevo en este código base: un modelo
`CascadeCancellationJob`, tenant-scoped, con un enum `CascadeCancellationTrigger`
(`ABSENCE` / `HOLIDAY`) y un enum de estado (`PENDING` / `PROCESSED`).
`AbsencesService.create` y `HolidaysService.create` insertan una fila
`PENDING` (`CascadeCancellationJobsService.enqueue`) **dentro de la misma
transacción** que crea la ausencia o el feriado, de modo que el hecho "esta
cascada queda pendiente" no puede perderse en un cierre abrupto entre esa
escritura y el disparo del evento que sigue.

Después de confirmar la transacción, el flujo eager de siempre sigue
intacto: se publica el evento (`absenceRegistered`/`holidayRegistered`,
ahora con el id del job agregado a su payload) y el adaptador correspondiente
llama a `AppointmentsService.cancelForAbsence`/`.cancelForHoliday` con ese id.
Ambos métodos delegan en un método nuevo compartido,
`cancelActiveAppointmentsInRange` — extracción de la lógica que antes estaba
duplicada casi íntegra entre los dos, ver más abajo—, envuelto por
`runCascadeCancellation`: si la cascada corre sin lanzar, marca el job
`PROCESSED`; si lanza, registra el intento fallido sobre el job (contador de
intentos y último error) y **no vuelve a lanzar** — la petición original,
cuya ausencia o feriado ya se confirmó, no debe fallar por esto. Un
`CascadeCancellationJobCron` nuevo, con la misma cadencia de 15 minutos y el
mismo recorrido por organización que el resto de los trabajos programados del
módulo, barre periódicamente los jobs que siguen `PENDING` y vuelve a invocar
exactamente el mismo método de `AppointmentsService` con el id del job — el
reintento es seguro porque `cancelForAbsence`/`.cancelForHoliday` sólo tocan
turnos RESERVADO/CONFIRMADO, así que un turno que la cascada ya canceló deja
de aparecer en la siguiente pasada.

## Decisiones y por qué

**Outbox transaccional en vez de mover la cancelación a la transacción de la
ausencia/feriado**, la primera de las dos soluciones que el propio hallazgo
proponía como alternativas. Se descartó porque violaría un límite
arquitectónico documentado explícitamente en `AbsencesService`: ese servicio
"emite el evento pero no implementa lógica de reasignación propia", y
moverla ahí exigiría inyectar `AppointmentsService` en el módulo de
profesionales (y en el de feriados), acoplando dos dominios que hoy sólo se
comunican a través de un puerto de extensión. La fila `CascadeCancellationJob`
resuelve el mismo problema de durabilidad sin ese acoplamiento: quien la
escribe conoce sólo los datos que ya publica el evento (organización,
profesional si corresponde, rango de fechas, actor), nunca la lógica de
cancelación en sí.

**Un solo modelo genérico para las dos cascadas, no uno por disparador.** La
ausencia y el feriado comparten exactamente el mismo problema de durabilidad
y el mismo mecanismo de reintento; distinguirlos exige sólo un campo
`trigger` y que `professionalId` quede nulo para el feriado (que no se acota
a un profesional). Separar el modelo en dos habría duplicado el cron, el
servicio y el índice sin ninguna diferencia real de comportamiento.

**El job no guarda una referencia (FK) a la ausencia ni al feriado que lo
originó.** Guarda, en cambio, los mismos datos ya derivados que el evento
publica —organización, profesional opcional, rango de fechas, actor—, de
modo que sigue siendo ejecutable aunque la ausencia que lo originó se borre
antes de que el job corra. Añadir una referencia real habría exigido resolver
qué hacer con un job pendiente cuya ausencia ya no existe (¿debe seguir
cancelando turnos "como si" la ausencia siguiera vigente? ¿debe abortarse?),
pregunta que el hallazgo no plantea y que esta corrección no necesita
responder para cerrar el problema real: que la cascada no se pierda.

**Marcar el job `PROCESSED` en cuanto la cascada corre sin lanzar, no en
cuanto cada turno individual se cancela con éxito.** `cancelActiveAppointmentsInRange`
ya cancelaba cada turno en su propia transacción con su propio manejo de
errores desde antes de esta tarea (P3.6/TASK-39): un conflicto de
concurrencia sobre un turno puntual se registra y se sigue con el resto,
comportamiento que esta corrección no toca. El fallo que PROF-9 describe es
distinto — el fallo de la lectura inicial, antes de llegar siquiera a
iterar—, así que "procesado" significa "la cascada corrió de punta a punta",
no "cada turno individual se resolvió", que es una garantía que este código
ya no ofrecía ni antes de esta tarea.

**Extracción de `cancelActiveAppointmentsInRange`, sin cambio de
comportamiento.** `cancelForAbsence` y `cancelForHoliday` (TASK-127) eran,
antes de esta tarea, casi idénticos: la única diferencia real era el filtro
por `professionalId` y el `CancellationReason` con el que se tagea cada
turno. Envolver ambos con la misma protección de reintento habría duplicado
esa protección dos veces si el método compartido no se extraía primero — la
refactorización precede a la corrección de fiabilidad, no al revés, para que
esta última se escriba una sola vez.

## Alternativas descartadas

- **Reintento indefinido sin ningún registro de intentos**: descartado
  porque un job que falla repetidamente sin dejar rastro es tan difícil de
  diagnosticar como el problema original — el job guarda un contador de
  intentos y el último mensaje de error, aunque esta tarea no agrega ninguna
  alerta ni límite de reintentos sobre esos campos, dejados como base para
  una futura tarea de observabilidad si hiciera falta.
- **Un estado `FAILED` terminal tras N intentos**: descartado porque el
  hallazgo no pide abandonar la cascada bajo ninguna circunstancia — un turno
  que sigue reservado sobre una ausencia o un feriado es, precisamente, el
  problema que se quiere evitar, así que el cron reintenta indefinidamente en
  lugar de dejar de intentarlo.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma` (modificado): modelo nuevo `CascadeCancellationJob`
  (tenant-scoped, FK compuesta opcional a `Professional`), enums
  `CascadeCancellationTrigger`/`CascadeCancellationJobStatus`.
- `prisma/migrations/20260908155246_add_cascade_cancellation_job/` (nueva).
- `src/appointments/cascade-cancellation-jobs.service.ts` (nuevo):
  `CascadeCancellationJobsService` — `enqueue` (dentro de una transacción
  dada), `markProcessed`, `recordAttemptFailure`, `findPending`.
- `src/appointments/cascade-cancellation-job.cron.ts` (nuevo):
  `CascadeCancellationJobCron`, cada 15 minutos, mismo patrón de recorrido
  por organización que el resto de los crons del módulo.
- `src/appointments/appointments.service.ts` (modificado):
  `cancelForAbsence`/`cancelForHoliday` ganan un parámetro opcional
  `cascadeCancellationJobId`; lógica compartida extraída a
  `cancelActiveAppointmentsInRange`; método nuevo `runCascadeCancellation`.
- `src/appointments/appointments.module.ts` (modificado): registra y exporta
  `CascadeCancellationJobsService`; registra `CascadeCancellationJobCron`.
- `src/domain/ports/absence-events.port.ts` / `holiday-events.port.ts`
  (modificados): `cascadeCancellationJobId` agregado a ambos eventos.
- `src/appointments/absence-events.adapter.ts` / `holiday-events.adapter.ts`
  (modificados): reenvían el nuevo campo.
- `src/professionals/absences.service.ts` / `src/holidays/holidays.service.ts`
  (modificados): inyectan `CascadeCancellationJobsService`; `create` encola el
  job dentro de su propia transacción.
- `CLAUDE.md` (modificado): nuevo párrafo en "Integration ports" documentando
  el patrón de outbox transaccional como referencia para un futuro puerto
  cuyo despacho eager necesite la misma garantía.

## Tests y qué validan

- `src/appointments/cascade-cancellation-jobs.service.spec.ts` (nuevo):
  `enqueue` escribe a través del handle de transacción recibido, no del
  cliente plano; omite `professionalId` para un disparador `HOLIDAY`;
  `markProcessed`/`recordAttemptFailure`/`findPending`.
- `src/appointments/cascade-cancellation-job.cron.spec.ts` (nuevo): reintenta
  un job `ABSENCE`/`HOLIDAY` con las fechas ya convertidas a calendario (no
  el `Date` crudo de la columna) y el id del job; un job que falla no
  detiene el resto del lote; una organización sin usuario SYSTEM se omite.
- `src/professionals/absences.service.ts`, `src/holidays/holidays.service.spec.ts`,
  ambos adaptadores (`*.adapter.spec.ts`) y `appointments.service.spec.ts`/
  `appointments-rescheduling.service.spec.ts` (modificados): mocks
  actualizados para el nuevo colaborador y, en `holidays.service.spec.ts`,
  un caso nuevo que prueba que `create` encola el job en la misma transacción
  y que el evento publicado lleva su id.
- `test/professional-schedules.e2e-spec.ts` y `test/appointment-engine-integration.e2e-spec.ts`
  (modificados, contra Postgres real): tras registrar una ausencia o declarar
  un feriado con turnos ya reservados, la fila `CascadeCancellationJob`
  correspondiente queda en estado `PROCESSED` — la prueba de que el camino
  feliz efectivamente marca el job, no sólo que la cascada sigue cancelando
  turnos como antes.
- Ejecución: suite unitaria completa en verde (83 conjuntos, 809 pruebas),
  suite end-to-end completa en verde (51 conjuntos, 572 pruebas) y cobertura
  combinada (134 conjuntos, 1381 pruebas, 97.18 %/82.38 % líneas/ramas sobre
  la capa de servicios, por encima del umbral del 80 %), todas contra la
  instancia local de PostgreSQL con `--runInBand`; `tsc --noEmit` y `eslint`
  sin hallazgos. Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva. El diagrama de cancelación de turnos ya pendiente
(`figuras_pendientes.md`) puede incorporar, cuando se produzca, el paso de
encolado/reintento descrito aquí como una nota sobre la fiabilidad del
disparo, no como un flujo adicional visible al usuario.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-135-absence-holiday-appointment-reliability`, creada desde
  `origin/main` fresco (`19cb5dd`, con TASK-132 ya mergeado). Pusheada a
  `origin`; PR abierto, no fusionado aún.
- Ticket: TASK-135 ("Ausencias y feriados no gestionan de forma confiable los
  turnos que ya existen"), tarea de auditoría automática (hallazgos PROF-9 y
  PROF-10) validada por la usuaria antes de implementarse. Misma convención
  de bitácora dedicada para una corrección puntual dentro de la fase del
  ticket original que TASK-127 ([[FASE-3_PROMPT-32]]) y TASK-129
  ([[FASE-4_PROMPT-14]]). PROF-10 se documentó como ya resuelto por TASK-127
  al momento de implementar esta tarea — ver "Contexto" arriba.
