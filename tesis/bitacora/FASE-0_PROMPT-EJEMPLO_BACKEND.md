# Fase 0 — Fundaciones (backend) — entrada de ejemplo

> Esta entrada es una **muestra de validación** de la skill
> `documentacion-tesis`: ilustra el formato y nivel de detalle esperado de
> una entrada de bitácora real, a partir del trabajo de fundaciones ya
> presente en el repo backend. No reemplaza a las entradas que la skill
> generará automáticamente para tareas futuras.

## Qué se implementó

Se levantó el esqueleto del backend sobre NestJS y se conectó a PostgreSQL
mediante Prisma. Sobre esa base se agregaron, en commits sucesivos: el
modelo `Organization` y el mecanismo de scoping multi-tenant, autenticación
JWT con control de acceso por roles, un registro de auditoría, y los tres
puertos de integración externos (mensajería, cerradura, IA) con adaptadores
stub para desarrollo local.

## Decisiones y por qué

**Multi-tenancy impuesto a nivel de cliente de Prisma, no por repositorio.**
En lugar de exigir que cada método de cada repositorio agregue manualmente
`where: { organizationId }`, se implementó una extensión de Prisma
(`withTenantScoping`, en `src/prisma/tenant-scoping.extension.ts`) que
intercepta todas las operaciones sobre modelos que tienen `organizationId`
en su esquema y les inyecta el `organizationId` del contexto de tenant
actual, tomado de un `AsyncLocalStorage` (`TenantContextService`). Si no hay
un `organizationId` en contexto al ejecutar una de estas operaciones, la
extensión lanza `MissingTenantContextError` en lugar de dejar pasar una
consulta sin acotar. La justificación de esta decisión es que un control
manual, repetido en cada repositorio, depende de que cada desarrollador
recuerde agregarlo en cada consulta nueva; centralizarlo en el cliente hace
que olvidarlo sea un error en tiempo de ejecución detectable por los tests,
en vez de una fuga silenciosa de datos entre organizaciones.

**Auditoría como registro append-only separado del dominio.** El registro
de auditoría (`RegistroAuditoria`, expuesto a través de `AuditService`) se
modeló como una entidad propia con su propio modelo Prisma, en vez de
agregar columnas de auditoría a cada entidad de negocio. Se decidió así
porque la auditoría necesitada por la Ley 25.326 (protección de datos)
requiere poder reconstruir *quién* hizo *qué* sobre *qué* entidad y
*cuándo*, para cualquier entidad del dominio, sin acoplar cada modelo de
negocio a los detalles de cómo se registra esa traza.

**Autenticación y autorización como guards globales, no por ruta.** Se optó
por un `JwtAuthGuard` global (con opt-out explícito vía `@Public()`) y un
`RolesGuard` global (con `@Roles(...)` por ruta) en lugar de aplicar guards
ruta por ruta. La alternativa descartada — decorar cada controlador
individualmente — se descartó porque el costo de olvidar un guard en una
ruta nueva (dejarla desprotegida por omisión) es mayor que el costo de
tener que declarar `@Public()` explícitamente en los pocos casos que sí
deben quedar abiertos.

**Integraciones externas detrás de puertos con adaptadores stub.** Los tres
puntos de integración externa previstos (mensajería tipo WhatsApp, cerradura
inteligente tipo TTLock, y procesamiento de IA conversacional) se definieron
como interfaces de dominio (`MessagingPort`, `LockPort`, `AIPort` en
`src/domain/ports/`) resueltas por inyección de dependencias contra un token
(`Symbol`), con adaptadores *stub* (`src/infrastructure/adapters/`) como
única implementación por ahora. Se decidió así para poder desarrollar y
testear el resto del sistema sin depender de credenciales ni disponibilidad
de los proveedores reales, y para que conectar el proveedor real más
adelante sea un cambio de adaptador, no un cambio en la lógica de negocio
que los consume.

## Alternativas descartadas

- Alcanzar el scoping multi-tenant mediante un middleware que reescribiera
  las consultas SQL crudas: descartado por acoplar la lógica de tenancy al
  transporte HTTP en lugar de al acceso a datos, dejando sin cobertura
  cualquier acceso a Prisma que no pasara por ese middleware (jobs
  programados, scripts, etc.).
- Guardar el estado del tenant actual en una variable de módulo o en el
  request de Express directamente: descartado en favor de
  `AsyncLocalStorage`, que evita fugas de contexto entre requests
  concurrentes sin depender de pasar el `organizationId` explícitamente por
  cada capa de llamadas.

## Entidades / puertos / adaptadores tocados

- Modelos Prisma: `Organization`, `User`, `RegistroAuditoria`,
  `ConfiguracionOrganizacion`, `ObraSocial`, `Diagnostico`.
- `TenantContextService` (`src/tenant/tenant-context.service.ts`) y la
  extensión `withTenantScoping` (`src/prisma/tenant-scoping.extension.ts`).
- `AuditService` (`src/audit/audit.service.ts`).
- Guards y decoradores de auth: `JwtAuthGuard`, `RolesGuard`, `@Public()`,
  `@Roles()`, `@CurrentUser()` (`src/auth/`).
- Puertos de dominio: `MessagingPort`, `LockPort`, `AIPort`
  (`src/domain/ports/`) y sus adaptadores stub
  (`src/infrastructure/adapters/`).

## Tests y qué validan

- `tenant-context.service.spec.ts` — que el `organizationId` fijado con
  `run()` es recuperable dentro del mismo contexto asincrónico y no se filtra
  fuera de él.
- `audit.service.spec.ts` — que `AuditService.registrar` persiste un
  `RegistroAuditoria` con los campos esperados.
- `integrations.module.spec.ts` — que los tres puertos resuelven a sus
  adaptadores stub por inyección de dependencias.
- `config-tenant.service.spec.ts` — configuración por organización.

## Figuras pendientes

- Diagrama de la arquitectura hexagonal del backend (capas dominio /
  aplicación / infraestructura y dirección de las dependencias). Ver
  `tesis/figuras_pendientes.md`.
- Diagrama de secuencia del scoping multi-tenant (request → interceptor de
  tenant → `AsyncLocalStorage` → extensión de Prisma). Ver
  `tesis/figuras_pendientes.md`.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-19-config-catalogs`.
- Commits relevantes de esta fase: TASK-14 (Prisma/Postgres), TASK-16 (auth
  JWT y roles), TASK-15 (Organization y tenant scoping), TASK-17 (registro de
  auditoría), TASK-18 (puertos de integración), TASK-20 (pipeline de CI).
