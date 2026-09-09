# Fase 3 — Motor de Turnos (backend) — `isSlotFree` sin validación de grilla real ni solapamiento de intervalo (TASK-151, corrección a TASK-35/TASK-36/TASK-113)

## Contexto

TASK-151 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, la misma auditoría que dio
origen a TASK-150 ([[FASE-3_PROMPT-36]]). El detalle completo está en el
artefacto `https://claude.ai/code/artifact/5c2ee112-c426-4d6e-a8e2-499ea7bbbc74`
("auditoria-psique-back.md"), hallazgos PROF-3 y PROF-4.

PROF-3 señalaba que solo la reserva de un paciente nuevo bajo una franja
extra configurada validaba el instante elegido contra una grilla real
(`getNewPatientSlots`, vía `AppointmentsService.assertValidNewPatientStart`):
la reserva de un paciente recurrente y toda reprogramación pasaban
exclusivamente por `AvailabilityService.isSlotFree`, que solo comprueba
ocupación (otro turno, feriado, ausencia, hold), nunca si el instante
coincide con la grilla de horarios habituales, cadencia y duración del
profesional. PROF-4 señalaba, de forma independiente, que tanto `isSlotFree`
como el cálculo de la agenda (`loadBookedTimes`, hoy renombrado
`loadBookedIntervals`) comparaban ocupación por **igualdad exacta de
instante**, no por solapamiento de intervalo `[scheduledAt,
scheduledAt+duration)`. El escenario que la auditoría citaba como ejemplo:
reconfigurar `slotCadence` de 60 a 90 minutos con un turno ya reservado a las
10:00 (duración original 45 minutos, hasta las 10:45) hace que la grilla
recalculada ofrezca 10:30 como libre — un instante que en rigor se solapa con
la sesión ya reservada, porque ninguno de los dos instantes coincide
literalmente con el otro.

## Qué se implementó

**PROF-4 (solapamiento de intervalo).** Se introdujo en
`availability.service.ts` un par de funciones puras, `rangesOverlap`
(compara dos rangos `[start, start+duration)`) e `isOccupied` (si un
candidato se solapa con algún intervalo de una lista), y un tipo
`OccupiedInterval { scheduledAt, duration }`. Estas dos funciones son ahora
la única definición de "ocupado" del archivo, usada por tres sitios que
antes comparaban cada uno a su manera: `isSlotFree` (ahora recibe `duration`
como parámetro obligatorio y compara contra un `findMany` acotado a un día
antes/después del candidato, no un `findFirst` por igualdad exacta),
`getSlots` (su exclusión de instantes ya reservados) y la rama de franja
extra de `getNewPatientSlots` (`PRIMER_TURNO_DIA`/`ULTIMO_TURNO_DIA`). El
método antes privado `loadBookedTimes` (devolvía un `Set<string>` de
instantes exactos) se renombró `loadBookedIntervals` y ahora devuelve
`OccupiedInterval[]`, con la propia `duration` de cada turno ya reservado —
nunca la `consultationDuration` *actual* del profesional, que puede haber
cambiado desde que ese turno se reservó.

**PROF-3 (validación de grilla).** Se agregó `AvailabilityService
.isOnHabitualGrid(professionalId, scheduledAt, schedule)`, construido sobre
el mismo `loadGridByDay` que ya usa `getSlots` — la "función compartida" que
la auditoría pedía extraer — para responder si un instante es un inicio real
de la grilla habitual (horario laboral, cadencia, duración), sin mirar
ocupación. En `AppointmentsService` se agregó `assertOnHabitualGrid` (lanza
`BadRequestException` si no lo es) y se la conectó en dos puntos que antes
no pasaban por ningún control de grilla: `book()`, para todo turno que no
sea la primera sesión de un paciente nuevo bajo una franja extra configurada
(la reserva recurrente, y la primera sesión sin franja configurada, que se
reserva "como cualquier otra" según el comentario ya existente en el
código); y `rescheduleCore`/`applyRescheduleWrite`, para toda
reprogramación, sin excepción de `isFirstSession`. `applyRescheduleWrite`
repite el chequeo (contra `ConflictException`, no `BadRequestException`) por
la misma razón por la que ya repetía el chequeo de "debe ser futuro": una
reprogramación con confirmación pendiente puede aceptarse horas después de
registrada, y la configuración del profesional pudo cambiar en el ínterin.

## Decisiones y por qué

**No incrustar la validación de grilla dentro de `isSlotFree` en sí, a
pesar de que la auditoría lo sugiere literalmente.** La solución propuesta
por la auditoría dice "llamarla también desde `isSlotFree`", lo que
implicaría que *todo* llamador de `isSlotFree` —incluida la segunda mitad de
un turno doble bajo `PRIMER_TURNO_DIA`/`ULTIMO_TURNO_DIA`— quedara sujeto al
control de grilla habitual. Eso es incorrecto: esas dos modalidades colocan
deliberadamente sus dos mitades *fuera* de la grilla habitual (TASK-121,
corrección a TASK-37) precisamente para no competir con los turnos ya
ofrecidos a todo el mundo. Incrustar el control dentro de `isSlotFree` habría
rechazado toda reserva de paciente nuevo bajo esas dos modalidades — un
defecto nuevo al corregir uno viejo. La solución implementada logra el mismo
objetivo (ningún turno recurrente ni reprogramación pasa sin control de
grilla) con `assertOnHabitualGrid` como un chequeo explícito y separado en
`AppointmentsService`, aplicado exactamente donde `assertValidNewPatientStart`
no se aplica ya.

**Extender el control de grilla a toda reprogramación, no solo a la que
mueve una primera sesión.** El hallazgo cita "reprogramación" en general,
sin distinguir según el tipo de turno, y antes de esta corrección ninguna
reprogramación pasaba por ningún control de grilla, sin excepción. Se
consideró eximir del control a la reprogramación de una primera sesión
colocada originalmente por una franja extra (para permitirle moverse a otra
posición igualmente fuera de la grilla habitual), pero se descartó: la
reprogramación mueve un turno a la vez y no tiene ningún mecanismo que
mueva junto con él la otra mitad de su turno doble, así que permitir esa
reprogramación ya dejaría la pareja desincronizada por una razón ajena a
esta corrección. Exigir la grilla habitual en toda reprogramación evita
además ese caso, sin quitarle capacidad real a ningún flujo hoy soportado.

**Repetir el chequeo de grilla en `applyRescheduleWrite`, no solo en
`rescheduleCore`.** Es la misma razón, documentada ya en el código para el
chequeo de "instante futuro", por la que `TASK-115` dejó ambos chequeos
duplicados entre el registro de una oferta y su aplicación diferida: la
configuración del profesional (horario, cadencia, duración) puede cambiar
en las horas que pasan entre que se registra una `RescheduleOffer` y que el
paciente la acepta.

## Alternativas descartadas

- **Acotar la ventana de la consulta de conflicto de `isSlotFree` al mismo
  día calendario del instante candidato**, en vez de un día antes y después.
  Se descartó: `loadEdgeExtraSlotsByDay` ya documenta que un turno de
  `PRIMER_TURNO_DIA`/`ULTIMO_TURNO_DIA` puede legítimamente cruzar la
  medianoche cuando el límite de la jornada cae cerca de ella, así que una
  ventana de exactamente un día podría dejar fuera un conflicto real.
- **Reutilizar `getSlots`/`getNewPatientSlots` (que ya excluyen instantes
  ocupados) como el propio control de grilla**, en vez de un método nuevo
  ajeno a la ocupación. Se descartó: un instante ocupado desaparecería de
  esas listas por la razón equivocada, y el mensaje de error resultante
  ("no es un instante válido de la grilla") habría sido engañoso para lo que
  en realidad es un conflicto de ocupación, ya cubierto con su propio
  mensaje por `isSlotFree`.

## Entidades / puertos / adaptadores tocados

- `src/availability/availability.service.ts` (modificado): `rangesOverlap`,
  `isOccupied`, `OccupiedInterval` (nuevos, privados al módulo);
  `isSlotFree` (firma: nuevo parámetro `duration`, conflicto por
  solapamiento vía `findMany` en vez de `findFirst` por igualdad exacta);
  `isOnHabitualGrid` (nuevo, público); `loadBookedTimes` renombrado
  `loadBookedIntervals` (devuelve intervalos, no instantes); `getSlots` y
  `getNewPatientSlots` (su rama de franja extra) migrados a `isOccupied`.
- `src/appointments/appointments.service.ts` (modificado): `book`
  (agrega el chequeo de grilla habitual para toda reserva que no sea una
  primera sesión bajo franja extra; sus dos llamadas a `isSlotFree` ahora
  pasan `duration`); `rescheduleCore` (agrega el chequeo previo a la
  escritura); `applyRescheduleWrite` (repite el chequeo antes de la
  escritura real, y su llamada a `isSlotFree` pasa `appointment.duration`);
  `assertOnHabitualGrid` (nuevo, privado).
- Sin cambios de esquema: ningún modelo nuevo, ninguna migración — la
  corrección es exclusivamente de validación, no de dato almacenado.

## Tests y qué validan

- `src/availability/availability.service.spec.ts` (unitaria): se reescribió
  la suite de `isSlotFree` sobre la nueva firma (`findMany` en vez de
  `findFirst`, con ventana de un día antes/después) y se agregaron casos de
  solapamiento real sin coincidencia de instante exacto (un turno existente
  que empieza 30 minutos después del candidato pero comparte 15 minutos con
  él) y el caso límite de dos turnos exactamente adyacentes (no se solapan).
  Se agregó una suite nueva, `AvailabilityService.isOnHabitualGrid`, que
  prueba pertenencia/no pertenencia a la grilla por horario laboral,
  cadencia, feriado y ausencia. Las suites de `getSlots` y
  `getNewPatientSlots` ganaron un caso cada una probando exclusión por
  solapamiento (no por igualdad) cuando la duración del turno ya reservado
  difiere de la `consultationDuration` actual del profesional — el escenario
  literal que cita la auditoría.
- `src/appointments/appointments.service.spec.ts` y
  `appointments-rescheduling.service.spec.ts` (unitarias): se agregaron
  casos que prueban el rechazo por grilla habitual (`book`/reprogramación) y
  que confirman, por omisión de la llamada, que una primera sesión bajo
  franja extra configurada nunca pasa por ese control.
- Impacto en la suite de extremo a extremo: la validación de grilla,
  correcta pero antes inexistente, expuso que varias fixtures de
  `test/appointments-booking.e2e-spec.ts`,
  `test/appointments-rescheduling.e2e-spec.ts`,
  `test/appointment-engine-integration.e2e-spec.ts`,
  `test/appointment-reassignment.e2e-spec.ts` y
  `test/access-code-invalidation-expiration.e2e-spec.ts` reservaban o
  reprogramaban turnos contra profesionales sin horario laboral configurado,
  o hacia instantes construidos como `Date.now() + N horas` sin redondear a
  un múltiplo de media hora — instantes que la ocupación por igualdad
  exacta nunca exigió que fueran reales, pero que la grilla sí exige ahora.
  Se corrigieron las cinco fixtures: se agregó horario laboral amplio
  (todos los días, 00:00-23:30) a los profesionales de prueba que lo
  necesitaban, y se redondeó al próximo medio-hora todo instante usado como
  destino de una reserva o reprogramación. En
  `appointments-rescheduling.e2e-spec.ts`, redondear expuso además una
  colisión latente ya presente en el diseño de esas fixtures — varias
  pruebas comparten un mismo profesional sin limpieza entre pruebas y
  confiaban en la resolución de milisegundos real para no chocar entre sí —
  y se corrigió introduciendo un contador de horas monotónico compartido
  por todo el archivo, que le da a cada instante usado en el archivo su
  propio bloque de media hora, sin colisión posible.
- Ejecución: suite unitaria completa en verde (873 pruebas, 86 suites),
  suite de extremo a extremo completa en verde (606 pruebas, 51 suites)
  contra la instancia local de PostgreSQL con `--runInBand`, y
  `npm run test:cov` (unitaria + extremo a extremo combinadas) en verde sin
  ningún archivo del área tocada por debajo del umbral del 80% exigido
  sobre la capa de servicios; `eslint --fix` sin hallazgos nuevos; `tsc
  --noEmit` sin errores. Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna figura nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-151-slot-free-grid-and-overlap-validation`,
  creada desde `origin/main` fresco (`1b3d734`, con TASK-150 ya fusionada).
- Ticket: TASK-151 ("`isSlotFree` no valida contra la grilla de horarios ni
  contra la duración de la sesión"), tarea de auditoría automática
  (hallazgos PROF-3 y PROF-4) validada por la usuaria antes de
  implementarse. Misma convención de bitácora dedicada para una corrección
  puntual dentro de la fase del ticket original que TASK-127/TASK-135/
  TASK-137/TASK-149/TASK-150 ([[FASE-3_PROMPT-32]]/[[FASE-3_PROMPT-33]]/
  [[FASE-3_PROMPT-34]]/[[FASE-3_PROMPT-35]]/[[FASE-3_PROMPT-36]]).
