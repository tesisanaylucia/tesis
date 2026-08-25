# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Auditoría de integridad, normalización y relaciones del esquema

## Qué se implementó

Se realizó una auditoría del estado real de la base de datos de desarrollo
(`psique_dev`, PostgreSQL 16 vía Prisma), previa a continuar con la
implementación de los módulos siguientes, contrastando tanto el archivo
`prisma/schema.prisma` como el historial de migraciones aplicadas contra la
base real (mediante `prisma migrate status`/`deploy` y consultas directas al
catálogo de PostgreSQL: `pg_constraint`, `pg_indexes`, `_prisma_migrations`).
La auditoría se organizó en torno a los tres pilares que el propio esquema
ya declara como reglas del proyecto (integridad, normalización, relaciones
mediante claves foráneas compuestas — ver el bloque de comentarios al inicio
de `schema.prisma`) y confirmó que el diseño existente ya sostenía esas tres
propiedades de forma consistente: las 38 foreign keys documentadas existen
1:1 en la base real, los 5 `CHECK` mencionados en comentarios (rango de
fechas de `Absence`, `duration > 0` y consistencia de `holdUntil`/`holdReason`
en `Appointment`, `order > 0` en `WaitlistEntry`, consistencia rol/profesional
en `User`) están efectivamente presentes, y cada FK compuesta tiene su
índice de respaldo.

De la auditoría surgieron tres hallazgos concretos, los tres corregidos en
esta misma tarea:

1. Un vacío de integridad relacional en `WaitlistOffer`: la FK compuesta
   hacia `WaitlistEntry` sólo garantizaba que ambas filas pertenecieran a la
   misma organización, no que fueran del mismo paciente.
2. Un ledger de migraciones de Prisma inconsistente (`_prisma_migrations`)
   que bloqueaba `prisma migrate dev` en el ambiente local.
3. Datos de prueba acumulados en el catálogo global `HealthInsurer`, dejados
   por corridas de tests e2e anteriores.

## Decisiones y por qué

**El hallazgo de `WaitlistOffer` se corrigió ampliando la FK compuesta
existente de dos a tres columnas, no agregando una validación a nivel de
aplicación.** `WaitlistOffer` tenía `waitlistEntryId` (nullable, FK
compuesta `(organizationId, waitlistEntryId) → WaitlistEntry(organizationId,
id)`) y `patientId` (columna propia, FK compuesta independiente hacia
`Patient`), pero nada ataba a nivel de base de datos que el paciente de la
oferta fuera el mismo que el del `WaitlistEntry` señalado. En la práctica
ambos valores siempre coinciden porque `WaitlistResponseAdapter.registerOffer`
los deriva del mismo candidato de la cola de espera, pero esa garantía sólo
la sostenía el código de aplicación — exactamente el tipo de promesa que la
regla 3/4 del propio esquema (documentada en el encabezado de
`schema.prisma`) señala como insuficiente y reemplaza por una FK compuesta.
Se siguió ese mismo patrón ya establecido en el código (visible en
`PatientProfessional`, `Appointment` y el propio `WaitlistEntry`) en lugar de
introducir un mecanismo nuevo: se agregó `@@unique([organizationId, id,
patientId])` en `WaitlistEntry` (reemplazando el `@@unique([organizationId,
id])` anterior, que sólo existía para servir de destino a esta FK y quedó
subsumido por el nuevo índice de tres columnas) y se amplió la FK de
`WaitlistOffer.waitlistEntry` a
`(organizationId, waitlistEntryId, patientId) → (organizationId, id,
patientId)`. Con `MATCH SIMPLE` (el comportamiento por defecto de
PostgreSQL), la restricción queda sin verificar mientras `waitlistEntryId`
sea `NULL` — el estado ordinario de una oferta ya aceptada, cuando el
servicio limpia esa columna antes de borrar el `WaitlistEntry` — por lo que
el cambio no afecta ese flujo existente.

**El ledger de migraciones corrupto se resolvió con el mecanismo estándar de
Prisma, no reseteando la base.** La tabla `_prisma_migrations` tenía dos
filas para la misma migración, `20260821050000_seed_chatbot_config`: un
primer intento que había fallado en PostgreSQL con el error `42804 — could
not determine polymorphic type because input has type unknown` (causado por
`to_jsonb(E'...')` sobre un literal de texto sin *cast* explícito), y un
segundo intento, exitoso, después de que el commit `TASK-53` agregara el
*cast* `::text` a esa misma línea. El *fix* en sí era correcto, pero el
intento fallido nunca se marcó como resuelto en el historial, así que
cualquier sesión nueva de `prisma migrate dev` se negaba a continuar y
sugería `prisma migrate reset` (que habría descartado el estado de la base
local). Se optó por `prisma migrate resolve --rolled-back
20260821050000_seed_chatbot_config` — el comando que Prisma documenta para
esta situación exacta — en lugar de un reseteo, porque no modifica datos ni
el esquema, sólo reconcilia el registro de qué se aplicó; se comprobó que
`prisma migrate deploy` (el camino que corre en CI/producción) nunca se vio
afectado por este problema, sólo el flujo interactivo de `migrate dev` en
desarrollo local.

**La limpieza de `HealthInsurer` se trató como higiene de datos, no como un
defecto de diseño.** `HealthInsurer` es, deliberadamente, la única entidad
del esquema sin organización propia (es un catálogo del mundo real que cada
profesional acepta o no, según el comentario de diseño en el propio modelo),
lo cual significa que ningún mecanismo de limpieza de tests que borra por
cascada desde `Organization` llega a tocarla nunca. Varios specs e2e del
módulo de pacientes crean su propia aseguradora con nombres identificables
(`e2e-patients-abmc-insurer`, etc.) para no interferir con las cuatro del
seed, pero esas filas quedan para siempre en cualquier base contra la que
corran. Se borraron las tres filas de prueba (ninguna tenía una fila
referenciándola, verificado antes de borrar) y se dejó registrada, como
recomendación para una tarea futura, la falta de limpieza de catálogos
globales en la suite e2e.

## Entidades tocadas

- `prisma/schema.prisma`: `WaitlistEntry.@@unique` (de dos a tres columnas)
  y `WaitlistOffer.waitlistEntry` (FK compuesta de dos a tres columnas),
  ambos con el comentario de diseño actualizado explicando la regla que
  cierran.
- `prisma/migrations/20260825210000_waitlist_offer_entry_patient_consistency/migration.sql`:
  nueva migración, escrita a mano con el mismo DDL que generó
  `prisma migrate diff` contra el schema actualizado (`DROP`/`ADD
  CONSTRAINT`, `DROP`/`CREATE INDEX`), aplicada con `prisma migrate deploy`
  en lugar de `migrate dev` por el problema de ledger descrito arriba.
- Ningún archivo de `src/` cambió: el hallazgo no requirió tocar
  `WaitlistResponseAdapter` ni `WaitlistReassignmentService`, porque ambos ya
  escriben `patientId` y `waitlistEntryId` de forma consistente.

## Tests

No se agregaron tests nuevos — la tarea fue una auditoría y endurecimiento
de esquema, no una funcionalidad nueva. Se verificó que el cambio no rompiera
nada existente:

- `npx tsc --noEmit`: sin errores, contra el cliente de Prisma regenerado.
- Suites unitarias de `waitlist` (`waitlist.service.spec.ts`,
  `waitlist-reassignment.service.spec.ts`): 40/40 en verde.
- Suites e2e de `waitlist` contra la base PostgreSQL real
  (`waitlist.e2e-spec.ts`, `waitlist-offer-timeout.e2e-spec.ts`): 20/20 en
  verde.
- `prisma migrate status` sin drift entre `schema.prisma` y la base real,
  antes y después del cambio.

La base de desarrollo no tenía datos reales al momento de la auditoría (0
organizaciones, 0 pacientes, 0 turnos) — sólo el catálogo `HealthInsurer`
mostraba filas, resultado de corridas de tests e2e anteriores. Por eso la
auditoría fue de esquema y de historial de migraciones, no de datos de
producción: no había datos de producción que revisar.

## Figuras pendientes

Ninguna nueva. El diagrama entidad-relación de `WaitlistEntry`/`WaitlistOffer`
ya está registrado como pendiente desde la tarea que introdujo
`WaitlistOffer` (TASK-82, P4.5); si esa figura llega a incorporarse, debería
reflejar la FK de tres columnas en lugar de la de dos.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `main` (tarea realizada directamente sobre `main`,
  sin ticket Jira asociado — auditoría solicitada explícitamente por la
  autora antes de continuar con los próximos módulos, no parte de la
  planificación de tickets del backlog).
- Cambios sin commitear al cierre de la tarea: quedan en el working tree de
  `back/` a la espera de revisión.
