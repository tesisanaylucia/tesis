# Fase 6 — Cerradura TTLock (backend) — adaptador hardcodeado con códigos aleatorios para pruebas, detrás de `LOCK_ADAPTER` (TASK-211)

## Qué se implementó

TASK-211 ("hardcodear codigo de cerradura") no corrige un defecto: atiende
una restricción material del proyecto. El adaptador real del `LockPort`
—`TTLockAdapter`, construido en P6.1 (TASK-55) y endurecido por P6.5
(TASK-59)— está terminado, pero no existe todavía ni la cerradura física
ni el gateway ni una cuenta de la Open Platform de TTLock contra la cual
apuntarlo. Mientras esa configuración no exista, el flujo completo "turno
confirmado → `CODIGO_ACCESO` activo → PIN enviado al paciente por
WhatsApp" no podía ejercitarse en absoluto: `resolveLockId` no encontraba
la clave `ttlock_config` del tenant, lanzaba `LockOperationError`,
`generateForAppointment` lo registraba como `ACCESS_CODE_ERROR` y devolvía
`null` —el comportamiento correcto ante una configuración ausente— y
ningún código llegaba nunca al paciente. El módulo quedaba, en la
práctica, no probable de punta a punta.

Se agregó `HardcodedLockAdapter`, una implementación del mismo `LockPort`
que genera los códigos localmente y no habla con ningún servicio externo.
Se liga a `LOCK_PORT` en lugar de `TTLockAdapter` únicamente cuando la
variable de entorno `LOCK_ADAPTER` vale exactamente `hardcoded`; sin
definir, con el valor `ttlock` o con cualquier otro valor queda ligado el
adaptador real. Cuando está activo, cada generación de código produce un
PIN aleatorio de seis dígitos y un identificador opaco propio, y la
verificación de instalación se responde contra un registro en memoria de
lo que el propio proceso emitió.

## Decisiones y por qué

**El disparador del envío no se movió.** El pedido original podía leerse
como "enviar el código cuando se *solicita* un turno". Consultada la
cuestión, se mantuvo el disparador vigente: `AppointmentsService.confirm`
sigue siendo el único llamador de `generateForAppointment`, conforme al
propio P6.2 ("una vez que `CODIGO_ACCESO` queda en estado=activo... enviar
al paciente"), de modo que reservar un turno sigue sin producir código y
confirmarlo produce uno aleatorio. La alternativa —generar también al
reservar— habría alterado la máquina de estados del turno y el requisito
del SRS para resolver un problema que es de configuración de entorno, no
de diseño del flujo; se descartó por eso.

**Una variable de entorno global, no una clave de `OrganizationConfig`.**
Todo el resto de la configuración de la cerradura es por tenant
(`ttlock_config`: `clientId`, `clientSecret`, `lockId`, `gatewayId`),
porque cada clínica posee su propia cerradura y su propia cuenta. Qué
adaptador está ligado, en cambio, es una propiedad del *entorno* que se
está probando y no de la configuración de un inquilino: dejarlo como clave
por tenant permitiría que una clínica pusiera su propia puerta sobre una
cerradura falsa, que es exactamente el riesgo que esta tarea debe evitar
mientras exista. Se descartó por eso, y la variable se documentó en
`.env.example` con la advertencia explícita de no usarla en un entorno que
atienda a una clínica real.

**El valor por defecto falla hacia el adaptador real.** La comprobación
exige la cadena exacta `hardcoded`; cualquier otro valor, incluido un
error de tipeo o la variable sin definir, deja ligado `TTLockAdapter`. La
dirección del fallo no es arbitraria: la bandera decide si los códigos de
acceso físico son reales, así que un error de configuración debe degradar
hacia la integración verdadera —que como máximo fallará de forma visible—
y nunca hacia una cerradura falsa que reporta éxito sobre una puerta que
nunca se abrió.

**El adaptador imita la forma del real, no su contenido.** El PIN se sortea
con el mismo generador criptográfico y la misma longitud de seis dígitos
que `TTLockAdapter.generatePin`, en lugar de ser un literal fijo, por dos
razones concretas y no por prolijidad: el PIN se chequea contra los demás
códigos activos con ventana de validez superpuesta (CER 5, TASK-170), de
modo que un valor constante colisionaría consigo mismo en el segundo turno
y agotaría los reintentos de ese chequeo; y la unicidad de la longitud es
lo que sostiene el cálculo de rareza de colisión (una en novecientas mil
por sorteo) que esa misma corrección documenta. El identificador opaco se
sortea como UUID y nunca se reutiliza, igual que el `keyboardPwdId` de
TTLock es un handle global. El adaptador reporta además un intento por
llamada a través del mismo callback `onAttempt` que la política de
reintentos de TASK-59 usa, de modo que la traza operativa en `LockLog`
—una fila `CODE_ATTEMPT` por operación— conserva su forma en los dos
modos.

**El PIN sigue sin registrarse en ninguna línea de log.** Una cerradura
falsa no es razón para empezar a registrar PINs: las filas de
`CODIGO_ACCESO` y los mensajes al paciente que este modo produce son por
lo demás indistinguibles de los de producción —el PIN se cifra en reposo
igual que siempre (TASK-190)—, así que el criterio de P8.1 se mantiene sin
excepción. Las líneas de advertencia que el adaptador emite nombran el
identificador del código y la cerradura, nunca los dígitos.

**El modo se marca como activamente incorrecto, no como un simple stub.**
Es la diferencia relevante con los stubs que precedieron a los adaptadores
reales (`StubLockAdapter`, `StubMessagingAdapter`): aquéllos respondían
valores inertes en entornos sin integración, mientras que éste convive con
un flujo que cree estar operando una puerta. Ningún passcode llega a una
cerradura, y `remoteUnlock` —la apertura manual que RF-CE-07 nombra como
alternativa al código ad-hoc (CER 3, TASK-168)— reporta éxito sin abrir
nada. Por eso la ligadura emite una advertencia al arranque, cada llamada
emite la suya, y todo `AccessCode.ttlockPasscodeId` producido en este modo
lleva el prefijo `hardcoded-`: una fila así queda identificable en la base
de datos cuando se vuelva al adaptador real, donde ese handle no
corresponde a ningún passcode que TTLock pueda eliminar.

**`resolveLockId` admite un `lockId` de reemplazo sólo mientras la bandera
está activa.** Sin eso, el flujo que la bandera existe para ejercitar
seguiría fallando antes de llegar al puerto, con el mismo
`ACCESS_CODE_ERROR` que produce una configuración real ausente. Si el
tenant sí tiene un `lockId` configurado, ése prevalece: el adaptador
hardcodeado ignora qué cerradura se le indica, pero los logs y las filas
de `LockLog` siguen nombrando la puerta real de la clínica. La función
recibió por eso un segundo parámetro (el `ConfigService` global desde donde
se lee la bandera), propagado desde sus dos llamadores —
`AccessCodeService` y `AccessCodeExpirationCron`—, que ya recibían ese
servicio para resolver la clave de cifrado.

**La selección se resuelve con una factoría y los dos adaptadores
registrados, no construyendo el elegido a mano.** Cada uno tiene su propia
lista de dependencias —el real necesita el enlace de `fetch`,
`ConfigTenantService` y el contexto de tenant; el hardcodeado, ninguna—, y
dejar que el contenedor de inyección resuelva ambos mantiene esa lista
fuera del módulo. Instanciar el que no se usa cuesta un objeto con un mapa
vacío, o un adaptador cuyo constructor sólo lee dos valores de
configuración: ninguno abre conexiones ni llama a nada. La advertencia de
arranque se ubicó en la factoría y no en el constructor del adaptador, de
modo que el adaptador no necesita conocer la bandera que lo selecciona y la
línea se emite una sola vez, para el modo realmente vigente.

## Entidades / puertos / adaptadores tocados

`LockPort` (`src/domain/ports/lock.port.ts`) — sin cambios en la interfaz:
el adaptador nuevo implementa los cuatro métodos ya declarados
(`createTemporaryCode`, `deleteCode`, `verifyCodeInstalled`,
`remoteUnlock`), que es justamente lo que el patrón de puertos y
adaptadores hace posible sin tocar la lógica de negocio.

`HardcodedLockAdapter` y sus constantes
(`src/infrastructure/adapters/hardcoded-lock.adapter.ts`,
`hardcoded-lock.constants.ts`) — nuevos. `IntegrationsModule`
(`src/infrastructure/integrations.module.ts`) — la ligadura de `LOCK_PORT`
pasa de `useClass` a una factoría que elige entre los dos adaptadores.
`resolveLockId` (`src/access-codes/resolve-lock-id.ts`) — segundo
parámetro y reemplazo del `lockId` ausente mientras la bandera está
activa. `AccessCodeService` y `AccessCodeExpirationCron` — propagan el
`ConfigService` que ya recibían. Sin cambios de esquema de base de datos.

## Tests agregados o modificados

`hardcoded-lock.adapter.spec.ts` (nuevo): cubre la forma del PIN (seis
dígitos, sin cero inicial), que el identificador opaco no contiene los
dígitos y lleva el prefijo previsto, que veinticinco generaciones
concurrentes no repiten identificador y no devuelven un PIN constante, el
reporte de un único intento exitoso por llamada, la verificación de
instalación afirmativa para un código propio y negativa para un handle
desconocido o para el mismo código pedido sobre otra cerradura, la
eliminación idempotente (de un código inexistente y de uno ya eliminado),
la apertura remota que resuelve sin abrir nada, y que el PIN no aparece en
ninguna línea de log. Cubre además la bandera en sí: verdadera sólo para
la cadena exacta, ignorando mayúsculas y espacios, y falsa para cualquier
otro valor, incluida la variable sin definir.

`integrations.module.spec.ts`: el armado del módulo de pruebas se extrajo
de `beforeEach` a una función que acepta configuración adicional, para que
el mismo grafo pueda compilarse una segunda vez con la bandera puesta sin
duplicar la configuración. Se agregaron dos pruebas sobre la ligadura
—que es lo verdaderamente riesgoso de esta tarea, ya que una factoría mal
armada dejaría el modo temporal inerte o, mucho peor, dejaría el adaptador
falso ligado en un entorno que se cree conectado a una cerradura real—:
que `LOCK_PORT` resuelve a `TTLockAdapter` con la variable sin definir, en
`ttlock` y con un valor arbitrario, y que resuelve a
`HardcodedLockAdapter` con `hardcoded`, emitiendo un PIN de seis dígitos y
verificándolo como instalado sin tenant configurado y sin que el enlace de
`fetch` de TTLock reciba una sola llamada.

Suite unitaria completa verde: 97 suites, 1080 pruebas; `eslint` y
`tsc --noEmit` limpios. Suite e2e: 52 de 54 suites verdes. Las dos fallas
(`patients-import`, `patients-entities`) se reproducen en
`origin/development` sin estos cambios: son fijaciones de prueba
dependientes de la fecha, cuya `lastConsultationDate` del 2025-10-10
envejeció más allá de la ventana que clasifica a un paciente como
recurrente. No se agregó cobertura e2e propia para el modo hardcodeado:
exigiría levantar una segunda instancia de la aplicación con otra
configuración de entorno, y la prueba de ligadura a nivel de módulo ya
cubre lo que esa instancia probaría.

## Figuras pendientes

Ninguna nueva. Esta tarea no agrega un camino de ejecución propio al
flujo ya diagramado: sustituye quién responde detrás del puerto, sin
cambiar la secuencia de llamadas que las figuras pendientes de P6.2
(TASK-56) y TASK-189 ya describen.

## Pendiente que esta tarea deja abierto

La bandera y el adaptador son andamiaje temporal, por pedido explícito
("momentáneamente... hasta que implementemos la integración real con
ttlock"). La tarea que valide la integración real contra la cerradura y el
gateway de la clínica debe eliminar `HardcodedLockAdapter`, sus
constantes, la factoría de selección en `IntegrationsModule`, el
reemplazo de `lockId` en `resolveLockId` y la variable `LOCK_ADAPTER` de
`.env.example`, dejando la ligadura directa a `TTLockAdapter` que existía
antes de esta tarea.

## Componente y referencia

Backend. Rama `feature/TASK-211-hardcoded-lock-random-codes`, creada desde
`origin/development`.
