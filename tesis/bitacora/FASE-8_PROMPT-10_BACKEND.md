# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Lectura del registro de auditoría de un paciente (TASK-179, SEC 8, corrección/extensión a P8.2 y TASK-67)

## Qué se implementó

Se cerró un hallazgo de una auditoría de código contra la especificación de
requisitos (SEC 8): el sistema escribía una entrada de auditoría por cada
acción sobre un paciente desde la Fase 0, pero no existía ningún camino
para leer esa traza — solo consultas directas a la base de datos. Los
comentarios del propio esquema ya justificaban la clave foránea
`AuditLog.patientId` invocando "todo lo que se le hizo a este paciente"
como la consulta de cumplimiento que ese campo existe para responder, sin
que ningún endpoint la respondiera realmente. Se agregó un cuarto endpoint
al módulo de derechos del titular ya existente (TASK-67, P8.2):
`GET /admin/pacientes/:id/auditoria`, solo para administradores, que
devuelve la traza de auditoría de un paciente paginada y ordenada de la
más reciente a la más antigua.

## Decisiones y por qué

**El endpoint se agregó al módulo de derechos del titular ya existente
(`PatientDataRightsController`/`PatientDataRightsService`), no a uno
nuevo ni al controlador ordinario de pacientes.** Leer "todo lo que se le
hizo a este paciente" es, en los mismos términos que ya usa el código para
el endpoint de exportación, el derecho de acceso del Art. 14 de la Ley
25.326 — pero sobre la traza de acciones, no sobre el propio registro del
paciente, que es lo único que el endpoint de exportación ya cubre. Ambos
son la misma clase de operación de cumplimiento, distinta de la edición
administrativa ordinaria (P2.2), así que corresponden al mismo módulo,
ADMIN-only, ya separado del ABMC de pacientes por esa razón. La consulta
de la especificación de requisitos citaba textualmente
`GET /pacientes/:id/auditoria` (sin el prefijo `admin/`), pero se descartó
ese literal a favor de mantener la ruta bajo `/admin/pacientes/:id`, junto
a exportación, supresión y rectificación: el propio hallazgo es
generado automáticamente a partir de una auditoría de código y está
etiquetado explícitamente como pendiente de validación humana antes de
implementarse, y las tres rutas hermanas ya establecen la convención real
del sistema para esta clase de endpoint.

**La entrada de auditoría del paciente se devuelve con sus columnas
directas — `userId` en vez de un objeto anidado con el actor** (nombre o
email de quien hizo la acción), siguiendo el mismo patrón que cualquier
otro presentador del código expone una relación como clave foránea plana
(`AppointmentResponse.patientId`/`.professionalId`, por ejemplo) en lugar
de desnormalizarla. Quien necesite resolver la identidad de ese `userId`
ya cuenta con `GET /users`, también restringido a administradores.

**La propia lectura de la traza se audita con un nombre de acción propio,
`VIEW_PATIENT_AUDIT_LOG`, igual que ya hacía la exportación con
`EXPORT_PATIENT_DATA`.** Ver quién consultó alguna vez el historial
completo de acciones sobre un paciente es, en sí mismo, información de
cumplimiento — y la razón por la que el endpoint de exportación ya se
auditaba con su propio nombre de acción (distinguir una solicitud formal
de derechos de una edición rutinaria) aplica igual de bien a esta lectura,
que responde una pregunta distinta ("quién miró el historial" en vez de
"se exportó el registro").

**La paginación replica el mismo contrato `limit`/`offset` que ya usa
`GET /notificaciones`** (`ListAuditLogQueryDto`, con el mismo par de
constantes de tamaño de página por defecto/máximo que ya existían para
notificaciones, ahora una copia propia en `audit.constants.ts` para no
acoplar un módulo de cumplimiento a las constantes de otro dominio), en
lugar de inventar un esquema de paginación nuevo: la traza de un paciente
puede acumular años de entradas bajo la política de retención ya
documentada (cinco años desde cada entrada antes de archivarse), así que
necesita la misma protección contra traer todo de una vez que ya tiene
cualquier otro listado del sistema.

## Entidades / puertos / adaptadores tocados

Ninguna migración de base de datos: `AuditLog` y su índice
`(organizationId, patientId)` ya existían desde que la Fase 0 introdujo el
propio modelo, exactamente pensados para esta consulta.

- `src/audit/audit.presenter.ts` (nuevo): selección de columnas y
  función de mapeo para una entrada de `AuditLog`, mismo patrón que
  `notification.presenter.ts`.
- `src/audit/audit.constants.ts` (nuevo): tamaño de página por
  defecto/máximo para este listado.
- `src/audit/dto/list-audit-log-query.dto.ts` (nuevo): `limit`/`offset`
  validados, mismo esquema que `ListNotificationsQueryDto`.
- `src/audit/audit.service.ts`: nuevo método `findByPatient`, que arma la
  consulta paginada contra el cliente de Prisma acotado por inquilino.
- `src/patient-data-rights/patient-data-rights.service.ts`: nueva acción
  `VIEW_PATIENT_AUDIT_LOG_ACTION` y método `auditLog`, que primero
  verifica que el paciente sea visible para quien pide (mismo guard que ya
  usa la exportación) antes de leer la traza y auditar la propia lectura.
- `src/patient-data-rights/patient-data-rights.controller.ts`: nueva ruta
  `GET :id/auditoria`.
- `CLAUDE.md`: ampliada la sección de derechos del titular (antes tres
  endpoints, ahora cuatro) con la descripción de la nueva ruta.
- `postman/psique-backend.postman_collection.json`: regenerado por el
  hook del proyecto a partir de los controladores, incorporando la ruta
  nueva.

## Tests

- `src/audit/audit.service.spec.ts`: cobertura unitaria de
  `findByPatient` — consulta acotada por `patientId`, orden descendente
  por `timestamp`, y que un `limit`/`offset` explícito reemplaza a los
  valores por defecto.
- `src/patient-data-rights/patient-data-rights.service.spec.ts`: cobertura
  unitaria del nuevo método `auditLog` — verifica visibilidad del paciente
  antes de leer, delega la lectura paginada en `AuditService`, y audita
  `VIEW_PATIENT_AUDIT_LOG` con los campos esperados.
- `test/patient-data-rights.e2e-spec.ts` (ampliado): traza devuelta de la
  más reciente a la más antigua contra Postgres real, con la propia
  lectura auditada; paginación con `?limit=&offset=`; ninguna entrada de
  otro paciente del mismo tenant se filtra a la respuesta; 403 para un
  profesional; 404 para un paciente de otro tenant (mismo patrón de
  aislamiento que ya prueban export/erase/rectify en el mismo archivo).

Suite completa en verde al cierre: 90 suites unitarias (975 pruebas) y la
suite de extremo a extremo de este módulo (14 pruebas) contra PostgreSQL
real, verificación de tipos y lint sin errores.

## Figuras pendientes

Se agregó una figura pendiente (Figura 58) para el diagrama de secuencia
de esta lectura, contrastándola con la de exportación ya pendiente — ver
`figuras_pendientes.md`.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-179-patient-audit-log-endpoint`,
  creada desde `main` para esta tarea, con pull request abierto hacia
  `main`.
