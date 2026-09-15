# Fase 1 — Profesionales (backend) — el alta de un profesional con matrículas duplicadas devolvía un 500 sin manejar (TASK-197, corrección a P1.1/P1.3)

## Contexto

Una auditoría automatizada de código contra las fuentes de verdad (anteproyecto
de tesis y SRS), corrida el 28 de agosto de 2026, encontró que el alta de un
profesional (`ProfessionalsService.create`) no traducía la violación de la
restricción de unicidad `(professionalId, type, number)` de `License` cuando
el propio cuerpo de la petición traía dos matrículas repetidas dentro de su
lista de matrículas iniciales. La restricción en sí —descrita en la sección
anterior de esta subsección, TASK-92— ya existía en el esquema desde antes de
esta tarea; lo que faltaba era su traducción en este punto de escritura en
particular. `LicensesService.create`/`update`, los dos puntos de escritura del
alta incremental de una matrícula sobre un profesional ya existente, sí
envuelven esa misma violación desde TASK-92 y la convierten en un `400`
legible. El alta de un profesional con matrículas inline, sin embargo, dejaba
que `Prisma.PrismaClientKnownRequestError` (código `P2002`) se propagara sin
capturar, resultando en un `500` interno ante una entrada simplemente mal
formada, no ante una falla real del sistema.

## Qué se implementó

Se extrajo el chequeo que traducía la violación —hasta entonces un método
privado de `LicensesService`— a una función pura y compartida,
`throwIfDuplicateLicense`, en un archivo nuevo del módulo
(`src/professionals/license-conflict.ts`), y se envolvió la transacción de
`ProfessionalsService.create` con el mismo patrón try/catch que ya usan los
dos métodos de `LicensesService`: si la escritura falla con el código de
violación de unicidad, se traduce a un `BadRequestException` con el mismo
mensaje; cualquier otro error se repropaga sin modificar.

## Decisiones y por qué

**La función se extrajo en vez de duplicarse o de exponerse como método
público de `LicensesService`.** `LicensesService` depende de
`ProfessionalsService` en su propio constructor —lo usa para anclar cada
operación sobre matrículas a un profesional existente y activo del
inquilino del llamador—, así que hacer que `ProfessionalsService` dependiera
a su vez de `LicensesService` para reusar el chequeo habría cerrado un ciclo
de inyección de dependencias entre ambos servicios del mismo módulo. Extraer
el chequeo a una función sin estado, sin dependencias de Nest, resolvió la
reutilización que pedía el propio hallazgo sin introducir esa dependencia
circular.

**Se mantuvo el mismo criterio ya adoptado en TASK-92: `400`, no `409`.** El
mensaje y el código de estado son idénticos a los que ya devuelve el alta
incremental de una matrícula, por tratarse exactamente de la misma
restricción y el mismo razonamiento ya documentado en la subsección anterior
—una entrada redundante con datos ya visibles para quien la envía, no una
concurrencia entre dos escrituras que compiten por un recurso inexistente—,
sin motivo para que dos caminos de alta de la misma entidad respondieran de
forma distinta ante la misma violación.

## Alternativas descartadas

- **Validar la ausencia de duplicados en el propio DTO** (por ejemplo, con un
  validador `class-validator` que recorriera el arreglo `licenses` antes de
  llegar al servicio): descartada por apartarse del arreglo sugerido por el
  propio hallazgo, que pedía específicamente reusar el patrón try/catch ya
  validado en `LicensesService`, y por duplicar en la capa de validación una
  garantía que la base de datos ya impone de forma correcta e ineludible.

## Entidades / puertos / adaptadores tocados

- `src/professionals/license-conflict.ts` (nuevo): función compartida
  `throwIfDuplicateLicense`, sin estado ni dependencias de Nest.
- `src/professionals/licenses.service.ts`: su método privado homónimo se
  eliminó a favor de la función compartida; sin cambio de comportamiento.
- `src/professionals/professionals.service.ts`: `create()` ahora envuelve su
  transacción en try/catch y traduce la violación de unicidad de la misma
  forma que `LicensesService`.
- Ningún cambio de esquema: la restricción ya existía desde TASK-92.

## Tests y qué validan

- Prueba unitaria nueva sobre `ProfessionalsService.create` (mock de Prisma):
  que una `PrismaClientKnownRequestError` con código `P2002` se traduce a
  `BadRequestException` sin llegar a auditar la escritura, que cualquier otro
  error se repropaga sin modificar, y que el alta sin conflicto sigue
  auditando con los mismos campos de siempre.
- Prueba de extremo a extremo nueva en `test/professionals-abm.e2e-spec.ts`,
  junto a la ya existente del tope de tres matrículas por el mismo camino de
  alta inline: crear un profesional con dos matrículas inline de igual tipo y
  número responde `400`, no `500`, y no deja ningún profesional creado en la
  base.
- Ejecución: suite unitaria y de extremo a extremo completas en verde,
  análisis estático sin advertencias. Datos ficticios, sin contenido clínico
  ni datos personales reales.

## Figuras pendientes

Ninguna figura nueva; la corrección no introduce un flujo o diagrama distinto
de los ya registrados para el módulo.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-197-duplicate-license-500-fix` (creada a
  partir de `origin/main`).
- Ticket: TASK-197 ("[BAJO] 500 sin manejar al crear un profesional con
  matrículas duplicadas"), hallazgo de auditoría multi-agente del 28 de
  agosto de 2026 (fuentes: anteproyecto de tesis y SRS), prioridad baja.
