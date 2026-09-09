# Fase 3 — Motor de Turnos (backend) — desfase sistemático de 3 horas entre la hora real y `scheduledAt` guardado (TASK-150, corrección a TASK-35 y otras)

## Contexto

TASK-150 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, validada por la usuaria
antes de implementarse (hallazgo TUR-2, severidad crítica). El detalle
completo de la auditoría está en el artefacto
`https://claude.ai/code/artifact/5c2ee112-c426-4d6e-a8e2-499ea7bbbc74`
("auditoria-psique-back.md").

El hallazgo señalaba que `AvailabilityService` construye la grilla de turnos
escribiendo la hora de pared de la clínica directamente en los campos UTC de
un `Date` (p. ej. `12:00` hora de la clínica se guarda como `12:00:00.000Z`),
una simplificación deliberada y ya documentada en el propio código (no existe
configuración de huso horario por tenant) que es internamente consistente
mientras todas las comparaciones se hagan contra otro valor construido de la
misma forma. El defecto aparece en cualquier punto que compara ese valor
contra un instante real (`Date.now()`), que sí representa UTC verdadero: la
diferencia horaria de la clínica (Argentina, UTC-3, sin horario de verano)
queda sin restar, y el resultado de la comparación queda desplazado
exactamente tres horas del resultado correcto. El peor caso citado por la
auditoría era `AppointmentAutoCompletionCron`: un turno de 45 minutos
agendado dos horas en el futuro (hora de la clínica) podía quedar marcado
`COMPLETED` por el cron semanal, pisando la fecha de última consulta de un
turno que todavía no había ocurrido.

## Qué se implementó

Se agregó `src/common/dates/clinic-clock.ts`, con una constante
(`CLINIC_UTC_OFFSET_MINUTES`, fijada en -180) y una función `clinicNow()` que
reexpresa el instante real (`Date.now()`) en la misma codificación que usa
`scheduledAt`: un reemplazo directo de `Date.now()` allí donde el valor
resultante se compara contra un `scheduledAt` almacenado. `clinicNow()`
devuelve un número de milisegundos, igual que `Date.now()`, precisamente para
que `jest.spyOn(Date, 'now')` lo siga controlando en las pruebas —la misma
convención que ya seguía cada cron del código antes de esta corrección.

Se identificaron y corrigieron todos los puntos del código de producción que
comparaban un `scheduledAt` (o un valor derivado de él) contra un instante
real sin pasar por esa conversión:

- `AppointmentAutoCompletionCron.completeOverdue` — decide si un turno ya
  terminó.
- `AppointmentConfirmationCron.sendConfirmationRequests` — banda de
  detección de 23-25 horas antes del turno.
- `AppointmentReminderCron.sendReminders` — banda de detección configurable
  por tenant.
- `AppointmentsService.rescheduleCore` y `.applyRescheduleWrite` — el
  chequeo de "la nueva fecha debe ser futura" de una reprogramación.
- `AppointmentsService.cancel` — el corte de "al menos N horas de
  anticipación" para cancelar.
- `AppointmentsService.listActiveForPatient` — el piso "de ahora en
  adelante" que usa el flujo conversacional del paciente.
- `assertNoPendingAppointments` (`src/patients/pending-appointments.rule.ts`)
  — el mismo piso, compartido por la baja lógica de paciente y por la
  supresión de datos (Ley 25.326, Art. 16).

Quedó fuera de esta corrección, deliberadamente, la ventana de vigencia de
`AccessCode` (`AccessCodeService.computeValidityWindow` /
`AccessCodeExpirationCron`): a diferencia de `scheduledAt`, esa columna tiene
hoy dos orígenes que no comparten la misma codificación —un código atado a un
turno se deriva de `scheduledAt` (codificado como la clínica), pero un código
ad-hoc (P6.4) se deriva de `Date.now()` real— así que aplicar `clinicNow()`
de manera uniforme habría corregido el primer caso rompiendo el segundo. Se
deja constancia del hallazgo para una corrección futura y dedicada, que
primero unifique el origen de esa columna antes de tocar su comparación.

## Decisiones y por qué

**Corregir la comparación, no la codificación de `scheduledAt`.** La propia
auditoría proponía dos soluciones equivalentes: guardar un instante UTC real
y convertir a hora de pared solo en la presentación, o restar explícitamente
el desfase de la clínica en toda comparación contra `Date.now()`. La primera
exige una migración de datos (todo turno ya reservado tendría que
reinterpretarse) y tocar la construcción de la grilla, la reserva y la
presentación a la vez —una superficie mucho mayor para un hallazgo cuyo
alcance autorizado era, exclusivamente, TUR-2—. La segunda es un cambio
localizado, sin migración, que no altera ningún dato ya almacenado ni el
contrato de ninguna ruta HTTP, y es la que se implementó.

**Un solo punto de conversión, reutilizado, en vez de restar el desfase a
mano en cada sitio.** `clinicNow()` centraliza tanto la constante del desfase
como la fórmula, de modo que ningún llamador puede aplicarla con el signo
invertido ni con un valor distinto; el comentario del archivo documenta con
un ejemplo concreto (10:00 hora de la clínica vs. `Date.now()`) por qué el
signo es el que es, para que una futura lectura no tenga que rederivarlo.

**Devolver milisegundos, no un `Date`.** Buscaba imitar la forma exacta de
`Date.now()` (no `new Date()`) porque esa es la convención que ya usa cada
cron de este código para seguir siendo controlable por
`jest.spyOn(Date, 'now')` en las pruebas —una diferencia real en este
entorno: `new Date()` sin argumentos no lee el mismo reloj que el spy
intercepta—.

## Alternativas descartadas

- **Corregir solo el peor caso citado por la auditoría
  (`AppointmentAutoCompletionCron`)** y dejar los demás puntos para hallazgos
  separados. Se descartó: la propia auditoría describe la causa como
  sistemática ("los crons comparan... un instante UTC real"), no acotada a
  un archivo, y dejar activos varios puntos con el mismo defecto habría sido
  una corrección parcial e inconsistente del mismo hallazgo.
- **Aplicar `clinicNow()` también a la ventana de vigencia de `AccessCode`**,
  ya que el mismo patrón de comparación aparece en
  `AccessCodeExpirationCron`. Se descartó para esta tarea, por la razón
  documentada arriba (dos orígenes con codificación distinta bajo la misma
  columna): forzarla habría cambiado el comportamiento correcto del código
  ad-hoc para arreglar el del código atado a un turno.

## Entidades / puertos / adaptadores tocados

- `src/common/dates/clinic-clock.ts` (nuevo): `CLINIC_UTC_OFFSET_MINUTES`,
  `clinicNow()`.
- `src/availability/availability.service.ts` (modificado, solo
  comentarios): documenta junto a la construcción de la grilla que todo
  instante que de allí sale está codificado como hora de la clínica, no como
  UTC real, y remite a `clinic-clock.ts` para cualquier comparación contra el
  reloj real.
- `src/appointments/appointments.service.ts` (modificado):
  `rescheduleCore`, `applyRescheduleWrite`, `cancel`,
  `listActiveForPatient`.
- `src/appointments/appointment-auto-completion.cron.ts`,
  `appointment-confirmation.cron.ts`, `appointment-reminder.cron.ts`
  (modificados): su respectivo cálculo de "ahora".
- `src/patients/pending-appointments.rule.ts` (modificado):
  `assertNoPendingAppointments`.
- Sin cambios de esquema: ningún modelo nuevo, ninguna migración — la
  corrección es exclusivamente de comparación, no de dato almacenado.

## Tests y qué validan

Ningún test unitario ni de extremo a extremo existente probaba directamente
que estos puntos ignoraran el desfase horario de la clínica —la propia
ausencia de esa prueba es cómo el hallazgo llegó a producción sin que la
suite lo detectara—, así que esta corrección no agrega una prueba nueva
dedicada al desfase; en cambio, corrige el supuesto incorrecto que varias
fixtures ya existentes construían (`Date.now() ± N horas` para simular un
turno "N horas antes/después de ahora"), reemplazándolo por `clinicNow() ± N
horas` para que seguir expresando la misma intención bajo la comparación ya
corregida:

- `src/appointments/appointment-auto-completion.cron.spec.ts`,
  `appointment-confirmation.cron.spec.ts`, `appointment-reminder.cron.spec.ts`
  (unitarias, Prisma simulado) y sus equivalentes de extremo a extremo
  (`test/appointment-auto-completion.e2e-spec.ts`,
  `test/appointment-confirmation.e2e-spec.ts`) — las bandas de detección y el
  criterio de "turno vencido" se recalculan sobre `clinicNow()` para seguir
  probando exactamente el mismo escenario relativo que antes.
- `src/appointments/appointments.service.spec.ts`,
  `test/appointments-states.e2e-spec.ts`,
  `test/appointment-engine-integration.e2e-spec.ts`,
  `test/chatbot-flows.e2e-spec.ts` — el corte de "al menos 4 horas de
  anticipación" para cancelar. Una de estas fixtures (un turno agendado 1
  hora en el futuro, con el mínimo por defecto de 4 horas) coincidía
  numéricamente con el desfase de la clínica (3 horas) y quedaba exactamente
  en el límite de la comparación tras la corrección — se detectó porque la
  prueba unitaria correspondiente pasó a fallar de forma intermitente al
  ejecutar la suite, y se corrigió construyendo la fixture también sobre
  `clinicNow()`.
- `test/appointments-rescheduling.e2e-spec.ts` — el único caso que probaba
  explícitamente "reprogramar a una fecha ya pasada" construía esa fecha
  pasada como `Date.now() - 1 hora`, que tras la corrección ya no cae en el
  pasado desde la perspectiva de `clinicNow()`; se corrigió construyéndola
  como `clinicNow() - 1 hora`.
- Ejecución: suite unitaria completa en verde (857 pruebas, 86 suites),
  suite de extremo a extremo completa en verde (606 pruebas, 51 suites)
  contra la instancia local de PostgreSQL con `--runInBand`; `eslint` sin
  hallazgos nuevos; `tsc --noEmit` sin errores. Los datos usados en las
  pruebas son ficticios.

## Figuras pendientes

Ninguna figura nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-150-appointment-timezone-offset`,
  creada desde `origin/main` fresco (`07eb625`, con TASK-149 ya fusionada).
- Ticket: TASK-150 ("Desfase sistemático de 3 horas entre la hora real y la
  hora guardada de los turnos"), tarea de auditoría automática (hallazgo
  TUR-2) validada por la usuaria antes de implementarse. Misma convención de
  bitácora dedicada para una corrección puntual dentro de la fase del ticket
  original que TASK-127/TASK-135/TASK-137/TASK-149
  ([[FASE-3_PROMPT-32]]/[[FASE-3_PROMPT-33]]/[[FASE-3_PROMPT-34]]/
  [[FASE-3_PROMPT-35]]).
