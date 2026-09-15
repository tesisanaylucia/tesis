# Fase 6 — Cerradura TTLock (backend) — reissueAccessCode reenvía el PIN nuevo al reprogramar (TASK-166, corrección a P6.3/TASK-57 y P6.4/TASK-58, CER 1)

## Qué se implementó

TASK-166 es una tarea generada por una auditoría de código contra las
fuentes de verdad (SRS "Secretaria Virtual — PSIQUE NEUROCIENCIAS", sección
Cerradura Electrónica), fechada 2026-08-26, hallazgo CER 1, severidad
crítica. El hallazgo: `AppointmentsService.reissueAccessCode` (el enganche
de P6.3/TASK-57 que revoca el código de un turno reprogramado y genera uno
nuevo para la fecha nueva) nunca notificaba el PIN nuevo al paciente. El
único mensaje que efectivamente salía era el aviso de reprogramación
(`APPOINTMENT_RESCHEDULE`, sin el código), así que el paciente se enteraba
del horario nuevo pero quedaba con un PIN ya anulado y sin forma de conocer
el reemplazo — sin intervención manual de secretaría no podía ingresar al
consultorio, contradiciendo el propósito del módulo declarado en el propio
SRS ("sustituye la apertura manual").

La corrección:

- `reissueAccessCode` reenvía `ACCESS_CODE_DELIVERY` con el PIN nuevo
  cuando efectivamente se genera un código de reemplazo, en lugar de
  limitarse a revocar y regenerar en silencio.
- El armado y envío del mensaje se extrajo de `sendAccessCode` (el método
  de P6.4/TASK-58 que ya lo hacía al confirmar) a un método privado
  compartido, `sendAccessCodeDelivery(appointment, accessCode)`, en vez de
  duplicar la plantilla y el envío entre los dos flujos.
- `reissueAccessCode` pasó a recibir el turno ya reprogramado completo
  (`AppointmentWithSelect`) en lugar de sólo su id, para tomar la franja
  horaria ya actualizada sin una segunda lectura — antes bastaba con el id
  porque el método no necesitaba ningún otro campo del turno.

## Decisiones y por qué

**El criterio de envío es simétrico al de la confirmación, no
incondicional.** `reissueAccessCode` sólo reenvía cuando
`generateForAppointment` efectivamente devuelve un código activo —el mismo
criterio "no enviar si el código no quedó activo" que ya regía en
`sendAccessCode` desde TASK-58—, así que no hizo falta ninguna verificación
nueva: un resultado `null` (LockPort falló, o `verifyCodeInstalled` no
confirmó la instalación) simplemente nunca llega al envío, igual que en la
confirmación.

**"Regenerar es condicional" (TASK-57) se mantuvo sin cambios: sólo se
reenvía si efectivamente había un código activo para reemplazar.** Un turno
todavía `RESERVADO` en el momento de la reprogramación nunca tuvo un código
activo (la única puerta de entrada a la generación sigue siendo la
confirmación, TASK-56), así que regenerar — y por lo tanto notificar —
sería prematuro. `revokeForAppointment` ya devolvía un booleano
(`hadActiveCode`) desde TASK-57 para distinguir este caso sin una segunda
consulta redundante; la corrección reutiliza ese mismo booleano en lugar de
agregar una verificación propia.

**El método compartido se extrajo en lugar de duplicar la lógica de
plantilla/envío entre `sendAccessCode` y `reissueAccessCode`.** Ambos
flujos ya usaban la misma plantilla (`ACCESS_CODE_DELIVERY`) con los mismos
dos parámetros (`scheduledAt`, `pin`) y el mismo canal (`notifyPatient`);
mantenerlos como dos copias habría dejado abierta la posibilidad de que
divergieran en el futuro sin que nada lo impidiera. `sendAccessCodeDelivery`
es ahora el único punto que renderiza y envía el PIN, para ambos llamadores.

**`reissueAccessCode` pasó a recibir el turno completo, no sólo su id.**
El mensaje necesita `scheduledAt` (la fecha nueva) además de `patientId`;
antes de esta tarea el método sólo usaba el id para las llamadas a
`AccessCodeService`, así que agregar el envío hizo evidente que ya
convenía recibir el turno ya reprogramado completo en lugar de agregar una
lectura extra sólo para el mensaje — el mismo turno que su único llamador
(`applyRescheduleWrite`) ya tiene a mano tras el `$transaction` que aplica
la escritura.

**Ambos llamadores de `reissueAccessCode` quedan cubiertos por la misma
corrección, sin un segundo punto de enganche.** `applyRescheduleWrite` es
el método compartido entre la reprogramación inmediata (`reschedule`) y la
aplicación diferida de una `RescheduleOffer` ya aceptada
(`acceptRescheduleOffer`, TASK-115) — como `reissueAccessCode` se llama
desde ese único punto compartido, corregirlo ahí alcanza para los dos
caminos, la misma razón por la que TASK-57 lo enganchó ahí originalmente.

## Entidades / puertos / adaptadores tocados

- `AppointmentsService.sendAccessCode`: la construcción/envío del mensaje
  se reemplazó por una llamada a `sendAccessCodeDelivery`.
- `AppointmentsService.sendAccessCodeDelivery` (privado, nuevo): renderiza
  `ACCESS_CODE_DELIVERY` y llama a `notifyPatient`; compartido por
  `sendAccessCode` y `reissueAccessCode`.
- `AppointmentsService.reissueAccessCode`: firma cambiada de
  `(appointmentId: string, actorId: string)` a
  `(appointment: AppointmentWithSelect, actorId: string)`; agrega la
  llamada a `sendAccessCodeDelivery` tras una regeneración exitosa.
- `AppointmentsService.applyRescheduleWrite` (único llamador de
  `reissueAccessCode`): pasa a pasarle el turno completo en lugar del id.

Sin cambios de esquema/migración.

## Tests agregados o modificados

- `appointments-rescheduling.service.spec.ts`: caso nuevo en el describe de
  `reschedule` que verifica que se renderiza y envía `ACCESS_CODE_DELIVERY`
  con el PIN nuevo cuando la reprogramación regenera un código activo, y
  que el aviso de reprogramación se sigue enviando además (no en lugar de);
  caso simétrico que verifica que no se envía nada cuando no había código
  activo que reemplazar. Caso equivalente en el describe de
  `acceptRescheduleOffer`, para probar que el camino diferido (vía
  `applyRescheduleWrite`) no puede divergir del inmediato.

Suite dirigida verde: `appointments-rescheduling.service.spec.ts` +
`appointments.service.spec.ts`, 126 tests.

## Figuras pendientes

- Ampliar el diagrama de secuencia del envío del código al confirmar
  (P6.4/TASK-58, más arriba en esta misma sección) para incluir la rama de
  reprogramación: código activo revocado → código nuevo generado →
  `ACCESS_CODE_DELIVERY` con el PIN nuevo, junto a la rama sin código
  activo que antes reprogramaba en silencio.

## Componente y referencia

Backend. Rama `feature/TASK-166-reschedule-pin-notification` (nombre real
de la rama pusheada: `task-166-reschedule-pin-notification`), creada desde
`main`. Commit `d25c796` ("TASK-166: reissueAccessCode reenvía el PIN
nuevo al reprogramar (CER 1)"), fusionado a `main` vía PR #130 (commit de
merge `d8285f4`). Ver también `FASE-6_PROMPT-8_BACKEND.md` (TASK-188), que
verifica que esta corrección ya cubre un segundo hallazgo de auditoría
sobre el mismo problema.
