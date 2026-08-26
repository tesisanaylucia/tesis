# Fase 6 — Cerradura TTLock (backend) — Tolerancia a fallos de la cerradura (TASK-59)

## Qué se implementó

Se cerró el último ticket declarado del Módulo Cerradura Electrónica: la
robustez de la cadena nube → TTLock Open Platform → Gateway WiFi G2 →
cerradura, mediante reintentos, registro de cada intento y un comportamiento
seguro ante fallas definitivas.

**Política de reintentos en `TTLockAdapter`.** `createTemporaryCode` y
`verifyCodeInstalled` (no `deleteCode`, fuera del alcance del propio ticket)
ahora ejecutan toda su operación —resolución de token incluida, no sólo la
llamada HTTP final— dentro de un método privado `withRetry`, con backoff
exponencial y un máximo de tres intentos totales. Se agregó además un
timeout por pedido (`TTLOCK_TIMEOUT_MS`, 10 segundos por defecto,
configurable como `OPENAI_TIMEOUT_MS`) vía `AbortSignal.timeout`, algo que el
adaptador no tenía desde TASK-55: sin timeout no había ningún error de
"tiempo agotado" que reintentar. `LockOperationError` ganó un campo
`transient` que el propio punto de lanzamiento fija (verdadero para un 429,
un 5xx o un timeout/error de red; falso para cualquier otro 4xx o para el
sobre `{errcode, errmsg}` propio de TTLock sobre una respuesta 200), en
lugar de que cada sitio de captura vuelva a inferirlo a partir del mensaje.

**Registro de cada intento.** `LockPort.createTemporaryCode`/
`verifyCodeInstalled` ganaron un cuarto parámetro opcional, `onAttempt`, que
`TTLockAdapter` invoca después de cada intento (éxito o fracaso,
reintentado o no) con el número de intento, si tuvo éxito y, si no, el
código HTTP y el mensaje. El puerto en sí no conoce turnos ni códigos de
acceso —sólo conoce `lockId`—, así que quien persiste ese evento es
`AccessCodeService`, el único que tiene el `appointmentId` a mano: un
`onAttempt` compartido entre la llamada a `createTemporaryCode` y la
llamada a `verifyCodeInstalled` de una misma `generateForAppointment`
escribe una fila `CODIGO_ACCESO_INTENTO` (`CODE_ATTEMPT`) en `LockLog` por
cada intento, y recuerda el número del último intento por si ese mismo
llamado termina en falla definitiva y lo necesita.

**Comportamiento ante falla definitiva.** Cuando `LockPort` agota sus
reintentos, o nunca llega a llamarse (sin `lockId`/credenciales
configurados), o `verifyCodeInstalled` responde que no quedó instalado,
`AccessCodeService.recordFailure` sigue sin persistir ningún código activo
ni enviar nada al paciente —eso ya lo garantizaba el diseño de TASK-56— y
ahora, además de la fila `ACCESS_CODE_ERROR` que ya escribía en `LockLog`
desde esa misma tarea (con el número de intento real que `LockPort` hizo,
no siempre 1 como hasta ahora), escribe un asiento nuevo en
`REGISTRO_AUDITORIA` (`AuditLog`) con `accion=ACCESS_CODE_ERROR` — el
propio ticket nombra esa tabla por su nombre real en español, distinto del
uso genérico de "auditoría" que usa para el registro de intentos. La
notificación al profesional sigue siendo la que ya existía desde TASK-56/57
(`InAppNotificationsService`/`NotificationType.LOCK_ERROR`), no
`MessagingPort` como redacta el ticket: ver más abajo.

**Hilo del actor a través del ciclo de vida del código.**
`generateForAppointment`/`revokeForAppointment` pasaron a exigir un
`actorId`, que faltaba por completo hasta esta tarea porque nada anterior
necesitaba atribuir un asiento de `AuditLog` a un usuario concreto.
`AppointmentsService` ya tenía ese `actorId` disponible en `confirm`,
`cancel`, `cancelForAbsence` y `rescheduleCore` (es el mismo actor que
firma el asiento `CONFIRM`/`CANCEL`/`RESCHEDULE` de la propia transición),
así que sólo hizo falta encadenarlo a través de los métodos privados
intermedios (`sendAccessCode`, `revokeAccessCode`, `reissueAccessCode`).

## Decisiones y por qué

**"Máximo 3 intentos" se interpretó como 3 intentos totales, no 3
reintentos.** El propio ticket lista "intervalos 1s/2s/4s" para esos tres
intentos, pero con sólo tres intentos totales existen apenas dos
intervalos entre ellos (1s, luego 2s): el tercer valor nunca llega a
usarse. Se mantuvo la misma lectura que ya fijó TASK-46 sobre una frase en
español idéntica ("máximo 3 intentos") para el adaptador de IA —tres
intentos totales, no tres reintentos—, en vez de reinterpretar la misma
frase de manera distinta entre dos tickets consecutivos del mismo
proyecto. El "4s" del ticket queda documentado en el propio código como el
siguiente término de la misma secuencia de duplicación, simplemente
inalcanzable con un presupuesto de tres intentos.

**El reintento vive en el adaptador, no en `AccessCodeService`, tal como
pide el propio ticket ("política de reintentos en el TTLockAdapter").**
Esto entra en tensión con que sólo `AccessCodeService` conoce el
`appointmentId` necesario para registrar cada intento en `LockLog`: se
resolvió agregando un callback opcional (`onAttempt`) a la firma del
puerto, que el adaptador invoca en cada intento sin conocer para qué sirve
—ni `LockLog` ni `AuditLog` existen desde el punto de vista de
`LockPort`—, y que el llamador usa para persistir lo que le interese. La
alternativa —mover el bucle de reintentos al servicio, que sí tiene el
contexto— habría cumplido el criterio de aceptación pero no la instrucción
explícita del ticket sobre dónde debe vivir la política.

**El registro "de cada intento" fue a `LockLog`, no a `AuditLog`, pese a
que el ticket dice "registra en auditoría" para ambos casos.** El propio
código de este repositorio ya distinguía, desde TASK-55, entre `LockLog`
(el registro operativo de la integración con la cerradura, deliberadamente
separado de `AuditLog` para no contaminar la traza de cumplimiento con
ruido técnico) y `AuditLog`/`REGISTRO_AUDITORIA` (la traza de Ley 25.326).
El propio comentario de `LockLog.attemptNumber` en el esquema, escrito
desde TASK-55, ya anticipaba un evento "`CODE_ATTEMPT`" para este propósito
específico. Se leyó la palabra "auditoría" en minúscula del ticket como
referencia genérica a ese registro operativo, y la mención explícita y en
mayúsculas de "REGISTRO_AUDITORIA" —el nombre real, en español, de la
tabla `AuditLog` desde TASK-17— como la que sí exige la tabla de
cumplimiento. Sólo la falla definitiva escribe en ambas.

**La notificación al profesional sigue sin usar `MessagingPort`, aunque el
ticket lo pide explícitamente.** Ni `Professional` ni `User` tienen una
columna de número de celular en este esquema —la misma limitación que
TASK-58 ya había encontrado para la entrega del PIN en la apertura ad-hoc,
documentada en ese momento con la misma conclusión—. Se mantuvo el
mecanismo ya existente desde TASK-56 (`InAppNotificationsService`/
`NotificationType.LOCK_ERROR`), que el propio comentario de ese tipo de
notificación en el esquema ya señalaba como "lo único que puede significar
hoy" la tolerancia a fallos de esta tarea.

**El asiento nuevo en `AuditLog` no se ató a la misma transacción que la
actualización de estado del código anulado**, a pesar de que la regla
general de este proyecto exige que un asiento de auditoría se confirme
junto con la mutación que describe. Se documentó como una excepción
deliberada: ese asiento describe un fallo diagnóstico de la integración
física, de la misma naturaleza que el `LockLog` con el que siempre viaja
—ambos de mejor esfuerzo—, no el registro de cumplimiento de "qué cambió",
que ya queda correctamente atado a su propia transacción en el asiento
`CANCEL`/`CONFIRM`/`RESCHEDULE` de la transición del turno que disparó todo
esto.

## Entidades / puertos / adaptadores tocados

- `LockOperationError` (`lock.errors.ts`): campo nuevo `transient`.
- `LockPort` (`lock.port.ts`): tipos `LockAttempt`/`OnLockAttempt` nuevos;
  `createTemporaryCode`/`verifyCodeInstalled` ganan el parámetro opcional
  `onAttempt`.
- `TTLockAdapter`: método privado `withRetry`; timeout por pedido vía
  `fetchWithTimeout`; clasificación de estado transitorio
  (`isTransientStatus`).
- `ttlock.constants.ts`: `TTLOCK_MAX_ATTEMPTS`, `TTLOCK_RETRY_BASE_DELAY_MS`,
  `DEFAULT_TTLOCK_TIMEOUT_MS`.
- `access-codes.constants.ts`: evento `CODE_ATTEMPT` de `LockLog`, acción
  `ACCESS_CODE_ERROR` de `AuditLog`.
- `AccessCodeService`: `generateForAppointment`/`revokeForAppointment` ganan
  `actorId`; método privado nuevo `createAttemptLogger`; `recordFailure`
  ahora también escribe en `AuditLog` y recibe el número de intento real.
- `AppointmentsService`: `sendAccessCode`/`revokeAccessCode`/
  `reissueAccessCode` encadenan `actorId` hacia `AccessCodeService`.
- `prisma/schema.prisma`: comentario de `NotificationType.LOCK_ERROR`
  actualizado (ya no dice "no cableado todavía").
- `CLAUDE.md`: sección de puertos de integración amplía la descripción de
  `LockPort` con la política de reintentos.
- `.env.example`: sección `TTLOCK_API_BASE_URL`/`TTLOCK_TIMEOUT_MS`, ausente
  hasta ahora pese a que el adaptador ya la leía desde TASK-55.

Sin cambios de esquema/migración: ninguna columna nueva, `action`/`detail`
de `AuditLog` y `event`/`attemptNumber`/`detail` de `LockLog` ya existían.

## Tests agregados o modificados

- `ttlock.adapter.spec.ts`: batería nueva `fault tolerance (P6.5, TASK-59)`
  — reintento tras dos 5xx y éxito al tercero, reportando cada intento por
  `onAttempt`; agotamiento tras tres 429 consecutivos; ausencia de
  reintento ante un 4xx definitivo (401); reintento tras un timeout
  simulado; clasificación `transient` correcta en `deleteCode`, que nunca
  reintenta. Usa temporizadores falsos (`jest.useFakeTimers` +
  `advanceTimersByTimeAsync`), mismo patrón que la cobertura de reintentos
  de `OpenAiAdapter`.
- `access-code.service.spec.ts`: todas las llamadas existentes a
  `generateForAppointment`/`revokeForAppointment` actualizadas con
  `actorId`; nuevas aserciones sobre el asiento de `AuditLog` en cada rama
  de falla; casos nuevos que verifican el registro de `CODE_ATTEMPT` por
  intento y que el número de intento de `ACCESS_CODE_ERROR` refleja el real
  reportado por `LockPort`, no siempre 1.
- `appointments.service.spec.ts` / `appointments-rescheduling.service.spec.ts`:
  aserciones de llamado a `generateForAppointment`/`revokeForAppointment`
  actualizadas con el nuevo argumento `actorId`.
- `test/access-code-generation.e2e-spec.ts`: captura el id del usuario admin
  creado en el fixture para pasarlo como `actorId` a la llamada directa al
  servicio; aserción sobre `createTemporaryCode` actualizada con el nuevo
  parámetro `onAttempt`.

Suite completa verde: 71 suites unitarias / 725 tests, 46 suites e2e / 529
tests (`--runInBand`), lint y `tsc --noEmit` limpios.

## Figuras pendientes

- Diagrama de secuencia de la política de reintentos de `TTLockAdapter`
  (intento → ¿error 429/5xx/timeout? → notificación del intento al
  llamador vía `onAttempt` → espera exponencial → siguiente intento, hasta
  tres, o error definitivo que expone `transient=false` sin reintentar),
  contrastado con la Figura 29 (el mismo ciclo del lado del adaptador de
  IA) para remarcar la diferencia: aquí cada intento se reporta hacia
  afuera, no sólo se registra en el log de la aplicación.
- Diagrama de secuencia de la falla definitiva de generación de un código
  (reintentos agotados o `LockPort` no disponible → sin código activo, sin
  envío al paciente → asiento `ACCESS_CODE_ERROR` en `LockLog` y en
  `REGISTRO_AUDITORIA` → notificación `LOCK_ERROR` al profesional → turno
  igual queda CONFIRMADO), contrastado con la Figura 48 para remarcar que
  ahora hay dos escrituras de registro, no una.

## Componente y referencia

Backend. Rama `feature/TASK-59-lock-fault-tolerance`, creada desde
`origin/main` (incluye ya fusionados TASK-55 a TASK-58). No commiteada
todavía en esta sesión (pendiente de autorización explícita para
commit/push, según lo pedido).
