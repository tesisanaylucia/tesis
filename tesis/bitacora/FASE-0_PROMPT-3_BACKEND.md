# Fase 0 — Fundaciones (backend) — corrección de la traza de auditoría con `turnoId` (TASK-73)

## Qué se implementó

Se corrigió el modelo `AuditLog` (originalmente `RegistroAuditoria`, tarea
TASK-17) para incorporar el campo `turnoId`, que el diagrama
entidad-relación de la base de datos define explícitamente en la entidad
`REGISTRO_AUDITORIA` como `id_turno [FK]` hacia `TURNO`, y que la
implementación original solo cubría de forma genérica a través de los
campos `entity`/`entityId`. Se agregó `turnoId String? @db.Uuid` al modelo
Prisma, junto con un índice (`@@index([turnoId])`) para soportar consultas
de historial de auditoría por turno, mediante una migración manual
(`prisma/migrations/20260717170000_add_audit_log_turno_id/`) que agrega la
columna nullable y su índice sobre la tabla existente. `AuditService.log`
se extendió con un parámetro opcional `turnoId` en `CreateAuditLogParams`,
que se persiste tal cual (`undefined` si no se provee, lo que Prisma
almacena como `NULL`).

No se agregó una relación (`@relation`) de Prisma entre `AuditLog.turnoId`
y una entidad `Turno`, porque esa entidad todavía no existe: su creación
corresponde a TASK-34 ("P3.1 — Entidades de Turno, Feriado y Lista de
espera"), que al momento de esta corrección seguía en estado "Por hacer".
`turnoId` queda por ahora como una columna simple sin restricción de clave
foránea a nivel de base de datos; la relación real se agregará cuando
TASK-34 cree el modelo `Turno` en el esquema.

Tampoco se modificó ningún módulo para que llame a `AuditService.log`
pasando `turnoId` en cambios reales sobre turnos (lo que pedía el punto 4
de la consigna original, "los módulos que registran cambios sobre turnos
(M3, M4) llamen a AuditService pasando el turnoId"): ni el motor de turnos
(fase 3) ni notificaciones (fase 4) existen todavía en el código, por lo
que no hay ningún llamador real al que instrumentar. Esto sigue el mismo
criterio aplicado en TASK-17: no instrumentar módulos que aún no existen.

## Decisiones y por qué

**`turnoId` como columna simple, sin FK, en vez de bloquear la tarea hasta
TASK-34.** Se consultó explícitamente a la usuaria cómo proceder dado que
el criterio de aceptación pedía una FK real a `TURNO` pero esa entidad no
existe todavía. Se optó por agregar el campo ahora (sin relación) en lugar
de posponer toda la corrección o crear un modelo `Turno` mínimo dentro de
esta misma rama, para no invadir el alcance de TASK-34 (que define la
entidad completa con sus propios campos, estados e índices) ni dejar
`AuditService` sin la forma esperada del parámetro mientras tanto.

**Tipo `String @db.Uuid` para `turnoId`, no `Int`.** El diagrama ER
original usa la convención `id_turno`, pero todas las claves primarias ya
migradas a inglés en este esquema (`Organization.id`, `User.id`,
`AuditLog.id`, etc.) son `String` con `@default(dbgenerated("gen_random_uuid()"))`
y `@db.Uuid`. Se mantuvo esa misma convención para `turnoId`, anticipando
que el futuro modelo `Turno` de TASK-34 seguirá el mismo patrón que el
resto del esquema.

## Alternativas descartadas

- **Crear un modelo `Turno` mínimo en esta misma rama** solo para poder
  declarar la FK real: descartado porque el alcance completo de `Turno`
  (estados, índices compuestos, relación con `Profesional`/`Paciente`) le
  corresponde a TASK-34, y crear una versión mínima ahora hubiera obligado
  a rehacer la migración cuando esa tarea se implemente de verdad.
- **Bloquear TASK-73 hasta que TASK-34 esté hecha:** descartado por
  decisión de la usuaria; se prefirió avanzar con la columna simple ahora.

## Entidades / puertos / adaptadores tocados

- `AuditLog` en `prisma/schema.prisma` (`prisma/migrations/20260717170000_add_audit_log_turno_id/`).
- `AuditService.log` y `CreateAuditLogParams` (`src/audit/audit.service.ts`).

## Tests y qué validan

- `src/audit/audit.service.spec.ts` — casos agregados/ajustados:
  persiste un registro pasando `turnoId` explícito, y persiste un registro
  sin `turnoId` (queda `undefined`, es decir `NULL` en la base).
- `test/audit.e2e-spec.ts` — dos casos agregados contra Postgres real:
  un registro de auditoría sobre un cambio de turno queda con el `turnoId`
  correcto tanto en el valor devuelto como en lo efectivamente almacenado;
  y un registro de una entidad no relacionada a turnos (`Patient`) admite
  `turnoId = null`.
- Se ejecutó la suite completa: 6 suites / 20 tests unitarios y 7 suites /
  23 tests end-to-end, todos en verde, contra la instancia local de
  PostgreSQL (`docker-compose`) con la nueva migración aplicada vía
  `prisma migrate deploy`.

## Figuras pendientes

Ninguna nueva. Esta corrección no introduce un elemento arquitectónico
nuevo; cuando TASK-34 agregue la entidad `Turno` y su relación real con
`AuditLog`, corresponderá revisar el diagrama entidad-relación pendiente
(ver `tesis/figuras_pendientes.md`) para reflejar la FK ya conectada.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-73-audit-turno-fk`.
- Ticket: TASK-73 ("[CORRECCIÓN] P0.6 – Traza de auditoría"), corrección
  sobre el trabajo original de TASK-17. Depende de TASK-34 (pendiente) para
  la relación de clave foránea real.
