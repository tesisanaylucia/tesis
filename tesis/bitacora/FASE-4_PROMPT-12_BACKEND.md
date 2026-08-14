# Fase 4 — Notificaciones y Scheduler (backend) — verificación de una duplicación ya resuelta (TASK-111, sobre TASK-101/TASK-43/TASK-45)

## Qué se implementó

TASK-111 no pedía código nuevo: pedía unificar en un solo helper el
formateo de hora en reloj de pared que, según una auditoría multi-agente
de `psique-back` en `main` (2026-08-12), estaba duplicado entre
`appointment-confirmation.cron.ts:32-39` (`formatScheduledAt`) y
`appointment-reminder.cron.ts:38-41` (`formatTime`), cada uno declarando
su propia función `pad` inline.

La verificación se hizo sobre una rama nueva creada desde `origin/main`
recién actualizado (`git fetch origin`), no contra una referencia local
potencialmente desactualizada — la misma precaución dejada como lección
en TASK-38 y aplicada de nuevo en TASK-81. Esa copia muestra que la
duplicación que describe el ticket **ya no existe**: ambos crons importan
`formatScheduledAt`/`formatTime` desde el archivo compartido
`src/notifications/notification-template.format.ts`, que contiene un
único `pad(value)` privado. La consolidación se hizo en TASK-101 (commit
`b39e22e`, PR #64, fusionado a `main` antes de la fecha de la auditoría
que originó este ticket) — como parte de mover las llamadas de
reagendado/cancelación por ausencia/oferta de lista de espera a pasar por
`NotificationTemplateService`, TASK-101 ya había extraído ambas funciones
de formateo (y su único `pad`) fuera de los dos crons hacia ese archivo
compartido, dejando el problema que TASK-111 describe resuelto como
efecto colateral. Un `grep` de `padStart` en `src/` confirma que no queda
ningún otro formateador de hora duplicado en el repositorio.

Es el mismo patrón que TASK-81 sobre TASK-78: una auditoría automatizada
reportando sobre un estado del código anterior a un merge que ya lo había
corregido.

## Decisiones y por qué

**No se modificó código de producción ni de tests.** El helper compartido
ya existe, ya lo usan ambos crons, y no queda duplicación que extraer.

**No se abrió PR en `psique-back`.** La rama `feature/TASK-111-cron-time-format-helper`
se creó desde `origin/main` fresco y se pusheó (mismo precedente de
trazabilidad que TASK-81: toda tarea de Jira, incluidas las de
verificación, tiene una rama propia), pero no lleva commits — no hay diff
que ofrecer contra `main`, así que un PR ahí no tendría contenido que
revisar ni mergear.

**Se dejó un comentario en TASK-111 documentando la verificación** (con
referencia explícita al commit de TASK-101 y a la fecha relativa de la
auditoría) y se transicionó el ticket a "Listo", en vez de reabrirlo o
dejarlo "En curso" — los criterios de aceptación ya se cumplen sin
cambios: ambos crons usan el mismo helper y el texto generado es
idéntico al de antes de esta verificación.

## Alternativas descartadas

- **Extraer igual un helper `formatWallClockTime` nuevo** (la sugerencia
  literal del ticket, en `src/common/dates/calendar-date.ts` o un archivo
  hermano) **aunque ya no hubiera duplicación que resolver**: descartada
  porque habría sido puro churn — mover código ya consolidado de un
  archivo a otro sin ninguna ganancia de mantenibilidad, y
  `calendar-date.ts` documenta explícitamente que solo cruza entre
  `"YYYY-MM-DD"` y `Date` a nivel de día calendario, no de hora de reloj;
  mezclar ahí un formateador de `HH:mm` habría sido una capa nueva de
  ambigüedad, no una simplificación.
- **Correr la suite e2e completa como parte de esta verificación**:
  descartada por no aportar información nueva sobre la pregunta que
  TASK-111 hace (¿sigue habiendo dos implementaciones del mismo helper?);
  se corrieron en cambio los tests unitarios de los tres archivos
  directamente involucrados (21/21 verdes), suficiente para confirmar que
  el comportamiento no cambió porque no se tocó nada.

## Entidades / puertos / adaptadores tocados

Ninguno.

## Tests y qué validan

No aplica — tarea de verificación documental, sin cambios de código. Se
corrieron (sin modificarlos) `appointment-confirmation.cron.spec.ts` y
`appointment-reminder.cron.spec.ts` como confirmación adicional de que el
comportamiento actual sigue verde (21/21 tests).

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-111-cron-time-format-helper`
  (creada desde `origin/main` fresco, sin commits propios). Pusheada a
  `origin`.
- Ticket: TASK-111 (Jira), "[MEJORA] Helper de formato de hora duplicado
  entre appointment-confirmation.cron.ts y appointment-reminder.cron.ts".
  Referencia también a TASK-101 (consolidación real del helper) y a
  [[FASE-4_PROMPT-4]] (implementación original de
  `AppointmentReminderCron`, TASK-45).
