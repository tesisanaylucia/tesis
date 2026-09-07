# Fase 3 — Motor de Turnos (backend) — el alta de un feriado no cancelaba los turnos ya reservados en esa fecha (TASK-127, corrección a TASK-78/TASK-116)

## Contexto

`HolidaysService.create` (P3.b, TASK-78, [[FASE-3_PROMPT-8]]) inserta la fila
`Holiday` y su entrada de auditoría, sin recorrer los turnos ya agendados
sobre esa fecha. Del lado de la lectura, `AvailabilityService` sí excluye
feriados —tanto al construir la agenda ofrecible como al verificar un
instante puntual—, y TASK-116 ([[FASE-3_PROMPT-27]]) cerró la única ruta de
escritura que todavía no pasaba por esa verificación: el motor de
reasignación de lista de espera, que hasta entonces podía asignar un turno
liberado sobre una fecha ya declarada feriado. Esa misma entrada de bitácora
dejó constancia explícita de que la corrección no alcanzaba a los turnos
agendados *antes* de que la fecha se declarara feriado, que permanecían en
la agenda sin más. El escenario de falla: un turno se agenda con semanas de
anticipación, la clínica declara después esa fecha como feriado, y el turno
sigue RESERVADO/CONFIRMADO — aparece en la agenda del profesional (que es
deliberadamente ciega al estado), dispara recordatorio y pedido de
confirmación por los crons correspondientes, y el paciente se presenta un
día en que la clínica no atiende. TASK-127 cierra esa carencia, ya
registrada como observación pendiente.

## Qué se implementó

- `AppointmentsService.cancelForHoliday(organizationId, date, actorId)`:
  cancela todo turno RESERVADO/CONFIRMADO cuya `scheduledAt` cae en la fecha
  dada, **a través de todos los profesionales de la organización** (a
  diferencia de `cancelForAbsence`, que acota por `professionalId`). Reutiliza
  exactamente el mismo patrón que `cancelForAbsence` (P3.6, TASK-39): cada
  turno se cancela en su propia transacción con su propio `try/catch`, para
  que un conflicto de concurrencia sobre uno no impida cancelar el resto ni
  haga fallar el alta del feriado; cada cancelación revoca el código de
  acceso si lo tenía, dispara el mismo enganche hacia `ReassignmentPort` que
  cualquier otra cancelación, notifica al profesional
  (`InAppNotificationsService`) y al paciente con la plantilla
  `APPOINTMENT_CANCELLATION` ya existente.
- Nuevo valor `HOLIDAY` en el enum `CancellationReason` (antes sólo
  `PROFESSIONAL_ABSENCE`/`NO_CONFIRMATION`), vía migración manual
  (`ALTER TYPE ... ADD VALUE`, mismo patrón que las dos anteriores).
- Nuevo puerto `HolidayEventsPort` (`src/domain/ports/holiday-events.port.ts`),
  mismo shape que `AbsenceEventsPort`: un único método
  `holidayRegistered(event)` con `organizationId`/`holidayId`/`date`/
  `actorId`. `HolidaysService.create` publica el evento después de que la
  transacción de alta del feriado confirma (nunca si el alta falla, p. ej. el
  409 de fecha duplicada). `AppointmentHolidayEventsAdapter`
  (`src/appointments/holiday-events.adapter.ts`) es el consumidor real,
  vinculado al token `HOLIDAY_EVENTS_PORT` en `AppointmentsModule` y
  exportado desde ahí; `HolidaysModule` importa `AppointmentsModule` para
  resolver el token, sin necesitar nada más de él.
- `HolidaysService.remove` documenta explícitamente (comentario, no código
  nuevo) que borrar un feriado nunca restaura los turnos que su alta
  canceló, y que el conteo de turnos afectados que ya reportaba pasa a ser 0
  en el caso común ahora que el alta se encarga de vaciar la fecha por
  adelantado.

## Decisiones y por qué

**Cancelar y disparar la reasignación, no marcar para reprogramación
manual.** Era uno de los puntos que la propia tarea pedía definir antes de
implementar. Desde TASK-116 disparar la reasignación sobre una fecha feriado
es inocuo — el motor la reconoce y libera la retención sin ofrecérsela a
nadie —, así que hacerlo mantiene esta cancelación indistinguible de
cualquier otra desde el punto de vista de `fireReassignment`/
`notifyCancellation`/`revokeAccessCode`, en lugar de inventar una tercera
variante de "turno recién cancelado" en un servicio que ya distingue dos
(cancelación ordinaria, cancelación por ausencia).

**Puerto de extensión (`HolidayEventsPort`), aunque no hacía falta para
evitar un ciclo de importación.** A diferencia de `AbsencesService`, que
necesitó una tercera clase de módulo (`AbsencesModule`) situada por encima
de `ProfessionalsModule` y `AppointmentsModule` porque `AppointmentsModule`
ya importa `ProfessionalsModule`, `HolidaysModule` no tenía ningún ciclo que
evitar: nada en la cadena de módulos que `AppointmentsModule` importa
depende de `HolidaysModule`, así que éste podría haber importado
`AppointmentsService` directamente. Se mantuvo igual el mismo patrón de
puerto que `AbsenceEventsPort`, por la razón que motiva esa entrada de
arquitectura en general (CLAUDE.md, "Integration ports"): que un módulo
señale un hecho propio sin depender de la lógica del consumidor. El módulo
de feriados sigue sin saber qué hace el de turnos con la fecha que declara.

**No restaurar turnos al borrar un feriado, y no tratar distinto una fecha
pasada — ambos puntos que la tarea pedía dejar dichos explícitamente aunque
quedaran fuera de alcance.** Lo primero es simétrico a la decisión ya
tomada para `AbsenceCancelledEvent`: reclamar el mismo horario podría ya no
ser posible, y no hay un evento de "feriado cancelado" análogo porque no hay
nada que deshacer. Lo segundo no necesitó una regla nueva: el filtro por
estado RESERVADO/CONFIRMADO ya excluye cualquier turno COMPLETADO o AUSENTE,
de modo que un alta retroactiva sólo alcanza a turnos que nadie resolvió de
una forma u otra — exactamente el caso que la tarea pedía no tocar.

## Alternativas descartadas

- **Consultar los turnos afectados filtrando por `professionalId` uno a uno**
  (recorriendo la lista de profesionales de la organización), en lugar de una
  única consulta sin ese filtro: descartada porque el cliente de Prisma
  acotado por inquilino ya scopea la consulta a la organización activa, y
  filtrar además por profesional habría exigido N consultas (una por
  profesional) para lograr exactamente lo que una sola consulta sin ese
  filtro ya resuelve.
- **Reutilizar `cancelForAbsence` con un rango de un solo día en lugar de un
  método nuevo**: descartada porque `cancelForAbsence` filtra por
  `professionalId`, y generalizar esa firma para aceptar "todos los
  profesionales" habría mezclado dos formas de alcance distintas (un
  profesional en un rango; todos los profesionales en un día) en una única
  función, en lugar de dos métodos con forma idéntica pero alcance propio,
  cada uno más simple de leer que una versión generalizada de ambos.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma` (modificado): `HOLIDAY` agregado a `CancellationReason`.
- `prisma/migrations/20260907190000_add_appointment_cancellation_reason_holiday/`
  (nueva): `ALTER TYPE "CancellationReason" ADD VALUE 'HOLIDAY'`.
- `src/domain/ports/holiday-events.port.ts` (nuevo): `HolidayEventsPort`,
  `HolidayRegisteredEvent`, token `HOLIDAY_EVENTS_PORT`.
- `src/appointments/holiday-events.adapter.ts` (nuevo):
  `AppointmentHolidayEventsAdapter`, consumidor real del puerto.
- `src/appointments/appointments.service.ts` (modificado): método nuevo
  `cancelForHoliday`; comentarios de `revokeAccessCode`/`notifyCancellation`/
  `fireReassignment` actualizados para nombrar el tercer origen de
  cancelación.
- `src/appointments/appointments.module.ts` (modificado): vincula y exporta
  `HOLIDAY_EVENTS_PORT` con `AppointmentHolidayEventsAdapter`, mismo patrón
  que `ABSENCE_EVENTS_PORT`.
- `src/holidays/holidays.service.ts` (modificado): inyecta
  `TenantContextService` y `HOLIDAY_EVENTS_PORT`; `create` publica
  `holidayRegistered` tras confirmar la transacción; comentario nuevo en
  `remove` sobre la no restauración.
- `src/holidays/holidays.module.ts` (modificado): importa `AppointmentsModule`
  para resolver el token.

## Tests y qué validan

- `src/appointments/appointments-rescheduling.service.spec.ts` (modificado):
  nuevo `describe('cancelForHoliday')` — cancela todos los turnos
  RESERVADO/CONFIRMADO de la fecha a través de profesionales distintos
  (probando que la consulta no filtra por `professionalId`), tagea el motivo
  `HOLIDAY`, dispara la reasignación y el aviso al paciente con la plantilla
  correspondiente, no hace nada si no hay turnos ese día, y sigue cancelando
  el resto si uno de los turnos cambia de estado concurrentemente.
- `src/appointments/holiday-events.adapter.spec.ts` (nuevo): el adaptador
  invoca `cancelForHoliday` con los campos del evento.
- `src/holidays/holidays.service.spec.ts` (modificado): `create` publica
  `holidayRegistered` con el tenant/id/fecha/actor correctos tras confirmar,
  y no lo publica si el alta falla (violación de unicidad).
- `test/holidays.e2e-spec.ts` (modificado, contra Postgres real): fija el
  orden de limpieza entre `AuditLog` y `Appointment` en `cleanup()`/
  `afterEach` — `AuditLog.appointmentId` es una FK real (TASK-34) y esta
  tarea es la primera en esta suite que efectivamente escribe una entrada de
  auditoría referida a un turno, así que borrar el turno primero pasó a
  violar la restricción, algo que ningún test anterior de este archivo había
  ejercitado. Se agregaron tres casos: alta de feriado cancela turnos
  RESERVADO/CONFIRMADO de dos profesionales distintos de la misma
  organización, con motivo `HOLIDAY` y entrada de auditoría por cada uno; no
  afecta turnos de otro estado (COMPLETADO) ni de otra fecha; no tiene efecto
  alguno si no hay turnos ese día.
- Ejecución: suite unitaria completa en verde (80 conjuntos, 788 pruebas),
  suite end-to-end completa en verde (51 conjuntos, 562 pruebas) y cobertura
  combinada (131 conjuntos, 1350 pruebas, 97.25 %/82.48 % líneas/ramas sobre
  la capa de servicios, por encima del umbral del 80 %), todas contra la
  instancia local de PostgreSQL con `--runInBand`; `tsc --noEmit` y `eslint`
  sin hallazgos. Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva. La corrección no agrega un flujo que la tesis no describa ya:
extiende el diagrama de cancelación de turnos (motivo de ausencia) con un
tercer origen, y el diagrama del ciclo de vida del feriado ya pendiente
(`figuras_pendientes.md`) puede incorporar esta cancelación cuando se
produzca.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-127-holiday-cancels-appointments`,
  creada desde `origin/main` (`e2d32a5`, con TASK-125 ya mergeado). Pusheada
  a `origin`.
- Ticket: TASK-127 (Jira), "[MEJORA] El alta de un feriado no cancela los
  turnos ya reservados en esa fecha". Misma convención de bitácora dedicada
  para tareas puntuales dentro de la fase del ticket original que
  TASK-79/TASK-81/TASK-86/TASK-94/TASK-95/TASK-96/TASK-100/TASK-108/
  TASK-110/TASK-113/TASK-114/TASK-116/TASK-117/TASK-123/TASK-115/TASK-121
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]], [[FASE-3_PROMPT-15]],
  [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]], [[FASE-3_PROMPT-18]],
  [[FASE-3_PROMPT-19]], [[FASE-3_PROMPT-23]], [[FASE-3_PROMPT-24]],
  [[FASE-3_PROMPT-25]], [[FASE-3_PROMPT-26]], [[FASE-3_PROMPT-27]],
  [[FASE-3_PROMPT-28]], [[FASE-3_PROMPT-29]], [[FASE-3_PROMPT-30]],
  [[FASE-3_PROMPT-31]]). Referencia directa: la observación pendiente dejada
  en [[FASE-3_PROMPT-27]] (TASK-116) es exactamente la carencia que esta
  tarea cierra.
