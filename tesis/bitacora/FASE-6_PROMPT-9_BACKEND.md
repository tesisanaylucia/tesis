# Fase 6 — Cerradura TTLock (backend) — limpieza del passcode en la cerradura cuando falla la verificación de instalación (TASK-189)

## Qué se implementó

TASK-189 ("[MEDIO] Sin limpieza del passcode de TTLock cuando falla la
verificación de instalación") es la tercera tarea generada por la
auditoría multi-agente fechada 2026-08-28 (las otras dos fueron TASK-166,
ya corregida en una fase anterior, y TASK-188, un duplicado obsoleto de esa
misma corrección — ver `FASE-6_PROMPT-8_BACKEND.md`). A diferencia de esas
dos, esta describe un vacío real y vigente: en
`AccessCodeService.obtainVerifiedUniqueAccessCode`, si
`createTemporaryCode` tiene éxito pero `verifyCodeInstalled` devuelve
`false` o la propia llamada lanza un error (reintentos de TASK-59
agotados), el código nunca se elimina de la cerradura. Como ninguna fila
`AccessCode` llega a persistirse en esa rama, `AccessCodeExpirationCron`
—que barre por `validUntil` en la base de datos— tampoco puede
encontrarlo: el passcode puede quedar instalado en la cerradura física sin
ningún registro recuperable en ningún lado, justo el vacío contra el que
el SRS pide "invocar a la API de la cerradura para eliminarlo, evitando su
reutilización".

Se agregaron dos llamadas a `deleteCodeBestEffort` dentro de
`obtainVerifiedUniqueAccessCode`: una en el `catch` que envuelve la propia
llamada a `verifyCodeInstalled` (antes de relanzar el error tal cual), y
otra en la rama `!verified` (antes de lanzar `UnverifiedLockCodeError`).
Ambas reutilizan el método de mejor esfuerzo que CER 5 (TASK-170) ya había
introducido para descartar un candidato con PIN colisionante, en lugar de
duplicar su lógica de `try/catch`+`logger.warn`.

## Decisiones y por qué

**Reutilizar `deleteCodeBestEffort` en vez de escribir una limpieza
propia.** El método ya expresaba exactamente el contrato que hacía falta
—intentar `lock.deleteCode` y tragarse cualquier error propio, dejando el
barrido de expiración como red de seguridad final— para el caso de
colisión de PIN; extenderlo a estos dos call sites nuevos evita una
tercera copia de ese mismo `try/catch` en el archivo. Se amplió el
comentario del método para reflejar que ahora tiene tres llamadores en
lugar de uno.

**No se cambia el tipo de error que sale de `obtainVerifiedUniqueAccessCode`.**
El `catch` de `verifyCodeInstalled` relanza el error original después de
la limpieza, y la rama `!verified` sigue lanzando `UnverifiedLockCodeError`
exactamente como antes — ambos caminos siguen llegando al mismo `catch` de
`generateForAppointment`/`generateAdhoc` que ya distinguía los dos casos
para `recordFailure`. La limpieza es un efecto colateral de mejor esfuerzo,
no un cambio en el contrato de excepciones que el resto del servicio ya
depende de mantener.

**La prueba unitaria que documentaba el comportamiento anterior se
reescribió, no se descartó.** El propio ticket cita
`access-code.service.spec.ts:813`, donde una prueba mockeaba `deleteCode`
para que fallara y afirmaba que nunca se llamaba — evidencia de que la
omisión era conocida y estaba probada como tal, no un descuido sin
cobertura. Esa prueba (ubicada, tras los cambios acumulados desde el
26-08-2026, en el describe de `generateForAppointment`) ahora verifica lo
contrario: que `deleteCode` sí se invoca con el `codeId` del candidato, y
se agregó una prueba nueva, específica para este ticket, que confirma que
una falla de la propia eliminación no cambia el resultado reportado al
llamador (el flujo de `ACCESS_CODE_ERROR` sigue disparándose igual).

## Entidades / puertos / adaptadores tocados

`AccessCodeService` (`src/access-codes/access-code.service.ts`) —
`obtainVerifiedUniqueAccessCode` y el comentario de `deleteCodeBestEffort`,
sin cambios de esquema ni de `LockPort` (la interfaz del puerto ya
declaraba `deleteCode`, este ticket sólo agrega dos llamadas nuevas a un
método privado existente).

## Tests agregados o modificados

`access-code.service.spec.ts`: las pruebas "does not persist an active
code when verifyCodeInstalled returns false" y "records the failure when
verifyCodeInstalled itself throws" (describe `generateForAppointment`)
ahora aserten que `deleteCode` se invoca con el `lockId`/`codeId` del
candidato; se agregó "still records the failure when the best-effort lock
cleanup itself fails", que fuerza a `deleteCode` a rechazar y confirma que
el resultado reportado (código nulo, `ACCESS_CODE_ERROR` en `LockLog`,
notificación al profesional) no cambia. La prueba equivalente de
`generateAdhoc` ("throws instead of persisting a code when
verifyCodeInstalled returns false") ganó la misma aserción, ya que
comparte el método corregido con `generateForAppointment`.

`test/access-code-generation.e2e-spec.ts`: la prueba "does not activate a
code and records the failure when LockPort cannot verify the install" se
amplió con la misma aserción contra el `LOCK_PORT` simulado, leyendo el
`codeId` real que la propia llamada mockeada de `createTemporaryCode`
generó (el mock asigna un id distinto por invocación dentro de este
archivo, no un literal fijo).

Suite dirigida verde: 35 pruebas unitarias en
`access-code.service.spec.ts`, 4 pruebas e2e en
`access-code-generation.e2e-spec.ts` (contra Postgres real). Suite
completa (`--runInBand`) verde tanto en unitarios como en e2e.

## Figuras pendientes

Ampliar el diagrama de secuencia de la verificación fallida durante la
generación (ya registrado desde P6.2/TASK-56) para incluir el intento de
eliminación de mejor esfuerzo antes de registrar `ACCESS_CODE_ERROR` — ver
`cap4_desarrollo.md` §4.7.

## Componente y referencia

Backend. Rama `feature/TASK-189-lock-cleanup-on-verify-failure`, creada
desde `origin/main`.
