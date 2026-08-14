# Fase 3 — Motor de Turnos (backend) — `rankCandidates` reconsultaba por cada candidato en vez de usar el batching de `refreshTypes` (TASK-108, corrección a TASK-40)

## Contexto

Una auditoría multi-agente de `psique-back` en `main` (ángulo reuse/
simplificación/eficiencia, 2026-08-12) encontró que
`WaitlistReassignmentService.rankCandidates`
(`src/waitlist/waitlist-reassignment.service.ts`) recorría el `for` sobre
las entradas de la lista de espera y, dentro del loop, llamaba
`this.patientProfessionals.find(patientId, professionalId)` una vez por
candidato. `find` a su vez llama
`this.inactivity.refreshTypes([link])` con un arreglo de un solo elemento
— pese a que `PatientInactivityService.refreshTypes`
(`src/patients/patient-inactivity.service.ts`) está pensado para recibir
muchos links de una sola vez y aplicar la regla de inactividad (y su
`updateMany` de persistencia) sobre el conjunto completo.

El costo real: `rankCandidates` corre en cada llamada a
`advanceWaitlist` — es decir, en la cancelación original de un turno y de
nuevo cada vez que un candidato rechaza una oferta o esta expira
(`WaitlistOfferTimeoutCron`) — así que una lista de espera de N pacientes
costaba del orden de 2-3×N round trips secuenciales a la base de datos
por cada paso del recorrido (uno por `patientProfessional.findFirst`
dentro de `find`, más el `updateMany`/lectura que `refreshTypes` hace
cuando corresponde demover el tipo). El propio archivo, unas líneas más
abajo (`advanceWaitlist`, `Promise.all` sobre `rankCandidates` y
`triedWaitlistEntryIds`), ya usa batching donde puede — el loop dentro de
`rankCandidates` era la única secuencia de N round trips independientes
del módulo.

## Qué se implementó

- `PatientProfessionalsService` gana un método nuevo,
  `findManyByProfessional(professionalId, patientIds)`
  (`src/patients/patient-professionals.service.ts`): carga en una sola
  consulta (`patientProfessional.findMany` con `patientId: { in:
  patientIds }`) el vínculo de todos los candidatos con ese profesional, y
  llama `refreshTypes` una única vez sobre el conjunto completo, en vez de
  una vez por vínculo. Devuelve un `Map<patientId, link>` — un paciente
  sin vínculo todavía simplemente está ausente del mapa.
- `rankCandidates` reemplaza el loop `for` con `await
  this.patientProfessionals.findManyByProfessional(professionalId,
  entries.map(e => e.patientId))` y construye la lista rankeada leyendo
  cada entrada del mapa (`link?.priority ?? null`, `link?.type ??
  PatientType.NEW`) en vez de un `try/catch` por candidato.

## Decisiones y por qué

**`findManyByProfessional` no llama `patients.assertOwned` por cada
paciente, a diferencia de `find`.** No hace falta: cada `patientId` que
llega a `rankCandidates` proviene de una `WaitlistEntry`, que ya tiene una
clave foránea compuesta real hacia `Patient` (regla 4 del esquema,
`CLAUDE.md`) resuelta por la propia extensión de tenant-scoping al leer
`waitlistEntry.findMany` — el chequeo de pertenencia al tenant que `find`
hace por si mismo para un llamador arbitrario ya está garantizado aquí
por la fila que originó el `patientId`, así que repetirlo sería un round
trip redundante, no una validación adicional.

**Un paciente sin vínculo queda simplemente ausente del mapa, en vez de
lanzar y capturar una excepción por candidato.** El comportamiento
original usaba el `NotFoundException` de `find` (vía `loadLinkOrThrow`)
como señal de "sin vínculo todavía", capturada en el propio
`rankCandidates` para asignar la prioridad más baja sin abortar el
ranking completo — una tabla de control por control de flujo del lenguaje
que además obligaba a una llamada por candidato para poder fallar
individualmente. Con la carga batcheada no hay ninguna excepción que
capturar: la ausencia de una clave en el `Map` devuelto ya es la misma
señal, resuelta con el operador `??` en el sitio de lectura.

**El resultado del ranking no cambia.** La comparación (prioridad, orden,
fecha de creación) sigue siendo exactamente la misma función; sólo cambió
de dónde vienen `priority`/`type` para cada candidato — de N llamadas a
`find` a una lectura de un `Map` poblado por una sola consulta batcheada
— tal como pide el criterio de aceptación del ticket.

## Alternativas descartadas

- **Mantener `find` para casos individuales y agregar el batching sólo
  dentro de `rankCandidates`, sin tocar `PatientProfessionalsService`**:
  descartada porque el batching real (la consulta `findMany` y el
  `refreshTypes` sobre el conjunto) es responsabilidad de
  `PatientProfessionalsService` — es el único lugar que conoce el
  `select` correcto del vínculo y el único que ya orquesta
  `PatientInactivityService`. Duplicar esa lógica dentro de
  `WaitlistReassignmentService` habría creado un segundo lugar con acceso
  directo a `patientProfessional.findMany`, algo que el resto del módulo
  de pacientes evita deliberadamente (todo acceso a `PatientProfessional`
  pasa por `PatientProfessionalsService`).
- **Traducir el `Map` devuelto a una lista posicional alineada con
  `entries`**: descartada por ser estrictamente más frágil — un `Map`
  indexado por `patientId` no depende de que ambos arreglos conserven el
  mismo orden, mientras que una lista posicional sí.

## Entidades / puertos / adaptadores tocados

- `PatientProfessionalsService.findManyByProfessional` (nuevo método,
  `src/patients/patient-professionals.service.ts`).
- `WaitlistReassignmentService.rankCandidates`
  (`src/waitlist/waitlist-reassignment.service.ts`): reemplaza el loop
  `for` con la llamada batcheada.

## Tests agregados o modificados

- `src/patients/patient-professionals.service.spec.ts` (primer spec
  unitario dedicado a este servicio — el resto de
  `PatientProfessionalsService` sigue cubierto sólo por e2e, como antes
  de esta tarea): tres casos para `findManyByProfessional` — arreglo
  vacío no consulta Prisma; N `patientIds` disparan exactamente una
  llamada a `patientProfessional.findMany` y exactamente una a
  `refreshTypes` sobre el conjunto completo, sin importar N; un paciente
  sin vínculo queda ausente del `Map` en vez de lanzar.
- `src/waitlist/waitlist-reassignment.service.spec.ts`: los dos casos
  existentes de ranking (prioridad más alta primero; candidato sin
  vínculo tratado como prioridad más baja sin abortar) se adaptaron para
  mockear `findManyByProfessional` devolviendo un `Map` en vez de `find`
  devolviendo o rechazando por candidato. Se agregó un caso nuevo que
  arma una lista de 8 candidatos y verifica que
  `findManyByProfessional` se llama exactamente una vez, con el arreglo
  completo de `patientId`s de todas las entradas — la prueba directa del
  criterio de aceptación del ticket ("un número de queries independiente
  de N").

Suite completa verde tras el cambio: 39 suites unitarias / 434 pruebas;
37 suites e2e / 439 pruebas (`--runInBand`). Lint y verificación de tipos
(`tsc --noEmit`) sin errores. Docker (`back-db-1`) ya estaba corriendo.

## Figuras pendientes

Ninguna nueva — es una corrección de eficiencia sobre un algoritmo ya
documentado (P3.7, ranking de lista de espera) sin cambio de esquema ni
de comportamiento observable.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-108-rank-candidates-batch-refresh-types`
  (creada desde `origin/main` fresco, tras el merge de TASK-106). Pusheada
  a `origin`, PR abierta, no fusionada aún.
- Ticket: TASK-108 ("[MEJORA] rankCandidates re-consulta por cada
  candidato en vez de usar el batching de refreshTypes"), corrección a
  TASK-40 (P3.7, implementó el algoritmo de ranking). Misma convención de
  bitácora dedicada para correcciones pequeñas dentro de la fase del
  ticket original que TASK-79/TASK-81/TASK-86/TASK-94/TASK-95/TASK-96/
  TASK-100 ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]],
  [[FASE-3_PROMPT-15]], [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]],
  [[FASE-3_PROMPT-18]], [[FASE-3_PROMPT-19]]).
