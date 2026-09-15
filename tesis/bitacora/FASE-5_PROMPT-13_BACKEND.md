# Fase 5 — Capa conversacional y WhatsApp (backend) — idempotencia y manejo de errores del webhook (TASK-183, corrección/extensión a P5.8)

## Contexto

Una auditoría multi-agente contra las fuentes de verdad (anteproyecto de
tesis y SRS), corrida el 28 de agosto de 2026, generó el hallazgo crítico
TASK-183 sobre `WhatsappWebhookController.receive` (P5.8, TASK-53): la
extracción del mensaje entrante nunca capturaba el `id` que Meta asigna a
cada mensaje (el "wamid", `messages[].id`), así que no existía ningún campo
sobre el cual deduplicar; y el cuerpo del método no envolvía en `try/catch`
ni la llamada a `OrquestadorService.procesar` ni a
`MessagingPort.sendMessage`, de modo que cualquier falla —OpenAI agotando
sus propios reintentos, el tope de iteraciones de herramientas
(`ToolCallLimitExceededError`), un fallo de entrega— se propagaba sin
capturar, Nest respondía 500 y el paciente nunca recibía contestación
alguna. El propio comentario dejado en `chatbot.errors.ts` por TASK-48/49
anticipaba explícitamente un "futuro llamador" que atraparía esos errores
una vez existiera un controlador HTTP real; ese llamador nunca se
construyó hasta esta tarea. El hallazgo señalaba además una consecuencia
concreta de la falta de idempotencia: WhatsApp reintenta un webhook que no
respondió 200, así que el reintento re-ejecutaba el turno completo — un
`request_prescription` reintentado creaba una segunda `PrescriptionRequest`
en estado PENDING, porque nada a nivel de base de datos lo impide.

## Qué se implementó

Se agregó `id` (el wamid) a `IncomingTextMessage`, extraído por
`extractIncomingTextMessage` con la misma disciplina que el resto de los
campos requeridos (`from`, `text.body`): su ausencia hace que el mensaje se
trate como no procesable, igual que un mensaje sin remitente. Se agregó
`ProcessedWhatsappMessageStore`, un almacén en memoria, del mismo proceso,
que `receive()` consulta con `claim(messageId, ttlMinutes)` antes de
resolver la organización o invocar al orquestador: la primera vez que un
wamid se reclama devuelve verdadero y lo recuerda; una repetición dentro de
la ventana de TTL devuelve falso, y el controlador responde `200` sin volver
a ejecutar el turno. Se envolvió el tramo `procesar`/`sendMessage` en
`try/catch`: cualquier error se registra por log y el método responde `200`
igual —un `500` sólo garantiza un reintento duplicado de Meta, no arregla
nada— además de intentar, en un segundo `try/catch` propio que nunca
propaga, enviar al paciente un mensaje de respaldo genérico en lugar de
dejarlo sin ninguna respuesta.

## Decisiones y por qué

**El almacén de deduplicación no reutiliza `ConversationSessionStore` tal
cual, aunque comparte su forma.** Ambos son un `Map` en memoria, de proceso
único, con expiración perezosa —suficiente para el despliegue de una sola
instancia del piloto, ya que `CLAUDE.md` no nombra infraestructura de
caché compartida—, pero difieren en cómo se limpian: un `sessionId` se
vuelve a leer cada vez que la misma conversación continúa, así que la
expiración perezosa por clave (sólo se verifica cuando esa clave se vuelve a
leer) mantiene acotado el mapa por sí sola. Un wamid, en cambio, se reclama
como mucho una vez en el caso normal y nunca se vuelve a leer, así que nada
dispararía jamás su propia expiración; `claim()` en cambio barre en cada
llamada las entradas más viejas que el TTL, barrido que se mantiene barato
precisamente porque el TTL es corto —el mapa nunca contiene más que los
mensajes recibidos en esa ventana reciente—.

**El TTL de deduplicación se fijó corto (15 minutos) a propósito.** Antes de
esta tarea, todo el mecanismo de reintento de WhatsApp existía porque el
webhook podía responder algo distinto de 200; con el `try/catch` agregado,
`receive()` responde 200 en prácticamente cualquier caso, así que el
reintento por fallo de WhatsApp debería dejar de dispararse casi por
completo. La ventana sólo tiene que cubrir la garantía de entrega "al menos
una vez" que WhatsApp mantiene de forma independiente al reintento por
error, no una ventana larga pensada para una caída prolongada del sistema.

**Se agregó una respuesta de respaldo al paciente, más allá de lo que el
hallazgo pedía textualmente.** La lista de correcciones sugeridas por el
hallazgo (capturar el `id`, mantener un set con TTL, envolver en
`try/catch` y responder 200 igual, agregar el e2e faltante) no menciona
explícitamente enviar un mensaje de error al paciente — pero el propio
hallazgo nombra "el paciente nunca recibe respuesta" como parte del
problema, y un `try/catch` que sólo registra por log y responde 200
evita el reintento duplicado sin resolver esa parte: el paciente sigue sin
recibir nada. `sendFallbackErrorReply` cierra esa brecha con el mismo
patrón de "mensaje enlatado" que ya usa `guardrail.constants.ts` para sus
propias respuestas fijas, envuelto en su propio `try/catch` que nunca
propaga, para que una segunda falla (el propio `MessagingPort` caído) no
convierta un error ya manejado en uno sin manejar.

**No se agregó una restricción de unicidad a `PrescriptionRequest`.** El
hallazgo menciona la ausencia de esa restricción como una consecuencia
observada de la falta de idempotencia, no como parte de la lista de
correcciones sugeridas, y agregarla habría bloqueado el caso legítimo de un
paciente que solicita una segunda receta en un turno distinto. La
corrección real está en el nivel del webhook, que es donde efectivamente se
originaba la repetición.

## Alternativas descartadas

- **Reutilizar `ConversationSessionStore` directamente para el
  seguimiento de wamids**, en lugar de una clase nueva: descartada porque
  su estrategia de expiración perezosa por clave no limpia un identificador
  que nunca se vuelve a leer (ver la decisión de arriba).
- **Agregar una restricción de unicidad a `PrescriptionRequest`** como
  segunda capa de defensa: descartada por bloquear el caso legítimo de
  múltiples solicitudes de receta a lo largo del tiempo; ver la decisión
  correspondiente arriba.
- **No enviar ningún mensaje de respaldo al paciente**, limitándose
  estrictamente a la lista de correcciones sugeridas por el hallazgo:
  descartada porque el propio hallazgo describe la falta de respuesta al
  paciente como parte del problema, no sólo el reintento duplicado.

## Entidades / puertos / adaptadores tocados

- `src/whatsapp/whatsapp-webhook.types.ts`: `IncomingTextMessage` gana el
  campo `id`; `extractIncomingTextMessage` lo exige como cualquier otro
  campo requerido.
- `src/whatsapp/processed-whatsapp-message.store.ts` (nuevo):
  `ProcessedWhatsappMessageStore`, el almacén de deduplicación en memoria.
- `src/whatsapp/whatsapp-webhook.constants.ts` (nuevo):
  `PROCESSED_MESSAGE_ID_TTL_MINUTES` y el texto de respaldo
  `WEBHOOK_PROCESSING_ERROR_MESSAGE`.
- `src/whatsapp/whatsapp-webhook.controller.ts`: `receive()` reclama el
  wamid antes de resolver la organización, y envuelve el tramo
  `procesar`/`sendMessage` en `try/catch` con respuesta de respaldo.
- `src/whatsapp/whatsapp.module.ts`: registra
  `ProcessedWhatsappMessageStore` como proveedor del módulo.
- `src/chatbot/chatbot.errors.ts`: comentario actualizado — el "futuro
  llamador" que anticipaba ya existe y es este controlador.
- Ningún cambio de esquema.

## Tests y qué validan

- `src/whatsapp/whatsapp-webhook.types.spec.ts`: caso nuevo que prueba que
  un mensaje sin `id` se trata como no procesable.
- `src/whatsapp/processed-whatsapp-message.store.spec.ts` (nuevo): reclamo
  simple, rechazo de un `id` repetido dentro del TTL, reclamos
  independientes para `id`s distintos, y el propio límite del TTL (se
  reclama de nuevo justo después de vencer, se sigue rechazando justo
  antes).
- `test/whatsapp-webhook.e2e-spec.ts`: dos casos nuevos de idempotencia
  —el mismo wamid reenviado no vuelve a invocar al orquestador ni a
  reenviar la respuesta; un wamid distinto del mismo remitente sí se
  procesa— y un caso de manejo de errores que fuerza a `AIPort` a
  rechazar la promesa y verifica que el webhook igual responde `200` y que
  el paciente recibe el mensaje de respaldo.
- Ejecución: suite unitaria completa en verde (91 conjuntos, 984 pruebas) y
  suite de extremo a extremo completa en verde (53 conjuntos, 644 pruebas,
  `--runInBand`); análisis estático (ESLint) y verificación de tipos
  (`tsc --noEmit`) sin advertencias. Datos ficticios, sin contenido clínico
  ni datos personales reales.

## Figuras pendientes

Ninguna figura nueva; la corrección no introduce un flujo o diagrama
distinto del ya registrado para el webhook de WhatsApp (P5.8).

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-183-whatsapp-webhook-idempotency-errors` (creada a partir
  de `origin/main`).
- Ticket: TASK-183 ("[CRÍTICO] Webhooks de WhatsApp sin idempotencia ni
  manejo de errores"), hallazgo de una auditoría multi-agente del
  28 de agosto de 2026 sobre `src/whatsapp/whatsapp-webhook.controller.ts`
  y `whatsapp-webhook.types.ts`, prioridad máxima.
