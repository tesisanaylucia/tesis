# Fase 4 — Notificaciones y Scheduler (backend) — autocancelación a las 4h sin respuesta (TASK-44, P4.3)

## Qué se implementó

`AppointmentAutoCancellationCron`
(`src/appointments/appointment-auto-cancellation.cron.ts`), el segundo
trabajo programado del módulo: corre cada 15 minutos y busca, en cada
organización, los TURNO en estado RESERVADO cuya `confirmationRequestedAt`
—la marca que deja `AppointmentConfirmationCron` (TASK-43, P4.2) al enviar
la solicitud de confirmación— tiene 4 horas o más de antigüedad. Cada
candidato se cancela (`estado → CANCELADO`,
`motivo_cancelacion → NO_CONFIRMATION`, un valor nuevo agregado al enum
existente) y dispara `ReassignmentPort.appointmentCancelled` para que el
algoritmo de reasignación de M3 (TASK-40) ofrezca el turno liberado a la
lista de espera del profesional, exactamente el mismo mecanismo que ya usa
`AppointmentsService.cancel` para una cancelación ordinaria.

El recorrido por organización, la apertura de contexto de tenant por
organización (`TenantContextService.run`) y la atribución de la entrada de
auditoría al usuario `SYSTEM` de cada organización siguen el mismo patrón
que los otros dos jobs del proyecto (`AppointmentConfirmationCron`,
`AppointmentAutoCompletionCron`): un cron no tiene una request de la que
heredar el tenant, así que abre el suyo propio por cada organización
existente, y omite (con una advertencia en el log) una organización que no
tenga un usuario `SYSTEM` sembrado.

## Decisiones y por qué

**El cálculo del plazo es una simple comparación de marcas de tiempo, sin
ninguna lógica especial para fin de semana.** El propio ticket aclara
explícitamente que la mención a "contemplar fines de semana" del documento
de requisitos original no significa que el plazo de 4 horas se extienda si
la solicitud de confirmación se envió un viernes y el turno es un lunes,
sino que el job debe seguir corriendo también en fin de semana para
detectar esos casos a tiempo. Como el job de `@nestjs/schedule` corre en
el mismo proceso de la aplicación de forma continua, sin ninguna condición
que lo pause los fines de semana, esa aclaración no requirió ningún código
adicional más allá de comparar `confirmationRequestedAt` contra "ahora
menos 4 horas" — el propio test de cruce de fin de semana (ver más abajo)
verifica esto sin necesitar ninguna rama especial en la implementación.

**Se agregó un valor nuevo al enum `CancellationReason` existente
(`NO_CONFIRMATION`) en lugar de dejar `motivo_cancelacion` en null o crear
un mecanismo paralelo.** El enum ya distinguía la cancelación masiva por
ausencia del profesional (`PROFESSIONAL_ABSENCE`, TASK-39) de una
cancelación ordinaria (que lo deja en null); la autocancelación por falta
de respuesta es, de la misma manera, un motivo que vale la pena poder
distinguir después de una cancelación pedida por el paciente o el
profesional a través de `PATCH /turnos/:id/cancelar`, así que se extendió
el mismo mecanismo en vez de introducir uno nuevo. Como Prisma 6.19.3 no
soporta agregar un valor a un enum de PostgreSQL dentro de una migración
declarativa generada automáticamente sin recrear el tipo, se escribió la
migración a mano con `ALTER TYPE ... ADD VALUE`, el mismo patrón ya usado
para agregar el rol `SYSTEM` (TASK-72).

**El job no reutiliza `AppointmentsService.cancel` ni pasa por la ventana
de aviso mínimo de cancelación (las "4 horas de anticipación" que sí
aplican a una cancelación pedida por una persona).** Esa ventana existe
para impedir que un paciente o profesional cancele a último momento sin
avisar; no tiene sentido aplicada a una cancelación que el propio sistema
dispara porque nadie respondió a tiempo. El job escribe la transición de
estado directamente contra el cliente de Prisma con alcance de tenant,
siguiendo el mismo patrón ya usado por
`AppointmentsService.cancelForAbsence` (que tampoco pasa por esa ventana,
por la misma razón) y por los otros dos crons del proyecto.

**La actualización de estado está protegida contra condiciones de
carrera con la misma técnica que el resto del proyecto**: la escritura
lleva `status: RESERVED` en su propia cláusula `where`, así que si el
paciente confirma o cancela el turno entre la lectura y la escritura del
job, la actualización no afecta ninguna fila y el job simplemente no
genera ni la entrada de auditoría ni el disparo de reasignación para ese
turno — no hay una segunda cancelación compitiendo con lo que la persona
ya decidió.

## Entidades / puertos / adaptadores tocados

- `CancellationReason` (enum de Prisma): nuevo valor `NO_CONFIRMATION`,
  agregado con una migración manual (`ALTER TYPE ... ADD VALUE`).
- `AppointmentAutoCancellationCron` (nuevo): job de `@nestjs/schedule`,
  registrado como proveedor privado de `AppointmentsModule`, igual que los
  otros dos crons del módulo.
- `REASSIGNMENT_PORT` (puerto ya existente, TASK-40): consumido
  directamente por el nuevo cron, sin pasar por `AppointmentsService`, el
  mismo puerto que ya dispara una cancelación ordinaria o una cancelación
  masiva por ausencia.
- `AuditService`: una entrada `action: CANCEL` por turno autocancelado,
  con `detail: { reason: NO_CONFIRMATION }`.

## Tests agregados

- `appointment-auto-cancellation.cron.spec.ts` (9 pruebas unitarias, con
  Prisma simulado): cancela un turno con solicitud de confirmación de
  4h01m de antigüedad y dispara la reasignación; no toca un turno de
  3h59m; no selecciona un turno ya CONFIRMADO ni uno ya CANCELADO; cruza
  el fin de semana correctamente (solicitud enviada un viernes a las
  23:00, turno el lunes, corrida del job el lunes a la mañana); una
  organización sin usuario `SYSTEM` no interrumpe la corrida; una
  actualización con cero filas afectadas (condición de carrera) omite la
  auditoría y la reasignación; una falla en un turno del lote no impide
  que el resto se procese; la corrida recorre cada organización bajo su
  propio contexto de tenant.
- Ejecución: 34 suites unitarias / 374 pruebas en verde (9 nuevas); 34
  suites e2e / 414 pruebas en verde (sin cambios en el conteo de e2e —
  esta tarea no agregó pruebas e2e nuevas, solo unitarias, siguiendo el
  criterio de que la lógica del job es enteramente reproducible con
  Prisma simulado y reloj controlado). Lint limpio
  (`npm run lint` sobre todo el repo).

## Figuras pendientes

Una: diagrama de secuencia de la autocancelación por falta de confirmación
(cron de confirmación envía la solicitud → transcurren 4 horas sin
respuesta → cron de autocancelación detecta el turno vencido → transición
a CANCELADO con motivo NO_CONFIRMATION → disparo de ReassignmentPort).
Agregada a `figuras_pendientes.md`.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-44-appointment-auto-cancellation-cron`
  (creada desde `origin/main` fresco, tras el merge de TASK-43). Pusheada
  a `origin`, no fusionada aún.
- Ticket: TASK-44 (P4.3 — Autocancelación a las 4h sin respuesta), tercera
  tarea de Módulo 4. Depende de TASK-43 (confirmación a 24h,
  [[FASE-4_PROMPT-2]]), TASK-40 (algoritmo de reasignación por prioridad,
  [[FASE-3_PROMPT-7]]) y TASK-38 (máquina de estados del turno,
  [[FASE-3_PROMPT-5]]). Deja pendiente, declarado fuera de alcance por el
  propio ticket: el envío real de WhatsApp (M5/P5.1, TASK-46).
