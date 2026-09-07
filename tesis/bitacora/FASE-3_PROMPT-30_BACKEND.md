# Fase 3 — Motor de Turnos (backend) — el adaptador provisorio de confirmación de reprogramación bloqueaba siempre, y no sólo transitoriamente (TASK-115, corrección a TASK-39)

## Contexto

La reprogramación individual de un turno por administración o por el propio
profesional, y la reorganización manual de la agenda, quedaron implementadas
en P3.6 (TASK-39, [[FASE-3_PROMPT-6]]) exigiendo la confirmación previa del
paciente antes de aplicar el cambio, según pide el documento de requisitos.
Como en ese momento no existía ningún canal real por el que el paciente
pudiera responder (esa respuesta llegaría por WhatsApp, un componente de una
fase posterior), la confirmación quedó detrás de un puerto de dominio propio
resuelto por un adaptador provisorio que respondía que no de forma
incondicional — la misma solución, y la misma limitación, que en ese momento
tenía también la reasignación automática de un turno cancelado hacia la
lista de espera (P3.7, TASK-40). Una auditoría de código sobre
`psique-back/main` (2026-08-14) encontró que esa limitación nunca se
resolvió con la misma oportunidad para los dos puertos: P4.5 (TASK-82,
[[FASE-4_PROMPT-5]]) rediseñó el de la lista de espera para dejar de
bloquear, pero el de reprogramación siguió sin tocarse, de modo que, a la
fecha de la auditoría, tanto `PATCH /turnos/:id/reprogramar` (cuando quien
llama no es el proceso automatizado) como `POST
/profesionales/:id/reorganizar-agenda` fallaban siempre, sin ninguna
excepción, sobre dos puntos de acceso que el sistema exponía como
disponibles y documentados.

## Qué se implementó

- El puerto de dominio `RescheduleResponsePort` se rediseñó siguiendo
  exactamente el mismo patrón asíncrono que TASK-82 ya había aplicado al
  puerto de la lista de espera: en lugar de un único método que pregunta y
  espera la respuesta (`awaitConfirmation`), expone `registerOffer` (persiste
  la oferta y devuelve su identificador de inmediato) y `getStatus` (consulta
  el estado sin bloquear).
- Se agregó el modelo `RescheduleOffer` (Prisma), con el mismo cuatro estados
  que ya tiene el modelo equivalente de la lista de espera
  (`PENDING`/`ACCEPTED`/`REJECTED`/`EXPIRED`), aunque hoy sólo `PENDING` es
  alcanzable en producción — ver más abajo. El adaptador que resuelve el
  puerto en producción pasó de ser un valor fijo (siempre "no") a persistir y
  leer esa tabla.
- `AppointmentsService.rescheduleCore` deja de bloquear el pedido HTTP en
  espera de una respuesta: cuando la reprogramación requiere confirmación,
  registra la oferta, envía el mensaje al paciente y devuelve de inmediato un
  resultado que declara el turno como pendiente de confirmación, sin escribir
  su nueva fecha. Dos métodos nuevos, `acceptRescheduleOffer` y
  `rejectRescheduleOffer`, son el punto de entrada que resuelve una oferta
  cuando la confirmación finalmente llega — hoy sólo los llaman las pruebas,
  ya que el canal real de WhatsApp (M5) todavía no existe, la misma situación
  en la que se encontraban los métodos equivalentes de la lista de espera
  antes de que se les conectara un webhook real.
- El punto de acceso individual (`PATCH /turnos/:id/reprogramar`) y el de
  reorganización en lote (`POST /profesionales/:id/reorganizar-agenda`)
  devuelven ahora, para cada turno que necesitó confirmación, un estado
  explícito de "pendiente de confirmación" junto con el identificador de la
  oferta registrada, en lugar de un error o de una reprogramación aplicada
  sin que nadie la haya aceptado.

## Decisiones y por qué

**No se agregó ningún trabajo programado que expire una oferta de
reprogramación sin responder.** La lista de espera sí tiene ese trabajo,
porque el documento de requisitos fija en cuatro horas el tiempo que un
candidato tiene para responder antes de que la oferta pase al siguiente
candidato — ese plazo es una regla de negocio explícita, no una decisión de
esta corrección. La reprogramación no tiene, en el documento de requisitos,
ningún plazo equivalente; inventar uno habría introducido una regla de
negocio que ninguna fuente respalda, exactamente lo que este trabajo evita
en general (las reglas de negocio se tratan como datos configurables que la
clínica decide, no como valores que el código elige por su cuenta). Se
adoptó en cambio la alternativa que el propio hallazgo de la auditoría
planteaba explícitamente: un estado de "confirmación pendiente" indefinido,
visible para quien llama, en lugar de un fallo inmediato o una
reprogramación simulada. Una oferta de reprogramación queda entonces
pendiente hasta que el canal real la resuelva, o hasta que un trabajo futuro
—si el documento de requisitos llega a fijar un plazo— agregue el trabajo
programado correspondiente.

**La escritura real que mueve el turno se extrajo a un método compartido**
(`applyRescheduleWrite`) entre el camino que escribe de inmediato (el
proceso automatizado, que nunca necesita confirmación porque el pedido del
propio paciente ya es su aceptación) y el que la aplica de forma diferida al
aceptar una oferta. Ambos caminos repiten, deliberadamente, la verificación
de que la nueva fecha sigue en el futuro: la del camino inmediato ocurre
segundos después de la primera comprobación, pero la del camino diferido
puede ocurrir horas después de que se registró la oferta, y una oferta
aceptada sobre una fecha que ya pasó no debe aplicarse silenciosamente como
si aún fuera válida.

**El método que resuelve una oferta relee el turno en el momento de
aplicarla, en lugar de reutilizar el que tenía en memoria cuando la oferta
se registró.** El mismo turno pudo cancelarse o reprogramarse por otra vía
mientras la oferta seguía pendiente; releerlo permite que la aceptación se
descarte silenciosamente cuando el turno ya no está en un estado
reprogramable, en lugar de sobrescribir un estado que cambió por un motivo
legítimo. La misma razón llevó a mantener la oferta como aceptada aunque la
escritura posterior falle por cualquier otro motivo (un conflicto de horario
concurrente, por ejemplo): reescribirla como rechazada dejaría en el registro
de auditoría un rechazo que el paciente nunca expresó.

## Alternativas descartadas

- **Fijar un plazo de expiración propio para la oferta de reprogramación**,
  reutilizando por ejemplo el mismo valor de cuatro horas que ya tiene la
  lista de espera: descartada por no tener respaldo en el documento de
  requisitos, que no fija ningún plazo para este caso — ver la decisión
  correspondiente más arriba.
- **Mantener el diseño síncrono y limitarse a corregir el valor fijo del
  adaptador** (por ejemplo, haciendo que responda que sí siempre): descartada
  de plano, porque habría dado por aceptada cualquier reprogramación sin que
  el paciente realmente la haya confirmado, el defecto exactamente opuesto al
  que originó la corrección.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: modelo nuevo `RescheduleOffer` y enum
  `RescheduleOfferStatus`, migración
  `20260907120000_reschedule_offer_async`.
- `src/domain/ports/reschedule-response.port.ts` (rediseñado):
  `registerOffer`/`getStatus` en lugar de `awaitConfirmation`.
- `src/infrastructure/adapters/reschedule-response.adapter.ts` (nuevo,
  reemplaza a `StubRescheduleResponseAdapter`, eliminado).
- `src/infrastructure/integrations.module.ts`: el puerto pasa a resolverse
  con el adaptador real.
- `src/appointments/appointments.service.ts`: `rescheduleCore` reescrito;
  método nuevo `applyRescheduleWrite` (escritura compartida);
  `acceptRescheduleOffer`/`rejectRescheduleOffer` nuevos.
- `src/appointments/appointment.presenter.ts` y los dos controladores
  correspondientes (`appointments.controller.ts`, `agenda.controller.ts`):
  la respuesta HTTP distingue ahora entre una reprogramación aplicada y una
  pendiente de confirmación.
- `src/chatbot/tools/appointment.tools.ts`: sin cambio de comportamiento —
  el proceso automatizado nunca necesita confirmación, así que la
  herramienta del chatbot sigue devolviendo el turno reprogramado sin más.

## Tests y qué validan

- Suite unitaria de reprogramación (`appointments-rescheduling.service.spec.ts`,
  reescrita): la reprogramación por administración o por el propio
  profesional registra la oferta y no escribe nada; el proceso automatizado
  sigue escribiendo de inmediato; los dos métodos nuevos de resolución de
  oferta, incluyendo los casos de oferta inexistente, ya resuelta
  concurrentemente, turno ya no reprogramable, y fecha ofrecida ya vencida
  para cuando se acepta.
- Suite end-to-end (`test/appointments-rescheduling.e2e-spec.ts`, reescrita):
  el puerto ya no se reemplaza por un valor de prueba — se ejercita el
  adaptador real contra la base de datos, y la resolución de una oferta se
  hace llamando directamente al método de servicio correspondiente, la misma
  técnica que ya usaba la prueba end-to-end del vencimiento de ofertas de
  lista de espera. Se agregó también un caso que acepta dos ofertas de un
  mismo lote de reorganización de forma independiente, donde una tiene éxito
  y la otra falla por un conflicto de horario, sin que el resultado de una
  afecte a la otra.
- Suite end-to-end de invalidación de códigos de acceso
  (`test/access-code-invalidation-expiration.e2e-spec.ts`, ajustada): las dos
  pruebas que dependían de una reprogramación aplicada de inmediato ahora
  aceptan la oferta explícitamente antes de comprobar la invalidación y
  reemisión del código.
- Ejecución: suite unitaria completa en verde (78 conjuntos, 778 pruebas) y
  suite end-to-end completa en verde (51 conjuntos, 557 pruebas), ambas
  contra la instancia local de PostgreSQL con `--runInBand`; cobertura
  combinada 97,24 % líneas / 82,67 % ramas, por encima del umbral del 80 %
  que exige la configuración de cobertura del repositorio; `tsc --noEmit` y
  `eslint` sin hallazgos. Los datos usados en las pruebas son ficticios.

## Figuras pendientes

- Figura nueva (número tentativo a asignar): diagrama de estados de
  `RescheduleOffer` (`PENDING` → `ACCEPTED`/`REJECTED`; `EXPIRED` alcanzable
  sólo si un trabajo futuro lo agrega), para 4.4, en el mismo estilo que la
  figura ya pendiente del estado de la oferta de lista de espera.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-115-reschedule-response-port-async`,
  creada desde `main`.
- Ticket: TASK-115 (Jira), "[CORRECCIÓN] TASK-39 – RescheduleResponsePort
  stub hace fallar siempre reorganizar-agenda y reprogramación por
  ADMIN/profesional". Misma convención de bitácora dedicada para tareas
  puntuales dentro de la fase del ticket original que TASK-94/TASK-95/
  TASK-96/TASK-100/TASK-108/TASK-110/TASK-113/TASK-114/TASK-116/TASK-117/
  TASK-123 ([[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]], [[FASE-3_PROMPT-18]],
  [[FASE-3_PROMPT-19]], [[FASE-3_PROMPT-23]], [[FASE-3_PROMPT-24]],
  [[FASE-3_PROMPT-25]], [[FASE-3_PROMPT-26]], [[FASE-3_PROMPT-27]],
  [[FASE-3_PROMPT-28]], [[FASE-3_PROMPT-29]]), y misma referencia directa que
  [[FASE-4_PROMPT-5]] (TASK-82), cuyo patrón de rediseño se reutilizó aquí
  íntegramente.
