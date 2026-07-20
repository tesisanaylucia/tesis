# Fase 1 — Profesionales (backend) — Duración de consulta y franja extra para pacientes nuevos (TASK-24)

## Qué se implementó

Sobre el módulo de Profesionales ya construido en P1.1 (entidades), P1.2 (ABM)
y P1.3 (horarios y ausencias) se agregó la configuración de agenda que cada
profesional define para sí mismo: la duración de sus turnos y la franja horaria
extra destinada a la primera sesión de un paciente nuevo. Concretamente se
persisten tres campos de la entidad profesional —duración de la consulta (en
minutos), franja extra para pacientes nuevos (en horas) y la modalidad de
ubicación de esa franja extra— y se expone un endpoint dedicado para editarlos:
`PATCH /profesionales/:id/configuracion`.

Dos de los tres campos (`consultationDuration` y `newPatientExtraSlot`) ya
existían en el esquema desde P1.1, declarados opcionales precisamente porque su
configuración se difería a esta tarea. El tercero, la modalidad de la franja
extra, no estaba en el modelo entidad-relación original y se incorporó en esta
tarea como un enumerado nuevo (`NewPatientSlotMode`) con una migración de Prisma
que crea el tipo y agrega la columna nullable. Los tres valores del enumerado
—franja extra como primer turno del día (antes de la franja habitual), como
último turno del día (después de la franja habitual), o dentro de la franja
habitual ocupando dos turnos consecutivos— provienen literalmente de la fuente
de verdad (SRS, Módulo Turnos, "Pacientes nuevos / primera sesión", y Módulo
Profesionales, "Configuración de la franja horaria extra para pacientes
nuevos").

Esta tarea solo almacena la configuración. La lógica que aplica la franja extra
al generar los slots de la agenda, y el límite de un paciente nuevo por día,
pertenecen a M3/P3.4 (TASK-37); el aviso de la duración de la sesión al paciente
al confirmar el turno pertenece a M4/M5. El alcance quedó, por tanto, acotado a
la persistencia y el endpoint de edición con sus validaciones.

## Decisiones y por qué

**La configuración se editó con un endpoint propio, no ampliando el `PATCH`
general de datos del profesional.** El ticket especifica una ruta separada,
`PATCH /profesionales/:id/configuracion`, y esa separación se respetó por
cohesión: los datos generales del profesional (nombre, especialidad, tipo de
atención) y su configuración de agenda son preocupaciones distintas, con DTO y
validaciones propias, y mantenerlas en endpoints separados deja explícito qué
conjunto de campos toca cada operación. El DTO de edición general se había
dejado deliberadamente sin estos campos en P1.2, anticipando esta tarea. Se
prefirió, sin embargo, no crear un controlador ni un servicio nuevos para la
configuración: a diferencia de los horarios y las ausencias —que son entidades
propias y justifican su sub-recurso con controlador dedicado—, la configuración
son campos de la propia entidad profesional, de modo que el método se sumó al
controlador de profesionales existente y la lógica a su servicio, reutilizando
la verificación de pertenencia al tenant (`findOwnedOrThrow`) y el registro de
auditoría ya presentes.

**El método de servicio respeta la semántica de un PATCH parcial.** Solo se
escriben los campos presentes en el cuerpo de la petición; los ausentes se
pasan como `undefined`, que Prisma interpreta como "sin cambio", de modo que
editar únicamente la duración no borra la franja extra ni su modalidad. Se
prefirió esta semántica parcial a exigir el envío de los tres campos en cada
petición porque un profesional puede querer ajustar un solo parámetro desde la
app sin reenviar el resto de su configuración.

**Las validaciones de los criterios de aceptación viven en el DTO, con
class-validator.** La duración de la consulta debe ser un entero estrictamente
positivo y la franja extra un entero no negativo (se admite cero, que
representa "sin tiempo extra"); la modalidad, cuando se envía, debe ser uno de
los tres valores del enumerado. Estas son restricciones de forma sobre campos
aislados —no reglas sobre la relación entre varios—, por lo que corresponden al
DTO y no al servicio, en coherencia con el criterio ya adoptado en P1.3 para
distinguir validación de forma (DTO) de regla de negocio (servicio). Todos los
campos se marcaron opcionales para habilitar el PATCH parcial.

**La modalidad se modeló como enumerado en inglés, en coherencia con el resto
del esquema.** Aunque el vocabulario de dominio del proyecto es el castellano,
los enumerados de persistencia ya existentes (`CareType`, `ReassignmentMode`,
`LicenseType`, `Weekday`) están en inglés, siguiendo la convención de que solo
las entidades y el vocabulario de dominio se nombran en español mientras que el
resto del código va en inglés. El nuevo enumerado (`FIRST_SLOT_OF_DAY`,
`LAST_SLOT_OF_DAY`, `WITHIN_SCHEDULE`) se sumó a esa convención, con comentarios
en el esquema que mapean cada valor a su semántica en castellano tomada del SRS.
Se lo dejó nullable, en coherencia con los otros dos campos de configuración,
porque un profesional recién dado de alta aún no ha definido su modalidad.

**La autorización reutilizó el guard de propiedad existente.** El endpoint queda
accesible al administrador para cualquier profesional del tenant y al propio
profesional para su registro, aplicando el `ProfessionalOwnershipGuard` ya usado
en la edición general y en las capacidades de agenda, sin introducir lógica de
autorización nueva.

## Alternativas descartadas

- **Ampliar el `PATCH /profesionales/:id` general con los campos de
  configuración**: descartada porque el ticket pide una ruta dedicada y porque
  separar datos generales de configuración de agenda mantiene DTO y validaciones
  cohesivos y explícitos.
- **Crear un controlador y un servicio nuevos para la configuración** (como se
  hizo con horarios y ausencias): descartada por sobredimensionada; la
  configuración son campos de la propia entidad profesional, no una entidad
  aparte, de modo que se reutilizó el controlador y el servicio existentes.
- **Exigir los tres campos en cada petición**: descartada en favor de un PATCH
  parcial, para permitir ajustar un único parámetro sin reenviar el resto.
- **Nombrar el enumerado y sus valores en español**: descartada por coherencia
  con los enumerados de persistencia ya existentes, todos en inglés; la
  semántica en castellano quedó en los comentarios del esquema.

## Entidades / puertos / adaptadores tocados

- Esquema de Prisma: enumerado nuevo `NewPatientSlotMode` y columna nullable
  `newPatientSlotMode` en `Professional`; migración
  `20260718010608_add_new_patient_slot_mode` (crea el tipo y agrega la columna).
- Módulo `src/professionals/`: DTO nuevo `dto/update-professional-config.dto.ts`
  con las validaciones; método `updateConfiguration` agregado a
  `professionals.service.ts`; endpoint `PATCH :id/configuracion` agregado a
  `professionals.controller.ts` bajo el guard de propiedad; el campo de
  modalidad se sumó a la interfaz de respuesta y a la función de presentación en
  `professional.presenter.ts`. Se reutilizaron sin cambios el
  `ProfessionalOwnershipGuard`, el `AuditService`, la verificación
  `findOwnedOrThrow` y el cliente de Prisma acotado por tenant.
- Colección de Postman: regenerada para incluir el endpoint de configuración,
  con un cuerpo de ejemplo que muestra los tres campos y su descripción.

## Tests y qué validan

- `src/professionals/professionals.service.spec.ts` (unitario, nuevo, con el
  cliente de Prisma y el servicio de auditoría simulados): valida que la
  configuración se persiste para cada uno de los tres valores de modalidad, que
  un PATCH parcial deja los campos omitidos sin tocar (pasándolos como
  `undefined`), que se escribe una entrada de auditoría marcada como cambio de
  configuración, y que se rechaza con "no encontrado" un profesional fuera del
  tenant del solicitante sin escribir ni auditar.
- `test/professional-config.e2e-spec.ts` (end-to-end, sobre PostgreSQL local y
  atravesando la capa HTTP con JWT reales): valida los criterios de aceptación
  del ticket —configuración y persistencia de los tres valores de modalidad por
  el propio profesional, con su auditoría; aceptación de una franja extra de
  cero; PATCH parcial que conserva los campos omitidos; configuración por el
  administrador; rechazo (400) de una duración no positiva, de una franja
  negativa, de una modalidad desconocida y de una duración no entera; rechazo
  (403) al configurar a otro profesional; y aislamiento por tenant (404) sobre
  un profesional de otra organización.
- Ejecución: suites unitaria (9 suites / 35 tests) y end-to-end (12 suites / 66
  tests) completas en verde; `eslint` y `nest build` sin errores. Los datos
  usados son ficticios.

## Figuras pendientes

- Ninguna nueva. La tarea agrega un endpoint de configuración sin flujo ni
  entidad que amerite un diagrama propio.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-24-consultation-duration-config` (creada a
  partir de `main`, con P1.1, P1.2 y P1.3 ya fusionados).
- Ticket: TASK-24 ("P1.4 – Duración de consulta y franja extra para pacientes
  nuevos"). Depende de TASK-21 (P1.1) y TASK-22 (P1.2). Deja para M3/P3.4
  (TASK-37) la aplicación de la franja extra al generar slots y el cupo de un
  paciente nuevo por día, y para M4/M5 el aviso de duración al paciente.
