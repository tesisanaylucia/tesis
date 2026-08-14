# Fase 3 — Motor de Turnos (backend) — la auditoría de reagendado logeaba los timestamps en vez de solo el nombre del campo (TASK-100, corrección a TASK-39)

## Contexto

Una auditoría multi-agente de `psique-back` en `main` (ángulo convenciones
de `CLAUDE.md`, 2026-08-12) detectó que `rescheduleCore`
(`src/appointments/appointments.service.ts`, usado tanto por
`reschedule()` como por `reorganizeAgenda()`) escribía la entrada de
auditoría del reagendado con los valores reales de la fecha:

```ts
detail: {
  previousScheduledAt: previousScheduledAt.toISOString(),
  newScheduledAt: newScheduledAt.toISOString(),
},
```

`CLAUDE.md`, sección "Audit trail", es explícito: *"An entry's `detail`
names fields, never values"*, reforzado por el comentario de
`src/audit/changed-fields.ts` ("Names only, never values: an audit entry
records that something changed on a resource, not the personal or
clinical data it carried"). `rescheduleCore` era el único path de
mutación de turnos que no seguía la regla — el path hermano
`guardedStatusUpdate`/`applyFieldUpdate` (mismo archivo, escrituras de
pago/orden vía `PATCH /turnos/:id/pago` y `.../orden`) sí usa
`detail: { fields }` con el nombre del campo modificado, nunca su valor.
Una exportación de cumplimiento ("qué cambió en este turno") terminaba
revelando también los timestamps reales de agenda del paciente — el dato
personal que la regla existe justamente para mantener fuera de esa
columna.

## Qué se implementó

- `rescheduleCore` ahora escribe `detail: { fields: ['scheduledAt'] }` en
  la entrada de auditoría `RESCHEDULE`, en vez de los dos timestamps
  ISO. `scheduledAt` es el único campo que este flujo modifica
  (`confirmationRequestedAt`/`reminderSentAt` se resetean como efecto
  colateral de idempotencia, ya explicado en [[FASE-3_PROMPT-17]], no
  como el cambio que la auditoría documenta), así que el nombre queda
  fijo como literal, siguiendo el mismo patrón que
  `setPayment`/`setReferralOrder` (`applyFieldUpdate`, un array literal
  de un solo campo conocido) en vez de derivarlo con `changedFields(dto)`
  — ese helper existe para DTOs con múltiples campos opcionales
  (`class-transformer` instancia todas las propiedades declaradas), y
  `rescheduleCore` no recibe un DTO de ese tipo, sino un `Date` ya
  validado.
- La variable local `previousScheduledAt`, que solo existía para
  alimentar el `detail` retirado, se eliminó junto con su asignación.

## Decisiones y por qué

**Literal `['scheduledAt']`, no `changedFields`.** El ticket ofrecía
ambas formas como equivalentes ("`['scheduledAt']` o equivalente vía
`changedFields`"). Se eligió el literal porque `rescheduleCore` no opera
sobre un DTO con campos opcionales — recibe un `newScheduledAt: Date` ya
resuelto por sus dos únicos llamadores (`reschedule`,
`reorganizeAgenda`) — y es exactamente la misma situación que
`setPayment`/`setReferralOrder` ya resuelven con un array literal en vez
de invocar `changedFields` sobre un objeto de un solo campo.

**No se tocó qué otros campos escribe la transacción.** El ticket es
explícito en que la lógica de reagendado en sí queda fuera de alcance
(remite al ticket de `confirmationRequestedAt`/`reminderSentAt`, ya
resuelto en TASK-95); esta corrección es exclusivamente sobre qué se
audita, no sobre qué se escribe en la fila del turno.

## Alternativas descartadas

- **Usar `changedFields(dto)`** tal como lo hace el path de
  actualización de campos con DTO — descartada porque no hay un DTO con
  campos opcionales en este flujo; habría sido una capa de indirección
  sin un objeto real que recorrer.

## Entidades / puertos / adaptadores tocados

- `AppointmentsService.rescheduleCore`
  (`src/appointments/appointments.service.ts`): cambia el `detail` de la
  entrada de auditoría `RESCHEDULE` de los dos timestamps a
  `{ fields: ['scheduledAt'] }`; elimina la variable
  `previousScheduledAt`, ya sin otro uso en el método.

## Tests agregados o modificados

- `src/appointments/appointments-rescheduling.service.spec.ts`: el caso
  existente de `describe('reschedule', …)` se actualizó para esperar
  `detail: { fields: ['scheduledAt'] }` en vez de los dos timestamps
  ISO, y se renombró para dejar constancia del criterio (TASK-100).
- `test/appointments-rescheduling.e2e-spec.ts`, `PATCH
  /turnos/:id/reprogramar`: el caso existente se actualizó para el mismo
  criterio contra Postgres real — además de `toEqual({ fields:
  ['scheduledAt'] })`, agrega una aserción negativa explícita
  (`JSON.stringify(audit?.detail)` no contiene el nuevo `scheduledAt`)
  como prueba directa del criterio de aceptación del ticket ("el
  `detail` no contiene ningún valor de fecha/hora").

Suite completa verde tras el cambio: 38 suites unitarias / 422 pruebas;
37 suites e2e / 439 pruebas (`--runInBand`). Lint y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva — es una corrección puntual sobre qué se audita en un flujo
de escritura ya documentado (P3.6, reprogramación de turnos); no cambia
ningún diagrama existente.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-100-reschedule-audit-fields`
  (creada desde `origin/main` fresco, tras el merge de TASK-99). Pusheada
  a `origin`, PR abierta, no fusionada aún.
- Ticket: TASK-100 ("[CORRECCIÓN] Auditoría de reagendado de turno logea
  valores en vez de nombres de campo"), corrección a TASK-39 (P3.6,
  implementó `reschedule`/`reorganizeAgenda`/`rescheduleCore`). Misma
  convención de bitácora dedicada para correcciones pequeñas dentro de la
  fase del ticket original que TASK-79/TASK-81/TASK-86/TASK-94/TASK-95/TASK-96
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]], [[FASE-3_PROMPT-15]],
  [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]], [[FASE-3_PROMPT-18]]) y
  TASK-91/TASK-92/TASK-97/TASK-98 ([[FASE-4_PROMPT-7]],
  [[FASE-1_PROMPT-9]], [[FASE-4_PROMPT-9]], [[FASE-2_PROMPT-13]]).
