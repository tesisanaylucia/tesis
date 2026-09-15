# Fase 3 — Motor de Turnos (backend) — verificación del estado real del puerto de confirmación de reprogramación (TASK-184, verificación sobre TASK-115)

## Qué se implementó

TASK-184 no pedía código nuevo, aunque su redacción sí lo sugería: pedía,
en los hechos, determinar si el defecto que describe —
`PATCH /turnos/:id/reprogramar` y `POST /profesionales/:id/reorganizar-agenda`
fallando siempre porque `RESCHEDULE_RESPONSE_PORT` seguía atado en
producción a `StubRescheduleResponseAdapter.awaitConfirmation`, fijo en
"no"— seguía presente en `main`. El propio ticket citaba como fuente una
auditoría multi-agente del 2026-08-28.

La verificación se hizo sobre `origin/main` recién actualizado (`git fetch
origin`), la misma precaución documentada para TASK-38 y aplicada de nuevo
en TASK-81. El árbol de `origin/main` no contiene ningún
`stub-reschedule-response.adapter.ts`: `RescheduleResponsePort` ya expone
`registerOffer`/`getStatus` (no `awaitConfirmation`), `integrations.module.ts`
bindea `RESCHEDULE_RESPONSE_PORT` a un `RescheduleResponseAdapter` real que
persiste `RescheduleOffer` vía `TenantScopedPrismaService`, y
`AppointmentsService.acceptRescheduleOffer`/`.rejectRescheduleOffer`
resuelven cada oferta — exactamente el rediseño async que el "fix sugerido"
del ticket pedía, siguiendo el patrón de `WaitlistResponseAdapter`. Ese
trabajo ya está documentado en [[FASE-3_PROMPT-30]] (TASK-115, fusionado a
`main` el 2026-09-07 según la migración `20260907120000_reschedule_offer_async`),
tres semanas antes de la fecha de la auditoría que originó TASK-184. La
explicación más probable, la misma que ya se dio para TASK-81/TASK-78, es
que la auditoría corrió contra una copia del repositorio anterior a ese
merge.

Se revisó también, contra el anteproyecto de tesis (fuente de verdad en
Drive), la única diferencia real entre lo implementado en TASK-115 y el
"fix sugerido" del ticket: éste pedía además "un cron de timeout análogo a
`WaitlistOfferTimeoutCron`". [[FASE-3_PROMPT-30]] ya había decidido, de
forma explícita, no agregarlo porque el documento de requisitos no fija
ningún plazo de expiración para una oferta de reprogramación (a diferencia
de las cuatro horas que sí fija para la lista de espera). No se encontró en
el anteproyecto ningún plazo para este flujo que contradijera esa decisión.

Se corrió la suite relacionada (`integrations.module.spec.ts`,
`appointments-rescheduling.service.spec.ts` y los tests de
`src/infrastructure/adapters/`) contra el checkout de verificación: 6
conjuntos, 91 pruebas, todas en verde — incluyendo el caso de
`integrations.module.spec.ts` que resuelve `RESCHEDULE_RESPONSE_PORT` sin
sobreescribir el provider, exactamente el binding de producción cuya
ausencia de cobertura señalaba el ticket.

## Decisiones y por qué

**No se modificó código de producción ni de tests.** El puerto async y su
adaptador real ya existen, están fusionados a `main` desde TASK-115 y
tienen cobertura sobre el binding real de producción; no había ningún hueco
que TASK-184 identificara correctamente y que siguiera abierto.

**No se agregó el cron de timeout que sugería el "fix sugerido".** Sería
repetir, sin ningún elemento nuevo, la misma decisión ya tomada y
documentada en TASK-115: el documento de requisitos no define un plazo
para este flujo, y fijar uno de todos modos introduciría una regla de
negocio que ninguna fuente respalda — lo mismo que el proyecto evita en
general tratando las reglas de negocio como datos configurables, no como
valores que el código elige por su cuenta.

**Se dejó un comentario en TASK-184 documentando la verificación** (con
referencia directa a TASK-115) y se transicionó el ticket a "Listo", en
lugar de dejarlo abierto o reabrir TASK-115 — el mismo tratamiento que
TASK-81 le dio a un hallazgo equivalente sobre TASK-78.

**Se creó de todos modos una rama dedicada**
(`feature/TASK-184-verify-reschedule-response-adapter`, desde `main`
fresco), sin commits de código, siguiendo el mismo precedente de
trazabilidad de TASK-80/TASK-81.

## Alternativas descartadas

- **Confiar directamente en el hallazgo de la auditoría y reconstruir un
  `RescheduleResponseAdapter` "nuevo"**: descartada al confirmar que ya
  existe, con el mismo diseño que el ticket pedía, desde TASK-115 —
  reimplementarlo habría duplicado código y arriesgado introducir una
  regresión sobre un flujo ya correcto y probado.
- **Agregar igualmente el cron de timeout "por las dudas", ya que el ticket
  lo pedía explícitamente**: descartada por la misma razón que TASK-115 ya
  documentó — ninguna fuente de verdad fija un plazo para este flujo.

## Entidades / puertos / adaptadores tocados

Ninguno.

## Tests y qué validan

No aplica — tarea de verificación documental, sin cambios de código. La
verificación en sí consistió en `git fetch origin` + inspección del árbol y
el historial de `origin/main` para `reschedule-response.port.ts`/
`.adapter.ts`/`integrations.module.ts`, más una corrida acotada de la suite
relacionada (ver más arriba) para confirmar que sigue en verde.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-184-verify-reschedule-response-adapter`
  (creada desde `main` fresco, sin commits propios). Pusheada a `origin`.
- Ticket: TASK-184 (Jira), "[ALTO] Reprogramación de turnos rota en
  producción — puerto de confirmación es un stub". Referencia también a
  TASK-115 (corrección ya fusionada, [[FASE-3_PROMPT-30]]) y a
  [[FASE-3_PROMPT-14]] (TASK-81, mismo patrón de verificación sobre un
  hallazgo de auditoría desactualizado).
