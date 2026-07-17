# Fase 0 — Fundaciones (backend) — corrección del rol `SYSTEM` (TASK-72)

## Qué se implementó

Se corrigió el modelo `User` para incluir un tercer valor del enum `Role`,
`SYSTEM`, que el diagrama entidad-relación de la base de datos define junto
a `admin` y `profesional` pero que la implementación original de
autenticación y roles (fase 0, corrección sobre TASK-16) había omitido. El
enum `Role` en `prisma/schema.prisma` pasó de dos a tres valores
(`ADMIN`, `PROFESSIONAL`, `SYSTEM`), incorporados mediante una migración
`ALTER TYPE "Role" ADD VALUE 'SYSTEM'`. `AuthService.login` se modificó
para rechazar explícitamente con `403 Forbidden` cualquier intento de
inicio de sesión manual de un usuario con rol `SYSTEM`, verificación que se
ejecuta después de validar la contraseña (para no revelar por temporización
o por código de estado si una cuenta existe o qué rol tiene antes de
autenticar la contraseña correctamente). El script de seed del usuario
administrador no requirió cambios de comportamiento — ya sembraba
exclusivamente `Role.ADMIN` — y se dejó un comentario aclarando que
`SYSTEM` no se siembra como cuenta humana ahí. `AuditService.log` tampoco
requirió cambios: recibe el `userId` como identificador opaco, sin relación
de clave foránea hacia `User` en el esquema, por lo que ya admitía
persistir una entrada de auditoría cuyo actor fuera un usuario de rol
`SYSTEM`; se agregó cobertura de tests para dejar esa capacidad verificada
explícitamente en vez de asumida.

Como trabajo necesario para poder validar lo anterior, se corrigieron
además cinco archivos de test end-to-end (`test/auth.e2e-spec.ts`,
`test/audit.e2e-spec.ts`, `test/catalogs.e2e-spec.ts`,
`test/config-tenant.e2e-spec.ts`, `test/tenant-scoping.e2e-spec.ts`) que
habían quedado referenciando identificadores en español previos al
renombrado a inglés de esquema y servicios (por ejemplo
`Role.PROFESIONAL`, `prisma.registroAuditoria`, `prisma.obraSocial`,
`prisma.configuracionOrganizacion`, `AuditService.registrar`), lo que
impedía compilar y ejecutar la suite completa de tests end-to-end del
repositorio. Esta corrección no introduce comportamiento nuevo: solo
alinea los tests con los nombres ya vigentes en el código de producción.

## Decisiones y por qué

**Verificar el rol después de la contraseña, no antes.** `AuthService.login`
primero confirma existencia del usuario y coincidencia de contraseña
(`401` si falla cualquiera de las dos) y recién luego evalúa si el rol es
`SYSTEM` (`403`). Se decidió este orden para que un atacante sin la
contraseña correcta no pueda usar el código de respuesta del endpoint de
login para distinguir cuentas `SYSTEM` de cuentas humanas.

**No agregar lógica nueva en `RolesGuard` ni en `AuditService`.** La
consigna original de la tarea mencionaba "actualizar el guard de roles" y
"revisar" `AuditService`, pero al inspeccionar el código se constató que
ninguno de los dos necesitaba cambios de comportamiento: `RolesGuard` ya
excluye implícitamente `SYSTEM` de cualquier endpoint porque ese rol nunca
se declara en un decorador `@Roles(...)` de una ruta humana, y
`AuditService.log` ya acepta cualquier `userId` como cadena opaca sin
validarlo contra una relación de `User`. Se prefirió dejar constancia por
tests de que el comportamiento ya deseado se sostiene, en vez de agregar
una verificación redundante que no correspondía a ninguna falla real
detectada.

## Alternativas descartadas

- Agregar una relación de clave foránea explícita de `AuditLog.userId`
  hacia `User` para "formalizar" que un actor de auditoría es siempre un
  usuario válido: descartado por no ser parte del alcance de esta
  corrección y por no haber sido solicitado; el comportamiento requerido
  (que un actor `SYSTEM` pueda registrar auditoría) ya se cumplía sin esa
  relación.

## Entidades / puertos / adaptadores tocados

- Enum `Role` en `prisma/schema.prisma` (`prisma/migrations/20260717120000_add_system_role/`).
- `AuthService.login` (`src/auth/auth.service.ts`).
- `prisma/seed.ts` (solo comentario aclaratorio, sin cambio de comportamiento).

## Tests y qué validan

- `src/auth/auth.service.spec.ts` (nuevo) — que `login` devuelve un token
  para `ADMIN` y `PROFESSIONAL` con credenciales válidas, que rechaza con
  `ForbiddenException` a un usuario `SYSTEM` aun con credenciales
  correctas, y que rechaza con `UnauthorizedException` un email
  inexistente o una contraseña incorrecta — cubriendo los tres valores del
  enum `Role`.
- `test/auth.e2e-spec.ts` — caso agregado: `POST /auth/login` con un
  usuario `SYSTEM` responde `403`.
- `test/audit.e2e-spec.ts` — caso agregado: `AuditService.log` persiste
  correctamente una entrada cuyo `userId` pertenece a un usuario de rol
  `SYSTEM`, dentro de un contexto de tenant activo.
- Se corrigieron (sin agregar casos nuevos de negocio) `test/catalogs.e2e-spec.ts`,
  `test/config-tenant.e2e-spec.ts` y `test/tenant-scoping.e2e-spec.ts` para
  que vuelvan a compilar contra los nombres en inglés vigentes.
- Se ejecutó la suite completa contra una instancia local de PostgreSQL
  (vía `docker-compose`) con `prisma migrate deploy` aplicando la nueva
  migración: 6 suites / 19 tests unitarios y 7 suites / 21 tests end-to-end,
  todos en verde.

## Figuras pendientes

Ninguna nueva; esta corrección no introduce un elemento arquitectónico
nuevo, solo un valor adicional de un enum ya documentado en la Figura 1
pendiente (diagrama de arquitectura hexagonal, ver
`tesis/figuras_pendientes.md`).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-72-system-role`.
- Ticket: TASK-72 ("[CORRECCIÓN] P0.5 – Autenticación y usuarios con
  roles"), corrección sobre el trabajo original de TASK-16.
