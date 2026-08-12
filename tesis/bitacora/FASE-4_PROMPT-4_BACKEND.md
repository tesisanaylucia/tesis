# Fase 4 — Notificaciones y Scheduler (backend) — recordatorio configurable y placeholder de expiración de códigos (TASK-45, P4.4)

## Qué se implementó

Dos trabajos programados independientes, tal como los describe el ticket:

1. `AppointmentReminderCron`
   (`src/appointments/appointment-reminder.cron.ts`): corre cada hora y
   busca, en cada organización, los TURNO en estado CONFIRMADO cuya
   `scheduledAt` cae dentro de la ventana de recordatorio configurada
   para ese inquilino (`OrganizationConfig`, clave
   `appointment_reminder_hours_before`, 24 horas por defecto). Para cada
   turno candidato renderiza la plantilla RECORDATORIO_TURNO
   (`NotificationTemplateKey.APPOINTMENT_REMINDER`) con el nombre del
   profesional y la hora del turno, la envía por `MessagingPort` (el
   adaptador de prueba) y marca `reminderSentAt` (columna nueva en
   `Appointment`) para no reenviar en una corrida posterior. El
   recorrido por organización, la apertura de contexto de tenant y la
   atribución a un usuario `SYSTEM` siguen el mismo patrón que
   `AppointmentConfirmationCron` y `AppointmentAutoCancellationCron`.
2. `AccessCodeExpirationCron`
   (`src/access-codes/access-code-expiration.cron.ts`): el placeholder
   que pide el ticket para el futuro job de expiración de códigos de
   acceso de la cerradura TTLock. Corre cada 15 minutos, no recibe
   ninguna dependencia inyectada y solo registra en el log que es un
   placeholder — no hay ninguna entidad `AccessCode` todavía contra la
   que consultar (esa entidad llega en M6/P6.1, TASK-55; la lógica real
   de expiración es M6/P6.3, TASK-57). Vive en un módulo nuevo,
   `AccessCodesModule`, pensado como el lugar donde esa futura entidad y
   su servicio van a vivir.

Se agregaron dos migraciones: una que agrega la columna
`Appointment.reminderSentAt`, y otra que siembra
`appointment_reminder_hours_before=24` para las organizaciones
existentes, siguiendo el mismo patrón que
`appointment_cancellation_min_hours` y `patient_inactivity_months`
(regla de negocio de alcance tenant sembrada por migración, no solo en
el script de desarrollo, para que exista como fila real desde el primer
entorno productivo). `prisma/seed.ts` se actualizó con el mismo valor
por defecto para organizaciones sembradas después.

## Decisiones y por qué

**El recordatorio solo selecciona turnos CONFIRMADO, no RESERVADO.** El
ticket lo pide así explícitamente, y es coherente con el resto del
módulo: el recordatorio es una cortesía para un turno que el paciente ya
confirmó, un paso distinto y posterior a la solicitud de confirmación
misma (P4.2), que sí opera sobre RESERVADO.

**La ventana de recordatorio es configurable por inquilino, a diferencia
de la ventana fija de 23h-25h del job de confirmación.** El propio
ticket lo pide en esos términos ("por defecto X horas antes del turno,
configurable por tenant"), así que se modeló como una fila más en
`OrganizationConfig`, con el mismo mecanismo de valor por defecto y
siembra por migración ya usado para `appointment_cancellation_min_hours`
y `patient_inactivity_months` — un valor que no es un entero positivo se
descarta a favor del valor por defecto, no se confía ciegamente en lo
que haya en la fila, siguiendo el mismo criterio de
`PatientInactivityService.threshold`. Se eligió 24 horas como valor por
defecto porque coincide con la ventana que ya usa el job de confirmación
y con el propio texto de la plantilla base ("...mañana a las {time}").
La banda de detección alrededor de ese valor configurado sigue siendo de
±1 hora, la misma técnica ya usada por el job de confirmación para que
un turno no pueda atravesar sin ser detectado la brecha entre dos
corridas horarias consecutivas.

**La idempotencia se resolvió con una columna nueva (`reminderSentAt`) en
lugar de reutilizar alguna existente**, exactamente el mismo
razonamiento que llevó a `confirmationRequestedAt` en el job de
confirmación: ninguna columna existente significa "ya se envió el
recordatorio", y superponerle ese significado a otra hubiera sido
ambiguo además de insuficiente para el caso de una ventana que puede
verse en más de una corrida.

**El job de expiración de códigos se dejó deliberadamente sin cuerpo
real**, porque no hay entidad `AccessCode` en el esquema todavía — el
propio comentario transicional que ya existía en el modelo `LockLog`
señala que esa entidad llega recién en M6/P6.1 (TASK-55). El ticket pide
explícitamente solo el andamiaje: firma final, corre sin errores, no
hace nada. Se optó por un módulo nuevo (`src/access-codes/`) en lugar de
ubicar el placeholder dentro de algún módulo existente, para que ya
exista el lugar natural donde la futura entidad, su servicio y la lógica
real de expiración (TASK-57) se van a agregar sin tener que reorganizar
nada. El comentario en el código deja escrita la consulta que el job va
a resolver una vez que la entidad exista (`validUntil < ahora` y
`status = ACTIVE`), para que la tarea futura no tenga que redescubrir esa
decisión.

**El job de expiración corre cada 15 minutos, distinto de la hora del
recordatorio.** Además de dejar concretada la independencia entre ambos
jobs que pide el ticket ("pueden configurarse con distintas
frecuencias"), la frecuencia más corta tiene una justificación de
dominio propia: un código de acceso vigente más allá de lo debido es una
exposición de acceso físico, de una naturaleza distinta a una demora en
el envío de un mensaje. Se usó una expresión cron literal en vez de
`CronExpression.EVERY_15_MINUTES` porque esa constante no existe en
`@nestjs/schedule`, el mismo motivo ya documentado en
`AppointmentAutoCancellationCron` (TASK-44).

## Entidades / puertos / adaptadores tocados

- `Appointment.reminderSentAt` (columna nueva, `DateTime?`): marca de
  idempotencia del recordatorio, análoga a `confirmationRequestedAt`.
- `appointment_reminder_hours_before` (clave nueva de
  `OrganizationConfig`): ventana de recordatorio por inquilino, con
  valor por defecto 24 sembrado por migración y por `seed.ts`.
- `AppointmentReminderCron` (nuevo): job de `@nestjs/schedule`,
  registrado como proveedor privado de `AppointmentsModule`, junto a los
  otros tres crons del módulo.
- `AccessCodesModule` / `AccessCodeExpirationCron` (nuevos): módulo y job
  registrados en `AppModule`; sin dependencias inyectadas todavía.
- `NotificationTemplateService` / `MessagingPort` (ya existentes):
  consumidos por `AppointmentReminderCron` de la misma forma que por
  `AppointmentConfirmationCron`.

## Tests agregados

- `appointment-reminder.cron.spec.ts` (11 pruebas unitarias, con Prisma
  simulado): envía el recordatorio a un turno CONFIRMADO dentro de la
  ventana por defecto de 24h y registra el intento; no selecciona un
  turno a 26h (fuera de la ventana); no selecciona un turno RESERVADO;
  no reenvía si `reminderSentAt` ya está seteado; usa la ventana
  configurada por el inquilino en lugar de la de 24h por defecto cuando
  existe una fila de configuración; vuelve al valor por defecto si el
  valor configurado no es un entero positivo; no marca como enviado un
  turno cuyo paciente no tiene celular registrado; una organización sin
  usuario `SYSTEM` no interrumpe la corrida; una actualización con cero
  filas afectadas (condición de carrera) omite la auditoría; una falla
  en un turno del lote no impide que el resto se procese; la corrida
  recorre cada organización bajo su propio contexto de tenant.
- `access-code-expiration.cron.spec.ts` (2 pruebas unitarias): corre sin
  lanzar errores y deja registrado el mensaje de placeholder; corridas
  sucesivas no producen ningún efecto observable más allá de ese mismo
  mensaje de log.
- Ejecución: 36 suites unitarias / 387 pruebas en verde (13 nuevas); 34
  suites e2e / 414 pruebas en verde, sin cambios en el conteo de e2e
  —igual que TASK-44, la lógica de ambos jobs es enteramente reproducible
  con Prisma simulado y reloj controlado, sin necesitar una prueba contra
  Postgres real—. Lint limpio (`npm run lint` sobre todo el repo) y
  verificación de tipos sin errores (`tsc --noEmit`). Migraciones
  aplicadas contra Postgres local (`prisma migrate deploy`) y script de
  siembra (`npm run seed`) verificados sin errores.

## Figuras pendientes

Una: línea de tiempo de los tres trabajos programados del turno
alrededor de una cita confirmada (recordatorio configurable por
inquilino antes del turno, y el placeholder de expiración de código de
acceso corriendo cada 15 minutos en paralelo, señalando que este último
todavía no produce ningún efecto). Agregada a `figuras_pendientes.md`
como figura 26.

## Marco Teórico ofrecido

El tema "2.4 Arquitectura de software (extensión: scheduling)" que
habilita la Fase 4 según `mapa_fases_capitulos.md` todavía no se ofreció
ni se redactó en ninguna de las tres tareas anteriores de esta fase
(TASK-42, TASK-43, TASK-44); tampoco se redactó en esta. Queda pendiente
de ofrecer a la usuaria.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-45-reminder-and-code-expiration-crons` (creada desde
  `origin/main` fresco, tras el merge de TASK-44). Pusheada a `origin`,
  no fusionada aún.
- Ticket: TASK-45 (P4.4 — Recordatorios y expiración de códigos
  (placeholder)), cuarta tarea de Módulo 4. Depende de TASK-42
  (motor de plantillas, [[FASE-4_PROMPT-1]]), TASK-43 (confirmación a
  24h, [[FASE-4_PROMPT-2]]) y TASK-34 (placeholder de CODIGO_ACCESO,
  [[FASE-3_PROMPT-1]]). Deja pendiente, declarado fuera de alcance por
  el propio ticket: la expiración real de códigos TTLock (M6/P6.3,
  TASK-57), la inclusión del código de acceso en el texto del
  recordatorio (una vez que M6 exista) y el envío real por WhatsApp
  (M5/P5.1, TASK-46).
