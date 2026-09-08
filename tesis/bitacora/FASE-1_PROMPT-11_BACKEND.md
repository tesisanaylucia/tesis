# Fase 1 — Profesionales (backend) — el ancla de los recursos anidados no respetaba la baja lógica (TASK-132, corrección a PROF-5)

## Contexto

Una auditoría automatizada de código contra las fuentes de verdad (SRS),
corrida el 20 de agosto de 2026, generó el hallazgo PROF-5: el método
`assertOwned` de `ProfessionalsService` —el punto de anclaje que usan las
matrículas, los horarios de atención, las ausencias, las obras sociales
aceptadas, el cálculo de disponibilidad y la reserva de turnos para
confirmar que un profesional pertenece al inquilino del llamante— no
filtraba la marca de baja lógica (`deletedAt`) descripta en 4.1 y 4.2. Solo
el listado de profesionales activos y el inicio de sesión respetaban esa
marca en todo el repositorio. La consecuencia concreta que el hallazgo
señaló: un administrador da de baja a un profesional, y tanto la consulta
de disponibilidad como la reserva de un turno nuevo siguen funcionando con
normalidad sobre ese profesional ya desvinculado. Por tratarse de un
hallazgo generado automáticamente, la tarea exigía validación humana antes
de intervenir, la cual se realizó como parte de esta misma implementación:
se confirmó el hallazgo contra el código vigente, se comprobó que seguía
aplicando, y se determinó su alcance real antes de corregir.

## Qué se implementó

Se agregó un segundo método a `ProfessionalsService`, `assertOwnedActive`,
que repite la misma consulta de `assertOwned` —existencia dentro del
inquilino del llamante, sin cargar relaciones— pero agregando el filtro
`deletedAt: null`, exactamente como ya hace el listado de profesionales
activos. Todos los puntos de anclaje de otros módulos que dependían de
`assertOwned` para confirmar que un profesional podía recibir un nuevo
compromiso de agenda pasaron a usar la variante activa: el cálculo de
disponibilidad y su variante para pacientes nuevos, la reserva de un turno,
la reorganización de la agenda, la verificación de elegibilidad de un
paciente nuevo, la incorporación a una lista de espera, la vinculación de
un paciente a un profesional, y las cuatro operaciones de matrículas,
horarios, ausencias y obras sociales aceptadas que el propio hallazgo
nombra explícitamente. El método original, `assertOwned`, se conservó sin
cambios y siguió siendo el que usan tanto los propios métodos de ciclo de
vida del profesional dentro de este módulo (edición, baja, configuración y,
en particular, la reactivación) como las lecturas de datos ya existentes de
otros módulos —el listado de turnos de un profesional, el listado y la
reordenación de una lista de espera, el buzón administrativo de
notificaciones—, que siguen siendo alcanzables para un profesional dado de
baja de la misma manera en que ya lo es su propia ficha completa.

## Decisiones y por qué

**No se modificó `assertOwned` en el lugar, tal como sugería el propio
hallazgo automatizado.** La solución propuesta por la auditoría era agregar
el filtro de baja directamente al `where` de `assertOwned`. Se descartó
porque ese método es también el que usa `reactivate`, el punto de acceso
que anula la baja de un profesional descripto más arriba en esta sección:
filtrar por `deletedAt: null` dentro de `assertOwned` habría hecho que la
reactivación de un profesional ya no pudiera encontrar al profesional que
necesita reactivar, precisamente el caso para el que existe. Se optó en
cambio por separar dos responsabilidades que el método único confundía: la
identidad y pertenencia al inquilino, que debe seguir siendo verificable
sin importar el estado de actividad, y la aptitud para recibir un nuevo
compromiso de agenda, que sí debe exigir que el profesional siga activo.

**La distinción entre lectura histórica y compromiso nuevo se aplicó de
forma pareja, no solo a los dos casos que ilustra el hallazgo.** Aunque el
escenario descripto en PROF-5 nombra solo la disponibilidad y la reserva de
turnos, el propio texto del hallazgo enumera además las matrículas, los
horarios, las ausencias y las obras sociales como recursos anclados por el
mismo método sin filtrar. Se decidió aplicar la variante activa a la
totalidad de las operaciones de esos cuatro módulos anidados —no solo a
sus escrituras— porque, a diferencia de la reserva o el listado de turnos,
ninguno de ellos distingue hoy entre una lectura administrativa y una
operación de escritura: los cuatro comparten un mismo anclaje en cada
método, activo o no. Distinguir de todos modos habría introducido una
asimetría nueva dentro de cada servicio sin una razón de negocio que la
sostenga. El listado de turnos de un profesional y el listado o
reordenación de su lista de espera, en cambio, sí se dejaron sobre el
método sin filtrar: son lecturas de un historial ya comprometido —turnos ya
reservados, un orden de espera ya existente—, no la creación de un
compromiso nuevo, y ocultarlas tras la baja habría impedido a un
administrador revisar el historial de un profesional después de
desvincularlo.

## Alternativas descartadas

- **Filtrar `deletedAt` directamente dentro de `assertOwned`**: descartada
  por el conflicto con `reactivate` señalado arriba, que rompería la
  reactivación de un profesional dado de baja.
- **Limitar la corrección a los dos casos literales del hallazgo
  (disponibilidad y reserva de turnos)**: descartada porque el propio
  hallazgo nombra explícitamente el mismo defecto en las matrículas, los
  horarios, las ausencias y las obras sociales, dejando esos cuatro
  recursos con el mismo comportamiento incorrecto si no se corregían a la
  vez.
- **Distinguir lectura de escritura dentro de cada uno de los cuatro
  servicios de recursos anidados (matrículas, horarios, ausencias, obras
  sociales)**: descartada por introducir una asimetría nueva y no
  justificada dentro de servicios que hoy anclan todas sus operaciones —
  lectura y escritura por igual— en un único método.

## Entidades / puertos / adaptadores tocados

- `src/professionals/professionals.service.ts`: nuevo método
  `assertOwnedActive`, junto al `assertOwned` existente, ambos documentados
  en el código con la distinción entre uno y otro.
- `src/availability/availability.service.ts`,
  `src/appointments/appointments.service.ts`,
  `src/waitlist/waitlist.service.ts`,
  `src/patients/patient-professionals.service.ts`,
  `src/professionals/licenses.service.ts`,
  `src/professionals/working-hours.service.ts`,
  `src/professionals/absences.service.ts`,
  `src/professionals/professional-health-insurers.service.ts`: cambiado el
  método de anclaje usado, de `assertOwned` a `assertOwnedActive`, en cada
  punto que representa un compromiso nuevo contra el profesional.
- Ningún cambio de esquema: la marca `deletedAt` ya existía desde la baja
  lógica original descripta en 4.1/4.2; esta corrección solo cambia qué
  consulta la respeta.

## Tests y qué validan

- Pruebas unitarias nuevas sobre `ProfessionalsService.assertOwned` /
  `assertOwnedActive`: fijan la forma exacta de la cláusula `where` de cada
  uno, y confirman que `assertOwned` sigue encontrando un profesional dado
  de baja (la propiedad de la que depende `reactivate`) mientras que
  `assertOwnedActive` responde 404 tanto para uno inexistente como para uno
  desactivado.
- Pruebas de extremo a extremo agregadas junto a la prueba de aislamiento
  por inquilino ya existente de cada ruta afectada —el mismo patrón, con un
  profesional desactivado en lugar de uno de otro inquilino—: disponibilidad,
  reserva de turnos, matrículas, horarios, ausencias, obras sociales
  aceptadas, incorporación a lista de espera y vinculación de un paciente a
  un profesional responden 404 sobre un profesional dado de baja; una de
  ellas, en el módulo de profesionales, prueba además que la reactivación
  restaura el acceso a la ruta que había quedado bloqueada.
- Ejecución: suite unitaria y de extremo a extremo en verde (132 conjuntos,
  1368 pruebas, `--runInBand`), análisis estático sin advertencias. Datos
  ficticios, sin contenido clínico ni datos personales reales.

## Figuras pendientes

Ninguna figura nueva; la corrección no introduce un flujo o diagrama
distinto de los ya registrados para el módulo.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-132-assert-owned-deleted-at` (creada
  a partir de `origin/main`). Commit `f3ed48c`.
- Ticket: TASK-132 ("Un profesional dado de baja conserva agenda y reservas
  activas"), hallazgo PROF-5 de la auditoría automatizada del 20 de agosto
  de 2026 sobre `src/professionals/professionals.service.ts:107-116`,
  prioridad alta, validado y corregido en la misma tarea.
