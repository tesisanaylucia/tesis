# Fase 3 — Motor de Turnos (backend) — reagendar un turno no reseteaba `confirmationRequestedAt`/`reminderSentAt` (TASK-95, corrección a TASK-39)

## Contexto

Una auditoría multi-agente de `psique-back` en `main` (ángulo motor de
turnos/crons, 2026-08-12) detectó que `rescheduleCore`
(`src/appointments/appointments.service.ts`, usado tanto por
`reschedule()` como por `reorganizeAgenda()`) actualizaba `scheduledAt`
al reprogramar un turno, pero nunca reseteaba `confirmationRequestedAt`
ni `reminderSentAt`. Esos dos campos son las guardas de idempotencia de
dos jobs programados de este mismo módulo (`AppointmentAutoCancellationCron`,
P4.3/TASK-44, y `AppointmentReminderCron`, P4.4/TASK-45), y ambos quedaban
desincronizados de la fecha real del turno tras un reagendamiento:

- `AppointmentAutoCancellationCron` filtra únicamente por
  `status: RESERVED` y antigüedad de `confirmationRequestedAt`, sin mirar
  `scheduledAt`. Un turno RESERVADO al que ya se le había pedido
  confirmación, reagendado a una fecha lejana antes de que se cumplieran
  las 4 horas de plazo, quedaba con `confirmationRequestedAt` apuntando
  todavía a la solicitud original — el cron lo cancelaba con motivo
  `NO_CONFIRMATION` en su próxima corrida, sin haberle preguntado nunca al
  paciente por la fecha nueva.
- Simétricamente, `AppointmentReminderCron` sólo considera candidatos con
  `reminderSentAt: null`. Un turno CONFIRMADO cuyo recordatorio ya se
  había enviado para la fecha original, y que luego se reagendaba, nunca
  recibía un recordatorio para la fecha nueva, porque el campo seguía
  marcado como ya enviado.

## Qué se implementó

- `rescheduleCore` ahora incluye `confirmationRequestedAt: null` y
  `reminderSentAt: null` en el mismo `guardedStatusUpdate` que escribe el
  nuevo `scheduledAt`, dentro de la misma transacción `SERIALIZABLE` que ya
  usaba para el resto de la escritura — un solo punto de escritura, no dos.

## Decisiones y por qué

**El reset se hizo incondicional, sin distinguir el estado del turno ni
cuál de los dos campos estaba efectivamente seteado.** Un turno RESERVADO
nunca tiene `reminderSentAt` seteado (ese cron sólo actúa sobre turnos
CONFIRMADO) y un turno recién confirmado puede o no conservar
`confirmationRequestedAt` de cuando todavía era RESERVADO; escribir
`null` sobre un campo que ya es `null` no tiene efecto, así que separar
casos por estado no aportaba nada y sólo hacía la guarda más frágil ante
un futuro estado intermedio no contemplado.

**No se tocó la ventana de detección de ninguno de los dos crons**, tal
como pedía el ticket explícitamente. La corrección deja que cada cron
vuelva a tratar el turno reagendado como si nunca se le hubiera pedido
confirmación ni enviado recordatorio — el propio mecanismo de
idempotencia existente (comparar contra `null`) es lo que hace que el
cron correspondiente lo vuelva a detectar como candidato en su próxima
corrida, con la ventana de detección de la fecha nueva, sin ningún caso
especial adicional.

## Alternativas descartadas

- **Filtrar también por `scheduledAt` dentro de cada cron** (por ejemplo,
  exigir que `scheduledAt` no se haya movido desde `confirmationRequestedAt`):
  se descartó porque no existe una relación temporal fija entre ambos
  campos — nada impide reprogramar un turno minutos después de pedir la
  confirmación — y hubiera dejado la responsabilidad de esta invariante
  repartida entre `rescheduleCore` y cada cron, en vez de en el único
  punto que efectivamente cambia `scheduledAt`.

## Entidades / puertos / adaptadores tocados

- `AppointmentsService.rescheduleCore`
  (`src/appointments/appointments.service.ts`): agrega
  `confirmationRequestedAt: null, reminderSentAt: null` a la escritura
  guardada de `scheduledAt`.

## Tests agregados o modificados

- `src/appointments/appointments-rescheduling.service.spec.ts`: nuevo caso
  en `describe('reschedule', …)` que arranca un turno CONFIRMADO con
  ambos campos seteados y verifica que la escritura enviada a
  `updateMany` los deja en `null`.
- `test/appointments-rescheduling.e2e-spec.ts`: nuevo caso dentro de
  `PATCH /turnos/:id/reprogramar` contra Postgres real — crea un turno
  CONFIRMADO, le fija ambos timestamps directamente en la base, reprograma
  vía la API y confirma por consulta directa que ambos quedan en `null`.
  No se agregó un e2e nuevo a nivel de los crons: siguiendo el precedente
  de TASK-44/TASK-45, la lógica de cada cron ya está cubierta por sus
  propios specs unitarios con Prisma simulado (verifican que actúan
  cuando el campo es `null` y no actúan cuando está seteado), así que la
  combinación de ese comportamiento ya probado con el nuevo test de
  `rescheduleCore` prueba el criterio de aceptación completo sin
  duplicar cobertura contra Postgres real.

Suite completa verde tras el cambio: 38 suites unitarias / 415 pruebas; 37
suites e2e / 435 pruebas (`--runInBand`). Lint y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva — es una corrección puntual sobre un flujo de escritura ya
documentado (P3.6, reprogramación de turnos); no cambia ningún diagrama
existente.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-95-reschedule-reset-cron-fields`
  (creada desde `origin/main` fresco, tras el merge de TASK-94). Pusheada
  a `origin`, PR abierta, no fusionada aún.
- Ticket: TASK-95 ("[CORRECCIÓN] Reagendar un turno no resetea
  confirmationRequestedAt/reminderSentAt"), corrección a TASK-39 (P3.6,
  implementó `reschedule`/`reorganizeAgenda`/`rescheduleCore`). Misma
  convención de bitácora dedicada para correcciones pequeñas dentro de la
  fase del ticket original que TASK-79/TASK-81/TASK-86/TASK-94
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]], [[FASE-3_PROMPT-15]],
  [[FASE-3_PROMPT-16]]) y TASK-91/TASK-92 ([[FASE-4_PROMPT-7]],
  [[FASE-1_PROMPT-9]]).
