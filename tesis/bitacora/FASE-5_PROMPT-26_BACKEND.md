# Fase 5 — Capa conversacional y WhatsApp (backend) — migración del adaptador de IA de GPT-4o mini a GPT-6 Luna (TASK-210, corrección a P5.1/TASK-46)

## Contexto

OpenAI anunció el 14 de octubre de 2026 la deprecación de `gpt-4o-mini`, el
modelo sobre el que se construyó `OpenAiAdapter` en P5.1 (TASK-46, entrada
`FASE-5_PROMPT-1`), con retiro de la API el 14 de enero de 2027, y nombró a
`gpt-6-luna` como su reemplazo recomendado. A partir de esa fecha el
adaptador dejaría de recibir respuestas y el chatbot completo quedaría fuera
de servicio, de modo que la migración no era optativa.

Esta entrada registra el cambio como un hecho nuevo y fechado. La entrada
`FASE-5_PROMPT-1` no se modificó: describe la decisión que efectivamente se
tomó en TASK-46, con GPT-4o mini, y reescribirla borraría el registro de una
decisión real. En el Capítulo 4 (4.6) sólo se actualizó el texto que afirma
el estado vigente del sistema.

## Qué se implementó

El trabajo se hizo en tres cambios sucesivos sobre la rama
`chore/openai-gpt-6-luna-migration` del repositorio `back`, aprobados uno
por uno.

- **Modelo y parámetros (commit `16e8329`, 15/10/2026).** El valor por
  defecto de `OPENAI_MODEL` pasó a `gpt-6-luna`, en el código
  (`openai.constants.ts`), en `.env.example`, `.env.test-deploy` y en
  `docker-compose.prod.yml`. Cada llamada envía además
  `reasoning_effort: 'none'`, expuesto como la variable opcional
  `OPENAI_REASONING_EFFORT`, con una advertencia en el arranque si se
  configura otro valor, y `store: false`.
- **Robustez del turno (commit `422539f`, 15/10/2026).** Una respuesta
  cortada por el tope de tokens (`finish_reason: 'length'`) ya no se envía al
  paciente: si trae solicitudes de herramienta se rechaza en el módulo de
  mapeo, y si es texto se lanza `TruncatedResponseError`. Se incorporó el
  campo `refusal` de OpenAI como un motivo de corte propio del puerto. La
  política de reintentos pasó a reintentar también las fallas de conexión y
  los vencimientos de tiempo, y a dejar de reintentar el error 429 de cuota
  agotada (`insufficient_quota`). El historial que se reenvía en cada turno
  quedó acotado por `CHATBOT_MAX_HISTORY_MESSAGES` (30 mensajes por
  defecto), recortando siempre en el comienzo de un turno completo. Las
  fallas se separaron en dos mensajes al paciente: uno para pedirle que
  reformule (respuesta rechazada o vacía) y otro de falla técnica, que lo
  remite al teléfono de la clínica (`CLINIC_CONTACT_PHONE`) y se registra en
  nivel de error con el código `AI_TURN_FAILED`.
- **Registros sin teléfonos (commit `ed75415`, 16/10/2026).** Todo número de
  celular que llegaba a una línea de registro se enmascara a sus últimos
  cuatro dígitos (`src/common/logging/mask-phone.ts`), también dentro de los
  mensajes y trazas de errores propios, de OpenAI y de Meta. El operador
  correlaciona ahora por el identificador del mensaje de WhatsApp (`wamid`).
  En el mismo cambio se bajó el tiempo máximo por llamada a OpenAI
  (`OPENAI_TIMEOUT_MS`) de 30 a 10 segundos.

## Decisiones y su porqué

- **`reasoning_effort: 'none'` es obligatorio, no una preferencia.** La
  documentación del modelo establece que, sobre la API de Chat Completions,
  `gpt-6-luna` admite *function calling* sólo con ese parámetro en `none`, y
  su valor por defecto es `medium`. Cambiar únicamente el nombre del modelo
  habría dejado al bot sin poder invocar ninguna herramienta mientras seguía
  respondiendo con texto verosímil: una falla silenciosa del chatbot entero.
  El valor es, además, adecuado al caso: un turno de agenda por WhatsApp es
  sensible a la latencia, y el razonamiento que la conversación necesita ya
  está en el manual de flujos y en las propias herramientas.
- **Mantener Chat Completions.** Obtener un nivel de razonamiento mayor
  exigiría migrar a la API de Responses y reescribir `openai.mappers.ts`. Se
  decidió no hacerlo, y evaluar Responses con esfuerzo `low` sólo si la
  calidad de selección de herramientas resultara inferior a la de GPT-4o
  mini. El puerto `AIPort` no cambió de forma, igual que en el cambio de
  proveedor de TASK-46.
- **El modelo de reemplazo se elige con el mismo criterio que el original.**
  El documento de evaluación de tecnologías de IA del Objetivo Específico 2
  priorizó el equilibrio costo–calidad entre los tiers económicos y la
  madurez del *function calling*. `gpt-6-luna` es el modelo más económico y
  rápido de su familia y conserva la misma API de herramientas, de modo que
  el fundamento de esa evaluación sigue aplicando.
- **`store: false`.** Chat Completions conserva por defecto las respuestas en
  las cuentas nuevas, y los mensajes contienen lo que escribe un paciente de
  salud mental. La regla del proyecto que excluye datos clínicos de los
  registros se extiende así al lado del proveedor.
- **Nunca enviar una respuesta cortada.** Una solicitud de reserva con
  argumentos truncados parece completa en los campos que llegaron, y media
  oración sobre un turno puede leerse como una instrucción completa.
- **Dos mensajes de falla.** Sólo la respuesta rechazada o vacía es algo que
  el paciente puede resolver reformulando; ante una cuota agotada o un
  vencimiento, invitarlo a reintentar lo hace chocar con la misma falla.
- **Enmascarar también el texto que el sistema no redacta.** El
  identificador de sesión es `organizationId:celular` por construcción, así
  que el número llegaba a los registros dentro de mensajes de error sin que
  ningún código lo formateara.
- **Bajar el tiempo máximo a 10 segundos.** Con los vencimientos ahora
  reintentados, tres intentos de 30 segundos sumaban cerca de 91 segundos de
  espera, tiempo suficiente para que Meta reentregue el webhook antes de que
  el primer turno se dé por fallido; con 10 segundos el peor caso ronda los
  31.

## Alternativas descartadas

- **Cambiar sólo el nombre del modelo.** Por lo indicado arriba: desactivaba
  todas las herramientas sin ningún error visible.
- **Migrar a la API de Responses.** Daría acceso a un razonamiento mayor
  combinado con herramientas, a costa de reescribir el módulo de mapeo; queda
  como opción a evaluar, no como necesidad.
- **Reescribir `FASE-5_PROMPT-1`.** Se descartó por el criterio de no
  alterar el registro histórico de la bitácora.

## Entidades, puertos y adaptadores tocados

- Adaptador: `openai.adapter.ts`, `openai.constants.ts` y
  `openai.mappers.ts`.
- Puerto: `ai.port.ts` (motivo de corte `refusal`; sin cambios en la forma de
  los mensajes).
- Chatbot: `orquestador.service.ts`, `conversation-session.store.ts`,
  `chatbot.constants.ts` y `chatbot.errors.ts`.
- WhatsApp: `whatsapp-webhook.controller.ts`,
  `whatsapp-webhook.constants.ts` y `whatsapp-cloud.adapter.ts`.
- Nuevo: `src/common/logging/mask-phone.ts`.
- Configuración: `.env.example`, `.env.test-deploy` y
  `docker-compose.prod.yml` (`OPENAI_MODEL`, `OPENAI_REASONING_EFFORT`,
  `OPENAI_TIMEOUT_MS`, `CHATBOT_MAX_HISTORY_MESSAGES`,
  `CLINIC_CONTACT_PHONE`).
- Ningún cambio de esquema, migración ni ruta HTTP.

## Tests y qué validan

- `openai.adapter.spec.ts`: el modelo por defecto es `gpt-6-luna` cuando
  `OPENAI_MODEL` no está definida, cada pedido lleva
  `reasoning_effort: 'none'` y `store: false`, el arranque advierte si el
  esfuerzo configurado desactivaría las herramientas, se reintenta un
  vencimiento de tiempo y no se reintenta el error de cuota agotada.
- `openai.mappers.spec.ts`: una respuesta cortada con solicitudes de
  herramienta se rechaza, y el campo `refusal` se traduce al motivo propio.
- `conversation-session.store.spec.ts`: el tope de historial corta sólo en el
  comienzo de un turno y nunca deja un resultado de herramienta sin su
  solicitud.
- `orquestador.service.spec.ts`: una respuesta de texto cortada no se envía,
  y ante un rechazo o un turno vacío se le pide al paciente que reformule.
- `whatsapp-webhook.constants.spec.ts`: el mensaje de falla técnica remite al
  teléfono de la clínica si está configurado, nunca invita a reintentar y
  tiene su propio código de registro.
- `mask-phone.spec.ts` y `whatsapp-webhook.controller.spec.ts`: el
  enmascarado no altera `wamid`, UUID ni fechas, y ningún camino de falla del
  webhook deja un número completo en los registros.
- Resultado: no se registró en esta entrada la cantidad de pruebas en verde
  de cada cambio (resultado pendiente de medición). La comparación de
  calidad de selección de herramientas entre GPT-6 Luna y GPT-4o mini con el
  modelo real tampoco se midió.

## Hallazgos derivados

Pendiente fuera de esta tarea: auditar los resultados de las trece
herramientas del chatbot antes de usarlo con pacientes reales, para que a
OpenAI llegue sólo lo estrictamente necesario (ningún diagnóstico, código
CIE-10, dato de historia clínica, identificador interno ni teléfono).

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `chore/openai-gpt-6-luna-migration` (creada a partir
  de `development`), commits `16e8329`, `422539f` y `ed75415`. Las ramas de
  corrección posteriores de la Fase 5 se crearon sobre ella; `development`
  todavía no la incorpora.
- Ticket: TASK-210 (documentación). La migración de código no tiene ticket
  propio.
