# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Revocación de tokens JWT (TASK-87, P8.1b, corrección/extensión a P8.1)

## Qué se implementó

Se implementó P8.1b ("Revocación de tokens JWT"), una tarea de corrección
sobre P8.1 (TASK-66, seguridad): un mecanismo para que un token JWT ya
emitido deje de ser válido antes de su expiración natural, cuando (a) el
profesional/usuario vinculado se desactiva, o (b) el usuario cierra sesión
explícitamente. El gap venía documentado literalmente en `CLAUDE.md`
("A JWT already issued stays valid until it expires — token revocation is
not implemented") desde que P8.1 se cerró sin cubrirlo, porque el propio
ticket de esa tarea nunca lo mencionaba.

Se agregó una columna `tokenVersion` (entero, valor por defecto cero) al
modelo `User`, copiada dentro del payload firmado de cada token en el
momento de emitirlo, y comparada contra el valor actual de la columna en
cada petición autenticada (`JwtStrategy.validate`): una discrepancia
responde 401, sin importar que la firma y la expiración del token sigan
siendo válidas. Dos puntos incrementan la columna: la desactivación de un
profesional (dentro de la misma transacción que ya escribía la baja lógica
y el asiento de auditoría) y un endpoint nuevo de cierre de sesión
explícito, `POST /auth/logout`.

## Decisiones y por qué

**Se optó por `tokenVersion`, la alternativa más simple que el propio
ticket ofrecía, en lugar de tokens de refresco rotativos con
almacenamiento server-side.** El ticket dejaba ambas opciones abiertas,
con la lista de revocación por versión como la explícitamente más simple;
dado el volumen esperado del piloto (una única organización, un plantel
acotado de profesionales), el costo adicional de una consulta a la tabla
`User` por petición autenticada es aceptable frente a la complejidad de
emitir, rotar y persistir un segundo tipo de token. Esa relación de
compromiso quedó documentada tanto en el comentario del propio campo en
`schema.prisma` como en `CLAUDE.md`.

**La verificación de `tokenVersion` corre antes de que exista contexto de
inquilino, igual que las dos comprobaciones ya existentes de
`AuthService.login`.** `JwtStrategy.validate` se ejecuta antes que el
interceptor que abre el `AsyncLocalStorage` del inquilino (ese interceptor
lee el resultado de `validate` para decidir qué organización abrir), así
que la búsqueda del usuario no puede pasar por el cliente de Prisma
acotado por tenant sin lanzar `MissingTenantContextError`. Se agregó
`UsersService.findById`, un tercer método sobre el cliente crudo de
Prisma, documentado con el mismo razonamiento que ya cubre a
`findByEmail`/`isLinkedProfesionalActive` desde TASK-120: es seguro sin
filtro manual porque `id` es la clave primaria global, no puede resolver
en una fila de otra organización aunque no se filtre por tenant.

**La revocación al desactivar un profesional incrementa la versión de
todas las cuentas vinculadas, no solo una.** `User.professionalId` no
lleva `@unique` en el esquema — nada impide, a nivel de modelo, que más de
una cuenta apunte al mismo profesional — así que
`UsersService.revokeTokensForProfessional` usa `updateMany` sobre el
filtro por `professionalId` en lugar de `update` sobre un identificador
supuesto único. La escritura se hizo dentro de la misma transacción que ya
envolvía la baja lógica y el asiento de auditoría en
`ProfessionalsService.deactivate`, con el mismo razonamiento que sostiene
esa transacción desde que se escribió: una desactivación que no llega a
comprometerse no puede dejar tokens revocados de un profesional que sigue
activo.

**El cierre de sesión explícito no es una ruta pública.** Necesita saber
qué cuenta revocar, y la única fuente confiable de esa identidad es el
propio token que se está invalidando — así que `POST /auth/logout` exige
el mismo bearer token válido que cualquier otra ruta protegida, en lugar
de aceptar, por ejemplo, un correo electrónico en el cuerpo del pedido.
Queda auditado (`entity: 'User'`, `action: 'LOGOUT'`) como toda acción de
control de acceso según la sección de auditoría de `CLAUDE.md`, pero de
forma autónoma y no dentro de la misma transacción que la revocación: a
diferencia de la desactivación, aquí no hay nada que revertir si el
asiento de auditoría falla — un token ya revocado no vuelve a ser válido
por no haberse podido registrar que se revocó.

**No existe un mecanismo separado de "forzar cierre de sesión" para el
caso de sospecha de compromiso de credenciales que menciona el ticket.**
Se reutiliza el mismo camino de revocación de una cuenta propia: sospechar
que las credenciales de una cuenta están comprometidas y revocar su
`tokenVersion` es exactamente la misma operación que un cierre de sesión
explícito, aplicada por quien administra el sistema en lugar de por la
propia usuaria. No había en el alcance del ticket un endpoint
administrativo distinto que lo pidiera, así que no se construyó uno
todavía.

**Un defecto real en la infraestructura de pruebas de extremo a extremo,
descubierto al validar el cambio.** Siete archivos de prueba ya existentes
(`patients-abmc`, `patients-import`, `patient-notes`, `patient-consent`,
`appointments-booking`, `appointments-states`,
`appointments-rescheduling`) simulan la única forma en que hoy se emite un
token `SYSTEM` firmándolo directamente con `JwtService`, porque
`AuthService.login` rechaza ese rol — no hay un camino real de inicio de
sesión que lo emita. Ninguno de esos siete incluía `tokenVersion` en el
payload simulado, así que trece pruebas repartidas en esos archivos
empezaron a fallar con 401 en lugar del código esperado (200, 201 o 403,
según el caso) apenas se activó la comprobación. Se corrigió agregando
`tokenVersion: systemUser.tokenVersion` a cada uno de los siete — el
mismo campo que ya leían del usuario recién creado para el resto del
payload —, sin tocar ningún otro comportamiento de esas suites. No hay
ningún punto del código de producción que emita un token `SYSTEM` de esta
forma; el patrón es exclusivo de las pruebas.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: columna `tokenVersion` (entero, por defecto
  cero) en `User`, con el comentario que documenta el mecanismo completo y
  la relación de compromiso aceptada.
- `prisma/migrations/20260827183658_add_user_token_version/`: migración
  generada por diferencia de esquema contra una base sombra descartable
  (`prisma migrate dev` sigue sin poder correr de forma no interactiva en
  este entorno, mismo rodeo documentado en tareas anteriores) y aplicada
  con `prisma migrate deploy`.
- `src/auth/interfaces/jwt-payload.interface.ts`: campo `tokenVersion`
  agregado al payload firmado.
- `src/auth/strategies/jwt.strategy.ts`: `validate` pasa a ser asíncrono,
  busca el usuario por id y compara `tokenVersion`.
- `src/auth/auth.service.ts`: `login` copia `tokenVersion` del usuario al
  payload; método nuevo `logout`.
- `src/auth/auth.controller.ts`: endpoint nuevo `POST /auth/logout`
  (204, protegido).
- `src/users/users.service.ts`: métodos nuevos `findById`,
  `revokeTokensForProfessional`, `revokeToken`; tipo `UserWriter`
  (mismo patrón que `AuditLogWriter`/`ConfigWriter`) para aceptar un
  handle de transacción opcional.
- `src/professionals/professionals.service.ts` y `.module.ts`:
  `deactivate` revoca los tokens del profesional dentro de su propia
  transacción; el módulo importa `UsersModule` para poder inyectar
  `UsersService`.
- `CLAUDE.md`: sección de Seguridad, reemplaza la mención del gap por la
  descripción del mecanismo implementado.

## Tests

- `src/auth/strategies/jwt.strategy.spec.ts` (nuevo): acepta un
  `tokenVersion` que coincide, rechaza uno atrasado (revocado) y rechaza
  un usuario que ya no existe.
- `src/auth/auth.service.spec.ts`: caso nuevo que verifica que
  `tokenVersion` viaja en el payload firmado; sección nueva para
  `logout` (revoca y audita).
- `src/users/users.service.spec.ts`: casos nuevos para `findById`
  (cliente sin acotar por tenant), `revokeTokensForProfessional`
  (`updateMany` por `professionalId`, con y sin handle de transacción
  explícito) y `revokeToken`.
- `src/professionals/professionals-deactivate.service.spec.ts` (nuevo,
  separado de `professionals.service.spec.ts` porque ese archivo ya
  declara su alcance como exclusivo de `updateConfiguration`): primera
  cobertura unitaria de `deactivate` — baja lógica, revocación de tokens
  y auditoría, las tres dentro del mismo handle de transacción; y el caso
  404 sin ningún efecto secundario.
- `test/professionals-abm.e2e-spec.ts`: la prueba existente "refuses
  login to a professional that was deactivated" se amplió con la
  aserción que el ticket pedía explícitamente — el token emitido *antes*
  de la desactivación, todavía sin expirar, es rechazado con 401 en la
  petición siguiente.
- `test/jwt-token-revocation.e2e-spec.ts` (nuevo): las dos aserciones
  restantes del ticket contra Postgres real — una cuenta sin cambios en
  `tokenVersion` sigue funcionando en pedidos sucesivos, y el cierre de
  sesión explícito invalida exactamente ese token sin afectar a otra
  cuenta ni impedir un inicio de sesión posterior — más un caso adicional
  que un test unitario no puede alcanzar: un token sin `tokenVersion` en
  absoluto en su payload (el caso de un token emitido antes de que la
  columna existiera) se rechaza igual que uno con el valor equivocado.
- Siete archivos de prueba existentes corregidos (ver la sección de
  decisiones): `tokenVersion` agregado al payload simulado del token
  `SYSTEM`.

Suite completa en verde al cierre: 77 suites unitarias (764 pruebas) y 50
suites de extremo a extremo (551 pruebas) contra PostgreSQL real,
verificación combinada de cobertura (127 suites/1315 pruebas) sin bajar
del umbral, lint y verificación de tipos sin errores.

## Figuras pendientes

Una nueva — ver `figuras_pendientes.md`: diagrama de secuencia de la
verificación de `tokenVersion` en cada petición autenticada, con sus dos
disparadores de revocación.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-87-jwt-revocation`, creada desde
  `main` para esta tarea. Sin commitear al cierre — pendiente de
  autorización explícita de la autora antes de commitear/pushear, según
  lo indicado para esta sesión.
