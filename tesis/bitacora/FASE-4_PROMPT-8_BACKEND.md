# Fase 4 — Notificaciones y Scheduler (backend) — FK compuesta e índices para `Notification.appointmentId`/`.prescriptionRequestId` (TASK-90, corrección a TASK-76)

## Qué se implementó

TASK-90 fue una tarea de corrección originada en una auditoría de la base
de datos viva de `psique-back` (2026-08-12): el modelo `Notification`
(TASK-76, P7.b) es hijo directo de `Professional` (sin `organizationId`
propio, patrón 2 del esquema), pero sus dos columnas opcionales de
referencia —`appointmentId` y `prescriptionRequestId`— eran FKs planas
contra `Appointment(id)` y `PrescriptionRequest(id)`, sin ningún
componente de profesional. A diferencia de todos los demás casos del
esquema donde una entidad con un padre tenant-scoped apunta a otra entidad
tenant-scoped (`AuditLog→Patient/Appointment`, `WaitlistOffer→
Appointment/WaitlistEntry`, `User→Professional`), que usan FK compuesta
contra un `@@unique` del target, nada impedía que una notificación de un
profesional apuntara a un turno o una solicitud de receta de otro
profesional, incluso de otra organización — verificado contra
`pg_constraint` en la base viva. Ninguna de las dos columnas tenía
tampoco índice de soporte (`pg_indexes` no devolvía ninguno para
`appointmentId`/`prescriptionRequestId` en `Notification`), a diferencia
del resto de las FKs opcionales del esquema.

Cambios en `prisma/schema.prisma`:

- Se agregó `@@unique([professionalId, id])` a `Appointment` y a
  `PrescriptionRequest`, para poder anclar la FK compuesta contra ellos
  (ambos ya tenían `professionalId` como columna propia).
- `Notification.appointmentId`/`.prescriptionRequestId` pasaron a ser FKs
  compuestas `(professionalId, targetId)` contra esos dos `@@unique`
  nuevos — no contra `organizationId`, porque `Notification` no tiene esa
  columna (patrón 2); el campo que su propio padre determina es
  `professionalId`, así que es contra ese campo que se compone la FK.
- Se agregaron los índices de soporte `@@index([professionalId,
  appointmentId])` y `@@index([professionalId, prescriptionRequestId])`
  en `Notification` (PostgreSQL no indexa columnas de FK
  automáticamente).
- Migración nueva:
  `20260813193022_notification_composite_fk_appointment_prescription`
  (generada vía `prisma migrate diff --shadow-database-url` contra una
  base sombra descartable, mismo procedimiento que TASK-82/TASK-92,
  porque `prisma migrate dev` no corre de forma no interactiva en este
  entorno). Sin pérdida de datos: las tablas involucradas estaban vacías
  en desarrollo.

## Decisiones y por qué

**`ON DELETE CASCADE`, no `RESTRICT`, en las dos FKs compuestas nuevas.**
Una FK compuesta con `professionalId` como columna `NOT NULL` no puede
usar `SET NULL` — Postgres intentaría anular también `professionalId`,
el mismo problema ya documentado en el esquema para
`User.professional`/`WaitlistOffer.waitlistEntry` (que por eso usan
`Restrict` y un `null` manual del lado del servicio antes de borrar).
Para `Notification` se descartó `Restrict` por dos razones: (1) una
notificación sobre un turno o una solicitud que ya no existe no tiene
nada que describir, así que la eliminación en cascada es lo
semánticamente correcto — el propio texto del ticket menciona "backing
de la FK y de la cascada/SET NULL", anticipando alguna forma de cascada;
y (2) `Notification` ya cascadea directamente desde `Professional`
(`onDelete: Cascade`), y `Appointment` también cascadea desde
`Professional` — con `Restrict` en la FK nueva, un futuro borrado físico
de `Professional` podría fallar según el orden en que Postgres procese
las dos cascadas simultáneas hacia `Appointment` y hacia `Notification`
(un problema de "cascada en diamante"). `Cascade` evita ese riesgo de
raíz.

**Se verificó el impacto en la suite e2e antes de decidir esto**, no solo
en teoría: varios specs (`in-app-notifications`, `waitlist-offer-timeout`,
`appointment-reassignment`, `appointment-engine-integration`,
`appointments-states`) crean notificaciones indirectamente al cancelar
turnos o aceptar reasignaciones, y sus rutinas de limpieza (`afterEach`/
`afterAll`) borran `Appointment` sin borrar `Notification` primero
(confiaban en que `Notification` solo cascadeaba desde `Professional`,
mucho más adelante en el orden de limpieza). Con `Restrict` esas
limpiezas habrían empezado a fallar por violación de FK; con `Cascade`
no fue necesario tocar el orden de limpieza de ningún spec — se confirmó
corriendo la suite completa.

**No se expuso `InAppNotificationsService.create()` por HTTP** ni se
tocó su firma — sigue fuera de alcance, tal como el ticket lo aclara. Los
dos únicos llamadores (`AppointmentsService.notifyCancellation`,
`WaitlistReassignmentService.notifyReassignment`) ya pasaban
`professionalId` y `appointmentId` de la misma entidad, así que no
necesitaron cambios: la FK compuesta nueva no les exige nada que no
estuvieran haciendo.

## Entidades / puertos / adaptadores tocados

- `Appointment` (Prisma): nuevo `@@unique([professionalId, id])`.
- `PrescriptionRequest` (Prisma): nuevo `@@unique([professionalId, id])`.
- `Notification` (Prisma): `appointment`/`prescriptionRequest` pasan de FK
  simple a FK compuesta `(professionalId, id)` con `onDelete: Cascade`;
  nuevos índices `@@index([professionalId, appointmentId])` y
  `@@index([professionalId, prescriptionRequestId])`.
- Migración nueva:
  `20260813193022_notification_composite_fk_appointment_prescription`.

## Tests y qué validan

Se agregaron dos casos e2e a `test/in-app-notifications.e2e-spec.ts`,
directamente contra Prisma (no vía HTTP, ya que `create()` no está
expuesto):

- Crear una `Notification` con `appointmentId` de un turno de **otro**
  profesional falla a nivel de base de datos (violación de la FK
  compuesta), no solo por convención del servicio.
- Mismo caso para `prescriptionRequestId` de una solicitud de **otro**
  profesional.

Verificación adicional (mismo método que usó la auditoría original y que
TASK-91):

- Las tres FKs y los dos índices nuevos existen en la base tras la
  migración, confirmado vía `\d "Notification"` y `pg_indexes` contra
  Postgres local.

Suite completa: 38 suites unitarias / 416 pruebas en verde; 37 suites e2e
/ 439 pruebas en verde (`--runInBand`, incluye los 2 casos nuevos). Lint
limpio y verificación de tipos (`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-90-notification-composite-fk-index` (creada desde
  `origin/main` fresco, tras el merge de TASK-96). Pusheada a `origin`;
  PR abierto, no fusionado aún.
- Ticket: TASK-90 ("[CORRECCIÓN] Notification.appointmentId/
  prescriptionRequestId sin FK compuesta ni índice"), corrección a
  TASK-76 (P7.b, [[FASE-4_PROMPT-6]], creó `Notification`). Misma
  convención de bitácora dedicada para correcciones pequeñas dentro de la
  fase del ticket original que TASK-91 ([[FASE-4_PROMPT-7]]), TASK-84
  ([[FASE-2_PROMPT-11]]) y TASK-79/TASK-81 ([[FASE-3_PROMPT-12]],
  [[FASE-3_PROMPT-14]]).
