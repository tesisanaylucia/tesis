# Fase 3 — Motor de Turnos (backend) — `DELETE /lista-espera/:id` podía fallar con violación de FK (TASK-96, corrección a TASK-40)

## Contexto

Una auditoría multi-agente de `psique-back` en `main` (ángulo motor de
turnos/crons, 2026-08-12) detectó que `WaitlistService.remove`
(`src/waitlist/waitlist.service.ts`) borraba la `WaitlistEntry` sin
anular antes `WaitlistOffer.waitlistEntryId` de las ofertas que todavía
la referencian. La foreign key (`prisma/schema.prisma`,
`WaitlistOffer.waitlistEntryId`) es `onDelete: Restrict`.

El único lugar que sí anulaba esa columna antes de borrar era
`WaitlistReassignmentService.reserveForCandidate` (P4.5/TASK-82), y solo
en el camino de **aceptación** de una oferta. Cuando un candidato recibe
una oferta bajo reasignación AUTOMÁTICA y la rechaza, o la oferta expira
(`WaitlistOfferTimeoutCron`), la fila `WaitlistOffer` (status
REJECTED/EXPIRED) queda apuntando a su `waitlistEntryId` para siempre —
la entrada del candidato permanece intacta en la lista, y cualquier
`DELETE /lista-espera/:id` posterior sobre esa entrada tiraba una
violación de FK de Postgres (P2003) sin manejar (500), en vez de
borrarla: el candidato quedaba sin forma de salir de la lista de espera
por el endpoint normal.

## Qué se implementó

- `WaitlistService.remove` ahora anula `waitlistEntryId` de toda
  `WaitlistOffer` que referencie la entrada (`updateMany` por
  `waitlistEntryId`) dentro de la misma transacción, inmediatamente antes
  del `delete` de la `WaitlistEntry` — mismo patrón de
  update-luego-delete que ya usaba `reserveForCandidate`, ahora también
  en el path de borrado explícito.

## Decisiones y por qué

**Se generalizó el mismo patrón ya existente en vez de la alternativa
que sugería el propio ticket** (capturar la violación P2003 y traducirla
a un error legible). Anular la columna antes de borrar dentro de la
transacción deja el estado consistente sin introducir un segundo camino
de manejo de errores para el mismo problema — `reserveForCandidate` y
`remove` son ahora los dos únicos lugares que borran una
`WaitlistEntry`, y ambos lo hacen de la misma forma. Detectar y traducir
P2003 hubiera dejado la corrección atada al mensaje de error de Postgres
en vez de a la invariante real (una `WaitlistEntry` borrada nunca debe
dejar una `WaitlistOffer` con una referencia colgante).

**`updateMany` en vez de buscar primero las ofertas afectadas.** No hace
falta conocerlas: el `where: { waitlistEntryId: id }` ya acota la
operación a exactamente las que importan, y una entrada sin ninguna
oferta asociada (el caso ya cubierto, sin ofertas) hace que el
`updateMany` afecte cero filas sin efecto ni error — no se necesitó
ninguna rama condicional nueva.

## Alternativas descartadas

- **Capturar la violación de FK (P2003) y traducirla** — la alternativa
  que el propio ticket ofrecía como opción B. Descartada por las razones
  de arriba: hubiera dejado dos estrategias distintas (anular vs.
  capturar) para el mismo problema, dependiendo de qué código llegó
  primero al borrado.

## Entidades / puertos / adaptadores tocados

- `WaitlistService.remove` (`src/waitlist/waitlist.service.ts`): agrega
  el `waitlistOffer.updateMany` previo al `waitlistEntry.delete`, dentro
  de la transacción existente.

## Tests agregados o modificados

- `src/waitlist/waitlist.service.spec.ts`: nuevo caso en
  `describe('remove', …)` que verifica que `waitlistOffer.updateMany` se
  llama con `{ waitlistEntryId: id }` → `{ waitlistEntryId: null }` antes
  que `waitlistEntry.delete` (orden de invocación verificado con
  `mock.invocationCallOrder`).
- `test/waitlist.e2e-spec.ts`, `DELETE /lista-espera/:id`: dos casos
  nuevos contra Postgres real — una entrada con una `WaitlistOffer`
  REJECTED asociada se borra con 204, y lo mismo para EXPIRED. El caso ya
  cubierto (entrada sin ofertas) es el test preexistente
  `'removes an entry from the waitlist'`, que sigue pasando sin cambios.
  El fixture nuevo (`createEntryWithRespondedOffer`) crea un turno
  CANCELLED y una `WaitlistOffer` ya respondida apuntando a la entrada,
  reproduciendo exactamente el estado que dejaba el rechazo/expiración de
  una oferta real, siguiendo el mismo patrón de fixture directo por
  Prisma que `waitlist-offer-timeout.e2e-spec.ts`.

Suite completa verde tras el cambio: 38 suites unitarias / 416 pruebas;
37 suites e2e / 437 pruebas (`--runInBand`). Lint y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva — es una corrección puntual sobre un flujo de borrado ya
documentado (P3.7, gestión de lista de espera); no cambia ningún
diagrama existente ni el de la máquina de estados de la oferta (figura
27, introducida en [[FASE-4_PROMPT-5]]).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-96-waitlist-remove-fk-fix` (creada
  desde `origin/main` fresco, tras el merge de TASK-95). Pusheada a
  `origin`, PR abierta, no fusionada aún.
- Ticket: TASK-96 ("[CORRECCIÓN] DELETE de una entrada de lista de
  espera puede tirar 500 por violación de FK"), corrección a TASK-40
  (P3.7, implementó el CRUD de lista de espera, incluyendo `remove`) que
  depende de TASK-82 (P4.5, creó `WaitlistOffer`). Misma convención de
  bitácora dedicada para correcciones pequeñas dentro de la fase del
  ticket original que TASK-79/TASK-81/TASK-86/TASK-94/TASK-95
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]], [[FASE-3_PROMPT-15]],
  [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]]) y TASK-91/TASK-92
  ([[FASE-4_PROMPT-7]], [[FASE-1_PROMPT-9]]).
