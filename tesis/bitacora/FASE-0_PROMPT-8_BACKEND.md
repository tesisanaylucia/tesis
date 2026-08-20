# Fase 0 — Fundaciones (backend) — `UsersService` migrado al cliente acotado por tenant (TASK-120)

## Qué se implementó

Se migró `UsersService` de consultar la base directamente con
`PrismaService` a hacerlo, salvo dos excepciones documentadas, a través de
`TenantScopedPrismaService` — el mismo cliente extendido que aplica el
acotamiento automático por organización a cualquier otro modelo con
`organizationId`. Antes de esta tarea, `UsersService` era la única pieza
del código donde ese acotamiento dependía de que cada método filtrara a
mano por `organizationId`; el método `findByOrganization` (único método de
lectura múltiple del servicio) lo hacía correctamente, recibiendo el
identificador de organización de su único invocador (`UsersController`),
pero un método nuevo que se agregara sin recordar ese filtro habría
quedado sin ninguna protección, en tiempo de desarrollo o de ejecución. Una
auditoría de código del 14 de agosto de 2026 (agente "Audit
notifications/AI/security/legal vs SRS") señaló esta grieta como la única
conocida en un mecanismo que en el resto del sistema es automático.

Se migró `findByOrganization`, renombrado a `findAll()`, al cliente
acotado, y se retiró su parámetro `organizationId`: una vez que la
extensión inyecta el filtro a partir del contexto de la solicitud, un
parámetro explícito con el mismo propósito es redundante y reabre la
posibilidad de que ambos valores diverjan — precisamente el riesgo que la
migración busca cerrar. `UsersController.findAll` se ajustó en
consecuencia, dejando de necesitar el usuario autenticado.

Dos métodos quedaron deliberadamente sobre el cliente sin extender:
`findByEmail` e `isLinkedProfessionalActive`. Ambos se invocan desde
`AuthService.login()` antes de que exista un JWT y, por lo tanto, antes de
que haya una organización en el contexto de la solicitud —
`TenantScopedPrismaService` directamente lanzaría
`MissingTenantContextError` si se intentara usarlo ahí—. Cada uno quedó
documentado en el código explicando por qué es seguro sin un filtro manual
de todos modos, y no solo por necesidad: `email` es una columna con
restricción de unicidad global en el esquema (`User.email @unique`), no
solo por organización, de modo que no existe una fila de otra organización
con la que pueda confundirse; y `professionalId` es una clave primaria
única en toda la base, con una clave foránea compuesta ya vigente
(`User.professional`) que garantiza a nivel de esquema que el profesional
vinculado a una cuenta pertenece siempre a la misma organización que esa
cuenta — un filtro adicional no podría cambiar qué fila devuelve la
consulta.

## Decisiones y por qué

**Retirar el parámetro `organizationId` de `findAll()` en lugar de
conservarlo junto al filtro automático.** Sostener ambos —el parámetro
explícito y el que la extensión agrega por contexto— no aporta ninguna
garantía adicional (el segundo siempre prevalece, ya que la extensión
sobrescribe el `where` con su propio valor) y sí dejaba abierta la
pregunta de qué pasaría si algún día llegaran a no coincidir. El patrón ya
existente en el resto del código (`HolidaysService.findAll()`,
`ProfessionalsService.findAllActive()`) tampoco recibe un identificador de
organización explícito por la misma razón, así que el cambio también
alinea a `UsersService` con la convención vigente en el resto del
repositorio.

**Documentar la excepción en el código, con el argumento específico de
cada método, en lugar de una nota genérica de "esto corre antes del
login".** Que un método corra antes de que exista contexto de tenant
explica por qué *no puede* usar el cliente acotado, pero no por qué es
*seguro* no filtrar manualmente. Ambas afirmaciones son necesarias, y son
independientes: un método sin contexto de tenant que filtrara por una
columna no única por organización sí sería un problema. Se optó por dejar
registrado, para cada uno de los dos métodos, cuál es la propiedad de la
columna consultada que ya lo acota a una única organización sin necesidad
de un filtro explícito.

**Formalizar la excepción en `CLAUDE.md` como una categoría propia, y no
solo como comentarios inline.** El documento de convenciones del
repositorio ya reconocía tres patrones para el diseño de un *modelo* nuevo
respecto del acotamiento por tenant (pertenencia directa, hijo de un padre
acotado, vínculo entre dos padres acotados). Esta excepción es de otra
naturaleza — no depende de cómo se modela `User`, que sigue siendo un
modelo acotado directamente por organización como cualquier otro, sino de
en qué momento del ciclo de una solicitud se ejecuta un método concreto —,
así que se agregó como un párrafo aparte en la sección de multi-tenancy,
siguiendo el mismo estilo que ya usan las excepciones existentes
(`LockLog`, `License`/`LicensesService`): qué se exceptúa, por qué, y qué
argumento necesitaría un caso nuevo para calificar también.

## Alternativas descartadas

Se consideró mantener `findByOrganization(organizationId)` con su
parámetro, ahora redundante con el filtro automático, en lugar de
retirarlo — evitaba tocar la firma pública del método a cambio de dejar
sin resolver la posibilidad de divergencia entre ambos valores descripta
arriba. Se descartó por ser exactamente el tipo de brecha silenciosa que
esta misma tarea busca cerrar, y porque el único invocador existente
(`UsersController`) no perdía nada al dejar de pasarlo.

Se consideró también aplicar un filtro manual adicional por
`organizationId` en `isLinkedProfessionalActive`, usando el
`organizationId` ya disponible en el `user` que `AuthService.login()`
acaba de resolver, como medida de defensa en profundidad. Se descartó por
no aportar ninguna garantía que la clave foránea compuesta
`User.professional` no aportara ya a nivel de esquema, y por requerir
cambiar la firma del método para aceptar un segundo parámetro sin ningún
caso en el que ese parámetro pudiera alguna vez cambiar el resultado.

## Entidades / puertos / adaptadores tocados

- `src/users/users.service.ts`: `findByOrganization(organizationId)` →
  `findAll()`, migrado a `TenantScopedPrismaService`; `findByEmail` e
  `isLinkedProfessionalActive` se mantienen sobre `PrismaService`, con
  comentarios ampliados documentando por qué cada uno es seguro sin
  acotamiento automático.
- `src/users/users.controller.ts`: `findAll` deja de requerir el usuario
  autenticado, ya innecesario tras el cambio anterior.
- `CLAUDE.md`: nuevo párrafo en "Multi-tenancy" que formaliza esta
  excepción a nivel de método de servicio, distinta de los tres patrones
  de diseño de modelo ya documentados.

## Tests y qué validan

- `src/users/users.service.spec.ts` (nuevo): cobertura unitaria de los tres
  métodos con los dos clientes mockeados por separado — `findByEmail` e
  `isLinkedProfessionalActive` consultan el cliente sin extender y nunca
  tocan el acotado; `findAll()` consulta el cliente acotado sin ningún
  argumento, verificando explícitamente que no hay ningún filtro manual de
  organización que un desarrollador pudiera omitir por error.
- `test/tenant-scoping.e2e-spec.ts` (preexistente, sin cambios): ya cubre
  de forma genérica, para cualquier modelo con `organizationId` —incluido
  `User`—, que una consulta sin filtro explícito a través del cliente
  acotado solo devuelve filas de la organización en contexto. Es la prueba
  contra una base de datos real que sostiene el criterio de aceptación del
  ticket ("un método nuevo hipotético... queda automáticamente scoped"),
  no reproducida en el nuevo archivo unitario para no duplicar contra un
  mock lo que ya se verifica contra Postgres.
- `test/auth.e2e-spec.ts` (preexistente, sin cambios): `GET /users` sigue
  devolviendo solo los usuarios de la organización del token, ahora por la
  vía automática en lugar del filtro manual retirado.
- Suite completa verde: 40 suites / 468 tests unitarios, y 38 suites / 455
  tests end-to-end con `--runInBand` contra PostgreSQL local (la corrida en
  paralelo por defecto mostró fallas intermitentes en suites de turnos y
  pacientes no relacionadas con este cambio, por contención sobre la base
  de datos compartida entre workers; desaparecen al correr secuencial).
  Lint (`eslint --max-warnings=0`) y `tsc --noEmit` sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-120-users-service-tenant-scoping`
  (creada a partir de `origin/main` fresco, `b74179a`, con TASK-119 ya
  fusionado).
- Ticket: TASK-120 ("[CORRECCIÓN] UsersService usa PrismaService crudo en
  vez de TenantScopedPrismaService"), corrección sobre el estado del
  código descripto en la propia auditoría del 14 de agosto de 2026 que la
  originó, sin un ticket de implementación previo específico al que
  corregir — `UsersService` viene de P0.5 (TASK-16) sin haber sido tocado
  desde entonces salvo por TASK-93.
