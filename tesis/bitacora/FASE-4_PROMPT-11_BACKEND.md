# Fase 4 — Notificaciones y Scheduler (backend) — helper compartido para el patrón "por organización + usuario SYSTEM" duplicado en 5 crons (TASK-109, mejora sobre TASK-43/TASK-44/TASK-45/TASK-82/TASK-89)

## Qué se implementó

TASK-109 fue una tarea de mejora hallada por la auditoría multi-agente de
`psique-back` sobre `main`, 2026-08-12, con ángulo reuso/simplificación/
eficiencia. Los cinco trabajos programados del sistema —el de solicitud de
confirmación (P4.2, TASK-43), el de autocancelación a las 4h sin respuesta
(P4.3, TASK-44), el de recordatorio configurable por inquilino (P4.4,
TASK-45), el de vencimiento de ofertas de lista de espera (P4.5, TASK-82) y
el de autocompletado semanal de turnos vencidos (TASK-89)— repetían, cada
uno por su cuenta, las mismas veinte líneas aproximadas: una consulta de
todas las organizaciones, un recorrido que abre el contexto de inquilino de
cada una, la resolución del usuario SYSTEM de esa organización, y un
`warn`-y-`skip` si no existía ninguno. Cada archivo copiaba, además, el
mismo comentario explicando la particularidad de `AsyncLocalStorage` que
exige que el callback pasado a `tenantContext.run` sea `async` y espere su
propio trabajo, o el contexto de inquilino se pierde apenas el cuerpo
síncrono del bucle retorna.

La duplicación no era solo redundancia de código: una mejora futura a esa
forma —agrupar la resolución del usuario SYSTEM en una sola consulta,
agregar métricas o trazas por organización— habría exigido editar cinco
archivos de manera idéntica, y era razonablemente probable que un sexto
trabajo programado futuro copiara uno de los cinco existentes en lugar de
reutilizar un mecanismo común. Se extrajo un helper compartido,
`runForEachOrganizationAsSystem` (`src/common/scheduling/`), que encapsula
el recorrido de organizaciones, la resolución del usuario SYSTEM y la
apertura del contexto de inquilino, documentando la particularidad de
`AsyncLocalStorage` en un único lugar en lugar de cinco. Los cinco trabajos
programados se migraron a usarlo.

## Decisiones y por qué

**Función exportada simple, no un servicio inyectable.** El helper recibe
como parámetros explícitos las dependencias que ya tenía cada trabajo
programado inyectadas (el cliente Prisma sin ámbito de inquilino, el
servicio de contexto de inquilino, el logger del propio cron y una
descripción textual para el mensaje de advertencia), en lugar de resolverse
por inyección de dependencias de NestJS. Se siguió el precedente ya
existente en el propio módulo común del backend,
`runSerializable` (`src/common/prisma/serializable-transaction.ts`), que
resuelve un problema de forma análoga —una operación transversal sobre
Prisma reutilizada por varios servicios— con la misma forma: una función
async con las dependencias como parámetros, no una clase con constructor.

**El callback de trabajo recibe el identificador de organización y el del
usuario SYSTEM ya resueltos, no vuelve a buscarlos.** Antes de la
extracción, cada método privado de cada cron repetía la consulta del
usuario SYSTEM y su comprobación de existencia. Con el helper, esa
responsabilidad se centraliza por completo: el callback que cada cron le
pasa recibe ambos valores como argumentos y ya no necesita tocar el cliente
Prisma sin ámbito de inquilino en absoluto para esa parte del trabajo. Tres
de los cinco crons (confirmación, recordatorio, autocompletado semanal) no
necesitaban el identificador de organización dentro de su propio método
privado —solo lo usaban para la consulta del usuario SYSTEM, ahora
eliminada—, así que ese parámetro se descartó de sus firmas; los otros dos
(autocancelación, que dispara una reasignación con el identificador de
organización como parte del payload; vencimiento de ofertas, que lo recibía
pero tampoco lo usaba y también se simplificó) se dejaron con la firma que
efectivamente necesitaban.

**Sin cambio de comportamiento observable, por alcance explícito del
ticket.** El helper preserva exactamente la semántica que tenía el código
duplicado: una corrida independiente por organización, cada una bajo su
propio contexto de inquilino, y un `warn`-y-`skip` —no una excepción que
aborte el resto de la corrida— cuando una organización no tiene usuario
SYSTEM. No se tocó la lógica de negocio particular de ningún cron
individual, tal como pedía la sección "fuera de alcance" del ticket.

## Entidades / puertos / adaptadores tocados

- `src/common/scheduling/run-for-each-organization-as-system.ts` (nuevo):
  la función `runForEachOrganizationAsSystem`, con el comentario sobre la
  particularidad de `AsyncLocalStorage` centralizado en un único lugar.
- `AppointmentConfirmationCron` (`src/appointments/appointment-confirmation.cron.ts`),
  `AppointmentAutoCancellationCron`
  (`src/appointments/appointment-auto-cancellation.cron.ts`),
  `AppointmentReminderCron` (`src/appointments/appointment-reminder.cron.ts`),
  `AppointmentAutoCompletionCron`
  (`src/appointments/appointment-auto-completion.cron.ts`) y
  `WaitlistOfferTimeoutCron` (`src/waitlist/waitlist-offer-timeout.cron.ts`):
  el método `run()` de cada uno pasó de un bucle manual sobre
  `prisma.organization.findMany` más `tenantContext.run` a una única llamada
  al helper; los métodos privados de cada uno perdieron la consulta y
  comprobación del usuario SYSTEM, ahora resuelta por el helper. Sin
  cambios de esquema ni de migración.

## Tests y qué validan

Los seis archivos de pruebas unitarias existentes de estos trabajos
programados —los cinco migrados más el del trabajo de expiración de
códigos de acceso, que no se tocó pero comparte el mismo tipo de simulacros
de `PrismaService`/`TenantContextService`— siguieron pasando sin
modificación alguna: cada uno simula `PrismaService` con
`organization.findMany`/`user.findFirst` y `TenantContextService.run` de la
misma forma que antes, y el helper, al recibir esas mismas dependencias
como parámetros en lugar de resolverlas por inyección, es indistinguible
para esos simulacros del código que reemplazó. Esto confirma, a nivel de
prueba, la ausencia de cambio de comportamiento observable que pedía el
criterio de aceptación del ticket, incluyendo el caso "una organización sin
usuario SYSTEM se omite sin lanzar una excepción" y el caso "cada
organización corre bajo su propio contexto de inquilino", ambos ya
cubiertos por los specs preexistentes de cada cron.

Suite completa sin cambios de resultado: 39 suites unitarias / 434 pruebas
en verde, 37 suites e2e / 439 pruebas en verde (`--runInBand`), verificación
de lint y de tipos (`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-109-shared-cron-organization-system-helper` (creada desde
  `origin/main` fresco, tras el merge de TASK-108). Pusheada a `origin`; PR
  abierto, no fusionado aún.
- Ticket: TASK-109 ("[MEJORA] Extraer a un helper compartido el boilerplate
  'por organización + usuario SYSTEM' duplicado en 5 crons"), mejora sobre
  TASK-43 (P4.2, [[FASE-4_PROMPT-2]]), TASK-44 (P4.3,
  [[FASE-4_PROMPT-3]]), TASK-45 (P4.4, [[FASE-4_PROMPT-4]]) y TASK-82
  (P4.5, [[FASE-4_PROMPT-5]]) — los cuatro crons listados como dependencia
  explícita del ticket— y también sobre `AppointmentAutoCompletionCron`
  (TASK-89, [[FASE-3_PROMPT-10]]), que comparte el mismo patrón aunque no
  figuraba entre las dependencias formales del ticket. Misma convención de
  bitácora dedicada
  para tareas de alcance transversal dentro de la fase que TASK-91
  ([[FASE-4_PROMPT-7]]), TASK-90 ([[FASE-4_PROMPT-8]]), TASK-97
  ([[FASE-4_PROMPT-9]]) y TASK-99 ([[FASE-4_PROMPT-10]]).
