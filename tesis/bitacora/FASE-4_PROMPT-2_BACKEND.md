# Fase 4 — Notificaciones y Scheduler (backend) — cron de confirmación a 24h (TASK-43, P4.2)

## Qué se implementó

`AppointmentConfirmationCron` (`src/appointments/appointment-confirmation.cron.ts`),
un job de `@nestjs/schedule` que corre cada hora y detecta los TURNO en
estado RESERVADO cuya `fecha_hora` cae en la ventana de detección de 24h
que pide el documento de requisitos (entre 23h y 25h desde el momento en
que corre el job). Para cada turno candidato, renderiza la plantilla
`CONFIRMACION_TURNO` a través de `NotificationTemplateService` (TASK-42,
P4.1) con el nombre del paciente, el nombre del profesional y la fecha y
hora del turno, y envía el texto resultante mediante `MessagingPort` (el
adaptador stub hoy; la integración real de WhatsApp llega en M5/P5.1,
TASK-46). El intento queda registrado en `REGISTRO_AUDITORIA`
(`action: CONFIRMATION_SENT`, con el `id_turno`).

Al igual que el cron semanal de auto-completado de turnos (TASK-89), este
job recorre todas las organizaciones existentes, abre su propio contexto
de tenant por organización (`TenantContextService.run`) y atribuye cada
entrada de auditoría al usuario `SYSTEM` de esa organización, porque un
cron no tiene una request de la que heredar el tenant.

La respuesta del paciente (SI/NO) queda explícitamente fuera de alcance
de esta tarea, tal como lo indica el propio ticket: la procesará el
chatbot de M5, llamando a `AppointmentsService.confirmar()` (ya
implementado desde el Motor de Turnos) o cancelando el turno.

## Decisiones y por qué

**Se agregó una columna nueva, `confirmationRequestedAt`, en lugar de
reutilizar `confirmedAt` para la idempotencia que pide el ticket.** El
texto del ticket describe el criterio de "no reenviar" en términos de
`fecha_confirmacion` no nula, pero esa columna ya tenía, desde el Motor
de Turnos (TASK-38), un significado distinto y ya construido: se completa
únicamente cuando el paciente responde que sí confirma, a través de
`AppointmentsService.confirmar()`. Reutilizarla para marcar "ya se envió
la solicitud" habría hecho que ambos significados —"ya se preguntó" y "ya
contestó que sí"— convivieran en la misma columna, y además habría hecho
inútil la propia idempotencia: la ventana de detección dura dos horas y
el job corre cada hora, de modo que sin una marca propia de "ya
preguntado" un turno sin respuesta se re-consultaría y se le reenviaría
el mensaje en la corrida siguiente. Se trató el texto del ticket como la
descripción funcional del requisito de idempotencia, no como un contrato
literal sobre qué columna usar — mismo criterio que ya se aplicó al
resolver el nombre de las claves de plantilla en TASK-42 y el código de
estado en TASK-78.

**El registro del intento se hizo en `REGISTRO_AUDITORIA`, no en una
tabla de log de notificaciones aparte.** El propio ticket ofrece ambas
alternativas ("o en una tabla de log de notificaciones"); se optó por la
auditoría porque ya es el mecanismo establecido en el proyecto para
"quién hizo qué, cuándo, sobre qué recurso" (Ley 25.326), ya admite un
vínculo tipado a `Appointment` y a `Patient`, y crear una tabla nueva
solo para este caso habría duplicado esa capacidad sin necesidad.

**Un paciente sin celular registrado se omite sin marcar el turno como
"ya solicitado".** El documento de requisitos no contempla explícitamente
este caso. Se siguió el mismo criterio de tolerancia que ya usa
`WaitlistReassignmentService.offer` para la oferta de reasignación: un
paciente sin número simplemente no puede ser contactado, y dejar el
turno sin marcar permite que una corrida posterior lo vuelva a intentar
si el dato se completa mientras el turno sigue dentro de la ventana.

**El envío y el registro no comparten una única transacción con la
base.** El envío por `MessagingPort` es una llamada a un servicio externo
(hoy un stub, mañana WhatsApp), y una escritura a la base de datos no
puede revertir un efecto ya ocurrido fuera de ella. Se envía primero el
mensaje y, solo si el envío no lanza una excepción, se marca
`confirmationRequestedAt` y se escribe la entrada de auditoría dentro de
una misma transacción — así la entrada de auditoría sí sigue committeando
junto con la escritura que describe, como exige la convención del
proyecto, incluso cuando el paso previo (el envío) queda fuera de ese
alcance transaccional por naturaleza.

**Formato de fecha del mensaje ("DD/MM/AAAA HH:mm"), leído de los campos
UTC del `Date` sin conversión de huso horario real.** No existía en el
proyecto ningún punto que formateara un instante (`DateTime`, no un
`DATE` de calendario) como texto legible para un paciente; se agregó un
formateador local a la tarea, siguiendo la misma simplificación ya
documentada para el motor de disponibilidad (TASK-35): el sistema no
tiene todavía configuración de huso horario por inquilino, así que la
hora de pared se trata como texto literal.

## Alternativas descartadas

- **Reutilizar `confirmedAt` para la idempotencia**, tal como sugiere
  literalmente el texto del ticket: descartada por las razones ya
  expuestas — colisiona con el significado que esa columna ya tiene desde
  TASK-38 y no resuelve el problema real de reenvío dentro de la ventana
  de dos horas con un cron que corre cada hora.
- **Registrar el intento en una tabla de log de notificaciones nueva**:
  descartada a favor de `REGISTRO_AUDITORIA`, ya construido para este
  propósito y ya vinculado a `Appointment`/`Patient`.
- **No enviar el mensaje a un turno sin celular y, aun así, marcarlo como
  ya solicitado** (para no volver a evaluarlo cada hora): descartada
  porque impediría que el turno se contactara si el dato de contacto se
  completara mientras el turno sigue dentro de la ventana de 24h.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: `Appointment.confirmationRequestedAt`
  (`DateTime?`, nulo hasta que el cron envía la solicitud).
- `prisma/migrations/20260812150000_add_appointment_confirmation_requested_at/`.
- `src/appointments/appointment-confirmation.cron.ts`:
  `AppointmentConfirmationCron`, inyecta `PrismaService`,
  `TenantScopedPrismaService`, `TenantContextService`,
  `NotificationTemplateService`, `MESSAGING_PORT` y `AuditService`.
- `src/appointments/appointments.module.ts`: registra el cron como
  provider (no exportado, igual que `AppointmentAutoCompletionCron`) e
  importa `NotificationsModule` para poder inyectar
  `NotificationTemplateService`.

No se tocó `MessagingPort` ni sus adaptadores: el stub existente
(`StubMessagingAdapter`) ya cubre lo que esta tarea necesita ("loguea el
intento"), y la integración real de WhatsApp es explícitamente de M5
(TASK-46).

## Tests y qué validan

- `appointment-confirmation.cron.spec.ts` (8 pruebas unitarias, con
  Prisma y los puertos mockeados): un turno exactamente a 24h se
  selecciona, se renderiza y envía la plantilla, se marca
  `confirmationRequestedAt` y se audita con `action: 'CONFIRMATION_SENT'`;
  un turno a 26h queda fuera de la ventana de detección; un turno con
  `confirmationRequestedAt` ya registrado no se reenvía; un paciente sin
  celular se omite sin marcar el turno; una organización sin usuario
  `SYSTEM` se salta sin interrumpir la corrida; una actualización
  concurrente (`count: 0`) no genera una segunda entrada de auditoría; un
  fallo en un turno no interrumpe el resto del lote de la organización; el
  recorrido cubre cada organización bajo su propio contexto de tenant. El
  mock de `findMany` filtra un arreglo en memoria replicando el `where`
  real de Prisma, para que las pruebas de ventana (24h/26h) ejerciten la
  cláusula real en vez de duplicar su aritmética de fechas como aserción
  aparte.
- `test/appointment-confirmation.e2e-spec.ts` (5 pruebas, Postgres real,
  mismo patrón que `appointment-auto-completion.e2e-spec.ts`): turno a
  24h queda con `confirmationRequestedAt` no nulo y `confirmedAt` sin
  tocar, con su entrada de auditoría; turno a 26h queda sin marcar; turno
  ya marcado no se reenvía ni duplica la auditoría; turno sin celular
  queda sin marcar; organización sin usuario `SYSTEM` no interrumpe la
  corrida.
- Ejecución: 33 suites unitarias / 365 pruebas en verde (8 nuevas); 34
  suites e2e / 414 pruebas en verde (`--runInBand`, 5 nuevas). Lint
  limpio (`npm run lint` sobre todo el repo).

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-43-appointment-confirmation-cron`
  (creada desde `origin/main` fresco, tras el merge de TASK-42). Pusheada
  a `origin`, no fusionada aún.
- Ticket: TASK-43 (P4.2 — Confirmación 24 h antes), segunda tarea de
  Módulo 4. Depende de TASK-42 (motor de plantillas,
  [[FASE-4_PROMPT-1]]) y de TASK-38 (máquina de estados del turno,
  [[FASE-3_PROMPT-6]] o equivalente — `AppointmentsService.confirmar()`).
  Es requisito declarado de P4.3 (autocancelación si no responde,
  TASK-44) y de la integración real de WhatsApp de M5/P5.1 (TASK-46),
  ninguna de las dos implementada todavía.
