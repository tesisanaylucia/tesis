# Fase 1 — Profesionales (backend) — Restricción de unicidad para matrículas (TASK-92, corrección a TASK-21/22)

## Contexto

Una auditoría de la base de datos viva de `psique-back` (2026-08-12) detectó
que `License` (TASK-21/22, P1.1/P1.2) no tenía ninguna restricción de
unicidad sobre la combinación `(professionalId, type, number)`, ni a nivel de
esquema Prisma ni en `LicensesService.create`/`.update`. El único control
existente hasta entonces era el tope máximo de tres matrículas por
profesional (`MAX_LICENSES_PER_PROFESSIONAL`), verificado dentro de una
transacción SERIALIZABLE, pero nada impedía cargar la misma matrícula
provincial o nacional dos veces para el mismo profesional: dos filas
distintas que dicen exactamente lo mismo, un dato redundante sin significado
adicional. TASK-85 (P1.1/P1.2, corrección, [[FASE-1_PROMPT-8]]) ya había
endurecido este mismo modelo por el lado del mínimo/tipos requeridos, pero
dejó la unicidad fuera de su alcance explícitamente.

## Qué se implementó

- `@@unique([professionalId, type, number])` en el modelo `License` de
  `prisma/schema.prisma`, en reemplazo del índice simple
  `@@index([professionalId])` preexistente — la restricción compuesta ya
  cubre cualquier búsqueda que el índice simple servía, al empezar por la
  misma columna.
- Migración Prisma correspondiente
  (`20260813170500_license_unique_professional_type_number`), que elimina el
  índice viejo y crea el índice único nuevo.
- `LicensesService.create` y `.update` ahora capturan la violación de
  unicidad de Postgres (código `P2002`, el mismo mecanismo que usan
  `holidays.service.ts`, `patients.service.ts` y
  `patient-professionals.service.ts`) y la traducen a un
  `BadRequestException` (400) legible, en vez de dejar propagar el error 500
  crudo de la constraint.

## Decisiones y por qué

**400, no el 409 que usa el resto del codebase para una violación de
unicidad concurrente.** El patrón establecido en `holidays.service.ts`,
`patients.service.ts` y `patient-professionals.service.ts` traduce `P2002` a
`ConflictException` (409), razonando que es una carrera entre dos escrituras
concurrentes que vale la pena reintentar. TASK-92 pide explícitamente un 400
en sus criterios de aceptación ("crear dos matrículas idénticas ... → falla
con 400"), y la semántica es distinta: no es una carrera entre dos
escrituras que compiten por el mismo recurso todavía inexistente (como una
fecha de feriado o un DNI), sino una entrada rechazada por ser redundante
con datos que ya existen y son perfectamente visibles para quien la envía —
más cerca de un error de validación de payload que de un conflicto de
concurrencia. Se siguió el texto literal del ticket en vez del precedente:
un caso inverso al de TASK-78 ([[FASE-3_PROMPT-9]]), donde el codebase se
apartó deliberadamente de un pedido textual del ticket (403) a favor de la
convención existente (404) porque el ticket solo describía el
comportamiento en prosa; aquí, en cambio, el pedido de 400 es un criterio de
aceptación explícito y comprobable por test, así que se le dio prioridad
sobre el precedente de 409.

**Se reemplazó el índice simple en vez de agregarlo junto al nuevo.** El
índice único compuesto `(professionalId, type, number)` ya sirve cualquier
consulta que solo filtre por `professionalId` (es el prefijo del índice), así
que mantener ambos habría sido redundante sin aportar cobertura extra.

**El manejo de errores se extrajo a un método privado compartido**
(`throwIfDuplicateLicense`) en vez de duplicar el `catch` en `create` y
`update`: ambos métodos necesitaban exactamente la misma traducción de
`P2002`, y el método no lanza nada si el error no coincide, dejando que el
`catch` que lo llama vuelva a lanzar el error original sin modificar — mismo
principio de "no ocultar un error que no se reconoce" que ya seguían los
demás servicios con este patrón.

## Alternativas descartadas

- **Upsert en vez de create+catch:** no aplica aquí, porque el objetivo no es
  "crear o actualizar", sino rechazar directamente el duplicado — un upsert
  silenciosamente sobrescribiría la matrícula existente en vez de informar el
  conflicto al usuario.
- **Validar con un `findFirst` antes del insert (dedup explícito) en vez de
  confiar en la constraint de base:** se descartó por la misma razón que el
  resto del codebase prefiere la constraint + traducción del error: un
  `findFirst` previo no cierra la ventana de carrera entre dos requests
  concurrentes (dos financiamientos exactamente iguales podrían pasar el
  `findFirst` a la vez), mientras que la constraint de Postgres es la única
  fuente de verdad que no puede fallar bajo concurrencia.

## Entidades / puertos / adaptadores tocados

- `License` (Prisma): `@@unique([professionalId, type, number])` reemplaza
  `@@index([professionalId])`.
- Migración nueva: `20260813170500_license_unique_professional_type_number`
  (`DROP INDEX` + `CREATE UNIQUE INDEX`, sin alteración de datos).
- `LicensesService` (`src/professionals/licenses.service.ts`): `create` y
  `update` envueltos en `try/catch` que delega en el nuevo método privado
  `throwIfDuplicateLicense`.

## Tests agregados o modificados

En `test/professionals-abm.e2e-spec.ts` (no existe spec unitario dedicado
para `LicensesService`; el módulo se cubre solo vía e2e desde su creación en
TASK-22):

- Crear dos matrículas idénticas (mismo `type` + `number`) para el mismo
  profesional → 400 en la segunda.
- Actualizar una matrícula para que coincida con otra del mismo profesional
  (`type` + `number` de una hermana) → 400, ejercitando la traducción de
  error en `update`, no solo en `create`.
- La misma combinación `type` + `number` para dos profesionales distintos
  sigue siendo válida (201 en ambos).

Suite completa verde tras el cambio: 38 suites unitarias / 411 pruebas; 36
suites e2e / 428 pruebas (`--runInBand`). Lint y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva — es una corrección de esquema y manejo de errores sobre un
modelo ya documentado (figura de matrículas, si existiera, no cambia de
forma).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-92-license-unique-constraint` (creada
  desde `origin/main` fresco, tras el merge de TASK-91). Pusheada a
  `origin`, no fusionada aún.
- Ticket: TASK-92 ("[CORRECCIÓN] License sin restricción de unicidad
  (professionalId, type, number)"), corrección a TASK-21/22 (P1.1/P1.2,
  [[FASE-1_PROMPT-2]], creó `License`) y relacionada con TASK-85
  ([[FASE-1_PROMPT-8]], validación mínima de matrículas, mismo modelo, lado
  del mínimo en vez del de la unicidad). Misma convención de bitácora
  dedicada para correcciones pequeñas dentro de la fase del ticket original
  que TASK-84 ([[FASE-2_PROMPT-11]]), TASK-79/TASK-81
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]]), TASK-86
  ([[FASE-3_PROMPT-15]]) y TASK-91 ([[FASE-4_PROMPT-7]]).
