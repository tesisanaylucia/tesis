# Fase 2 — Pacientes (backend) — Paginación de `GET /pacientes` (TASK-181, SEC 10)

## Contexto

TASK-181 es una tarea generada automáticamente por la misma auditoría de
código contra las fuentes de verdad (SRS), fechada 2026-08-26, que ya
produjo SEC 7 (TASK-178, [[FASE-6_PROMPT-6]]) y SEC 8 (TASK-179,
[[FASE-8_PROMPT-10]]), marcada "requiere validación humana" antes de
implementarse. El hallazgo (SEC 10): `FindPatientsQueryDto` sólo declaraba
`dni`/`includeInactive`, y `PatientsService.findAll` no aplicaba ningún
`take`/`skip` — devolvía el padrón completo del inquilino, con sus
relaciones anidadas, en cada llamada sin filtro por DNI. La propia
auditoría señaló el indicio de que se trataba de una inconsistencia y no de
una decisión deliberada: `ListAppointmentsQueryDto` (P3.9, TASK-79) ya tenía
`limit`/`offset` desde antes, de modo que el padrón de pacientes era el
único de los dos listados generales del sistema sin cota. El hallazgo
señaló además que el padrón de pacientes, a diferencia de la agenda de
turnos, sólo crece —la baja es lógica, nunca hay una eliminación física—,
lo que lo vuelve el listado con más posibilidades de convertirse en un
problema real a medida que el inquilino envejece. La solución que el propio
hallazgo proponía —agregar `limit`/`offset` a `FindPatientsQueryDto`,
espejando el DTO de turnos— fue, sin cambios, la que se implementó.

## Qué se implementó

Se agregaron a `FindPatientsQueryDto` los campos `limit`/`offset`, con
exactamente los mismos validadores y el mismo criterio que
`ListAppointmentsQueryDto` ya aplica: `limit` es un entero entre uno y un
tope superior, `offset` un entero no negativo, y ninguno de los dos tiene
valor por defecto a nivel del DTO —el valor por defecto se resuelve en el
servicio, no en la validación de entrada, el mismo criterio que ya regía
para el rango de fechas de disponibilidad y para el propio `limit`/`offset`
de turnos—. El tope superior y el valor por defecto se declararon como dos
constantes nuevas, `DEFAULT_PATIENTS_PAGE_SIZE`/`MAX_PATIENTS_PAGE_SIZE`
(cincuenta y doscientos), con el mismo valor y el mismo carácter de límite
antiabuso —no una regla de negocio— que ya tienen sus tres análogas
(`DEFAULT_APPOINTMENTS_PAGE_SIZE`/`MAX_APPOINTMENTS_PAGE_SIZE`,
`DEFAULT_AUDIT_LOG_PAGE_SIZE`/`MAX_AUDIT_LOG_PAGE_SIZE`,
`DEFAULT_NOTIFICATIONS_PAGE_SIZE`/`MAX_NOTIFICATIONS_PAGE_SIZE`).

`PatientsService.findAll` pasó de una única consulta `findMany` a dos
consultas en paralelo —una `findMany` con `skip`/`take` y una `count`,
ambas sobre la misma condición `where`—, el mismo patrón que
`AppointmentsService.findAll`, `AuditService.findByPatient` y
`InAppNotificationsService` ya aplican cada uno para su propio listado
paginado. A diferencia de esos tres, el método siguió devolviendo entidades
completas (`PatientWithRelations`) en vez del tipo de respuesta expuesto
por la API: se introdujo un sobre interno, `PaginatedPatients`
(`items`/`total`/`limit`/`offset`, con `items` como entidades), distinto
del sobre de la API, `PaginatedPatientsResponse` (mismo cuatro campos, con
`items` ya mapeado a `PatientResponse`). `PatientsController.findAll` es
quien hace ese mapeo antes de responder. La razón de la distinción es un
segundo llamador que `AppointmentsService.findAll` y las otras dos
análogas no tienen: `PatientTools.findOrCreatePatient` —la identificación
por DNI que el chatbot ejecuta antes de cada reserva, PAC-2/TASK-139—
también invoca `PatientsService.findAll`, y necesita el `deletedAt`/`id` de
la entidad cruda, no la forma que expone la API; devolver ya la forma de
presentación desde el servicio, como hacen las otras tres, habría roto ese
segundo llamador.

## Decisiones y por qué

**Se espejó el DTO de turnos al detalle, sin introducir un patrón propio.**
El propio hallazgo de la auditoría señalaba la inconsistencia con
`ListAppointmentsQueryDto` como el indicio de que faltaba paginación, no
como una sugerencia de forma aproximada: se reutilizaron los mismos
validadores, el mismo rango, y el mismo criterio de "el valor por defecto
lo resuelve el servicio" que ya regía en turnos y en la fecha de
disponibilidad, en lugar de inventar una variante distinta para pacientes.

**El servicio conserva las entidades completas en su propio sobre, en vez de
adoptar el patrón de mapear a la respuesta de la API dentro del servicio.**
Las tres paginaciones ya existentes (turnos, auditoría de un paciente,
notificaciones in-app) mapean a la respuesta de la API dentro del propio
servicio, porque ninguna tiene un segundo llamador interno que necesite la
entidad cruda. `PatientsService.findAll` sí lo tiene —la identificación del
chatbot por DNI—, así que replicar ese patrón sin más habría forzado a
reescribir ese llamador para acceder a un campo (`deletedAt`) que la
respuesta de la API deliberadamente no expone tal cual. Se prefirió, en
cambio, mantener la propia convención del módulo —el servicio siempre
devuelve entidades, la capa de presentación las traduce— y expresar la sola
paginación como un sobre nuevo de la misma forma (`items`/`total`/`limit`/
`offset`) que las otras tres, sin duplicar la entidad completa del listado
sin paginar.

**Se agregó `id` como desempate final del ordenamiento, más allá de lo que
pedía el hallazgo.** Al escribir la prueba de extremo a extremo de esta
misma tarea —dos páginas consecutivas de `limit=2` sin superposición—, la
prueba falló de forma intermitente al correrse después del resto de la
batería, no de forma aislada. La causa no era la paginación en sí sino el
ordenamiento que ya existía desde antes de esta tarea,
`orderBy: [{ lastName: 'asc' }, { firstName: 'asc' }]`: sin una columna
única como desempate final, dos pacientes empatados en nombre y apellido
—una situación común, varios de los propios datos de prueba de la batería
comparten el nombre por defecto— no tienen ningún orden garantizado entre
dos llamadas separadas, aunque no medie ninguna escritura entre ambas. Eso
es inocuo en un listado sin paginar, donde el orden entre empatados no
cambia qué filas se devuelven, pero deja de serlo en cuanto se agrega
`skip`/`take`: una fila empatada puede caer de un lado del límite de página
en una llamada y del otro lado en la siguiente, de modo que la página
siguiente repite una fila ya vista, o la anterior omite una que nunca
apareció en ninguna. Se agregó `id` como tercer criterio de ordenamiento,
lo mínimo que garantiza un orden determinístico sin alterar el criterio de
negocio (apellido y luego nombre) que el propio listado ya usaba. No es un
defecto que el hallazgo SEC 10 nombrara —la paginación en sí es lo único
que pedía— sino uno que la propia paginación, al agregarse sobre un
ordenamiento preexistente sin desempate único, podía introducir en
producción sin que ningún dato se perdiera realmente, sólo se leyera en un
orden inestable entre páginas.

## Alternativas descartadas

- **Mapear a `PatientResponse` dentro de `PatientsService.findAll`**, como
  hacen `AppointmentsService.findAll`/`AuditService.findByPatient`/
  `InAppNotificationsService`: descartado porque habría roto a
  `PatientTools.findOrCreatePatient`, que depende de recibir la entidad
  cruda de ese mismo método para decidir si el paciente encontrado está
  dado de baja.
- **Dejar el ordenamiento sin desempate**, confiando en que la paginación
  rara vez cae exactamente en un empate: descartada porque el propio
  suite de pruebas de este cambio demostró que la situación no es
  hipotética, y perder o repetir una fila al pasar de página es
  exactamente el defecto de integridad de listado que esta misma tarea
  existe para evitar.
- **Extender el mismo desempate a `AppointmentsService.findAll`
  (`scheduledAt`) y a `AuditService.findByPatient`
  (`timestamp`)**, que en principio comparten la misma vulnerabilidad
  teórica: quedó fuera de alcance de TASK-181, que nombra únicamente
  `GET /pacientes`, y no se tocó ningún archivo de esos dos módulos. Se
  deja señalado aquí, sin ticket propio todavía, como un hallazgo
  colateral de esta tarea para una futura auditoría o corrección.

## Entidades / puertos / adaptadores tocados

- `src/patients/dto/find-patients-query.dto.ts` (modificado): `limit`/
  `offset` nuevos, mismos validadores que `ListAppointmentsQueryDto`.
- `src/patients/patients.constants.ts` (modificado):
  `DEFAULT_PATIENTS_PAGE_SIZE`/`MAX_PATIENTS_PAGE_SIZE` nuevas (50/200).
- `src/patients/patient.presenter.ts` (modificado): `PaginatedPatients`
  (entidades, uso interno del servicio) y `PaginatedPatientsResponse`
  (`PatientResponse[]`, contrato de la API) nuevas.
- `src/patients/patients.service.ts` (modificado): `findAll` pagina con
  `skip`/`take` + `count` en paralelo, agrega `id` como desempate final del
  `orderBy`, y devuelve `PaginatedPatients` en lugar de un arreglo sin cota.
- `src/patients/patients.controller.ts` (modificado): `findAll` mapea el
  sobre del servicio a `PaginatedPatientsResponse` antes de responder.
- `src/chatbot/tools/patient.tools.ts` (modificado): la identificación por
  DNI ajusta su desestructuración a `{ items: [existing] }`, sin cambio de
  comportamiento — sigue leyendo, como antes, a lo sumo un elemento.

## Tests y qué validan

- `src/patients/patients.service.spec.ts` (ampliado, 2 pruebas nuevas):
  tamaño de página y offset por defecto cuando no se los da, y honra de un
  `limit`/`offset` propio con el total tomado de `count` — mismo par de
  pruebas que ya cubre `AppointmentsService.findAll`.
- `test/patients-abmc.e2e-spec.ts` (ampliado, 2 pruebas nuevas): paginación
  real contra Postgres con `limit`/`offset` propios y sin superposición
  entre dos páginas consecutivas (la prueba que expuso la falta de
  desempate en el ordenamiento), y rechazo con 400 de un `limit` por
  encima del tope. Ajustadas además todas las aserciones existentes que
  leían la respuesta de `GET /pacientes` como un arreglo directo, ahora
  bajo `body.items`.
- `test/patients-import.e2e-spec.ts` y `test/patient-notes.e2e-spec.ts`
  (ajustados, sin pruebas nuevas): mismo ajuste de `body.items` en sus
  propios usos de `GET /pacientes`.
- `src/chatbot/tools/patient.tools.spec.ts` (ajustado, sin pruebas nuevas):
  los catorce simulacros de `findAll` de este archivo se actualizaron a
  `{ items: [...] }`, sin cambiar lo que cada prueba verifica.
- Ejecución: suite unitaria en verde (143 suites / 1619 pruebas); suite
  end-to-end completa en verde (53 suites / 641 pruebas, `--runInBand`);
  compilación (`tsc --noEmit`) y análisis estático (`eslint`) sin errores
  sobre todos los archivos tocados. Todos los datos usados en las pruebas
  son ficticios.

## Figuras pendientes

Ninguna nueva. La corrección agrega paginación a un punto de acceso ya
existente, sin introducir un flujo ni una entidad que el material gráfico
pendiente de tareas anteriores no cubra ya.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-181-patients-pagination` (creada a
  partir de `main`). Commit `34e89be`, empujado a `origin` de
  `psique-back`.
- Ticket: TASK-181 ("GET /pacientes no pagina"), hallazgo SEC 10 de la
  auditoría de código vs. SRS del 2026-08-26, misma auditoría que SEC 7
  ([[FASE-6_PROMPT-6]]) y SEC 8 ([[FASE-8_PROMPT-10]]). Depende de
  `ListAppointmentsQueryDto` (P3.9, TASK-79), cuyo patrón de paginación
  reutiliza sin modificarlo.
