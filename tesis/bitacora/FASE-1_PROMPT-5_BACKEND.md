# Fase 1 — Profesionales (backend) — Filtro de edad, aceptación de pacientes nuevos y modalidad de reasignación (TASK-25)

## Qué se implementó

Se completó la configuración que cada profesional define para sí mismo con los
tres atributos de política de admisión y reasignación que faltaban: el filtro de
edad (el profesional atiende únicamente a personas adultas), la apertura o
cierre de la aceptación de pacientes nuevos, y la modalidad con que se reasigna
un turno liberado por una cancelación (automática o manual). Los tres pasaron a
ser editables a través del endpoint de configuración ya existente,
`PATCH /profesionales/:id/configuracion`.

El hallazgo que determinó el alcance real de la tarea fue que los tres campos ya
existían en el esquema desde P1.1 (TASK-21) —con sus valores por defecto: sin
filtro de edad, aceptando pacientes nuevos y reasignación automática— y ya se
exponían en la función de presentación del profesional. Lo que faltaba no era
modelarlos sino hacerlos editables. En consecuencia, **esta tarea no requirió
migración de base de datos ni agregó campos al esquema**: el modelo de datos ya
reflejaba la fuente de verdad, y el trabajo se concentró en la capa de entrada.

Como en la tarea anterior, el alcance se limita a persistir y exponer la
configuración. La validación de la edad del paciente durante la conversación
—que solicita documento y fecha de nacimiento únicamente a pacientes nuevos no
registrados, y reutiliza la fecha ya almacenada para los registrados— pertenece
a M5/P5.6 (TASK-51). La lógica de reasignación que da sentido a cada modalidad,
incluidas las ventanas de espera que cada una implica, pertenece a M3/P3.7
(TASK-40).

## Decisiones y por qué

**La configuración se consolidó en el endpoint existente en lugar de crear uno
nuevo.** El propio ticket contempla esta posibilidad de forma explícita, y se
optó por ella: los tres atributos nuevos son, igual que los de la tarea previa,
campos de la propia entidad profesional que el profesional edita desde su
aplicación, de modo que multiplicar endpoints por subconjunto temático de campos
habría fragmentado una misma operación —"el profesional ajusta su
configuración"— en varias rutas sin ganancia de cohesión, y habría obligado a
duplicar el guard de propiedad, la verificación de pertenencia al tenant y el
registro de auditoría. La separación que sí se mantuvo es la ya establecida en
la tarea anterior entre los datos identitarios del profesional y su
configuración, que siguen en endpoints distintos.

**Las validaciones siguieron el criterio ya adoptado en el módulo.** Los dos
indicadores se validan como booleanos y la modalidad de reasignación como uno de
los dos valores de su enumerado, todo en el DTO mediante class-validator, por
tratarse de restricciones de forma sobre campos aislados y no de reglas sobre la
relación entre varios. Los tres campos se declararon opcionales, en coherencia
con la semántica de modificación parcial que el endpoint ya respetaba.

**Se preservó deliberadamente la semántica de modificación parcial.** Este punto
resultó más relevante aquí que en la tarea anterior porque el indicador de
aceptación de pacientes nuevos tiene por defecto el valor verdadero: es
necesario que un valor falso explícito se escriba y no se confunda con "campo no
enviado". El servicio pasa a la capa de persistencia los campos ausentes como
indefinidos —interpretados como "sin cambio"— y los presentes con su valor, de
modo que cerrar la admisión de pacientes nuevos no exige reenviar la duración de
la consulta ni la franja extra.

**El enumerado de modalidad de reasignación no se tocó.** Ya existía en el
esquema desde P1.1, en inglés y con valor por defecto automático, en coherencia
con los demás enumerados de persistencia. La semántica de cada valor en
castellano se documentó en comentarios del DTO, junto con la referencia al
módulo posterior que implementará el comportamiento.

**Se refactorizó la prueba unitaria para que no se degrade al crecer.** Las
aserciones existentes comparaban el objeto de escritura completo campo por
campo, de modo que agregar atributos de configuración obligaba a repetir todos
los campos —la mayoría como indefinidos— en cada aserción. Se introdujo una
función auxiliar que construye ese objeto esperado a partir de los campos que
cada prueba realmente afirma, con lo que un futuro atributo de configuración se
agrega en un único lugar en vez de en cada aserción.

## Alternativas descartadas

- **Crear un endpoint separado para la política de admisión y reasignación**
  (por ejemplo `PATCH .../politica`): descartada porque el ticket admite
  explícitamente la consolidación y porque habría duplicado autorización,
  acotamiento por tenant y auditoría para operar sobre campos de la misma
  entidad y con el mismo criterio de acceso.
- **Agregar los campos al `PATCH /profesionales/:id` general**: descartada por
  el mismo motivo de cohesión que llevó, en la tarea anterior, a separar los
  datos identitarios de la configuración.
- **Escribir el objeto de persistencia propagando el DTO completo** en lugar de
  enumerar los campos: descartada por mantener la escritura explícita ya usada
  en el módulo, que documenta qué columnas toca la operación y evita que un
  campo futuro del DTO llegue a la base de datos por arrastre.
- **Conservar una prueba unitaria adicional** que verificaba de forma aislada
  que el valor falso explícito no se descartara: retirada por redundante, ya
  que la aserción del objeto de escritura y la prueba end-to-end sobre el
  listado cubren esa misma garantía; además introducía un acceso sin tipar que
  el linter rechazaba.

## Entidades / puertos / adaptadores tocados

- Esquema de Prisma: **sin cambios**. Los campos `adultsOnly`,
  `acceptsNewPatients` y `reassignmentMode` de `Professional`, y el enumerado
  `ReassignmentMode`, ya existían desde la migración del modelado inicial. No se
  generó migración.
- Módulo `src/professionals/`: se ampliaron el DTO de configuración
  (`dto/update-professional-config.dto.ts`) con los tres campos y sus
  validaciones, y el método `updateConfiguration` de `professionals.service.ts`
  con su escritura; se actualizó el comentario del endpoint en
  `professionals.controller.ts` para reflejar el alcance consolidado. No se
  modificaron la función de presentación —que ya devolvía los tres campos—, el
  `ProfessionalOwnershipGuard`, el `AuditService` ni el cliente de Prisma
  acotado por tenant.
- Colección de Postman: regenerada con el script del repositorio, que preserva
  cuerpos y descripciones existentes; se ampliaron el cuerpo de ejemplo y la
  descripción del endpoint de configuración con los tres campos nuevos.

## Tests y qué validan

- `src/professionals/professionals.service.spec.ts` (unitario, ampliado):
  además de lo ya cubierto, valida que se persiste cada una de las dos
  modalidades de reasignación y que se persisten el filtro de edad y el
  indicador de aceptación de pacientes nuevos; las aserciones existentes se
  reescribieron sobre la función auxiliar del objeto esperado.
- `test/professional-config.e2e-spec.ts` (end-to-end, ampliado): valida los
  criterios de aceptación del ticket —modificación con filtro de edad activo,
  admisión cerrada y cada una de las dos modalidades de reasignación, con
  verificación de la persistencia—; que una modificación de un campo de política
  no altera los parámetros de agenda; que el administrador puede fijar la
  política de cualquier profesional del tenant; y el rechazo de una modalidad
  desconocida y de valores no booleanos en los dos indicadores. Se agregó además
  una prueba específica para el criterio de aceptación que exige que el cierre
  de la admisión se refleje de forma inmediata: tras la modificación, se
  consulta el listado de profesionales —que es la vía por la que la capa
  conversacional obtiene los profesionales antes de ofrecer turnos— y se
  comprueba que el indicador aparece ya actualizado en la lectura siguiente, y
  lo mismo al reabrir la admisión.
- Ejecución: suites unitaria (9 suites / 38 tests) y end-to-end (12 suites / 74
  tests) completas en verde; `eslint` sin errores y verificación de tipos sin
  errores. Los datos usados son ficticios.

## Figuras pendientes

- Ninguna nueva. La tarea amplía un endpoint ya existente con campos de
  configuración, sin introducir flujo ni entidad que amerite un diagrama propio.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-25-age-filter-and-reassignment-config`
  (creada a partir de `main`, con P1.1 a P1.4 ya fusionados).
- Ticket: TASK-25 ("P1.5 – Filtro de edad, aceptación de nuevos y modalidades de
  reasignación/reprogramación"). Depende de TASK-21 (P1.1) y TASK-22 (P1.2).
  Deja para M5/P5.6 (TASK-51) la validación de edad en la capa conversacional y
  para M3/P3.7 (TASK-40) la lógica de reasignación de cada modalidad.
