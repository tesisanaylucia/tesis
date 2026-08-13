# Fase 4 — Notificaciones y Scheduler (backend) — índice faltante para la FK compuesta `WaitlistOffer.waitlistEntryId` (TASK-91, corrección a TASK-82)

## Qué se implementó

TASK-91 fue una tarea de corrección originada en una auditoría de la base
de datos viva de `psique-back` (2026-08-12): el modelo `WaitlistOffer`
(TASK-82, P4.5) tiene tres claves foráneas compuestas acotadas por
inquilino — hacia `Appointment`, hacia `Patient` y, opcionalmente, hacia
`WaitlistEntry` — siguiendo la regla 3 del esquema (`(organizationId,
targetId)` contra el `@@unique([organizationId, id])` del padre). Las dos
primeras ya tenían su índice de soporte
(`WaitlistOffer_organizationId_appointmentId_idx`,
`WaitlistOffer_organizationId_patientId_idx`), pero la tercera no: ni un
índice sobre `(organizationId, waitlistEntryId)` ni sobre `waitlistEntryId`
solo existían. La auditoría lo confirmó consultando `pg_indexes` contra la
base viva — de los cuatro índices no-PK de `WaitlistOffer`, ninguno cubría
esa columna.

PostgreSQL no indexa automáticamente las columnas de una clave foránea, así
que cada `UPDATE`/`DELETE` sobre `WaitlistEntry` forzaba un recorrido
secuencial completo de `WaitlistOffer` para resolver el `ON DELETE
RESTRICT` de esa FK — en particular, cada vez que
`WaitlistOfferTimeoutCron` (TASK-82) acepta o expira una oferta y elimina
la entrada de lista de espera correspondiente.

Se agregó `@@index([organizationId, waitlistEntryId])` al modelo
`WaitlistOffer` en `prisma/schema.prisma`, junto con la migración de Prisma
correspondiente. Sin cambios de lógica: es puramente un índice de soporte
para una relación que ya estaba correctamente acotada por inquilino.

## Decisiones y por qué

**Se indexó `(organizationId, waitlistEntryId)`, no solo
`waitlistEntryId`**, por consistencia con los otros dos índices de soporte
de FK compuesta del mismo modelo (`organizationId, appointmentId` /
`organizationId, patientId`) y porque toda lectura tenant-scoped en este
codebase filtra primero por `organizationId` — un índice que empieza por
esa columna sirve tanto al `WHERE organizationId = ? AND waitlistEntryId =
?` que la extensión de scoping por inquilino genera como al `WHERE
organizationId = ?` solo.

**No se tocó `WaitlistOfferTimeoutCron` ni `WaitlistReassignmentService`**,
tal como el propio ticket lo marca fuera de alcance: la corrección es
exclusivamente de esquema.

## Entidades / puertos / adaptadores tocados

- `WaitlistOffer` (Prisma): se agrega `@@index([organizationId,
  waitlistEntryId])`, con un comentario en el modelo que explica el porqué
  (PostgreSQL no indexa FKs automáticamente) y referencia a TASK-91.
- Migración nueva: `20260813164113_add_waitlist_offer_entry_index`
  (`CREATE INDEX` únicamente, sin alteración de datos).

## Tests y qué validan

No se agregaron tests nuevos — es un cambio de índice, sin superficie de
comportamiento observable para cubrir con una prueba. Verificación:

- El índice nuevo existe en la base tras la migración, confirmado vía
  `pg_indexes` contra Postgres local (mismo método que usó la auditoría
  original).
- Suite completa: 38 suites unitarias / 411 pruebas en verde; 36 suites e2e
  / 425 pruebas en verde (`--runInBand`). Lint limpio y verificación de
  tipos (`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-91-waitlist-offer-entry-index`
  (creada desde `origin/main` fresco, tras el merge de TASK-76). Pusheada a
  `origin`, no fusionada aún.
- Ticket: TASK-91 ("[CORRECCIÓN] Falta índice para la FK compuesta
  WaitlistOffer.waitlistEntryId"), corrección a TASK-82 (P4.5,
  [[FASE-4_PROMPT-5]], creó `WaitlistOffer`). Misma convención de bitácora
  dedicada para correcciones pequeñas dentro de la fase del ticket original
  que TASK-84 ([[FASE-2_PROMPT-11]]), TASK-79/TASK-81
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]]) y TASK-86
  ([[FASE-3_PROMPT-15]]).
