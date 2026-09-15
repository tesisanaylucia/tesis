# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Cifrado en reposo de AccessCode.pin (TASK-190, corrección/extensión a P8.1/TASK-66)

## Qué se implementó

Se cerró un hallazgo de severidad media de la auditoría multi-agente
fechada 2026-08-28 (la misma que generó TASK-166/TASK-188 y TASK-189, ya
documentadas en la Fase 6): `CLAUDE.md` justifica cifrar
`AccessCode.ttlockPasscodeId` porque es una "credencial viva contra una
puerta física" si el resultado de una consulta llega a filtrarse — un read
replica mal configurado, un `SELECT` ad-hoc, un pipeline de logging — sin
que haga falta que el propio disco de la base de datos se vea
comprometido. `AccessCode.pin`, el código de 6 dígitos que la paciente
efectivamente digita en el teclado de la cerradura, es exactamente el
mismo caso de amenaza — de hecho más directamente explotable, porque abre
la puerta tal cual, sin necesitar ninguna llamada a la API de TTLock — y
había quedado fuera del esquema de cifrado que P8.1 (TASK-66) sí aplicó a
su campo hermano. Un gap real contra la Ley 25.326 Art. 9 y la propia
justificación que el sistema ya usaba para el campo hermano.

Se extendió el mismo mecanismo `encryptField`/`decryptField` (AES-256-GCM,
`src/common/crypto/field-encryption.ts`) a `AccessCode.pin`, cifrado antes
de persistirse y descifrado en los mismos puntos donde ya se descifraba
`ttlockPasscodeId`.

## Decisiones y por qué

**El PIN se cifra en el mismo punto de escritura donde ya se cifraba el
identificador opaco, pero el valor en claro se reintroduce en el objeto
que el método devuelve, en lugar de descifrar de vuelta lo que Prisma
ecoa.** A diferencia de `ttlockPasscodeId` —que ningún llamador lee fuera
de `AccessCodeService` mismo—, `.pin` sí es necesario aguas abajo:
`AppointmentsService.sendAccessCodeDelivery` (compartido por la
confirmación y la reprogramación) y la respuesta HTTP de
`POST /cerradura/abrir-adhoc` leen `.pin` directamente, sin conocer el
mecanismo de cifrado. Como el valor en claro ya está disponible en la
misma función que lo acaba de cifrar (`code.value`, la respuesta de
`LockPort.createTemporaryCode`), sustituirlo de vuelta en el objeto
devuelto evita un descifrado redundante y, sobre todo, evita que
`AppointmentsService`/`AccessCodesController` tengan que conocer la clave
de cifrado o llamar a `decryptField` por su cuenta: toda la lógica de
cifrado queda contenida en `AccessCodeService`, y los dos consumidores
finales no necesitaron ningún cambio de código.

**`findActive` ahora descifra `.pin` además de `.ttlockPasscodeId`.** Es el
mismo método que resuelve la comprobación de idempotencia de
`generateForAppointment` ("si el turno ya tiene un código activo, no crear
otro, devolver el existente"), y ese valor de retorno llega exactamente al
mismo `sendAccessCodeDelivery` que un código recién creado — ambos caminos
de lectura debían coincidir en devolver el PIN en claro, o la rama
idempotente habría empezado a enviar el texto cifrado por WhatsApp.

**La comprobación de colisión de PIN (CER 5, TASK-170) dejó de poder
resolverse con una consulta de igualdad exacta en la base de datos.** Con
un vector de inicialización aleatorio en cada escritura, dos PIN idénticos
nunca vuelven a producir el mismo texto cifrado — la misma razón por la
que el `@@unique` de `ttlockPasscodeId` ya sólo garantiza unicidad del
texto cifrado, no del texto plano. `hasActivePinCollision` pasó de un
`findFirst` filtrando por `pin` exacto a un `findMany` que sólo filtra por
estado activo y superposición de ventana de vigencia —el invariante real
que CER 5 protege—, descifrando y comparando cada candidato en memoria
contra el PIN nuevo. El conjunto de candidatos se mantiene chico por
construcción: sólo son códigos activos cuya ventana se superpone, y una
colisión real ya era, antes de este cambio, un evento de 1 en 900000 por
sorteo.

**El seed del piloto (`prisma/seed-pilot.ts`) también cifra el PIN de su
fila de ejemplo.** Escribe directamente con Prisma, sin pasar por
`AccessCodeService` (no tiene contenedor de inyección de dependencias
desde el que resolver `LockPort`), pero ya reutilizaba
`encryptField`/`parseFieldEncryptionKey` para `ttlockPasscodeId` — extender
el mismo tratamiento a `pin` mantiene la fila con la forma exacta que el
flujo real habría producido.

## Entidades / puertos / adaptadores tocados

Ninguna migración de base de datos ni cambio de tipo de columna — `pin`
sigue siendo `String`, sólo cambia lo que esa columna contiene.

- `prisma/schema.prisma`: comentario de `AccessCode.pin` documenta el
  cifrado y el trade-off de la comprobación de colisión.
- `src/access-codes/access-code.service.ts`: `generateForAppointment` y
  `generateAdhoc` cifran `pin` al crear y reintroducen el valor en claro en
  el objeto devuelto; `findActive` descifra `pin` además de
  `ttlockPasscodeId`; `hasActivePinCollision` reescrito de `findFirst` a
  `findMany` + descifrar-y-comparar.
- `prisma/seed-pilot.ts`: cifra el `pin` de la fila de ejemplo.
- `CLAUDE.md`, `.env.example`: documentan la extensión del cifrado y el
  trade-off de la comprobación de colisión.

## Tests agregados o modificados

`access-code.service.spec.ts`: los bloques `describe` de
`generateForAppointment` y `generateAdhoc` ahora mockean también
`accessCode.findMany` (antes sólo `findFirst`) para la comprobación de
colisión, con un candidato cifrado con la misma clave de prueba que el
archivo ya usaba para `ttlockPasscodeId`. Las aserciones sobre el `pin`
que llega a `accessCode.create` pasaron de comparar el valor literal a
descifrarlo primero y comparar contra el valor plano, siguiendo el mismo
patrón que el archivo ya usaba para `ttlockPasscodeId`; se agregaron
aserciones nuevas confirmando que el objeto devuelto por el servicio sigue
llevando el PIN en claro. La prueba de idempotencia y la fila fija del
bloque `revokeForAppointment` pasaron a construir su PIN de fixture ya
cifrado, ya que `findActive` ahora intenta descifrarlo.

`test/access-code-generation.e2e-spec.ts` y
`test/access-code-delivery-and-adhoc-opening.e2e-spec.ts` (contra
PostgreSQL real): se agregó un helper `decryptStoredPin`, mismo patrón que
el `decryptStoredPasscode` ya existente, y se lo usó donde una prueba leía
`.pin` directamente de una fila o lo usaba como valor plano para forzar
una colisión — en particular, la prueba de colisión de PIN de extremo a
extremo ahora descifra el PIN ya almacenado antes de dárselo al `LockPort`
simulado como candidato, o la comprobación de colisión real nunca
encontraría la coincidencia forzada.

Suite completa verde: 145 suites (1636 pruebas) con `--runInBand`,
verificación de tipos y lint sin errores.

## Figuras pendientes

Ninguna — es una corrección del esquema de cifrado en reposo, sin un flujo
de interacción nuevo que las figuras ya pendientes de la Fase 6 (46, 50,
51) no muestren ya a ese mismo nivel de abstracción.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-190-encrypt-access-code-pin`, creada
  desde `main` para esta tarea, con pull request abierto hacia `main`.
