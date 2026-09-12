# Fase 4 — Notificaciones y Scheduler (backend) — dos de las siete plantillas de notificación no tenían migración de datos (TASK-159, corrección a TASK-42/TASK-101)

## Qué se implementó

TASK-159 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, validada por la usuaria
antes de implementarse. El hallazgo (TUR-11) señalaba que la migración
`20260812140000_seed_notification_templates` (P4.1, TASK-42; documentada
más arriba en esta sección) insertaba solo 5 de las 7 claves de
`NotificationTemplateKey` que existían al momento de la auditoría, y que
CLAUDE.md exige que una regla nueva a nivel de inquilino tenga su fila por
defecto en una migración, no solo en el seed.

Las dos claves faltantes, `APPOINTMENT_RESCHEDULE` y
`APPOINTMENT_RESCHEDULE_CONFIRMATION_REQUEST`, se habían agregado al
enumerado el 13/08/2026 (TASK-101, un día después de que la migración de
siembra original ya se hubiera escrito y aplicado el 12/08/2026), de modo
que el `ON CONFLICT DO NOTHING` de aquella migración nunca llegó a verlas.
Ninguna migración posterior cerró esa brecha. La corrección agregó una
migración de datos nueva, `20260912140000_seed_appointment_reschedule_templates`,
que inserta el texto base de esas dos claves en `OrganizationConfig` para
toda organización existente.

## Decisiones y por qué

**La migración reproduce exactamente la forma de la migración original y de
su misma corrección posterior** (`20260910120000_seed_appointment_notice_templates`,
que cerró la misma brecha para `APPOINTMENT_CANCELLED_NOTICE` y
`APPOINTMENT_REASSIGNED_NOTICE`, dos claves agregadas después de esa
migración original de forma análoga): un `INSERT ... SELECT` que cruza cada
`Organization` existente con la lista de claves y textos nuevos, con
`ON CONFLICT ("organizationId", "key") DO NOTHING`. La condición de no
sobrescritura es la misma en las tres migraciones: una organización que ya
hubiera personalizado alguna de estas dos claves —posible únicamente si lo
hizo antes de que existiera esta migración, ya que antes de ella la fila
simplemente no existía— conserva su propio texto.

**No fue necesario tocar `prisma/seed.ts` ni
`notification-template.constants.ts`.** El seed de desarrollo ya recorre
`Object.values(NotificationTemplateKey)` en lugar de declarar su propia
lista de claves, de modo que una organización creada después del hallazgo
ya recibía ambas filas sin cambio alguno; el defecto era exclusivamente que
una organización *ya existente* al momento en que se agregaron las dos
claves no tenía forma de recibirlas retroactivamente sin una migración. El
motor de plantillas (`NotificationTemplateService`) tampoco cambió: ya
prefiere la fila de `OrganizationConfig` del inquilino sobre
`DEFAULT_NOTIFICATION_TEMPLATES` cuando existe alguna, y sigue cayendo a la
misma constante cuando falta, de modo que ninguna combinación de "cuándo se
creó este inquilino" deja una plantilla sin definir ni con dos textos
distintos entre sí.

**Alcance de la corrección: sólo las dos claves que la propia auditoría
identificó**, no las demás claves del enumerado que tampoco tienen
migración propia (`ACCESS_CODE_DELIVERY` y `ACCESS_CODE_ADHOC_DELIVERY`,
agregadas el 26/08/2026 por TASK-58, después de la fecha de la auditoría
que originó este ticket). Esas dos quedan fuera del hallazgo TUR-11 tal
como está redactado y no se tocaron en esta tarea; si se confirma que
comparten el mismo defecto, es una corrección propia con su propio ticket,
no una ampliación silenciosa del alcance de éste.

## Entidades / puertos / adaptadores tocados

- `prisma/migrations/20260912140000_seed_appointment_reschedule_templates/migration.sql`
  (nueva): siembra retroactiva de
  `notification_template_appointment_reschedule` y
  `notification_template_appointment_reschedule_confirmation_request` para
  toda organización existente, con el mismo texto base que ya declaraba
  `DEFAULT_NOTIFICATION_TEMPLATES`.

No hubo cambios en `schema.prisma`, en ningún servicio ni en ningún DTO:
la tarea es puramente una migración de datos correctiva.

## Tests y qué validan

No se agregó ninguna prueba automatizada nueva: no existe, en este
proyecto, una prueba que verifique el contenido de una migración de siembra
por sí sola (las tres migraciones de este tipo en la sección —la original,
la de aviso/reasignación y esta— se verifican por inspección directa de la
base de datos tras aplicarlas, no por un `*.e2e-spec.ts`). Se verificó
manualmente, contra una base de datos Postgres local reiniciada desde cero
(`docker compose down -v` seguido de `docker compose up -d db`), que las
61 migraciones existentes —incluida la nueva— se aplican en orden sin
error (`prisma migrate deploy`) y que las dos filas quedan efectivamente
sembradas para la organización de desarrollo ya existente.

Suite completa sin regresiones: 143 suites / 1588 pruebas en verde
(`npm run test:cov`, unitarias + e2e combinadas) contra la base de datos
recién reiniciada. Una corrida anterior, contra la base de datos local sin
reiniciar, había mostrado la misma única falla preexistente ya documentada
en la entrada anterior de esta sección
([[FASE-4_PROMPT-16_BACKEND|FASE-4_PROMPT-16]]) —
`patients-entities.e2e-spec.ts`, "admits the same DNI in a different
organization"— causada esta vez por un paciente de prueba con ese DNI
dejado en la organización de desarrollo por una sesión de trabajo anterior;
se confirmó ajena a este cambio (la migración no toca la tabla de
pacientes) y se resolvió reiniciando la base de datos en lugar de tocar
código de producción para maquillar un dato de desarrollo obsoleto. `lint`
sin errores sobre el archivo agregado.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-159-reschedule-template-migration` (creada desde
  `origin/main` fresco, tras el merge de TASK-158).
- Ticket: TASK-159 ("Dos de las siete plantillas de notificación no tienen
  migración de datos"), tarea de auditoría automática (hallazgo TUR-11)
  validada por la usuaria antes de implementarse. Misma convención de
  bitácora dedicada para una corrección puntual que TASK-158
  ([[FASE-4_PROMPT-16_BACKEND|FASE-4_PROMPT-16]]) y TASK-157
  ([[FASE-4_PROMPT-15_BACKEND|FASE-4_PROMPT-15]]).
