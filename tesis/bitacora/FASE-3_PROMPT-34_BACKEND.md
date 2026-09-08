# Fase 3 — Motor de Turnos (backend) — gestión incompleta de feriados y ausencias: condición de carrera en la edición de feriados, sin edición para las ausencias (TASK-137, corrección a TASK-78/TASK-23)

## Contexto

TASK-137 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, validada por la usuaria
antes de implementarse, con dos hallazgos combinados (PROF-12 y PROF-13). El
detalle completo de la auditoría está en el artefacto
`https://claude.ai/code/artifact/5c2ee112-c426-4d6e-a8e2-499ea7bbbc74`
("auditoria-psique-back.md").

PROF-12 apuntaba a `HolidaysService.update`/`.remove`: ambos resolvían el
feriado con una lectura previa a cualquier transacción y recién después
abrían una transacción propia para escribir sobre el `id` ya leído. Dos
peticiones concurrentes sobre la misma fecha —dos `DELETE`, o un `PATCH` y un
`DELETE`— podían pasar esa lectura antes de que la otra confirmara y luego
competir por la escritura: la que llegaba segunda encontraba la fila ya
borrada y Prisma lanzaba un `P2025` sin capturar, que el manejador de
excepciones por defecto convertía en un 500 en lugar de un 404 o un 409.

PROF-13 apuntaba a `AbsencesService`: no existía ningún método ni ruta de
edición, solo alta y baja. Extender una ausencia un día —el caso que motivó
el hallazgo— exigía borrarla y volver a crearla, lo que además de perder la
razón original descartaba los turnos que la cascada de cancelación de la
ausencia original ya había cancelado (`AbsenceCancelledEvent` documenta
explícitamente que retirar una ausencia no los restaura) y volvía a barrer
el rango completo nuevo en busca de turnos que cancelar, en lugar de acotarse
a los días que la edición agrega de verdad.

## Qué se implementó

**PROF-12.** `update`/`remove` de `HolidaysService` pasan a resolver el
feriado y a escribir sobre él dentro de la misma transacción `SERIALIZABLE`,
usando el helper ya existente `runSerializable` —el mismo que ya envolvía la
creación de una ausencia (P1.3) o el reemplazo de la grilla horaria—, en
lugar de una lectura suelta seguida de una transacción aparte. Con la lectura
adentro de la transacción, una fecha ya borrada al momento en que arranca esa
transacción es simplemente un 404 (la propia búsqueda no encuentra nada), y
un conflicto genuino con otra transacción concurrente sobre la misma fila es
el 409 que `runSerializable` ya traduce desde el error de serialización de
Postgres — nunca más un 500 sin capturar.

**PROF-13.** Se agregó `PATCH /profesionales/:id/ausencias/:absenceId`
(`AbsencesService.update`), detrás del mismo `ProfessionalOwnershipGuard` que
ya protege el alta y la baja. Permite ajustar fecha de inicio, fecha de fin
y/o motivo, cualquier subconjunto de los tres en una misma petición. Reutiliza
el mismo chequeo de solapamiento que ya usa `create` —ahora excluyendo la
propia fila que se está editando— dentro de una transacción `SERIALIZABLE`
igual a la de aquel método, por el mismo motivo: "ninguna ausencia se
superpone con otra" es un invariante de lectura-y-escritura que Read
Committed no alcanza a proteger. Un `PATCH` vacío —ningún campo enviado— no
escribe nada ni deja una entrada de auditoría, siguiendo la misma convención
que ya usan `PatientsService.update`/`PatientProfessionalsService.update`.

La parte central del hallazgo es cómo se decide qué cascada de cancelación
disparar tras la edición: se calcula el sub-rango de días que la ausencia
nueva cubre y la anterior no —cero, uno o dos sub-rangos, según si el cambio
achica, agranda por un solo extremo, agranda por los dos extremos a la vez, o
mueve la ausencia a un rango que no toca en absoluto al anterior—, y solo
por esos días nuevos se encola un `CascadeCancellationJob` y se publica
`AbsenceRegisteredEvent`, el mismo evento y el mismo mecanismo que ya dispara
`AppointmentsService.cancelForAbsence` desde `create`. Un recorte puro, o una
edición que deja el rango exactamente igual (por ejemplo, un cambio de motivo
sin tocar fechas), no dispara ninguna cascada.

## Decisiones y por qué

**Reutilizar `AbsenceRegisteredEvent`/`cancelForAbsence` en vez de inventar un
evento de "ausencia editada".** El propio hallazgo pide que la edición ajuste
fechas "sin recancelar lo ya cancelado"; acotar el rango que se publica a solo
los días nuevos alcanza exactamente ese objetivo sin tocar el adaptador que
ya conecta el módulo de profesionales con el motor de turnos
(`AppointmentAbsenceEventsAdapter`), ni el patrón de outbox transaccional que
TASK-135 ya le dio a esa cascada. Inventar un tercer disparador habría
duplicado esa infraestructura para un caso que, mirado desde el motor de
turnos, es indistinguible de un alta parcial.

**No restaurar turnos cuando la edición achica el rango.** Ni el hallazgo ni
el SRS piden esa restauración, y hacerlo exigiría resolver si el horario
original sigue libre para reclamarlo — la misma pregunta, sin respuesta
provista, que ya llevó a documentar por qué retirar una ausencia tampoco
restaura nada (`AbsenceCancelledEvent`). Esta corrección extiende esa
decisión ya tomada a la edición en lugar de reabrirla.

**El chequeo de fin-no-antes-que-inicio y el de solapamiento corren dentro de
la transacción, sobre la fila leída ahí mismo, no sobre el DTO recibido
directamente.** Un campo omitido en el `PATCH` debe compararse contra el
valor vigente de la ausencia, no contra un valor por defecto inventado; leer
la fila dentro de la misma transacción serializable que hace la escritura
evita además que esa lectura quede, igual que en PROF-12, separada de la
escritura que depende de ella.

**Factorización de `getOwnedAbsenceOrThrow`.** El chequeo de pertenencia que
ya usaba `remove` (profesional dueño, ausencia perteneciente a ese
profesional) devolvía antes solo un booleano implícito vía excepción; se
extrajo la mitad que resuelve la fila para que `update` pudiera reusarla
tanto fuera de una transacción (el atajo del `PATCH` vacío) como dentro de
una (la edición real), sin duplicar la consulta.

## Alternativas descartadas

- **Capturar `P2025` en `update`/`remove` en vez de mover la lectura a la
  transacción**: el propio hallazgo ofrecía esta alternativa. Se descartó
  porque no cierra el caso de dos transacciones genuinamente concurrentes
  —ahí Postgres no responde "fila no encontrada" sino un conflicto de
  serialización real—, mientras que envolver la lectura junto con la
  escritura resuelve ambos casos (fila ya ausente y conflicto concurrente) a
  la vez, con el mismo mecanismo que el resto del código ya usa para este
  tipo de invariante.
- **Re-disparar la cascada sobre el rango completo nuevo en cada edición**:
  no habría duplicado turnos cancelados —el filtro por estado
  RESERVADO/CONFIRMADO ya lo impide— pero sí habría re-barrido a cada
  edición los días que ninguna edición tocó, contradiciendo la letra del
  hallazgo ("sin recancelar lo ya cancelado") aunque no su efecto observable
  final.

## Entidades / puertos / adaptadores tocados

- `src/holidays/holidays.service.ts` (modificado): `update`/`remove` ahora
  corren dentro de `runSerializable`; `getByDateOrThrow` pasa a recibir el
  handle de transacción en vez de usar siempre el cliente plano.
- `src/professionals/absences.service.ts` (modificado): método nuevo
  `update`; `assertNoOverlap` gana un parámetro opcional
  `excludeAbsenceId`; `assertOwnedAbsenceExists` se apoya en el nuevo
  `getOwnedAbsenceOrThrow`; método privado nuevo `newlyBlockedRanges`.
- `src/professionals/dto/update-absence.dto.ts` (nuevo): `UpdateAbsenceDto`,
  los tres campos de `CreateAbsenceDto` vueltos opcionales.
- `src/professionals/absences.controller.ts` (modificado):
  `PATCH /profesionales/:id/ausencias/:absenceId`, detrás de
  `ProfessionalOwnershipGuard`.
- `src/common/dates/calendar-date.ts` (modificado): función nueva
  `shiftDays`, la aritmética de un día que `newlyBlockedRanges` necesita
  para expresar "el día anterior a" / "el día siguiente a" un extremo del
  rango.
- Sin cambios de esquema: ningún modelo nuevo, ninguna migración.

## Tests y qué validan

- `test/holidays.e2e-spec.ts` (ampliado, contra Postgres real): dos
  `DELETE` concurrentes sobre el mismo feriado nunca responden 500 —cada
  resultado es 200, 404 o 409, y exactamente uno de los dos es 200—; un
  `PATCH` y un `DELETE` concurrentes sobre el mismo feriado tampoco 500 y el
  estado final de la fila coincide exactamente con si el `DELETE` llegó a
  responder 200 o no.
- `test/professional-schedules.e2e-spec.ts` (ampliado, contra Postgres
  real): edición de fechas y motivo con verificación de la entrada de
  auditoría (`detail: { professionalId, fields }`); fin antes que inicio
  rechazado sin tocar la fila; solapamiento con otra ausencia rechazado,
  pero reenviar el propio rango sin cambios no colisiona consigo mismo;
  ampliar una ausencia por ambos extremos a la vez dispara el evento solo
  para los dos sub-rangos nuevos, con un id de `CascadeCancellationJob`
  distinto al de la creación original; mover una ausencia a un rango
  disjunto trata el rango nuevo completo como recién bloqueado; achicar el
  rango o dejarlo igual no dispara ningún evento; cambiar solo el motivo
  deja una única entrada de auditoría sin evento; un `PATCH` vacío no
  escribe ninguna entrada de auditoría; 403/404 de pertenencia y 404
  entre inquilinos, siguiendo la misma convención que ya prueban el alta y
  la baja; dos ediciones concurrentes sobre la misma ausencia nunca dejan
  perder una en silencio (exactamente una 200, la otra 409).
- Ejecución: suite unitaria completa en verde (134 conjuntos combinados
  unidad + extremo a extremo, 1395 pruebas, cobertura de la capa de
  servicios por encima del umbral del 80 % — las dos únicas líneas sin
  cubrir en los dos servicios tocados son sendos `throw` defensivos ya
  documentados como "inalcanzables en la práctica", preexistentes a esta
  tarea), todo contra la instancia local de PostgreSQL con `--runInBand`;
  `tsc --noEmit` y `eslint` sin hallazgos. Los datos usados en las pruebas
  son ficticios.

## Figuras pendientes

Ninguna figura nueva. El diagrama de cancelación en cascada ya pendiente
(`figuras_pendientes.md`) puede, cuando se produzca, anotar que una edición
de ausencia también puede disparar la cascada sobre un sub-rango, sin que
eso cambie la forma del diagrama en sí.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-137-holiday-transaction-absence-edit`, creada desde
  `origin/main` fresco (`6b67511`, con TASK-136 ya fusionada). Pusheada a
  `origin`; PR abierto, no fusionado aún.
- Ticket: TASK-137 ("Gestión incompleta de feriados y ausencias (edición y
  condiciones de carrera)"), tarea de auditoría automática (hallazgos
  PROF-12 y PROF-13) validada por la usuaria antes de implementarse. Misma
  convención de bitácora dedicada para una corrección puntual dentro de la
  fase del ticket original que TASK-127/TASK-135
  ([[FASE-3_PROMPT-32]]/[[FASE-3_PROMPT-33]]).
