# Fase 4 — Notificaciones y Scheduler (backend) — `respondToOffer` audita fuera de transacción (TASK-97, corrección a TASK-82)

## Qué se implementó

TASK-97 fue una tarea de corrección hallada de forma independiente por dos
agentes distintos (uno enfocado en auditoría/fechas/reglas-como-datos, otro
en el motor de turnos y los crons) durante la auditoría multi-agente de
`psique-back` sobre `main`, 2026-08-12. `WaitlistReassignmentService.
respondToOffer` — el único punto donde se resuelve el destino de una
`WaitlistOffer` (aceptada, rechazada o vencida) — hacía el `updateMany` que
cambia el estado de la oferta sobre el cliente Prisma directo, sin
transacción, y llamaba a `audit.log(...)` recién varias líneas después, tras
una lectura intermedia del turno asociado, sin pasarle el handle de
transacción (`tx`). Esto contradice la regla del proyecto sobre traza de
auditoría: la entrada de auditoría debe confirmarse junto con la mutación
que describe, porque una escritura de auditoría autónoma —una mutación que
se confirma y una auditoría que falla después— deja un cambio sin registrar.

El caso concreto: `WaitlistOfferTimeoutCron` (o, a futuro, un webhook de
WhatsApp) vence o rechaza una oferta; ese cambio de estado dispara la
reasignación real de un turno vía `reserveForCandidate`/`advanceWaitlist`.
Si el proceso caía o `audit.log` fallaba justo después del `updateMany` ya
confirmado, la oferta quedaba EXPIRED/REJECTED/ACCEPTED sin ningún rastro en
la traza de auditoría — un hueco de cumplimiento respecto de la Ley 25.326,
a diferencia de `AppointmentAutoCancellationCron.cancelAndReassign`, que sí
envuelve su propio `updateMany` guardado y la auditoría correspondiente en
un mismo `$transaction`.

La corrección envuelve el `updateMany` guardado (`where: { status: PENDING
}`, el mismo patrón de guardas contra condiciones de carrera que usa el
resto del servicio) y el `audit.log` correspondiente en un mismo
`$transaction`, pasándole `tx` a `AuditService.log`, siguiendo exactamente
el patrón ya existente en `cancelAndReassign`. La lectura posterior del
`Appointment` —que solo alimenta la continuación del recorrido de la lista
de espera (aceptar y reservar, o avanzar al siguiente candidato), no la
mutación auditada en sí— quedó deliberadamente fuera de la transacción: no
forma parte de lo que debe confirmarse o revertirse junto con el cambio de
estado de la oferta.

## Decisiones y por qué

**Alcance mínimo, sin tocar la lógica de negocio.** El ticket es
explícitamente de corrección, no de rediseño: la guarda
`where: { status: PENDING }`, el orden de las operaciones y el
comportamiento ante una oferta ya resuelta (no-op) se mantuvieron
idénticos. El único cambio real es *dónde* corre cada operación —dentro de
`$transaction` en lugar de sobre el cliente directo— y que `audit.log`
recibe `tx`.

**La lectura del `Appointment` se dejó fuera de la transacción a
propósito.** Envolverla también habría sido inofensivo, pero el objetivo de
CLAUDE.md es que la mutación auditada y su entrada de auditoría compartan
destino atómico — no que toda lectura posterior a esa mutación viva en la
misma transacción. Ampliar el alcance de la transacción a una lectura que
no se audita ni se revierte no aporta nada y aumenta, sin necesidad, el
tiempo que la transacción mantiene el candado sobre la fila de la oferta.

## Entidades / puertos / adaptadores tocados

- `WaitlistReassignmentService.respondToOffer` (`src/waitlist/
  waitlist-reassignment.service.ts`): el `updateMany` guardado sobre
  `WaitlistOffer` y el `audit.log` correspondiente pasan a correr dentro de
  un mismo `$transaction`, con `tx` propagado a `AuditService.log`. Sin
  cambios de esquema ni de migración.

## Tests y qué validan

El único test unitario existente que cubre este método
(`waitlist-reassignment.service.spec.ts`) usaba, antes de esta tarea, un
cliente Prisma de prueba cuyo `$transaction` simplemente ejecutaba el
trabajo recibido sobre el mismo cliente en memoria (`(work) => work(client)`
), sin ninguna semántica de todo-o-nada — insuficiente para probar el
criterio de aceptación del ticket ("si `audit.log` fallara, el `updateMany`
del estado de la oferta debe hacer rollback"). Se reforzó ese doble de
prueba para que tome una instantánea de la tabla en memoria de
`WaitlistOffer` antes de ejecutar el trabajo de la transacción y, si el
trabajo lanza una excepción, restaure esa instantánea antes de repropagar
el error — el mismo comportamiento de una transacción interactiva real de
Prisma frente a una excepción sin capturar dentro de su callback.

Con ese doble de prueba ya honesto sobre atomicidad, se agregó un caso
nuevo: se hace que `audit.log` lance una excepción y se verifica que (a) el
estado de la oferta permanece en `PENDING` después de la llamada fallida
—es decir, el `updateMany` se revirtió junto con la auditoría que falló— y
(b) el recorrido de la lista de espera no avanzó (no se registró ninguna
oferta nueva), confirmando que nada de la mutación fallida quedó aplicado a
medias. Los tres casos existentes que verifican el contenido de la entrada
de auditoría (rechazo por falta de teléfono, aceptación, vencimiento) se
ajustaron para esperar también el segundo argumento (`tx`) que `audit.log`
ahora recibe siempre.

Suite completa: 38 suites unitarias / 417 pruebas en verde; 37 suites e2e /
439 pruebas en verde (`--runInBand`, sin cambios en la suite e2e — el
comportamiento observable de la API no cambió). Lint limpio y verificación
de tipos (`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-97-waitlist-offer-audit-transaction`
  (creada desde `origin/main` fresco, tras el merge de TASK-90). Pusheada a
  `origin`; PR abierto, no fusionado aún.
- Ticket: TASK-97 ("[CORRECCIÓN] WaitlistReassignmentService.respondToOffer
  audita sin transacción"), corrección a TASK-82 (P4.5,
  [[FASE-4_PROMPT-5]], creó `WaitlistOffer` y `respondToOffer`). Misma
  convención de bitácora dedicada para correcciones pequeñas dentro de la
  fase del ticket original que TASK-91 ([[FASE-4_PROMPT-7]]) y TASK-90
  ([[FASE-4_PROMPT-8]]).
