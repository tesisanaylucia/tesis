# Fase 3 — Motor de Turnos (backend) — FIRST_SLOT_OF_DAY/LAST_SLOT_OF_DAY reutilizaban slots existentes en vez de agregar tiempo extra (TASK-121, corrección a TASK-37)

## Contexto

La franja extra para paciente nuevo (P3.4, TASK-37, [[FASE-3_PROMPT-4]]) deja
que un profesional configure, en tres modalidades, dónde se ubica el doble
turno de una primera sesión. El documento de requisitos es explícito sobre
dos de esas tres: el primer turno del día "se agrega una hora antes de la
franja habitual", y el último "se agrega una hora después" — en ambos casos,
una franja genuinamente adicional, agregada por fuera del horario que el
profesional ya ofrece a sus pacientes en tratamiento. TASK-86
([[FASE-3_PROMPT-15]]) ya había resuelto, antes de esta tarea, que esa
magnitud fuera un tamaño fijo —dos veces la duración de la consulta— en
lugar de una cantidad de horas configurable, retirando el campo que la
representaba. Esta tarea es distinta: incluso con el tamaño fijo ya
decidido, la ubicación seguía sin ser genuinamente extra.

`AvailabilityService.getNewPatientSlots` calculaba, para estos dos modos, el
primer o el último turno de la grilla de horario habitual —la misma que
`getSlots` ofrece a cualquier paciente— y colocaba ahí el doble turno. Con un
horario de 09:00 a 11:00 y sesiones de 30 minutos, `FIRST_SLOT_OF_DAY`
devolvía las 09:00: el mismo horario que un paciente en tratamiento ya podía
reservar, no una franja anterior a las 09:00 que nadie más disputa. La propia
prueba unitaria escrita para el modo en TASK-37 lo confirmaba sin que se
interpretara como una falla. El defecto invertía la razón de ser explícita de
la regla, que existe para que una primera sesión —más propensa a
extenderse— no le quite capacidad al resto de la jornada. La detección
proviene de una auditoría de código sobre `psique-back/main` (2026-08-14,
agente "Audit waitlist/reassignment/holidays vs SRS"), la misma que dio
origen a TASK-113, TASK-114 y TASK-116.

## Qué se implementó

- `AvailabilityService` calcula ahora, para `FIRST_SLOT_OF_DAY`/
  `LAST_SLOT_OF_DAY` exclusivamente, un par de instantes por fuera de la
  grilla de horario habitual del día: el par que **termina** exactamente
  donde empieza el primer bloque de horario configurado (`FIRST_SLOT_OF_DAY`),
  o el que **empieza** exactamente donde termina el último bloque
  (`LAST_SLOT_OF_DAY`). Los dos turnos del par distan entre sí una cadencia
  (`schedule.step`), el mismo espaciado que separa a cualquier par de turnos
  consecutivos de la agenda desde TASK-114.
- `getNewPatientSlots` deja de enrutar estos dos modos a través de
  `getSlots`/la grilla habitual por completo: la comprobación de ocupación
  del par ahora consulta directamente el conjunto de turnos reservados en el
  rango (`loadBookedTimes`), no la lista de slots libres habituales. Efecto
  colateral corregido de paso: la implementación anterior negaba el doble
  turno por completo si la disponibilidad habitual no tenía nada libre ese
  día (porque dependía de `getSlots` para todo lo demás); ahora el bloque
  extra se sigue ofreciendo aun en una jornada completamente reservada, que
  es exactamente el escenario para el que la regla existe.
- Se extrajo `loadEligibleWorkingHoursByDay` (los bloques de horario por día
  elegible, descontando feriados y ausencias), compartido ahora entre
  `loadGridByDay` (la grilla habitual) y el nuevo `loadEdgeExtraSlotsByDay`
  (el par de borde), en lugar de que cada uno repitiera el mismo recorrido de
  días.
- Se extrajo también `groupByDayExcluding`, reemplazando la agrupación por
  día que antes vivía inline y se repetía entre la rama sin modo configurado
  y la rama `WITHIN_SCHEDULE`.

## Decisiones y por qué

**El par de borde no se valida contra la lista de slots libres habituales,
sino contra el conjunto crudo de turnos ocupados.** Es la consecuencia
obligada de sacar el cálculo de la grilla habitual: un instante anterior a
las 09:00 nunca aparece en `getSlots` (ni libre ni ocupado, directamente no
existe en esa grilla), de modo que la única fuente que puede responder "¿ya
está tomado?" es la misma que ya usa `getSlots` para restar lo ocupado de la
grilla —`loadBookedTimes`— pero consultada por su propio conjunto en lugar de
por diferencia. Esto es también lo que corrige, sin buscarlo, el efecto
colateral de una jornada completamente reservada: `bookedTimes` puede
contener todos los turnos habituales del día y aun así no mencionar el par de
borde, que ninguna reserva ordinaria puede ocupar.

**Se descartó seguir anclando el modo al primer/último turno *libre* que
todavía distinga "tomado" de "fuera de grilla", el criterio que TASK-37 ya
había fijado.** Ese criterio —no correrse al siguiente turno libre si el
literal ya está tomado— sigue siendo correcto y se conserva sin cambios; lo
que cambió es *dónde* está ese turno literal: antes, un instante dentro de la
grilla habitual; ahora, el par de borde recién calculado. La revalidación
"¿está tomado?" es la misma idea aplicada al lugar correcto.

**El espaciado entre los dos turnos del par sigue siendo la cadencia, no la
duración de la consulta.** Mismo razonamiento que TASK-114 ya dejó fijado
para `WITHIN_SCHEDULE` y para el emparejamiento general: la fuente de verdad
pide "dos turnos consecutivos de su agenda", y en una agenda con cadencia
configurada el turno siguiente empieza una cadencia después, no una duración
después. Cuando no hay cadencia configurada (`step === duration`), el
resultado coincide exactamente con la lectura literal del documento de
requisitos: el bloque extra mide dos veces la duración de la consulta.

**Se extrajo `loadEligibleWorkingHoursByDay` en lugar de duplicar el filtro
de feriados/ausencias dentro del nuevo método.** El cálculo de qué días son
elegibles —ni feriado ni cubiertos por una ausencia— ya vivía dentro de
`loadGridByDay`, mezclado con la generación de la grilla completa de
instantes. El nuevo método de borde necesita exactamente esa misma noción de
"día elegible" pero no la grilla completa, así que separarla evita que los
dos métodos puedan llegar a disentir sobre qué día cuenta como elegible —el
mismo criterio de definición única que ya rige el resto del repositorio— y,
como beneficio colateral, el cálculo de borde deja de generar la grilla
completa de instantes que no necesita.

## Alternativas descartadas

- **Seguir generando la grilla completa del día y tomar sus dos primeros o
  dos últimos instantes, pero desplazando el rango de `loadWorkingHoursByDay`
  hacia atrás o hacia adelante antes de generarla**: descartada por
  requerir sintetizar un bloque de horario de trabajo ficticio (un
  `startTime`/`endTime` que no proviene de ninguna fila real) sólo para
  reutilizar el recorrido existente, a cambio de un método nuevo más simple
  que calcula directamente los dos instantes que necesita sin generar el
  resto de la grilla.
- **Seguir validando el par de borde contra `getSlots`, restándole primero el
  rango extra a mano**: descartada porque `getSlots` seguiría dependiendo de
  la disponibilidad habitual para decidir si el día "tiene algo que ofrecer",
  reproduciendo el mismo defecto de fondo en un lugar distinto.

## Entidades / puertos / adaptadores tocados

- `src/availability/availability.service.ts` (modificado): `getNewPatientSlots`
  reestructurado; métodos privados nuevos `loadEligibleWorkingHoursByDay` y
  `loadEdgeExtraSlotsByDay`; método privado nuevo `groupByDayExcluding`;
  `loadGridByDay` reescrito para consumir `loadEligibleWorkingHoursByDay` en
  lugar de repetir el filtro de feriados/ausencias.

No se tocó el esquema de Prisma ni ningún puerto/adaptador: la corrección es
enteramente de cálculo dentro del servicio de disponibilidad.

## Tests y qué validan

- `src/availability/availability.service.spec.ts` (modificado): los dos casos
  de `FIRST_SLOT_OF_DAY`/`LAST_SLOT_OF_DAY` que fijaban el comportamiento
  anterior (ofrecer las 09:00/10:00 de la propia grilla) se reescribieron
  para exigir el par de borde (08:00/11:00 sobre un horario 09:00-11:00,
  sesiones de 30min); se agregó un caso nuevo que prueba que el bloque extra
  se sigue ofreciendo con la grilla habitual completamente reservada, uno que
  prueba que el bloque extra deja de ofrecerse cuando el par mismo ya está
  tomado (reemplazando el caso anterior, que probaba la condición equivocada:
  el primer turno *habitual* tomado, no el par de borde), y se actualizó el
  caso con cadencia configurada (duración 45, cadencia 60) al nuevo instante
  esperado (07:15, no 09:00).
- `test/appointment-engine-integration.e2e-spec.ts` (modificado): los dos
  casos que reservan un doble turno bajo `FIRST_SLOT_OF_DAY`/`LAST_SLOT_OF_DAY`
  contra Postgres real se actualizaron a los nuevos instantes esperados
  (08:00 y 12:00 sobre un horario 09:00-12:00).
- `test/appointments-booking.e2e-spec.ts` (modificado): el caso que compara
  la respuesta de `GET /profesionales/:id/disponibilidad?newPatient=true`
  contra lo que la reserva termina aceptando se actualizó al nuevo instante
  (08:00); el caso de rechazo por instante no ofrecido no cambió de
  aserción, sólo de comentario, porque 10:00 seguía sin ser el instante que
  el modo ofrece bajo ninguna de las dos versiones.
- Ejecución: suite unitaria completa en verde (78 conjuntos, 779 pruebas) y
  suite end-to-end completa en verde (51 conjuntos, 557 pruebas), ambas
  contra la instancia local de PostgreSQL con `--runInBand`; `tsc --noEmit` y
  `eslint` sin hallazgos sobre los archivos modificados. Los datos usados en
  las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva. La corrección no introduce un flujo que la tesis no describa
ya: cambia dónde cae el bloque extra dentro del mismo diagrama de los tres
modos de franja extra ya pendiente (`figuras_pendientes.md`, entrada 19).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-121-first-last-slot-extra-block`,
  creada desde `origin/main`. Commit `6311373`. Pusheada a `origin`.
- Ticket: TASK-121 (Jira), "[CORRECCIÓN] TASK-37 – FIRST_SLOT_OF_DAY/
  LAST_SLOT_OF_DAY reutilizan slots existentes en vez de agregar tiempo
  extra". Misma convención de bitácora dedicada para tareas puntuales dentro
  de la fase del ticket original que TASK-79/TASK-81/TASK-86/TASK-94/
  TASK-95/TASK-96/TASK-100/TASK-108/TASK-110/TASK-113/TASK-114/TASK-116/
  TASK-117/TASK-123/TASK-115 ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]],
  [[FASE-3_PROMPT-15]], [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]],
  [[FASE-3_PROMPT-18]], [[FASE-3_PROMPT-19]], [[FASE-3_PROMPT-23]],
  [[FASE-3_PROMPT-24]], [[FASE-3_PROMPT-25]], [[FASE-3_PROMPT-26]],
  [[FASE-3_PROMPT-27]], [[FASE-3_PROMPT-28]], [[FASE-3_PROMPT-29]],
  [[FASE-3_PROMPT-30]]). Misma auditoría de origen que TASK-113, TASK-114 y
  TASK-116 (2026-08-14, agente "Audit waitlist/reassignment/holidays vs
  SRS").
