# Fase 6 — Cerradura TTLock (backend) — Adaptador TTLock (TASK-55)

## Qué se implementó

Se implementó `TTLockAdapter`, la primera implementación real de
`LockPort` (declarado desde TASK-18, arquitectura hexagonal, hasta ahora
resuelto por un adaptador *stub*), contra la TTLock Open Platform API:
autenticación OAuth 2.0 de credenciales de cliente cacheada y renovada por
inquilino, creación de un passcode temporal de 6 dígitos, verificación de
si quedó instalado en la cerradura y su eliminación. Además se agregó el
modelo `AccessCode` (`CODIGO_ACCESO`), que no existía todavía en el
esquema, y se resolvió el "estado transicional" que `LockLog` arrastraba
desde TASK-34, cableando sus dos referencias hacia `Appointment` y
`AccessCode` como claves foráneas reales.

## Decisiones y por qué

**Se creó el modelo `AccessCode` en esta tarea, pese a que la propia
descripción del ticket da por hecho que TASK-34 ya lo había preparado.**
Inspeccionado el comentario real que TASK-34 dejó en `schema.prisma`
("CODIGO_ACCESO... will FK against Appointment; nothing here needs to
change for that to land"), lo que esa tarea dejó listo fue únicamente que
`Appointment` existiera como destino posible de esa clave foránea, no la
tabla en sí. En cambio, tres comentarios independientes dejados por
sesiones anteriores en archivos distintos —el "TRANSITIONAL STATE" de
`LockLog`, el *placeholder* de `AccessCodeExpirationCron` (TASK-45) y el
tipo de notificación `LOCK_ERROR` (TASK-76)— asignaban de forma consistente
la creación de `AccessCode` a esta misma tarea (M6/P6.1, TASK-55). Se
resolvió la contradicción a favor de esa evidencia interna, documentada y
repetida, en lugar de la referencia más superficial del propio ticket, y se
avisa aquí la decisión para que quede trazable.

**La firma de `LockPort.createTemporaryCode` no cambió** aunque la
descripción del ticket sugiere que el PIN se reciba como parámetro
(`crearPasscode(lockId, codigo, validoDesde, validoHasta)`). El diseño
vigente desde TASK-18 delega la generación del PIN en el propio adaptador
y devuelve `{codeId, value}`; revisar esa firma habría sido un cambio más
amplio que el alcance de esta tarea y sin motivo funcional, así que se
mantuvo, con el PIN generado con `randomInt` de `node:crypto` —no
`Math.random`— por tratarse de un valor que habilita acceso físico.

**La autenticación implementa una concesión de credenciales de cliente
estándar de OAuth 2.0 (`client_id`/`client_secret`, `grant_type=client_
credentials`), no la concesión de contraseña que usa la plataforma real de
TTLock en producción.** El propio ticket describe la autenticación como
"OAuth 2.0 (client credentials)" y enumera explícitamente sólo
`client_id`/`client_secret` como credenciales por inquilino, sin usuario ni
contraseña de cuenta TTLock — se siguió esa especificación literal en lugar
de la de la plataforma real, dejando la simplificación documentada en el
propio código para que sea fácil de corregir si una prueba contra el
entorno sandbox real la contradice.

**Las credenciales son por inquilino, leídas de `ConfigTenantService` bajo
una única clave `ttlock_config`** (`{clientId, clientSecret, lockId,
gatewayId}`), no cuatro claves sueltas — agrupa toda la configuración de
cerradura de un inquilino en un solo lugar, consistente con el pedido
explícito del ticket ("lockId, gatewayId, client_id, client_secret en la
configuración del tenant"). Sólo `clientId`/`clientSecret` los lee el
adaptador; `lockId` es el valor que un futuro llamador (TASK-56/57) leerá
de esa misma clave para pasarlo como parámetro de `LockPort`, y `gatewayId`
queda almacenado sin consumidor todavía, porque el ticket lo pide guardado
pero ningún método del puerto lo necesita (la API de TTLock enruta sola al
gateway que esté en línea).

**El token se cachea por organización y se renueva automáticamente antes
de expirar; las credenciales mismas se releen en cada llamada.** Cachear
también las credenciales habría dejado un secreto rotado por la clínica sin
efecto hasta que el proceso se reiniciara.

**Nunca se loguea el PIN, el token ni el secreto del cliente.** Cada línea
de error del adaptador lleva sólo la operación, el código HTTP y el mensaje
propio de TTLock (`errmsg`), nunca el cuerpo de la petición, que es el
único lugar donde cualquiera de los tres aparece.

**El adaptador contempla las dos formas en que TTLock informa un error**
—un rechazo a nivel HTTP y un sobre `{errcode, errmsg}` propio que puede
acompañar una respuesta 200— y en ambos casos lanza `LockOperationError`
con el código HTTP de la respuesta, satisfaciendo el criterio de aceptación
del ticket ("lanza error descriptivo con código HTTP de respuesta").

**`AccessCode` se modela como hijo de `Appointment`, sin columna de
organización propia** (patrón 2 de CLAUDE.md, igual que `License` o
`WorkingHour`): tiene un único padre acotado por inquilino del que hereda
esa garantía, y cada servicio que la toque deberá anclarse primero en la
pertenencia del turno.

**Las dos claves foráneas de `LockLog` (`appointmentId`, `accessCodeId`)
se cablean juntas y eliminan en cascada, no restringido.** Cablear una sin
la otra no habilita ninguna lectura por inquilino, porque el inquilino de
un evento de cerradura sólo puede resolverse a través del turno —el
comentario "TRANSITIONAL STATE" ya lo anticipaba así—. Se prefirió cascada
sobre restringir el borrado, a diferencia de `AuditLog`, porque `LockLog`
es un registro operativo sin la obligación de cumplimiento normativo que
sí pesa sobre la traza de auditoría, y porque restringir habría reproducido
el mismo conflicto de orden de eliminación que TASK-90 debió resolver
sobre `Notification`. La columna que antes se llamaba `turnoId` se
renombró a `appointmentId` al dejar de ser un marcador de posición, mismo
criterio que TASK-27/TASK-34 aplicaron sobre `AuditLog`.

## Entidades, puertos y adaptadores tocados

- `prisma/schema.prisma`: nuevo modelo `AccessCode` y enum
  `AccessCodeStatus` (activo/expirado/anulado); `LockLog.turnoId` renombrado
  a `appointmentId`, ambas columnas cableadas como claves foráneas reales
  (cascada); nueva migración
  `20260826180000_access_code_and_lock_log_fk` (incluye el `CHECK` de
  `validFrom <= validUntil`, escrita a mano porque Prisma 6.19 no admite
  `@@check`).
- `src/domain/ports/lock.errors.ts`: nuevo, `LockOperationError`.
- `src/infrastructure/adapters/ttlock.constants.ts`,
  `ttlock-fetch.provider.ts`, `ttlock.adapter.ts`: nuevos.
- `src/infrastructure/adapters/stub-lock.adapter.ts`: eliminado — nada más
  en el repositorio lo construye directamente, mismo criterio que los
  *stubs* de `MessagingPort`/`AIPort` cuando sus adaptadores reales
  llegaron.
- `src/infrastructure/integrations.module.ts`: `LOCK_PORT` pasa a resolver
  `TTLockAdapter`.
- `src/lock-log/lock-log.service.ts`: parámetro `turnoId` renombrado a
  `appointmentId`.
- `src/access-codes/access-code-expiration.cron.ts`,
  `access-codes.module.ts`: comentarios actualizados — el modelo y el
  adaptador ya existen, el barrido real sigue siendo TASK-57.
- `CLAUDE.md`: sección de puertos de integración y sección de
  multi-tenencia actualizadas para reflejar el adaptador real y el cierre
  del estado transicional de `LockLog`.

## Tests

- `src/infrastructure/adapters/ttlock.adapter.spec.ts`, nuevo: autenticación
  y token cacheado entre llamadas, creación/verificación/eliminación de
  passcode, error HTTP y error de aplicación de TTLock (`errcode≠0` con
  200), credenciales de inquilino ausentes, y que ningún mensaje de error
  contenga el secreto de cliente, el token o el PIN.
- `src/infrastructure/integrations.module.spec.ts`: ampliado para resolver
  `LOCK_PORT` con `TenantContextService`/`ConfigTenantService` reales,
  probando la conexión real de inyección de dependencias.
- `src/lock-log/lock-log.service.spec.ts` y
  `test/lock-log.e2e-spec.ts`: actualizados al nuevo nombre de columna;
  el caso e2e ahora construye una organización, profesional, paciente,
  turno y código de acceso reales (antes usaba un UUID inventado, posible
  sólo mientras la columna no era una clave foránea real), y se agregó un
  caso que verifica el rechazo por clave foránea inexistente.

Suite completa en verde al cierre de la tarea: 70 suites / 683 pruebas
unitarias y 43 suites / 514 pruebas de integración (`--runInBand`) contra
una base Postgres real. Lint (`eslint --fix`) y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

- Figura 46: diagrama de secuencia de la creación de un código temporal
  (adaptador → autenticación OAuth 2.0 cacheada por organización →
  generación del PIN → alta del passcode en TTLock → persistencia en
  `AccessCode`). Sección 4.7 Cerradura TTLock.
- Figura 47: diagrama entidad-relación actualizado del subdominio de
  cerradura (`AccessCode` como hijo de `Appointment`, y `LockLog` con sus
  dos claves foráneas ya reales hacia `Appointment` y `AccessCode`).
  Sección 4.7 Cerradura TTLock.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-55-ttlock-adapter`, creada desde
  `main` (990d825).
- Ticket: TASK-55 (Jira), "P6.1 – Adaptador TTLock", bajo el épico del
  Módulo 6 (Cerradura electrónica, TTLock).
- Dependencias declaradas por el ticket: TASK-13 (arquitectura hexagonal,
  ya fusionada) y TASK-34 (P3.1, entidad `Appointment`, ya fusionada) — no
  la entidad `CODIGO_ACCESO`, que esta misma tarea creó, según se explica
  arriba.
- Fuera de alcance, declarado en el propio ticket y respetado: la
  integración con el flujo de turnos, es decir *cuándo* se crea el código
  (TASK-56); el barrido real de vencimiento (TASK-57, M6/P6.3); la
  tolerancia a fallos de conectividad con reintentos (TASK-59, M6/P6.5).
