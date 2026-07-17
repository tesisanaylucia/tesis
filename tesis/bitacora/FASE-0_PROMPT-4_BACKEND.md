# Fase 0 — Fundaciones (backend) — separación de la traza de auditoría en tres categorías: `pacienteId`, `action` libre y `LockLog` (TASK-75)

## Qué se implementó

Se corrigió el modelo `AuditLog` (auditoría de acciones humanas) para
resolver la mezcla de tres categorías de eventos distintas que, de otro
modo, confluirían en una única tabla. El diagrama entidad-relación de la
base de datos y el documento de requisitos distinguen: acciones humanas
sobre un turno (ya cubiertas por la corrección de TASK-73), acciones
humanas sobre un paciente que no están ligadas a un turno (por ejemplo,
exportación de datos o supresión de un paciente, propias del módulo de
cumplimiento normativo), y eventos operativos generados por el adaptador
de la cerradura inteligente (confirmaciones de código enviadas, expiración
de códigos, aperturas manuales, errores de acceso, intentos), que no son
decisiones tomadas por una persona sino registros de la propia integración.

Para la segunda categoría se agregó a `AuditLog` un campo `pacienteId`
(`String?`, `@db.Uuid`, con índice propio), análogo en tratamiento a
`turnoId`: sin relación (`@relation`) todavía, porque la entidad `Paciente`
pertenece a una fase posterior del desarrollo (módulo de pacientes) que al
momento de esta tarea no existe en el esquema. `AuditService.log` y su
interfaz `CreateAuditLogParams` se extendieron con un parámetro opcional
`pacienteId`, persistido de la misma forma que `turnoId`.

Se verificó además que el campo `action` de `AuditLog` estuviera
implementado como texto libre en lugar de como un enum fijo de Prisma. La
implementación vigente (desde TASK-17) lo tenía como un enum `AuditAction`
con tres valores (`CREATE`, `UPDATE`, `DELETE`), lo cual entra en conflicto
con la necesidad de registrar, a futuro, acciones de cumplimiento normativo
sobre pacientes (por ejemplo, exportación o supresión de datos) que no se
conocen de antemano como un conjunto cerrado. Se convirtió la columna a
`String` mediante una migración manual que altera el tipo de la columna
(`ALTER COLUMN ... TYPE TEXT USING (...)`) y elimina el tipo enum
(`DROP TYPE`), y se actualizó la firma de `AuditService.log` para aceptar
`action: string`.

Para la tercera categoría (eventos operativos de la cerradura) se creó un
modelo separado, `LockLog`, con alcance tenant (`organizationId`,
obligatorio e indexado, siguiendo el mismo patrón de acotamiento por
organización que el resto del esquema), y campos para el turno asociado
(`turnoId`, opcional), el código de acceso afectado (`accessCodeId`,
opcional), el tipo de evento (`event`, texto libre, análogo a `action` en
`AuditLog` y por el mismo motivo), un número de intento (`attemptNumber`,
opcional, relevante solo para eventos de error o reintento) y un campo de
detalle en formato JSON (`detail`, para la respuesta de la API de TTLock,
el código HTTP de error o el mensaje asociado). Se agregó
`LockLogService.log(...)`, que persiste un evento a través del cliente de
Prisma acotado por tenant (el mismo mecanismo que usan `AuditService` y
`ConfigTenantService`), como módulo global independiente
(`src/lock-log/`) sin ninguna dependencia del módulo de auditoría.

No se modificó ningún servicio del módulo de cerradura (adaptador TTLock,
generación de códigos de acceso) para que efectivamente llame a
`LockLogService`, ni ningún endpoint administrativo de pacientes para que
llame a `AuditService` pasando `pacienteId`: al momento de esta tarea,
ninguno de esos módulos existe todavía en el código (ni el módulo de
pacientes ni el módulo de cerradura tienen implementación más allá de la
definición del puerto `LockPort`), por lo que no hay ningún llamador real
al que instrumentar. Esto sigue el mismo criterio aplicado en TASK-17 y en
la corrección de TASK-73: no instrumentar módulos que aún no existen.

## Decisiones y por qué

**`pacienteId` como columna simple sin FK, siguiendo el mismo patrón que
`turnoId`.** La entidad `Paciente` todavía no existe en el esquema (fase 2,
posterior a esta), de modo que agregar la relación real habría requerido
adelantar parte del alcance de esa fase. Se prefirió repetir la estrategia
ya validada con `turnoId` en TASK-73: agregar el campo ahora, sin relación,
y declarar la FK real cuando la fase correspondiente incorpore la entidad.

**Eliminar el enum `AuditAction` en vez de conservarlo junto a valores de
texto libre.** Se consideró mantener el enum para las tres acciones
CRUD originales y agregar un campo de texto adicional para acciones de
cumplimiento, pero esa alternativa habría dejado dos campos de acción
distintos en el mismo registro según la categoría del evento, lo cual
introduce una ambigüedad innecesaria sobre cuál campo consultar en cada
caso. Convertir el único campo `action` a texto libre evita esa
duplicación, al costo de perder la validación de valores en tiempo de
compilación que ofrecía el enum; ese costo se consideró aceptable porque
el conjunto de acciones válidas para pacientes todavía no está definido
por ningún módulo real.

**`LockLog` como tabla y servicio separados de `AuditLog`, en vez de un
campo adicional que distinga el origen del evento.** Se descartó agregar
una columna de "categoría" a `AuditLog` para diferenciar acciones humanas
de eventos operativos de la cerradura, porque mezclar ambos tipos de
evento en la misma tabla —aunque distinguibles por columna— contamina la
traza de auditoría con volumen operativo de una integración de hardware
(reintentos, errores de comunicación), que tiene una cardinalidad y un
propósito distintos (diagnóstico técnico, no accountability sobre datos de
pacientes). Separar la tabla también permite que cada una evolucione de
forma independiente: por ejemplo, una futura política de retención de
logs operativos de la cerradura no necesariamente debe aplicarse a la
traza de auditoría de cumplimiento normativo.

**Nombre de campo `event`/`attemptNumber` en inglés, aunque los nombres de
evento del documento de requisitos original están en español.** Se tradujo
`evento` a `event`, `numero_intento` a `attemptNumber` y los valores de
ejemplo del enunciado (`CONFIRMACION_ENVIADA`, `EXPIRACION_CODIGO`,
`APERTURA_ADHOC`, `ERROR_CODIGO_ACCESO`, `INTENTO_CODIGO`) a sus
equivalentes en inglés (`CONFIRMATION_SENT`, `CODE_EXPIRED`,
`ADHOC_UNLOCK`, `ACCESS_CODE_ERROR`, `CODE_ATTEMPT`) en los tests, siguiendo
la convención ya vigente en el repositorio de mantener el código, sus
identificadores y los valores almacenados en inglés, reservando el español
únicamente para el vocabulario de entidades de dominio (`Turno`,
`Paciente`), tal como ya se hizo con el enum `AuditAction`
(`CREAR`/`MODIFICAR`/`ELIMINAR` → `CREATE`/`UPDATE`/`DELETE`) antes de esta
tarea.

## Alternativas descartadas

- **Usar identificadores enteros autoincrementales para `LockLog.id` y
  `LockLog.accessCodeId`**, como sugería el enunciado original: descartado
  porque todo el resto del esquema (incluido `AuditLog`) usa UUID generado
  por la base de datos como clave primaria; introducir un modelo con un
  esquema de identificadores distinto habría roto la consistencia del
  resto de las entidades sin ningún beneficio funcional.
- **Registrar los eventos de cerradura como entradas de `AuditLog` con un
  `entity` distintivo** (por ejemplo, `entity: 'LockEvent'`): descartado
  por la misma razón que motivó crear una tabla separada; se prefirió una
  separación estructural (tabla y servicio propios) en lugar de una
  convención de nombrado dentro de la tabla existente.

## Entidades / puertos / adaptadores tocados

- `AuditLog` en `prisma/schema.prisma`: se agregó `pacienteId` y se
  convirtió `action` de enum a `String` (migración
  `prisma/migrations/20260717180000_add_patient_id_and_lock_log/`).
- `AuditService.log` y `CreateAuditLogParams`
  (`src/audit/audit.service.ts`): parámetro `pacienteId` agregado, tipo de
  `action` cambiado a `string`.
- Modelo `LockLog` nuevo en `prisma/schema.prisma` (misma migración).
- `LockLogService` y `LockLogModule` nuevos (`src/lock-log/`), registrados
  como módulo global en `AppModule`, sin relación de dependencia con
  `AuditModule`.

## Tests y qué validan

- `src/audit/audit.service.spec.ts`: se actualizaron los casos existentes
  para usar cadenas de texto en lugar del enum eliminado, y se agregó un
  caso que persiste un registro con `pacienteId` explícito y una acción de
  cumplimiento normativo (`PATIENT_DATA_EXPORT`) sin `turnoId`.
- `test/audit.e2e-spec.ts`: se actualizaron los casos existentes de la
  misma forma, y se agregó un caso contra Postgres real que persiste un
  registro con `pacienteId` y una acción de texto libre no perteneciente al
  conjunto CRUD original, verificando tanto el valor devuelto como el
  efectivamente almacenado.
- `src/lock-log/lock-log.service.spec.ts` (nuevo): persiste un evento con
  todos los campos opcionales presentes y otro sin ninguno de ellos,
  verificando que el cliente acotado por tenant reciba los datos esperados.
- `test/lock-log.e2e-spec.ts` (nuevo, contra Postgres real): persiste un
  evento `ACCESS_CODE_ERROR` con número de intento y detalle, verifica que
  el servicio rechaza registrar sin contexto de tenant (mismo mecanismo que
  `AuditService`), y un caso de aislamiento que registra un evento de
  cerradura y comprueba que la tabla `AuditLog` de esa misma organización
  permanece vacía.
- Se ejecutó la suite completa: 7 suites / 23 tests unitarios y 8 suites /
  27 tests end-to-end, todos en verde, contra la instancia local de
  PostgreSQL (`docker-compose`) con la nueva migración aplicada vía
  `prisma migrate deploy`.

## Figuras pendientes

Ninguna nueva. Cuando la fase 6 (Cerradura TTLock) implemente el adaptador
real y la fase 2 (Pacientes) incorpore la entidad `Paciente`, corresponderá
revisar el diagrama entidad-relación pendiente (ver
`tesis/figuras_pendientes.md`) para reflejar las relaciones de clave
foránea ya conectadas y, eventualmente, un diagrama específico de la
separación entre `AuditLog` y `LockLog`.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-75-patient-audit-lock-log` (creada a
  partir de `feature/TASK-73-audit-turno-fk`, todavía no fusionada a
  `main`).
- Ticket: TASK-75 ("[CORRECCIÓN] P0.6b – Separar eventos de auditoría:
  id_paciente en REGISTRO_AUDITORIA y nueva tabla LOG_CERRADURA"),
  corrección/ampliación sobre TASK-17 y TASK-73. Depende de TASK-34 (fase
  3, pendiente), TASK-55/56/59 (fase 6, pendientes) y TASK-67 (fase 8,
  pendiente) para las relaciones de clave foránea reales y para la
  instrumentación efectiva de `LockLogService` desde el adaptador TTLock.
