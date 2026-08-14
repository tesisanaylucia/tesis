# Fase 4 — Notificaciones y Scheduler (backend) — crons de confirmación/recordatorio envían antes de verificar idempotencia y estado (TASK-99, corrección a TASK-43/TASK-45)

## Qué se implementó

TASK-99 fue una tarea de corrección hallada por la auditoría multi-agente de
`psique-back` sobre `main`, 2026-08-12, con ángulo motor de turnos/crons.
`AppointmentConfirmationCron.requestConfirmation` (P4.2, TASK-43) llamaba a
`messaging.sendMessage(...)` antes de aplicar el guard de idempotencia
(`updateMany({ where: { id, confirmationRequestedAt: null } })`), y ninguno
de los dos pasos re-verificaba que el turno siguiera `RESERVED` — solo el
`findMany` inicial del batch había filtrado por `status`. El mismo patrón se
repetía en `AppointmentReminderCron.sendReminder` (P4.4, TASK-45): el
`sendMessage` corría antes del guard `reminderSentAt: null`, sin re-chequear
`status: CONFIRMED`.

El caso concreto: el paciente confirma (o el turno se cancela) en los
segundos/decenas de segundos que tarda el cron en iterar su batch después
del `findMany` inicial. El cron igual mandaba el WhatsApp de "confirmá tu
turno" (o "tenés turno mañana") y marcaba `confirmationRequestedAt`/
`reminderSentAt` sobre un turno que ya estaba CONFIRMADO/CANCELADO — un
mensaje contradictorio para el paciente, no detectado porque `result.count`
del `updateMany` solo protegía la escritura de auditoría, no el envío del
mensaje en sí.

La corrección invierte el orden en ambos crons: primero corre el
`updateMany` guardado —ahora también filtrado por `status` en el `where`, no
solo por `confirmationRequestedAt`/`reminderSentAt`— dentro de la misma
transacción que el `audit.log` correspondiente, y **solo si esa escritura
afecta una fila** se renderiza la plantilla y se envía el mensaje. Un
`updateMany` que devuelve `count: 0` significa que otra corrida ya registró
el envío, o que el turno cambió de estado desde el `findMany` del batch — en
cualquiera de los dos casos, ni mensaje ni entrada de auditoría.

## Decisiones y por qué

**El guard y la auditoría siguen atómicos entre sí, como exige CLAUDE.md.**
La regla del proyecto sobre traza de auditoría (la misma que motivó TASK-97
y TASK-98) es que la entrada de auditoría debe confirmarse junto con la
mutación que describe. Eso descarta intercalar el envío del mensaje *entre*
el `updateMany` y el `audit.log` — mantener la llamada de red fuera de la
transacción de base de datos sigue siendo la práctica correcta. La única
forma de invertir el orden respetando esa regla es la elegida: transacción
(guard + auditoría) primero, envío condicionado a su resultado después.

**Trade-off aceptado y documentado en el propio código:** con este orden, un
`sendMessage` que falla *después* de que el guard ya confirmó (caída de red,
proveedor caído) ya quedó marcado y auditado como enviado — no se reintenta
en la corrida siguiente, porque `confirmationRequestedAt`/`reminderSentAt`
ya no es `null`. Antes del fix, un `sendMessage` fallido no llegaba a marcar
nada (el `try/catch` del batch abortaba el resto del método), así que la
corrida siguiente sí reintentaba. Se aceptó esta regresión puntual porque es
inherente a cerrar la ventana de carrera que pedía el ticket: enviar primero
—el orden anterior— es exactamente el escenario que permitía el bug
(mensajear un turno que el guard habría rechazado). Documentado con un
comentario en ambos métodos para que quede explícito y no se lo confunda con
un descuido.

**Agregar `status` al `where` del `updateMany`, no solo al `findMany`
inicial.** El ticket lo dejaba como "evaluar" y no como requisito estricto,
pero sin ese filtro el guard seguiría siendo ciego a la causa real de la
carrera (el cambio de estado) y solo protegería contra una segunda corrida
del mismo cron — el caso que ya cubría antes del fix. Se decidió agregarlo
en los dos crons por simetría y porque es lo que hace que el `updateMany`
efectivamente "re-verifique el status actual", como pide la sección de
alcance funcional del ticket.

## Entidades / puertos / adaptadores tocados

- `AppointmentConfirmationCron.requestConfirmation`
  (`src/appointments/appointment-confirmation.cron.ts`): guard (`updateMany`
  con `status: RESERVED` agregado al `where`) y `audit.log` corren primero,
  dentro de la misma transacción; el render de plantilla y `sendMessage`
  quedan condicionados a que esa transacción haya reclamado la fila. Sin
  cambios de esquema ni de migración.
- `AppointmentReminderCron.sendReminder`
  (`src/appointments/appointment-reminder.cron.ts`): mismo cambio, con
  `status: CONFIRMED` agregado al `where` del guard.

## Tests y qué validan

En ambos specs unitarios (`appointment-confirmation.cron.spec.ts`,
`appointment-reminder.cron.spec.ts`) el doble de prueba de `updateMany` dejó
de devolver un `{ count: 1 }` fijo y pasó a reflejar el estado "vivo" de la
tabla en memoria (`stored`) contra el `where` recibido — condición necesaria
para poder escribir el criterio de aceptación literal del ticket: un caso
nuevo por cron hace que el mock de `findMany` del batch, como efecto
colateral de resolver su promesa, mute el `status` del turno candidato (a
`CONFIRMED` en el cron de confirmación, a `CANCELLED` en el de recordatorio)
antes de devolver la lista — modelando que el estado cambió en el mundo real
entre la lectura del batch y el procesamiento individual de ese candidato.
Se verifica que, en ese caso, ni `sendMessage` ni `audit.log` se llaman, y
que el campo de idempotencia (`confirmationRequestedAt`/`reminderSentAt`)
sigue en `null`. El caso normal (turno sigue en el estado esperado) se
mantiene sin cambios de comportamiento: sigue enviando el mensaje y marcando
el timestamp.

También se ajustó el test preexistente de "otra corrida ya lo registró"
(`updateMany` devuelve `count: 0`): antes esperaba que el mensaje se
enviara igual (comportamiento del orden viejo); ahora espera que no se envíe
ni se audite, coherente con el nuevo orden. Y el test de "una falla no
detiene el resto del batch" se actualizó para reflejar que ambos turnos
quedan auditados aunque el envío del primero falle —el guard y la auditoría
ya corrieron antes de intentar el envío— en vez del `log` con una sola
llamada que esperaba antes.

Suite completa: 38 suites unitarias / 422 pruebas en verde (411 previas + 11
nuevas entre ambos specs); 37 suites e2e / 439 pruebas en verde
(`--runInBand`, sin cambios en la suite e2e de confirmación — el
comportamiento observable de la API para el caso normal no cambió, y una
carrera real de temporización no es practicable de reproducir de forma
determinística contra Postgres real, por eso el criterio de aceptación se
cubrió a nivel unitario, igual que el patrón ya usado en TASK-97 para probar
atomicidad de transacción). Lint limpio y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-99-cron-idempotency-order` (creada
  desde `origin/main` fresco, tras el merge de TASK-98). Pusheada a
  `origin`; PR abierto, no fusionado aún.
- Ticket: TASK-99 ("[CORRECCIÓN] Crons de confirmación/recordatorio envían
  el mensaje antes de verificar idempotencia y estado"), corrección a
  TASK-43 (P4.2, [[FASE-4_PROMPT-3]], creó
  `AppointmentConfirmationCron`) y TASK-45 (P4.4, [[FASE-4_PROMPT-4]], creó
  `AppointmentReminderCron`). Misma convención de bitácora dedicada para
  correcciones pequeñas dentro de la fase del ticket original que TASK-91
  ([[FASE-4_PROMPT-7]]), TASK-90 ([[FASE-4_PROMPT-8]]) y TASK-97
  ([[FASE-4_PROMPT-9]]).
