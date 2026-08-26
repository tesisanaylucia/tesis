# Fase 6 — Cerradura TTLock (backend) — Invalidar/regenerar y expirar códigos (TASK-57)

## Qué se implementó

Se cerró el ciclo de vida completo de `CODIGO_ACCESO` que TASK-55 y TASK-56
habían dejado abierto en dos puntas: qué pasa con el código de un turno
cuando ese turno se cancela o se reprograma, y qué pasa cuando el código
simplemente vence sin que nadie lo haya tocado.

Se agregó `AccessCodeService.revokeForAppointment(appointmentId)`: busca el
`CODIGO_ACCESO` activo del turno (si no hay ninguno, no hace nada y lo
informa mediante un valor de retorno booleano), intenta eliminarlo de la
cerradura vía `LockPort.deleteCode` y, sin importar si esa llamada tuvo
éxito o falló, marca la fila como anulada. Este método se enganchó en tres
puntos de `AppointmentsService`: `cancel` (cancelación individual),
`cancelForAbsence` (cancelación masiva por ausencia del profesional) y
`rescheduleCore` (reprogramación, compartida por el endpoint de
reprogramación individual y el de reorganización de agenda). En
`rescheduleCore`, el valor de retorno de `revokeForAppointment` decide si
corresponde emitir un código nuevo para la nueva fecha
(`AccessCodeService.generateForAppointment`, ya existente desde TASK-56): un
turno que todavía no fue confirmado nunca tuvo un código activo, y generar
uno en ese punto sería prematuro, ya que la única puerta de entrada a la
generación sigue siendo la confirmación del turno.

Se completó además `AccessCodeExpirationCron` (`src/access-codes/`), que
desde TASK-45 existía sólo como *placeholder*: ahora, cada quince minutos y
por cada organización, selecciona los `CODIGO_ACCESO` activos cuya ventana
de validez ya venció, intenta eliminarlos de la cerradura, los marca como
expirados y deja un asiento en la traza de auditoría con la acción
`ACCESS_CODE_EXPIRED`.

## Decisiones y por qué

**La eliminación en la cerradura es un intento de mejor esfuerzo, nunca una
condición para anular o expirar el código**, tanto en `revokeForAppointment`
como en el barrido del cron. El propio ticket lo pide explícitamente: si la
API de TTLock falla al eliminar el passcode, el código debe marcarse como
anulado (o expirado) de todos modos, porque eventualmente dejará de ser
válido por su propia ventana de vigencia. Esto reproduce, en sentido
inverso, la misma filosofía que ya regía la generación desde TASK-56: una
falla de la integración física nunca debe convertirse en una falla de la
operación de negocio que la origina (cancelar, reprogramar o simplemente
dejar pasar el tiempo).

**El cron es una red de seguridad independiente de los *hooks* de
cancelación/reprogramación, no una reutilización de
`revokeForAppointment`.** El propio ticket lo dice de forma expresa: aunque
los *hooks* fallen al eliminar el código en TTLock, el cron debe eliminarlo
cuando venza su ventana. Esto hace que el cron necesite recorrer
`CODIGO_ACCESO` por fecha de vencimiento, no por turno, y no le corresponde
generar reemplazos — su única responsabilidad es limpiar lo vencido. Por
esa razón se implementó como una lectura y una escritura propias dentro del
cron, no como un llamador más de `revokeForAppointment`.

**La regeneración en `rescheduleCore` es condicional, no automática.** El
propio ticket lo redacta como una única rama: "si el turno tiene un código
activo, invalidarlo y generar uno nuevo" — no "invalidar si hay uno, y
generar siempre". Para que `rescheduleCore` supiera si correspondía generar
sin repetir la consulta que `revokeForAppointment` ya había hecho
internamente, `revokeForAppointment` se diseñó para devolver un booleano
("había o no un código activo que anular") en lugar de `void`, reutilizando
esa única lectura en el punto de llamada en vez de duplicarla.

**Se extrajo `resolveLockId` como una función compartida** (antes un método
privado de `AccessCodeService`, usado sólo por `generateForAppointment`) a
un módulo propio, `src/access-codes/resolve-lock-id.ts`, en cuanto un
segundo llamador —el cron de expiración— necesitó la misma resolución del
identificador de cerradura configurado por inquilino. La duplicación no se
toleró ni una sola vez adicional: en cuanto apareció el segundo caso de uso
se refactorizó, en lugar de esperar a un tercero.

**El método interno que registra una falla de `LockPort`
(`AccessCodeService.recordFailure`) se generalizó para distinguir entre una
generación y una revocación fallidas**, en vez de duplicar la lógica de
registro de errores (asiento en `LockLog`, notificación `LOCK_ERROR` al
profesional) para el nuevo caso. El mensaje que ambas rutas escriben ahora
proviene de una única variable compartida, así el registro operativo y la
notificación nunca pueden discrepar en la redacción de un mismo evento — un
efecto colateral positivo del refactor, no su motivación original. Además,
a diferencia de una generación fallida (que no tiene ninguna fila de
`AccessCode` todavía que referenciar), una revocación fallida sí tiene una
fila existente, así que el asiento de `LockLog` que documenta el error
ahora incluye `accessCodeId`, algo que la ruta de generación nunca pudo
tener y que la ruta de revocación sí necesitaba para que el registro
operativo apunte al código concreto que no pudo eliminarse.

**La acción de auditoría del cron se tradujo al inglés siguiendo la
convención ya establecida en el resto del código**, en vez de usar
literalmente "EXPIRACION_CODIGO" del ticket: el propio repositorio ya
registra acciones como `WAITLIST_OFFER_EXPIRED` o `REMINDER_SENT` en
mayúsculas y en inglés, así que se adoptó `ACCESS_CODE_EXPIRED` por
consistencia con ese vocabulario, el mismo criterio que ya se había aplicado
en tareas anteriores del proyecto al tratar el texto en español del ticket
como una descripción funcional y no como un literal a copiar tal cual. La
revocación por cancelación o reprogramación, en cambio, no generó un
asiento de auditoría propio — el criterio de aceptación explícito de
auditoría en este ticket es exclusivamente para el barrido del cron, y
`generateForAppointment` tampoco lo tiene desde TASK-56, así que no había
precedente en el propio módulo para agregarlo unilateralmente a la
revocación.

**Bug real encontrado y corregido durante la validación:** la consulta que
selecciona los códigos vencidos en el cron comparaba `validUntil` contra
`new Date()`, no contra `new Date(Date.now())` — una diferencia invisible
en producción, pero que rompe silenciosamente cualquier prueba que
controle el reloj mediante `jest.spyOn(Date, 'now')`, la convención que ya
usan los demás *crons* del proyecto (`AppointmentAutoCancellationCron`,
`AppointmentReminderCron`) para fijar un "ahora" determinístico en sus
pruebas. Se corrigió a `new Date(Date.now())` para que el cron sea
consistente con esa convención y las pruebas puedan controlar el tiempo de
forma confiable.

## Entidades / puertos / adaptadores tocados

- `AccessCodeService` (revocación agregada, registro de fallas
  generalizado).
- `AccessCodeExpirationCron` (implementación real, reemplaza el
  *placeholder* de TASK-45).
- `AppointmentsService` (`cancel`, `cancelForAbsence`, `rescheduleCore`):
  tres nuevos puntos de enganche hacia `AccessCodeService`.
- `LockPort.deleteCode` (ya declarado desde TASK-55): primer llamador real
  desde el dominio, más allá de las pruebas del propio adaptador.
- Nueva función compartida `resolveLockId`
  (`src/access-codes/resolve-lock-id.ts`).
- Sin cambios de esquema: `AccessCodeStatus.REVOKED`/`.EXPIRED` y las
  columnas que este ticket necesita ya existían desde TASK-55.

## Tests agregados o modificados

- `access-code.service.spec.ts`: nueva batería para `revokeForAppointment`
  (elimina y anula, no hace nada sin código activo, anula igual cuando
  `LockPort.deleteCode` falla o falta el `lockId` configurado).
- `access-code-expiration.cron.spec.ts`: reescrito por completo —de probar
  sólo el *placeholder*, a cubrir el barrido real (elimina y expira lo
  vencido, ignora lo aún vigente o ya resuelto, expira igual cuando falla
  la eliminación en la cerradura o falta el `lockId`, no deja que un código
  fallido detenga el resto del lote, recorre cada organización con su
  propio `SYSTEM`).
- `appointments.service.spec.ts` / `appointments-rescheduling.service.spec.ts`:
  nuevos casos para los tres puntos de enganche (revocación en cancelación
  individual y masiva, revocación condicional a la existencia de un código
  activo en la reprogramación, resiliencia ante una falla inesperada de
  `AccessCodeService`).
- `test/access-code-invalidation-expiration.e2e-spec.ts` (nuevo, contra
  Postgres real): cancelación con código activo invoca
  `LockPort.deleteCode` y anula el código; reprogramación anula el código
  anterior y emite uno nuevo con la ventana de validez de la nueva fecha;
  una reprogramación sobre un turno nunca confirmado no genera ningún
  código; el cron elimina, expira y audita un código vencido; una falla de
  `LockPort.deleteCode` no impide que el código quede anulado/expirado.

## Figuras pendientes

Diagrama de estados de `CODIGO_ACCESO` mostrando las tres transiciones
hacia un estado terminal (ACTIVO → ANULADO por cancelación, ACTIVO →
ANULADO por reprogramación seguido de un nuevo ACTIVO para la fecha
nueva, ACTIVO → EXPIRADO por el barrido del cron), señalando en cada una
que la eliminación en la cerradura es un intento de mejor esfuerzo que
nunca bloquea la transición de estado.

## Componente y referencia

Backend. Rama `feature/TASK-57-access-code-invalidation-expiration`
(todavía no fusionada al momento de esta entrada).
