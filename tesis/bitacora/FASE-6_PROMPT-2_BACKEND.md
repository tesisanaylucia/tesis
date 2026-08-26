# Fase 6 — Cerradura TTLock (backend) — Generar código al confirmar el turno (TASK-56)

## Qué se implementó

Se implementó `AccessCodeService.generateForAppointment`, el servicio que
decide *cuándo* y *cómo* se genera el código temporal de acceso —el hueco
que TASK-55 dejó explícitamente declarado ("integración con el flujo de
turnos: TASK-56")—, y se lo enganchó a `AppointmentsService.confirm`: la
única transición que lleva un turno a CONFIRMADO en todo el sistema, tanto
desde el endpoint HTTP como desde la herramienta del chatbot. Al confirmarse
un turno, el servicio calcula la ventana de validez, llama a `LockPort` para
crear el passcode temporal, verifica que haya quedado instalado en la
cerradura y sólo entonces persiste `CODIGO_ACCESO` con estado activo. Si la
verificación falla —o si `LockPort` falla antes de llegar a ese punto—, no
se persiste ningún código activo, y se deja un rastro operativo (`LockLog`)
y una notificación al profesional en lugar de fallar la confirmación misma.

## Decisiones y por qué

**El servicio nunca propaga un error de `LockPort` hacia
`AppointmentsService.confirm`.** El propio ticket pide que, ante una
verificación fallida, "no se persista como activo" y se dispare el "flujo de
tolerancia a fallos de TASK-59" —una tarea que todavía no existe—. Se
interpretó esa frase como lo único que puede significar concretamente hoy:
dejar un evento operativo (`LockLog`, `event: ACCESS_CODE_ERROR`) que el
futuro barrido de reintentos de TASK-59 pueda recorrer, más una notificación
al profesional para que un humano se entere mientras tanto. Ninguna de las
dos escrituras puede convertir una falla de la cerradura en una confirmación
fallida: un turno debe quedar CONFIRMADO aunque la integración física haya
fallado, la misma razón por la que `AppointmentsService.fireReassignment` y
`.notifyCancellation` ya son de mejor esfuerzo. El enganche en sí ocurre
después de que la transición de estado ya confirmó — nunca dentro de esa
misma transacción de base de datos —, porque llamar a `LockPort` implica una
petición HTTP externa, y el propio código base ya evita mantener abierta una
transacción mientras espera una respuesta externa (mismo criterio que
`requestRescheduleConfirmation` sobre la reprogramación).

**El evento de error reutiliza el literal `ACCESS_CODE_ERROR` que ya
existía en el código y en las pruebas desde TASK-55**, no uno nuevo: tanto
el comentario de `LockLog.attemptNumber` en el esquema como
`lock-log.service.spec.ts` y `test/lock-log.e2e-spec.ts` ya fijaban ese
literal antes de que esta tarea existiera, como si una sesión anterior ya
hubiera anticipado esta implementación. Se siguió esa evidencia en lugar de
inventar un nombre distinto, junto con `attemptNumber: 1` —el único intento
que hace esta tarea, sin reintentos todavía—, dejando ese campo listo para
que TASK-59 lo incremente cuando agregue el bucle de reintento real.

**La notificación `LOCK_ERROR` no lleva `appointmentId`**, aunque
semánticamente esté relacionada con un turno. El propio comentario del
esquema sobre `Notification.appointmentId` restringe esa columna a
`APPOINTMENT_CANCELLED`/`APPOINTMENT_REASSIGNED`, y el comentario de
`CreateNotificationParams` ya documentaba, desde antes de esta tarea, que
`LOCK_ERROR` "no lleva ninguna" de las dos referencias opcionales. Se
respetó esa restricción documentada en lugar de romperla por conveniencia.

**La ventana de validez usa la duración que el propio turno tiene grabada
(`Appointment.duration`), no una lectura en vivo de la configuración
vigente del profesional.** Es el mismo criterio que ya sigue
`AppointmentsService.book` al copiar la duración en el momento de la
reserva: un cambio posterior en la configuración del profesional no debe
alterar retroactivamente un código ya instalado para un turno existente.

**La verificación de idempotencia ("si el turno ya tiene un `CODIGO_ACCESO`
activo, no crear otro") se resuelve con una lectura indexada antes de tocar
`LockPort`**, no con una transacción serializable. El propio código base
reserva ese mecanismo para invariantes de lectura-y-escritura donde dos
llamadas concurrentes pueden pisarse (ver la sección de Concurrencia de
CLAUDE.md); aquí, en cambio, la máquina de estados del turno ya impide que
`confirm` se llame dos veces para el mismo turno bajo el flujo normal —la
segunda transición CONFIRMADO → CONFIRMADO es rechazada antes de llegar a
este servicio—, así que la única concurrencia real posible sería un futuro
llamador que invoque este método directamente y en paralelo (el barrido de
reintentos de TASK-59), fuera del alcance declarado de este ticket. Se dejó
la limitación documentada en el propio código en lugar de resolverla por
adelantado.

**El módulo `src/access-codes/` no necesitó reestructurarse**: ya existía
desde TASK-55 como el "hogar" declarado de `CODIGO_ACCESO`, con un
comentario explícito anticipando que "el servicio que crea/revoca códigos...
y los conecta al flujo de reserva/cancelación es TASK-56". Sólo se agregó el
nuevo servicio a sus proveedores y las dos importaciones que necesita
(`IntegrationsModule` para `LOCK_PORT`, `InAppNotificationsModule` para la
notificación de error).

## Entidades, puertos y adaptadores tocados

- `src/access-codes/access-codes.constants.ts`: nuevo — la ventana de
  validez (15 min antes, 1 h después de la duración) y los literales de
  `LockLog.event` que este servicio escribe.
- `src/access-codes/access-code.service.ts`: nuevo — `AccessCodeService`.
- `src/access-codes/access-codes.module.ts`: ampliado con el nuevo
  servicio y sus dos importaciones.
- `src/appointments/appointments.service.ts`: `confirm` ahora dispara
  `AccessCodeService.generateForAppointment` tras confirmar, envuelto en un
  respaldo de mejor esfuerzo (mismo patrón que `fireReassignment`/
  `notifyCancellation`).
- `src/appointments/appointments.module.ts`: importa `AccessCodesModule`.

No hubo cambios de esquema: `CODIGO_ACCESO` ya existía completo desde
TASK-55, exactamente con los campos que esta tarea necesitaba llenar.

## Tests

- `src/access-codes/access-code.service.spec.ts`, nuevo: los cuatro
  criterios de aceptación del propio ticket (verificación exitosa persiste
  el código activo; verificación fallida no lo persiste y dispara el
  registro de error; segunda llamada para el mismo turno es idempotente;
  fórmula de la ventana de validez verificada por aserción), más los casos
  adicionales de una falla de `LockPort` antes de la verificación (error de
  autenticación) y de un inquilino sin `lockId` configurado.
- `src/appointments/appointments.service.spec.ts`: el caso existente de
  `confirm` ahora también verifica que dispara la generación del código con
  el id del turno recién confirmado, más un caso nuevo que prueba que una
  falla inesperada del servicio de códigos no impide que el turno quede
  confirmado.
- `src/appointments/appointments-rescheduling.service.spec.ts`: sólo
  necesitó la nueva dependencia simulada para seguir compilando — no prueba
  la generación de códigos, que no toca la reprogramación.
- `test/access-code-generation.e2e-spec.ts`, nuevo: la conexión real por
  HTTP (`PATCH /turnos/:id/confirmar` → fila real de `AccessCode` en
  Postgres), con `LOCK_PORT` reemplazado por un doble controlable —el
  comportamiento propio de `TTLockAdapter` ya lo cubrió TASK-55—; incluye el
  caso de idempotencia invocando el servicio directamente una segunda vez, ya
  que la propia máquina de estados del turno impide alcanzar ese caso a
  través del endpoint.

Suite completa en verde al cierre de la tarea: 71 suites / 693 pruebas
unitarias y 44 suites / 517 pruebas de integración (`--runInBand`) contra
una base Postgres real. Lint (`eslint --fix`) y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

- Figura 48: diagrama de secuencia de la confirmación del turno con
  generación de código (`AppointmentsService.confirm` → transición de
  estado y auditoría, ya comprometidas → `AccessCodeService.
  generateForAppointment`, fuera de la transacción → verificación de
  idempotencia → `LockPort.createTemporaryCode` → `LockPort.
  verifyCodeInstalled` → rama de éxito, que persiste `CODIGO_ACCESO`
  activo, frente a la rama de falla, que registra `ACCESS_CODE_ERROR` y
  notifica al profesional sin afectar la confirmación ya comprometida).
  Sección 4.7 Cerradura TTLock.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-56-access-code-generation-on-confirm`,
  creada desde `main` (`e87d888`, ya con TASK-55 fusionada).
- Ticket: TASK-56 (Jira), "P6.2 – Generar código al confirmar el turno (con
  verificación)", bajo el épico del Módulo 6 (Cerradura electrónica,
  TTLock).
- Dependencias declaradas por el ticket, todas ya fusionadas a `main`:
  TASK-55 (P6.1, `LockPort`/`TTLockAdapter`), TASK-38 (P3.5, transición a
  CONFIRMADO) y TASK-34 (P3.1, entidad `CODIGO_ACCESO` — en la práctica
  completada por TASK-55, según su propia bitácora).
- Fuera de alcance, declarado en el propio ticket y respetado: el envío del
  código al paciente (TASK-58, P6.4); los reintentos y la tolerancia a
  fallos de conectividad (TASK-59, P6.5); la invalidación del código al
  reprogramar o cancelar el turno (TASK-57, P6.3).
