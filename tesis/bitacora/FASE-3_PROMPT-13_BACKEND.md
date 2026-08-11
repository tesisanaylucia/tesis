# Fase 3 — Motor de Turnos (backend) — cobertura de tests de la retención de 24h en modalidad manual (TASK-80, corrección a TASK-40)

## Qué se implementó

TASK-80 pedía formalmente, como corrección a TASK-40 (P3.7), que un turno
liberado por una cancelación bajo modalidad de reasignación MANUAL quedara
excluido de `DisponibilidadService.getSlots()` durante 24 horas, en lugar de
quedar ofrecible de inmediato — con tres criterios de aceptación puntuales:
un turno cancelado hace 1 hora no debe aparecer en `getSlots()`, uno
cancelado hace 25 horas sí debe aparecer, y la modalidad AUTOMATICA no debe
verse afectada por el cambio.

Al inspeccionar el código del repo antes de empezar, la implementación real
de esa retención ya existía: se agregó completa el mismo día, más temprano,
como parte de la revisión `/code-review ultra` documentada en
[[FASE-3_PROMPT-11]] (commit `a3db129`, hallazgos 3 y 4 de esa revisión) —
`Appointment.holdUntil`, la definición compartida de "ocupado" en
`AvailabilityService`, y `WaitlistReassignmentService.holdSlot`/`releaseHold`
fijando 24h en modalidad MANUAL y liberando explícitamente la modalidad
AUTOMATICA al concluir el recorrido de la lista de espera. TASK-80 llegó,
pues, como el ticket formal que documenta y da trazabilidad a un ajuste que
ya se había hecho por otra vía — no se reimplementó nada.

Lo que sí faltaba, y es el aporte real de esta tarea, era la prueba directa
de los tres criterios de aceptación tal como los pide el ticket, al nivel de
`getSlots()`: la cobertura existente probaba la retención dentro de
`isSlotFree` (con mocks, en `availability.service.spec.ts`) y el estado del
turno cancelado en los tests de extremo a extremo del algoritmo de
reasignación, pero ningún test llamaba a `GET /profesionales/:id/disponibilidad`
contra una base real para confirmar que un turno retenido 1 hora después de
cancelarse no aparece, y que uno retenido 25 horas después sí. Un test de
`appointment-reassignment.e2e-spec.ts` además tenía un título desactualizado
("leaves the turno available") que ya no describía el comportamiento real
desde el fix de la revisión ultrareview.

## Decisiones y por qué

**No se modificó código de producción.** El comportamiento pedido por el
ticket ya estaba implementado y correcto; reabrirlo habría sido trabajo
redundante. Se documenta esto explícitamente para que quede trazado por qué
un ticket de "corrección" no dejó cambios en `src/`.

**Los nuevos tests de aceptación se escribieron directamente contra
`GET /disponibilidad` con `holdUntil` fijado a mano vía Prisma**, en
`availability.e2e-spec.ts`, en lugar de pasar por el flujo completo de
cancelación en cada caso: son los dos casos límite exactos que el ticket
describe (retenido hace 1h, expirado hace 25h) y fijar `holdUntil`
directamente es lo que permite expresarlos sin depender del reloj real de
ejecución del test.

**El criterio "AUTOMATICA no se ve afectada" se probó en cambio a través del
flujo real de cancelación**, extendiendo los dos tests ya existentes en
`appointment-engine-integration.e2e-spec.ts` ("AUTOMATIC: cuando nadie
acepta" y "MANUAL") con una llamada a `GET /disponibilidad` después de
cancelar: ese es el punto en el que interesa probar que el algoritmo de
`WaitlistReassignmentService` en sí mismo libera la retención automática al
concluir el recorrido, no solo que la lectura respeta una columna que se le
fija a mano.

**Se corrigieron los títulos de los dos tests de modalidad MANUAL que
decían "leaves the turno available/free"**, tanto en
`appointment-reassignment.e2e-spec.ts` como en
`appointment-engine-integration.e2e-spec.ts`, y se sumó en el primero una
aserción directa de que `holdUntil` queda fijado a aproximadamente 24 horas
en el momento de la cancelación — ese archivo es el que posee la cobertura
del algoritmo de reasignación en sí (P3.7), así que es el lugar natural para
probar el efecto inmediato de la cancelación sobre la columna, complementario
a la prueba de lectura en `availability.e2e-spec.ts`.

## Alternativas descartadas

- **Volver a implementar la retención de 24h como si el ticket describiera
  un estado pendiente**: descartada tras confirmar en el código y en el
  commit `a3db129` que ya estaba hecha; hacerlo de nuevo habría arriesgado
  una segunda fuente de verdad para la misma regla.
- **Probar los tres criterios de aceptación únicamente a través del flujo de
  cancelación real** (cancelar, esperar, volver a consultar): descartada
  para los dos casos límite de tiempo (1h/25h) porque un test no debe
  depender de tiempo real transcurrido; fijar `holdUntil` a mano es la
  forma establecida en el resto de la suite de disponibilidad de expresar un
  estado de reloj sin esperarlo.

## Entidades / puertos / adaptadores tocados

Ninguno — tarea exclusivamente de tests, sin cambios de esquema, servicio ni
punto de acceso. Ver [[FASE-3_PROMPT-11]] para las entidades que sí tocó la
implementación real de esta regla (`Appointment.holdUntil`,
`AvailabilityService`, `WaitlistReassignmentService`,
`waitlist.constants.ts`).

## Tests y qué validan

- `test/availability.e2e-spec.ts`: dos casos nuevos a nivel `getSlots()` —
  un turno cancelado con `holdUntil` 23h en el futuro (simulando una
  cancelación MANUAL de hace 1h) queda excluido; uno con `holdUntil` 1h en
  el pasado (simulando una cancelación de hace 25h) vuelve a aparecer como
  libre, sin ningún job adicional.
- `test/appointment-engine-integration.e2e-spec.ts`: los dos casos de
  "Waitlist reassignment on cancellation" ahora piden disponibilidad real
  después de cancelar — AUTOMATICA (nadie acepta) muestra el turno libre de
  inmediato tras liberarse la retención; MANUAL no lo muestra.
- `test/appointment-reassignment.e2e-spec.ts`: el caso MANUAL ahora también
  verifica que `holdUntil` queda fijado a un valor entre 23.9 y 24 horas en
  el futuro al momento de la cancelación.
- Ejecución: suite unitaria completa en verde (30 suites / 335 pruebas).
  Suite end-to-end completa en verde en modo serie (33 suites / 411
  pruebas, `--runInBand`). Lint limpio. Los datos usados en las pruebas son
  ficticios.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-80-manual-reassignment-hold` (rama
  nueva desde `origin/main` fresco, que ya incluía el fix real de
  `a3db129`). Commit `8720a7b`. Pusheada a `origin`, pendiente de Pull
  Request en Bitbucket.
- Ticket: TASK-80 (Jira), "[CORRECCIÓN] TASK-40 – Bloqueo de 24h del turno
  liberado en modalidad de reasignación MANUAL".
