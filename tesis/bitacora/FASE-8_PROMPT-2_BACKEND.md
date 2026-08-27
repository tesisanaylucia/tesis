# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Seguridad: guards, rate limiting, secretos y cifrado en reposo (TASK-66, P8.1)

## Qué se implementó

Se implementó P8.1 ("Seguridad") del SRS: (1) revisión de completitud de
los guards de autorización en todos los endpoints del backend; (2) límite
de tasa de peticiones (`@nestjs/throttler`) en `POST /auth/login` y
`POST /webhook`, configurable por variable de entorno; (3) verificación de
que ningún secreto vive en el código fuente, ampliada con un escaneo
automático de secretos en la integración continua; (4) cifrado en reposo
con AES-256-GCM de `AccessCode.ttlockPasscodeId`, y la decisión —explícita,
no un pendiente— de no cifrar a nivel de columna `Patient.birthDate`,
`Patient.mobilePhone` ni `PatientProfessional.observaciones`.

## Decisiones y por qué

**La revisión de guards no encontró vacíos que corregir.** Se recorrió cada
controlador del repositorio (`@Controller`) comprobando la combinación de
`@Public()`, `@Roles(...)` y los guards de pertenencia
(`ProfessionalOwnershipGuard`/`ProfessionalSelfGuard`) contra lo que cada
ruta expone. Todas las rutas de lectura abiertas a más de un rol sin
`@Roles` explícito ya llevaban, de tareas anteriores, un comentario en el
propio controlador razonando por qué esa apertura es intencional (por
ejemplo, `AvailabilityController`/`WorkingHoursController`: la
disponibilidad y el horario de un profesional deben poder consultarse por
cualquier usuario autenticado del tenant, porque tanto la reserva de turnos
como el futuro chatbot los necesitan). La verificación cruzada de
`organizationId` no se implementa por ruta, sino que es una propiedad
transversal del acceso a la base de datos: la extensión de Prisma de
alcance por tenant (`tenant-scoping.extension.ts`) filtra y sella
automáticamente cada operación por la organización del token autenticado,
de modo que una fila de otra organización nunca es alcanzable desde
ninguna ruta, independientemente del guard puesto en ella.

**El acceso entre tenants responde 404, no 403, pese a que el texto del
ticket pide 403.** Esta es una decisión ya tomada y documentada en tareas
anteriores (TASK-78, TASK-94), no una decisión nueva de esta tarea: dado
que la extensión de alcance por tenant hace que una fila de otra
organización sea indistinguible de una fila inexistente en cualquier
consulta, responder 403 en ese caso revelaría que el recurso existe en
alguna organización, que es en sí mismo el dato que se busca no exponer.
Esta tarea se limitó a dejar una prueba de extremo a extremo única y
representativa de ese comportamiento (más el caso de rol incorrecto, que sí
responde 403 con el guard de roles) en lugar de duplicar esa cobertura en
cada módulo, que ya la tenía por separado.

**El límite de tasa se aplicó solo en los dos puntos que el SRS nombra, no
de forma global.** Se registró `ThrottlerModule` una única vez, con dos
perfiles nombrados (`auth-login`, `webhook`), pero deliberadamente sin
vincularlo como guard global: cada uno de los dos controladores lo activa
individualmente con `@UseGuards(ThrottlerGuard)` y descarta el perfil que
no le corresponde con `@SkipThrottle()`, de modo que el resto de la API
—incluida la app móvil del profesional y el propio flujo conversacional—
no queda sujeta a ningún límite. Ambos perfiles son configurables por
variable de entorno, con los valores de la especificación (diez intentos
por minuto, cien peticiones por segundo) como valor por defecto en el
código si la variable no está presente.

**Los límites por defecto tuvieron que elevarse en el entorno de pruebas y
de integración continua, no en el código de producción.** Al ejecutar la
suite completa de pruebas de extremo a extremo con el límite de diez
intentos de inicio de sesión por minuto activo, ochenta y ocho pruebas
distribuidas en diez archivos empezaron a fallar con "no autorizado": cada
archivo de prueba inicia sesión varias veces (una por combinación de
organización y rol que necesita), y varios archivos superan las diez
llamadas dentro del minuto que dura su propia ejecución. Se optó por
elevar el límite mediante la variable de entorno solo en el archivo `.env`
local y en la integración continua —dejando el valor por defecto del
código, el que efectivamente rige en producción, sin tocar— y por escribir
una prueba de extremo a extremo separada
(`test/security-hardening.e2e-spec.ts`) que restablece el valor real de la
especificación sobre su propia instancia aislada de la aplicación,
únicamente para poder demostrar el límite verdadero sin interferir con el
resto de la suite.

**El cifrado en reposo se acotó al identificador de la cerradura, con una
implementación puntual en dos servicios en lugar de una extensión genérica
de Prisma.** Se evaluó envolver el cliente de Prisma con una extensión que
cifrara y descifrara el campo de forma transparente, siguiendo el mismo
patrón que ya usa el alcance por tenant, pero se descartó: varias pruebas
de extremo a extremo ya existentes construyen y leen filas de
`AccessCode` directamente contra el cliente de Prisma sin alcance de
tenant, para preparar el estado de la prueba, y aplicar el cifrado a nivel
de cliente hubiera exigido modificar esas construcciones de datos de
prueba sin necesidad real, además de acoplar una decisión de seguridad de
un único campo a la infraestructura compartida de acceso a datos. Se optó,
en cambio, por cifrar y descifrar explícitamente en los dos únicos lugares
del código que tocan la columna —el servicio que genera y revoca códigos,
y el trabajo programado que los expira—, cada uno resolviendo la clave de
cifrado una sola vez desde la variable de entorno al construirse. El campo
numérico que la paciente efectivamente digita en la cerradura no se cifró:
ya estaba excluido de cualquier registro por una decisión de una tarea
anterior, y su ventana de validez corta reduce el valor de un cifrado
adicional.

**La ampliación de la clave foránea compuesta única a tres columnas quedó
sin efecto práctico sobre el nuevo campo cifrado.** `AccessCode` conserva
una restricción de unicidad sobre `ttlockPasscodeId`; al cifrarse con un
vector de inicialización distinto en cada escritura, dos códigos con el
mismo identificador real ya no producen el mismo valor almacenado, por lo
que esa restricción deja de detectar una duplicación real y pasa a
garantizar solo que el texto cifrado en sí no se repita. Se documentó esa
consecuencia en el propio comentario del campo en `schema.prisma` como una
compensación aceptada, no un defecto: la garantía original era una defensa
adicional sobre una propiedad que la propia plataforma de la cerradura ya
asegura (el identificador que devuelve es global y no se reutiliza), no el
único mecanismo que la sostenía.

## Entidades / puertos / adaptadores tocados

- `src/common/crypto/field-encryption.ts` (nuevo): primitivas de cifrado
  autenticado AES-256-GCM, sin conocimiento del dominio.
- `src/access-codes/access-code-encryption.ts` (nuevo): resolución de la
  clave de cifrado desde la variable de entorno, compartida por los dos
  únicos consumidores.
- `src/access-codes/access-code.service.ts` y
  `access-code-expiration.cron.ts`: cifran al escribir y descifran al leer
  `ttlockPasscodeId`.
- `src/app.module.ts`: registro de `ThrottlerModule` con los dos perfiles
  nombrados.
- `src/auth/auth.controller.ts` y
  `src/whatsapp/whatsapp-webhook.controller.ts`: activación puntual del
  guard de límite de tasa sobre su propio perfil.
- `prisma/schema.prisma`: comentario ampliado sobre `AccessCode.ttlockPasscodeId`
  documentando el cifrado y la consecuencia sobre la restricción de
  unicidad.
- `.github/workflows/ci.yml`: paso de escaneo de secretos y generación de
  una clave de cifrado efímera para la propia ejecución de la integración
  continua (no un secreto real).
- `CLAUDE.md`: sección de Seguridad ampliada con las cinco decisiones de
  esta tarea.

## Tests

- `src/common/crypto/field-encryption.spec.ts` (nuevo): cifrado/descifrado
  reversible, no determinismo del texto cifrado, rechazo con clave
  incorrecta y con texto cifrado alterado.
- `src/access-codes/access-code.service.spec.ts`: casos existentes
  adaptados para verificar que el valor persistido nunca es el
  identificador real sino que lo recupera al descifrarlo; nueva sección
  que espía cada método de registro de eventos para comprobar que el
  código numérico nunca aparece en ningún mensaje registrado.
- `src/access-codes/access-code-expiration.cron.spec.ts`: adaptado al
  mismo cifrado en el flujo de expiración.
- `test/access-code-generation.e2e-spec.ts` y
  `test/access-code-invalidation-expiration.e2e-spec.ts`: adaptados para
  descifrar antes de comparar contra el identificador real que el
  adaptador simulado de la cerradura devuelve.
- `test/security-hardening.e2e-spec.ts` (nuevo): acceso entre
  organizaciones con token válido (404), acceso con rol no autorizado
  (403), undécimo intento de inicio de sesión en la misma ventana (429) y
  petición ciento uno al punto de entrada de WhatsApp en la misma ventana
  (429).

Suite completa en verde al cierre: 74 suites unitarias (738 pruebas) y 47
suites de extremo a extremo (541 pruebas) contra PostgreSQL real, lint y
verificación de tipos sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-66-security-hardening`, creada desde
  `main` para esta tarea. Sin commitear al cierre — pendiente de
  autorización explícita de la autora antes de commitear/pushear, según lo
  indicado para esta sesión.
