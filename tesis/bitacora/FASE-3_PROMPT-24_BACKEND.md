# Fase 3 — Motor de Turnos (backend) — `WaitlistService.reorder` actualizaba filas secuencialmente dentro de una transacción SERIALIZABLE (TASK-110, mejora sobre TASK-40)

## Contexto

Una auditoría multi-agente de `psique-back` en `main` (ángulo reuse/
simplificación/eficiencia, 2026-08-12) encontró que
`WaitlistService.reorder` (`src/waitlist/waitlist.service.ts`) recorría
`for (const [index, id] of dto.order.entries())` y esperaba cada
`tx.waitlistEntry.update(...)` uno por uno, en vez de lanzar los updates
independientes en paralelo con `Promise.all`.

El costo real: con el tope de lote `MAX_WAITLIST_REORDER_BATCH_SIZE = 200`
(`waitlist.constants.ts`), un reordenamiento completo podía serializar
hasta 200 round trips de red de punta a punta en vez de solaparlos,
extendiendo innecesariamente cuánto tiempo la transacción SERIALIZABLE (y
su huella de locks) queda abierta en un endpoint administrativo de uso
frecuente.

## Qué se implementó

- El loop `for` secuencial dentro de `reorder` se reemplazó por
  `Promise.all(dto.order.map((id, index) => tx.waitlistEntry.update(...)))`,
  dado que cada `update` toca una fila distinta (`id`) y las N escrituras
  son independientes entre sí.
- Sin cambios de esquema, de firma pública ni de la validación previa del
  payload (que sigue rechazando un `order` que no contiene exactamente las
  entradas actuales del profesional).

## Decisiones y por qué

**Se mantuvo el `.map` con índice en vez de reconstruir el arreglo de
promesas con un `for` que las acumula.** `Array.prototype.map` recorre
`dto.order` en el mismo orden en que el `for` original lo hacía,
disparando cada llamada a `tx.waitlistEntry.update` de forma síncrona
antes de que ninguna se resuelva — el test unitario preexistente que
verifica el orden de invocación con `toHaveBeenNthCalledWith` sigue
pasando sin modificaciones, porque el orden de *inicio* de cada llamada
no cambió, sólo dejaron de esperarse una por una.

**No se tocó la lógica de reordenamiento en sí ni el tope de 200**, tal
como pide explícitamente el alcance del ticket — la única fila afectada
es la del loop, no la validación de conjunto ni el resto de la
transacción (que sigue auditando y devolviendo la lista reordenada dentro
del mismo `runSerializable`).

## Alternativas descartadas

- **Envolver los updates en un `tx.$transaction([...])` de tipo batch en
  vez de `Promise.all` dentro de la transacción interactiva existente**:
  descartada porque `reorder` ya corre dentro de una transacción
  interactiva vía `runSerializable(tx => ...)` (necesaria para leer el
  estado actual, validar el conjunto propuesto y recién después escribir,
  todo bajo aislamiento SERIALIZABLE) — un batch de nivel superior no
  puede anidarse dentro de esa transacción interactiva ya abierta, y
  `Promise.all` sobre el mismo `tx` sí resuelve el objetivo (paralelizar
  las N escrituras) sin cambiar el modelo transaccional.

## Entidades / puertos / adaptadores tocados

- `WaitlistService.reorder` (`src/waitlist/waitlist.service.ts`): único
  método modificado.

## Tests agregados o modificados

Ninguno nuevo — el comportamiento observable no cambia (mismo orden
final, mismo manejo de conflictos SERIALIZABLE), y el test unitario
existente en `waitlist.service.spec.ts` ("resequences entries to match
the submitted order", que verifica con `toHaveBeenNthCalledWith` que las
tres llamadas a `update` se disparan en el orden esperado) ya alcanza
para probar que el cambio no alteró el resultado. Suite completa verde
tras el cambio: 39 suites unitarias / 434 pruebas; 37 suites e2e / 439
pruebas (`--runInBand`). Lint y verificación de tipos (`tsc --noEmit`)
sin errores. Docker (`back-db-1`) ya estaba corriendo.

## Figuras pendientes

Ninguna nueva — es una mejora de eficiencia puntual sobre un endpoint ya
documentado (P3.7, gestión de lista de espera) sin cambio de esquema ni
de comportamiento observable.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-110-waitlist-reorder-parallel-updates`
  (creada desde `origin/main` fresco, tras el merge de TASK-109). Pusheada
  a `origin`, PR abierta, no fusionada aún.
- Ticket: TASK-110 ("[MEJORA] WaitlistService.reorder actualiza filas
  secuencialmente dentro de una transacción SERIALIZABLE abierta"), mejora
  sobre TASK-40 (P3.7, implementó `reorder`). Misma convención de bitácora
  dedicada para tareas puntuales dentro de la fase del ticket original que
  TASK-79/TASK-81/TASK-86/TASK-94/TASK-95/TASK-96/TASK-100/TASK-108
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]], [[FASE-3_PROMPT-15]],
  [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]], [[FASE-3_PROMPT-18]],
  [[FASE-3_PROMPT-19]], [[FASE-3_PROMPT-23]]).
