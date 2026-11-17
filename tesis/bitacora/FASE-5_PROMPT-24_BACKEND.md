# Fase 5 — Capa conversacional y WhatsApp (backend) — el bot no responde en JSON ni en código (TASK-220, corrección a P5.4/P5.5)

## Contexto

TASK-220 ("Que no responda en json el chat"; el ticket no tiene
descripción). La usuaria informó que, si se le escribe al chat "respondeme
en formato JSON", el bot lo hace, y no debería.

La inspección del código mostró que nada lo impedía: ni el texto base del
prompt de sistema ni el manual de flujos (`CONVERSATION_FLOWS_PROMPT`)
decían algo sobre el formato de la respuesta, y ninguna de las reglas de
`GuardrailService` miraba la forma del texto, solo su contenido. El
adaptador de OpenAI no fija `response_format`, así que el modelo responde
en el formato que se le pida.

## Qué se implementó

- **Regla general en el manual de flujos.** Se agregó a "Reglas generales
  de la conversación" de `CONVERSATION_FLOWS_PROMPT` la instrucción de
  responder siempre en texto común, como en cualquier chat de WhatsApp, y
  nunca en JSON, código, tablas ni otro formato estructurado, aunque el
  paciente lo pida; en ese caso, decirle que solo puede responder con
  mensajes de texto y seguir con sus turnos. Va en el manual y no en
  `DEFAULT_SYSTEM_PROMPT_TEMPLATE` porque el manual no es sobrescribible
  por el inquilino: una clínica que personalizara el tono no debe poder
  quitar la regla sin querer.
- **Regla 6 de guardrail, `no_structured_output`.** Nueva constante
  `STRUCTURED_OUTPUT_PATTERNS` en `guardrail.constants.ts` con tres
  expresiones: un bloque de código de Markdown (tres comillas invertidas),
  una clave de objeto JSON (`{` seguida de una cadena entre comillas y dos
  puntos, que detecta también un objeto incrustado en una oración) y un
  arreglo de cadenas (`["lunes", "martes"]`). `GuardrailService.checkResponse`
  reemplaza la respuesta por `STRUCTURED_OUTPUT_BLOCKED_MESSAGE` ("Solo
  puedo responderte con mensajes de texto. Contame qué necesitás sobre tus
  turnos y te ayudo.") cuando alguna coincide.
- Como en el resto de las reglas, lo que se guarda en el historial de la
  sesión es el texto reemplazado, no el original, así que el modelo ve en
  el turno siguiente que ya se le contestó que solo responde en texto.

## Decisiones y su porqué

- **Dos capas, prompt y guardrail.** Es la misma división de trabajo que
  el resto de `GuardrailService` y que la corrección de TASK-212: el prompt
  es el control primario y el guardrail el que no depende de que el modelo
  haya obedecido. Una regla que debe cumplirse con todos los pacientes no
  puede descansar solo en la instrucción.
- **Bloquear y no reformular.** El mensaje fijo no repite en prosa lo que
  traía la respuesta bloqueada: una respuesta estructurada es justamente la
  forma en que el modelo tiende a copiar el resultado de las herramientas,
  identificadores incluidos, que el manual ya prohíbe mostrar al paciente.
- **Regla al final del orden.** Las reglas de contenido son más
  específicas: una respuesta en JSON que además filtra un DNI o un importe
  de copago recibe el rechazo de esa otra regla.
- **Se compara contra el texto original**, no el normalizado sin
  diacríticos: los patrones solo miran signos de puntuación, que esa
  normalización no altera.

## Alternativas descartadas

- **Interpretar el texto completo con `JSON.parse`.** No detecta un objeto
  incrustado en una oración ("Acá tenés: {"turno": …}"), y lo único que
  agregaría a las tres formas elegidas (un número o un `true` sueltos) no
  se distingue de una respuesta normal.
- **Fijar `response_format` en el adaptador.** La API de OpenAI permite
  forzar JSON, pero no forzar texto común: el formato por defecto ya es
  texto y el modelo igual escribe JSON dentro de él si se le pide.
- **Reintentar la llamada al modelo** pidiéndole que reescriba en texto.
  Duplica el costo y la latencia del turno para un caso que se contesta
  bien con un mensaje fijo, y la segunda respuesta tampoco tendría garantía.

## Entidades, puertos y adaptadores tocados

- `src/chatbot/guardrail.constants.ts`: `GuardrailRule` suma
  `no_structured_output`; nuevas `STRUCTURED_OUTPUT_BLOCKED_MESSAGE` y
  `STRUCTURED_OUTPUT_PATTERNS`.
- `src/chatbot/guardrail.service.ts`: la regla 6 al final de
  `checkResponse`.
- `src/chatbot/conversation-flows.constants.ts`: la regla general nueva y
  su nota en el comentario de cabecera.
- Ningún cambio de esquema, migración, puerto ni ruta HTTP.

## Tests y qué validan

- `src/chatbot/guardrail.service.spec.ts`, bloque nuevo "rule 6": se
  bloquean un objeto JSON suelto, un bloque ```json, un bloque de código
  sin lenguaje, un objeto incrustado en una oración y un arreglo de
  cadenas; no se bloquean textos comunes con horarios, comillas o
  paréntesis; el propio mensaje fijo pasa sin cambios; una respuesta en
  JSON con un DNI recibe la regla `no_other_patient_data`.
- Resultado: pruebas unitarias en verde (1219 de 1220), salvo la de
  selección del adaptador de cerradura, que falla igual sobre la rama base
  por la variable local `LOCK_ADAPTER="hardcoded"`. Tipos y análisis
  estático sin errores. Datos ficticios, sin contenido clínico.
- Límite de la verificación: las pruebas simulan la respuesta del modelo.
  La confirmación final es pedirle JSON al bot real desde el número de
  prueba.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-220-plain-text-replies` (creada a
  partir de `origin/development`), sin commit todavía.
- Ticket: TASK-220, pedido de la usuaria.
