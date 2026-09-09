# Fase 3 — Motor de Turnos (backend) — una ausencia registrada no bloquea reservas ni evita que la lista de espera reofrezca esos horarios (TASK-149, corrección a TASK-35/TASK-116)

## Contexto

TASK-149 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, validada por la usuaria
antes de implementarse, con dos hallazgos combinados (PROF-1 y PROF-2). El
detalle completo de la auditoría está en el artefacto
`https://claude.ai/code/artifact/5c2ee112-c426-4d6e-a8e2-499ea7bbbc74`
("auditoria-psique-back.md").

PROF-1 apuntaba a `AvailabilityService.isSlotFree`: a diferencia de
`getSlots`, que ya excluye la ausencia de la agenda ofrecible, `isSlotFree`
—la verificación que respaldan la reserva directa (`POST /turnos`) y la
reprogramación— no leía la tabla de ausencias en absoluto. Un administrador o
el propio profesional podían reservar, o reprogramar, directamente sobre un
día de ausencia con solo nombrar el instante, sin pasar por la agenda que ya
lo excluía.

PROF-2 apuntaba a `WaitlistReassignmentService.advanceWaitlist`: el motor de
reasignación automática sólo comprobaba el feriado antes de reofrecer una
franja liberada a la lista de espera, no la ausencia. El escenario que ello
habilita: un profesional se ausenta del 7 al 11/9; un administrador igual
reserva un turno el 9/9 (posible por PROF-1); y si la ausencia cancela en
cascada los turnos ya reservados en su rango (mecanismo ya existente desde
TASK-127/TASK-135), la reasignación automática podía ofrecer —y dejar
aceptar— esos mismos horarios a la lista de espera, que terminaba reservando
dentro del rango de la propia ausencia que acababa de vaciarlos.

## Qué se implementó

Se agregó `AvailabilityService.isAbsent(professionalId, scheduledAt, client)`,
método nuevo con la misma forma y la misma razón de ser que ya tiene
`isHoliday` (TASK-116): verifica si el día calendario del instante cae dentro
de alguna ausencia registrada para ese profesional, aceptando el handle de
transacción del llamador igual que `isHoliday`/`isSlotFree`.

**PROF-1.** `isSlotFree` pasa a consultar `isAbsent` junto con `isHoliday` y
el conflicto de turno, de modo que la reserva directa y la reprogramación
—las dos rutas de escritura que ya comparten esa única verificación— heredan
la corrección sin tocarlas por separado.

**PROF-2.** `advanceWaitlist` incorpora el mismo chequeo de ausencia junto al
de feriado ya existente, en el mismo punto: una vez por recorrido, antes de
ofrecer la franja a nadie, liberando la retención sin registrar oferta si la
franja cae en cualquiera de las dos causas. La verificación dentro de la
transacción de reserva (`reserveInTransaction`), que ya repetía el chequeo de
feriado por si éste se declaraba mientras una oferta esperaba respuesta,
también incorpora la ausencia; la excepción que señala esa falla, hasta ahora
nombrada únicamente por el feriado (`SlotOnHolidayError`), se generalizó a
`SlotUnavailableError` para nombrar cualquiera de las dos causas sin duplicar
el mecanismo de liberación de retención y de propagación que ya la
acompañaba.

Se aprovechó además para corregir el mensaje de error de `isSlotFree`
(`slotTakenMessage`), que enumeraba las causas posibles de rechazo pero no
mencionaba la ausencia —inexactitud preexistente, del mismo tipo que ya había
motivado una corrección anterior (TASK-113) sobre el mismo mensaje.

## Decisiones y por qué

**Reutilizar la misma forma que `isHoliday`, no la verificación de franja
libre completa.** El motor de reasignación no puede pasar por `isSlotFree`
para su propio chequeo: el turno que está reasignando es la fila cancelada
que él mismo retiene, y colisionaría consigo mismo bajo cualquiera de los dos
tipos de acceso (mismo motivo, documentado en TASK-116, por el que `isHoliday`
ya existe como método separado). `isAbsent` se escribió con la misma
separación desde el principio, en lugar de intentar generalizar después.

**Generalizar `SlotOnHolidayError` a `SlotUnavailableError` en vez de
introducir una segunda clase de excepción.** Las dos causas —feriado y
ausencia declarados mientras una oferta espera respuesta— comparten
exactamente el mismo tratamiento aguas abajo (la oferta permanece aceptada,
la retención se libera, el error se propaga), así que distinguirlas con dos
clases habría duplicado ese tratamiento sin ganar nada; el mensaje de la
excepción sigue nombrando la causa concreta.

**Extender también la verificación dentro de la transacción de reserva
(`reserveInTransaction`), no sólo el chequeo de ofrecimiento.** El propio
hallazgo sólo pedía los dos puntos citados arriba (`isSlotFree` y el guard de
`advanceWaitlist`), pero la verificación de feriado dentro de la transacción
de reserva existe precisamente para cerrar la misma clase de carrera que
motiva PROF-2 —una condición declarada mientras el candidato tiene la oferta
pendiente—, y la ausencia comparte esa misma ventana temporal; dejarla sin la
verificación análoga habría reabierto el mismo hallazgo un paso más adelante
en el mismo flujo.

## Alternativas descartadas

- **No se consideró seriamente ninguna alternativa a duplicar la separación
  ya usada por `isHoliday`**: es exactamente la misma forma de problema
  (una regla de "instante ocupado" que el motor de reasignación necesita sin
  el resto de `isSlotFree`), y TASK-116 ya había descartado en su momento la
  alternativa de replicar la consulta en el módulo de lista de espera, razón
  que sigue aplicando sin cambios.

## Entidades / puertos / adaptadores tocados

- `src/availability/availability.service.ts` (modificado): método nuevo
  `isAbsent`; `isSlotFree` pasa a consultarlo; comentarios de `isSlotFree`/
  `isHoliday` actualizados para nombrar la ausencia junto al feriado.
- `src/waitlist/waitlist-reassignment.service.ts` (modificado):
  `advanceWaitlist` y `reserveInTransaction` incorporan el chequeo de
  ausencia junto al de feriado; `SlotOnHolidayError` renombrada a
  `SlotUnavailableError`.
- `src/appointments/appointments.service.ts` (modificado, menor):
  `slotTakenMessage` menciona la ausencia entre las causas posibles.
- Sin cambios de esquema: ningún modelo nuevo, ninguna migración.

## Tests y qué validan

- `src/availability/availability.service.spec.ts` (ampliado, Prisma
  simulado): `isSlotFree` es `false` dentro de una ausencia registrada aun
  sin conflicto de turno ni feriado; la consulta al calendario de ausencias
  usa el profesional y el instante correctos; corre a través del cliente de
  transacción recibido. Suite nueva para `isAbsent` en aislamiento, misma
  forma que la ya existente para `isHoliday`.
- `src/waitlist/waitlist-reassignment.service.spec.ts` (ampliado, Prisma
  simulado): no se ofrece nada y se libera la retención cuando la franja
  liberada cae en una ausencia; la ausencia se consulta una vez por
  recorrido, no por candidato; la aceptación falla explícitamente y libera la
  retención cuando la ausencia se registra mientras la oferta espera
  respuesta, dejando la oferta como aceptada en el registro.
- `test/appointment-engine-integration.e2e-spec.ts` (ampliado, contra
  Postgres real): una reserva directa sobre un día ya cubierto por una
  ausencia se rechaza (`PROF-1`, caso que ninguna prueba existente cubría
  porque `getSlots` ya excluía la ausencia de la agenda ofrecible y nada
  probaba la ruta de escritura por separado).
- `test/appointment-reassignment.e2e-spec.ts` (ampliado, contra Postgres
  real): las dos mismas pruebas ya existentes para el feriado (TASK-116),
  repetidas para la ausencia — no se ofrece la franja liberada dentro de una
  ausencia, y la aceptación de una oferta falla explícitamente si la ausencia
  se registra mientras la oferta está pendiente.
- **Corrección de una prueba de extremo a extremo ya existente que
  codificaba el propio defecto como comportamiento esperado**: en
  `test/appointment-engine-integration.e2e-spec.ts`, la prueba "Mass
  rescheduling on absence" registraba una ausencia sobre la fecha de un turno
  ya reservado, con un candidato en la lista de espera, y afirmaba como
  resultado esperado que la cascada de cancelación liberaba la franja y que
  el candidato la aceptaba de inmediato dentro del mismo rango de ausencia —
  exactamente el comportamiento incorrecto que este ticket corrige. Se
  reescribió para afirmar lo contrario: la franja no se ofrece a nadie y el
  candidato conserva su lugar en la lista.
- Ejecución: suite unitaria completa en verde (857 pruebas), suite de
  extremo a extremo de los archivos tocados y de los módulos relacionados
  (ausencia, feriado, disponibilidad, reasignación, reprogramación,
  seed) en verde contra la instancia local de PostgreSQL con
  `--runInBand`; `eslint` sin hallazgos nuevos. Los datos usados en las
  pruebas son ficticios.

## Figuras pendientes

Ninguna figura nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-149-absence-blocks-booking-and-waitlist`, creada desde
  `origin/main` fresco (`b96810d`, con TASK-147/TASK-148 ya fusionadas).
  Pusheada a `origin`; PR abierto, no fusionado aún.
- Ticket: TASK-149 ("Una ausencia registrada no bloquea reservas ni evita
  que la lista de espera reofrezca esos horarios"), tarea de auditoría
  automática (hallazgos PROF-1 y PROF-2) validada por la usuaria antes de
  implementarse. Misma convención de bitácora dedicada para una corrección
  puntual dentro de la fase del ticket original que TASK-127/TASK-135/
  TASK-137 ([[FASE-3_PROMPT-32]]/[[FASE-3_PROMPT-33]]/[[FASE-3_PROMPT-34]]).
