# Fase 6 — Cerradura TTLock (backend) — AuditLog para la generación/revocación exitosa (TASK-178, corrección a TASK-56/TASK-57)

## Qué se implementó

TASK-178 es una tarea generada automáticamente por una auditoría de código
contra las fuentes de verdad (SRS), fechada 2026-08-26, marcada "requiere
validación humana" antes de implementarse. El hallazgo (SEC 7): ni
`AccessCodeService.generateForAppointment` ni `.revokeForAppointment`
escribían nunca un asiento propio en `AuditLog` (`REGISTRO_AUDITORIA`) para
el caso exitoso — el único asiento con `entity="AccessCode"` que existía
hasta esta tarea era el de `recordFailure` (`ACCESS_CODE_ERROR`, agregado
por TASK-59, ver `FASE-6_PROMPT-5_BACKEND.md`), y ese es un registro de
fallo, no de éxito. El asiento `CONFIRM`/`CANCEL`/`RESCHEDULE` que
`AppointmentsService` ya escribe en la misma transacción de cada transición
registra que el *turno* cambió de estado, nunca que además se generó o se
revocó un código de acceso físico junto con esa transición — una pregunta de
cumplimiento (Ley 25.326) distinta, que `REGISTRO_AUDITORIA` por sí sola no
podía responder.

Se agregaron dos acciones nuevas de `AuditLog` a `access-codes.constants.ts`,
siguiendo la misma convención de nombres en inglés/mayúsculas que ya fijan
`ACCESS_CODE_ADHOC_OPENING`/`ACCESS_CODE_EXPIRED`/`ACCESS_CODE_ERROR`:
`ACCESS_CODE_GENERATE` y `ACCESS_CODE_REVOKE`. Cada una se escribe en la
misma transacción interactiva que la fila `AccessCode` que describe:

- `generateForAppointment` ya persistía el código dentro de la transacción
  serializable de `obtainVerifiedUniqueAccessCode` (CER 5, TASK-170 — tarea
  posterior a `FASE-6_PROMPT-4_BACKEND.md`, todavía sin su propia entrada de
  bitácora) — el callback `persist` que antes sólo hacía
  `tx.accessCode.create(...)` pasó a ser asíncrono y a escribir el asiento
  `ACCESS_CODE_GENERATE` con el mismo handle `tx`, inmediatamente después de
  la creación.
- `revokeForAppointment` no corría dentro de ninguna transacción hasta esta
  tarea: la actualización a `REVOKED` era una llamada suelta sobre
  `prisma.client`. Se envolvió en `prisma.client.$transaction(...)`, junto
  con el nuevo asiento `ACCESS_CODE_REVOKE`.

## Decisiones y por qué

**El asiento de éxito se escribe siempre, incluso cuando la eliminación en
TTLock falló y ya se registró un `ACCESS_CODE_ERROR`.** El propio diseño de
TASK-57 (`FASE-6_PROMPT-3_BACKEND.md`) deja `CODIGO_ACCESO` en estado
anulado "igualmente", sin importar si
`LockPort.deleteCode` tuvo éxito — la ventana de validez del código termina
eliminándolo igual cuando vence. El nuevo asiento `ACCESS_CODE_REVOKE`
describe exactamente esa garantía: la fila cambió de estado, con
independencia de lo que haya respondido la cerradura física. Ambos asientos
pueden coexistir para una misma revocación (uno diagnóstico sobre la
integración, otro de cumplimiento sobre el cambio de estado) sin
contradecirse.

**`revokeForAppointment` (TASK-57, `FASE-6_PROMPT-3_BACKEND.md`) pasó a
cargar el turno una única vez, al principio del método, en lugar de sólo
dentro de la rama de fallo de `deleteFromLockBestEffort` como hacía hasta
ahora.** El nuevo asiento
necesita `patientId` — "toda mutación relacionada con un paciente pasa
`patientId`", la misma regla de `schema.prisma` que ya cumplen
`recordFailure` y `generateAdhoc` — y antes de esta tarea sólo la rama de
fallo tenía ese dato a mano. Cargarlo una sola vez y pasarlo a
`deleteFromLockBestEffort` en lugar de que ese método lo vuelva a resolver
internamente evita una consulta duplicada en el camino de fallo, que antes
existía.

**La colisión de PIN (CER 5, TASK-170) no genera un asiento propio por cada
candidato descartado.** El asiento `ACCESS_CODE_GENERATE` vive dentro del
callback `persist`, que `obtainVerifiedUniqueAccessCode` sólo invoca una vez
que la comprobación de colisión dentro de la misma transacción serializable
ya dio negativo — un candidato descartado nunca llega a `persist`, así que
nunca llega a auditarse; sólo el intento que finalmente se persiste deja
rastro.

**No se tocó el comentario de clase que documenta por qué el asiento de
`recordFailure` queda deliberadamente fuera de la transacción de
`revokeForAppointment`** (ver `FASE-6_PROMPT-5_BACKEND.md): esa decisión
sigue vigente sin cambios — lo que cambió es que ahora, además de ese
asiento de mejor esfuerzo, existe uno transaccional distinto para el caso de
éxito. El comentario de clase se amplió para dejar constancia de ambos, sin
reabrir la decisión original de TASK-59.

## Entidades / puertos / adaptadores tocados

- `access-codes.constants.ts`: acciones nuevas `ACCESS_CODE_GENERATE_ACTION`
  (`'ACCESS_CODE_GENERATE'`) y `ACCESS_CODE_REVOKE_ACTION`
  (`'ACCESS_CODE_REVOKE'`).
- `AccessCodeService.generateForAppointment`: el `persist` que pasa a
  `obtainVerifiedUniqueAccessCode` ahora también escribe el asiento de
  auditoría, dentro de la misma transacción.
- `AccessCodeService.revokeForAppointment`: la actualización a `REVOKED`
  pasó de una llamada suelta a `prisma.client.$transaction(...)`, con el
  asiento de auditoría adentro.
- `AccessCodeService.deleteFromLockBestEffort` (privado): firma cambiada de
  `appointmentId: string` a `appointment: AppointmentForAccessCode`, ya
  resuelto por el llamador en lugar de cargarlo internamente sólo en la
  rama de fallo.

Sin cambios de esquema/migración: `AuditLog.action` ya era texto libre.

## Tests agregados o modificados

- `access-code.service.spec.ts`: casos nuevos que verifican el asiento
  `ACCESS_CODE_GENERATE`/`ACCESS_CODE_REVOKE` en la misma transacción que la
  fila `AccessCode`; el caso de colisión de PIN se actualizó para verificar
  que el asiento se escribe exactamente una vez (por el candidato que
  finalmente se persiste, no por el descartado); el caso de fallo de
  `deleteCode` se amplió para verificar que ambos asientos —`ACCESS_CODE_ERROR`
  y `ACCESS_CODE_REVOKE`— coexisten.
- `test/access-code-generation.e2e-spec.ts` /
  `test/access-code-invalidation-expiration.e2e-spec.ts`: una aserción
  nueva por archivo contra una fila real de `AuditLog` en Postgres (no un
  cliente mockeado), incluyendo `patientId`/`appointmentId`/`userId`.

Suite completa verde: 90 suites unitarias / 971 tests, 53 suites e2e / 634
tests (`--runInBand`), lint y `tsc --noEmit` limpios.

## Figuras pendientes

- Actualizar la figura pendiente de "falla definitiva de generación"
  (`FASE-6_PROMPT-5_BACKEND.md`) y la de revocación para mostrar que, además
  del asiento diagnóstico de `ACCESS_CODE_ERROR`, ahora existe un asiento de
  cumplimiento `ACCESS_CODE_GENERATE`/`ACCESS_CODE_REVOKE` transaccional con
  el cambio de estado del propio `CODIGO_ACCESO`.

## Componente y referencia

Backend. Rama `feature/TASK-178-access-code-audit-log`, creada desde
`origin/main` (incluye ya fusionado hasta TASK-163). Commit `93a2f62`
("TASK-178: audit AccessCode generation/revocation, not just its
failures"), pusheado a origin; PR pendiente de apertura manual por la
usuaria (sin `bb`/`gh` CLI en este entorno).
