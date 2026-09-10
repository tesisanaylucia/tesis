# Fase 3 — Motor de Turnos (backend) — reserva y reasignación de lista de espera no verifican que el instante siga siendo futuro (TASK-153, corrección a TASK-36/TASK-40)

## Contexto

TASK-153 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, la misma auditoría que dio
origen a TASK-150 ([[FASE-3_PROMPT-36]]) y TASK-151 ([[FASE-3_PROMPT-37]]),
validada por la usuaria antes de implementarse. El detalle completo está en
el artefacto `https://claude.ai/code/artifact/5c2ee112-c426-4d6e-a8e2-499ea7bbbc74`
("auditoria-psique-back.md"), hallazgos PROF-8 y TUR-9.

PROF-8 señalaba que `AppointmentsService.book()` ofrece el mismo chequeo de
"instante futuro" que ya tiene `rescheduleCore` desde su implementación
original, pero nunca lo aplicó: `getSlots` puede ofrecer un instante que ya
pasó (la propia grilla de horario habitual es ajena a "ahora", por diseño,
ya que también la usa `isOnHabitualGrid` para validar instantes pasados de
un turno existente), y nada en `book()` impedía reservarlo. TUR-9 señalaba,
de forma relacionada, que el recorrido de reasignación automática de lista
de espera (`WaitlistReassignmentService`) tampoco comprueba que
`event.scheduledAt` siga siendo futuro antes de cada oferta: con varios
candidatos, cada uno con hasta cuatro horas de ventana de respuesta
(P4.5/TASK-82), el recorrido puede alcanzar a un candidato tardío, o este
puede aceptar minutos antes de que cierre su propia ventana, cuando el
instante original del turno cancelado ya pasó o incluso ya empezó.

## Qué se implementó

**PROF-8 (reserva directa).** Se agregó en `AppointmentsService.book()`, justo
después de calcular `scheduledAt` a partir del DTO y antes de cualquier otra
validación, el mismo chequeo que ya tiene `rescheduleCore`:
`scheduledAt.getTime() <= clinicNow()` lanza `BadRequestException`. Se
extendió además `AvailabilityService` con un predicado de módulo,
`isFutureInstant`, usado para excluir de la oferta cualquier instante que el
reloj de la clínica ya alcanzó — en el bucle principal de `getSlots` (antes
de la consulta de ocupación, ya que es la exclusión más barata) y en la rama
de franja extra de `getNewPatientSlots` (`PRIMER_TURNO_DIA`/
`ULTIMO_TURNO_DIA`), que construye su propio par de instantes sin pasar por
`getSlots`. Deliberadamente **no** se tocó `loadGridByDay` ni
`isOnHabitualGrid`: ambos son, por diseño, ajenos a "ahora" —
`isOnHabitualGrid` valida pertenencia a la grilla de un turno existente,
que puede legítimamente estar en el pasado— así que el filtro de "instante
futuro" se aplicó solo donde la agenda *ofrece* un instante nuevo, no donde
se valida uno ya dado.

**TUR-9 (reasignación de lista de espera).** Se agregó en
`WaitlistReassignmentService` un predicado equivalente,
`isStillInTheFuture`, y dos chequeos, replicando la misma forma de
"chequeo temprano + reescritura guardada" que `rescheduleCore`/
`applyRescheduleWrite` ya usan para el mismo tipo de problema: uno en
`advanceWaitlist`, antes incluso de la consulta de feriado/ausencia (es la
forma más barata de terminar el recorrido), que libera la retención y
termina sin ofrecer nada si el instante ya no es futuro; y otro dentro de
`reserveInTransaction`, antes de la creación real del turno, que lanza
`SlotUnavailableError` — la misma clase que ya usa el chequeo de feriado/
ausencia de esa transacción, con el mismo tratamiento: la retención se
libera (el instante ya no sirve para nadie más) y la oferta permanece
`ACCEPTED` en el registro (reescribirla como `REJECTED` pondría en la traza
de auditoría, exigida por la Ley 25.326, un rechazo que el paciente nunca
expresó).

## Decisiones y por qué

**Usar `clinicNow()`, no `Date.now()`, en los tres puntos nuevos.** TASK-150
ya estableció que `scheduledAt` es la hora de pared de la clínica reetiquetada
como UTC, no un instante UTC real, y que toda comparación contra un
`scheduledAt` guardado debe pasar por `clinicNow()` en vez de `Date.now()`
directo — el propio chequeo de `rescheduleCore` que esta tarea replica ya lo
hace así. Los tres chequeos nuevos (`book()`, `AvailabilityService`,
`WaitlistReassignmentService`) siguen la misma convención.

**No incrustar el chequeo de "instante futuro" dentro de `loadGridByDay`.**
La solución propuesta por la auditoría se limita a "agregar el mismo chequeo
que ya tiene `rescheduleCore`" en `book()`, sin mencionar `getSlots`, pero se
consideró necesario extenderlo ahí también: dejar `book()` como única
barrera habría dejado la agenda ofreciendo instantes ya pasados hasta el
momento mismo de la reserva, una experiencia confusa para quien la consulta
antes de reservar (el chatbot, o el propio profesional). Se descartó
incrustarlo en `loadGridByDay` en sí, ya que esa función también respalda
`isOnHabitualGrid`, que debe seguir aceptando un instante pasado al validar
un turno existente — el filtro se aplicó, en cambio, en los puntos donde la
agenda construye una *oferta* nueva a partir de esa grilla.

**Repetir el chequeo en `reserveInTransaction`, no confiar solo en el de
`advanceWaitlist`.** Misma razón, ya documentada en el código para
`rescheduleCore`/`applyRescheduleWrite`: el chequeo temprano corre contra un
"ahora" que puede quedar obsoleto para cuando el candidato responde, hasta
casi cuatro horas después.

## Alternativas descartadas

- **Comparar `event.scheduledAt` contra "ahora" una sola vez, en
  `handleAppointmentCancelled`, en vez de en cada avance del recorrido
  (`advanceWaitlist`).** Se descartó: el recorrido puede extenderse por
  varios candidatos consecutivos, cada uno con su propia ventana de
  respuesta, así que el instante puede pasar de futuro a pasado en medio del
  recorrido — un chequeo único al principio no lo habría detectado.

## Entidades / puertos / adaptadores tocados

- `src/appointments/appointments.service.ts` (modificado): `book` (agrega el
  chequeo de instante futuro, antes de toda otra validación).
- `src/availability/availability.service.ts` (modificado): `isFutureInstant`
  (nuevo, privado al módulo); `getSlots` (excluye instantes no futuros de su
  bucle principal); `getNewPatientSlots` (excluye instantes no futuros de su
  rama de franja extra).
- `src/waitlist/waitlist-reassignment.service.ts` (modificado):
  `isStillInTheFuture` (nuevo, privado al módulo); `advanceWaitlist` (chequeo
  temprano, antes de la consulta de feriado/ausencia); `reserveInTransaction`
  (chequeo tardío, dentro de la transacción de reserva, reutilizando
  `SlotUnavailableError`).
- Sin cambios de esquema: ningún modelo nuevo, ninguna migración — la
  corrección es exclusivamente de validación, no de dato almacenado.

## Tests y qué validan

- `src/appointments/appointments.service.spec.ts` (unitaria): nuevo caso que
  prueba el rechazo de `book()` cuando `scheduledAt` ya no es futuro.
- `src/availability/availability.service.spec.ts` (unitaria): nuevo caso en
  `getSlots` que mockea `Date.now()` a mitad de una grilla y prueba que los
  instantes ya alcanzados quedan excluidos mientras los posteriores se
  siguen ofreciendo; nuevo caso equivalente para la rama
  `PRIMER_TURNO_DIA` de `getNewPatientSlots`.
- `src/waitlist/waitlist-reassignment.service.spec.ts` (unitaria): dos casos
  nuevos, paralelos a los ya existentes para feriado/ausencia — uno que
  prueba que `advanceWaitlist` libera la retención y no ofrece nada cuando
  el instante ya no es futuro, y otro que prueba que la aceptación de una
  oferta falla explícitamente (reteniendo el estado `ACCEPTED` de la oferta)
  cuando el instante deja de ser futuro entre el registro de la oferta y su
  aceptación.
- Impacto en fixtures existentes: los tres chequeos nuevos expusieron que
  varias fixtures unitarias fijaban `scheduledAt` a una fecha calendario
  concreta en vez de relativa a `clinicNow()`, y el paso del tiempo real las
  había dejado en el pasado sin que nada lo notara hasta ahora — el mismo
  tipo de fragilidad que TASK-150 ya había corregido en otras fixtures. Se
  corrigieron ancladas a `clinicNow()` (siguiendo la convención que ya usan
  las fixtures de reprogramación) en `appointments.service.spec.ts`,
  `appointments-rescheduling.service.spec.ts` y
  `waitlist-reassignment.service.spec.ts`; en
  `availability.service.spec.ts`, cuyas fixtures dependen además del día de
  la semana (horario laboral por `Weekday`), se optó por mockear
  `Date.now()` a un instante fijo muy anterior a las fechas de la fixture en
  vez de reescribirlas, para no perder esa dependencia deliberada del día.
  El mismo ajuste se replicó en `test/appointments-booking.e2e-spec.ts`,
  `test/availability.e2e-spec.ts` y `test/holidays.e2e-spec.ts` (extremo a
  extremo, contra PostgreSQL real): las tres mockean `Date.now()` a un
  instante fijo anterior a sus fixtures existentes; `availability.e2e-spec.ts`
  tiene además dos pruebas de retención MANUAL que comparan contra el reloj
  real sin pasar por `clinicNow()` (`AvailabilityService.isSlotFree` usa
  `new Date()`, no `Date.now()`, para esa comparación), así que sus propias
  fixtures de `holdUntil` se reescribieron para construirse también con
  `new Date()` en vez de `Date.now()`, de modo que sigan de acuerdo con el
  reloj que la comparación real usa pese al mock del archivo.
- Ejecución: suite unitaria completa en verde (888 pruebas, 87 suites);
  suite de extremo a extremo completa en verde salvo una prueba preexistente
  y ajena a esta corrección (`appointment-reassignment.e2e-spec.ts`,
  "MANUAL: the professional can still assign the held slot by hand during
  the 24h window"), intermitente en una ventana de aproximadamente treinta
  minutos por día en la que "ahora + 5 horas, redondeado a la media hora
  siguiente" cae exactamente en el límite de cierre de la jornada
  (`23:30`) configurada por esa fixture — reproducida contra el propio
  `origin/main`, sin ninguno de los cambios de esta tarea, así que queda
  fuera de alcance y solo documentada; `eslint` sin hallazgos nuevos; `tsc
  --noEmit` sin errores. Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna figura nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-153-future-instant-checks`, creada
  desde `origin/main` fresco (`43256ab`, con TASK-151 y TASK-152 ya
  fusionadas).
- Ticket: TASK-153 ("Reserva y reasignación de lista de espera no verifican
  que el instante siga siendo futuro"), tarea de auditoría automática
  (hallazgos PROF-8 y TUR-9) validada por la usuaria antes de
  implementarse. Misma convención de bitácora dedicada para una corrección
  puntual dentro de la fase del ticket original que TASK-150/TASK-151
  ([[FASE-3_PROMPT-36]]/[[FASE-3_PROMPT-37]]).
