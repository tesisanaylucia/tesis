# Fase 5 — Capa conversacional y WhatsApp (backend) — respuesta a audios, imágenes, stickers y otros mensajes no textuales (TASK-218, corrección a P5.8)

## Contexto

TASK-218 ("Ver cómo se van a procesar stickers, audios, imágenes, etc.";
el ticket no tiene descripción). La usuaria pidió que, cuando al chatbot de
WhatsApp le llegue un audio, una imagen, un sticker u otro formato que no
sea texto, responda que no puede procesar ese formato.

La inspección del código mostró que el webhook (P5.8, TASK-53) ya distinguía
estos mensajes, pero solo para descartarlos: `extractIncomingTextMessage`
devolvía `null` para todo mensaje cuyo `type` no fuera `text`, y el
controlador respondía 200 a Meta sin enviarle nada al paciente. Quien
mandaba una nota de voz o una foto se quedaba esperando una respuesta que
no iba a llegar. Que el chatbot atienda solo texto estaba decidido desde el
alcance del SRS, y el puerto de IA (`AIPort`) transporta únicamente texto.
Esta corrección no cambia eso: hace que el rechazo quede a la vista del
paciente.

## Qué se implementó

- **Clasificación del mensaje entrante.** `extractIncomingTextMessage` pasa
  a ser `extractIncomingMessage` y devuelve una unión discriminada
  (`IncomingMessage`): `{ kind: 'text', … }` para un mensaje de texto, o
  `{ kind: 'unsupported', id, phoneNumberId, from, type }` para cualquier
  otro tipo enviado por el paciente (audio, imagen, sticker, video,
  documento, ubicación, contacto y el tipo `unsupported` que Meta usa para
  lo que su API no puede entregar). Un mensaje no textual sin `id` (wamid),
  sin remitente o sin `phone_number_id` sigue devolviendo `null`, igual que
  uno de texto mal formado, porque sin esos datos no se puede deduplicar ni
  responder.
- **Tipos ignorados sin respuesta.** `SILENTLY_IGNORED_MESSAGE_TYPES`
  (`reaction`, `system`) siguen devolviendo `null`. Una reacción con emoji a
  un mensaje anterior, por ejemplo un 👍 sobre la confirmación del bot, no
  es algo que el paciente haya "dicho", y contestarle que no se puede
  procesar ese formato sería ruido. Un mensaje `system` es un aviso de Meta
  (por ejemplo, que el cliente cambió de número) y no lo envió el paciente.
- **Respuesta fija.** `UNSUPPORTED_MESSAGE_TYPE_REPLY`
  (`whatsapp-webhook.constants.ts`): "Por ahora solo puedo leer mensajes de
  texto, así que no pude procesar lo que enviaste (audio, imagen, sticker,
  archivo, etc.). ¿Me lo podés escribir?". El texto nombra los formatos que
  el paciente probablemente envió y le dice qué sí funciona, no solo qué
  no. Tiene la misma forma de mensaje predefinido que
  `webhookTechnicalFailureMessage` (TASK-183).
- **Flujo en el controlador.** `WhatsappWebhookController.receive` trata al
  mensaje no textual con el mismo reclamo de idempotencia
  (`ProcessedWhatsappMessageStore`) y la misma resolución de inquilino que
  al de texto, y recién después se bifurca: si `kind === 'unsupported'`,
  `replyToUnsupportedMessage` registra en el log el tipo recibido (con el
  número enmascarado) y envía la respuesta fija por `MessagingPort`. Nunca
  llama a `OrquestadorService`.

## Decisiones y su porqué

- **No pasar por el orquestador ni por OpenAI.** No hay texto que darle al
  modelo, así que la respuesta no consume tokens y no puede fallar por
  motivos de la IA. Además el historial de la conversación no cambia, así
  que el siguiente mensaje escrito del paciente sigue la conversación donde
  estaba.
- **Deduplicar y resolver el inquilino antes de responder.** Una reentrega
  del mismo sticker no debe contestarse dos veces (es el mismo motivo de
  TASK-183). Un número sin mapeo a una organización no recibe respuesta en
  ningún caso, sea texto o no, para que el comportamiento sea el mismo
  frente a una brecha de configuración del operador.
- **Un fallo al enviar no se convierte en 500.** Con la misma regla que el
  `try/catch` del turno (TASK-183): el mensaje ya quedó reclamado, así que
  un reintento de Meta se descartaría igual. Solo queda registrar el error
  (con el número enmascarado y el wamid para correlacionar) y responder 200.
- **Constante y no clave de configuración por inquilino.** El texto sigue
  el precedente de los demás mensajes predefinidos del webhook
  (`webhookTechnicalFailureMessage`), que también son constantes. Si una
  clínica necesitara redactarlo distinto, pasarlo a `OrganizationConfig`
  quedaría para un ticket posterior.

## Alternativas consideradas y descartadas

- **Responder también a las reacciones.** Se descartó por lo explicado
  arriba: es el caso más frecuente de mensaje no textual que no pide
  respuesta.
- **Mandar el mensaje no textual al modelo con un marcador, por ejemplo
  "[el paciente envió un audio]", y dejar que él redacte la respuesta.**
  Se descartó porque gasta una llamada a la IA en algo cuya respuesta es
  siempre la misma, y suma a la conversación un turno que el modelo podría
  malinterpretar.
- **Transcribir audios o interpretar imágenes.** Fuera del alcance del SRS
  (chatbot solo de texto) y del pedido de la usuaria. Además una imagen
  enviada a una clínica de salud mental podría contener información
  clínica, que el chatbot no debe manejar.

## Entidades / puertos / adaptadores tocados

- `src/whatsapp/whatsapp-webhook.types.ts`: `extractIncomingMessage`,
  `IncomingMessage`, `IncomingUnsupportedMessage`,
  `SILENTLY_IGNORED_MESSAGE_TYPES`.
- `src/whatsapp/whatsapp-webhook.constants.ts`:
  `UNSUPPORTED_MESSAGE_TYPE_REPLY`.
- `src/whatsapp/whatsapp-webhook.controller.ts`: bifurcación en `receive`,
  `replyToUnsupportedMessage`.
- Ningún cambio de esquema, migración, puerto ni ruta HTTP. `MessagingPort`
  se usa tal cual.

## Tests y qué validan

- `src/whatsapp/whatsapp-webhook.types.spec.ts`: cada tipo no textual
  (audio, imagen, sticker, video, documento, ubicación, contactos,
  `unsupported`) se clasifica como `unsupported` con id, número receptor,
  remitente y tipo; `reaction` y `system` devuelven `null`; un no textual
  sin wamid devuelve `null`; un texto con cuerpo vacío devuelve `null` (mal
  formado, no "no soportado").
- `src/whatsapp/whatsapp-webhook.controller.spec.ts`: para audio, imagen,
  sticker, video, documento y ubicación se envía la respuesta fija una vez
  y no se invoca al orquestador; un sticker reentregado con el mismo wamid
  se responde una sola vez; un número sin mapeo no recibe nada; una reacción
  no recibe nada; si el envío falla se responde 200 y el log no contiene el
  número en claro (sí el enmascarado y el wamid).
- `test/whatsapp-webhook.e2e-spec.ts`: un `POST /webhook` firmado con un
  mensaje de audio responde 200 y el puerto de mensajería recibe
  exactamente la respuesta fija, dirigida al remitente.
- Resultado: las suites del webhook y de conversación de extremo a extremo
  (22 pruebas) en verde; pruebas unitarias en verde (1199 de 1200), salvo la
  de selección del adaptador de cerradura, que falla igual sobre la rama
  base por la variable local `LOCK_ADAPTER="hardcoded"`. Análisis estático y
  tipos sin errores. Datos ficticios, sin contenido clínico.
- Límite de la verificación: las pruebas simulan el cuerpo que envía Meta.
  La confirmación final es mandarle un audio o un sticker real al número de
  prueba.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-218-unsupported-message-types`
  (creada a partir de `origin/development`), commit `2301dea`.
- Ticket: TASK-218, pedido de la usuaria, corrección al webhook de WhatsApp
  de P5.8 (TASK-53).
