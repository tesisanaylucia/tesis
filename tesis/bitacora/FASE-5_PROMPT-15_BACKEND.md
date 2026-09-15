# Fase 5 — Capa conversacional y WhatsApp (backend) — profundidad en la detección de urgencia (TASK-193, corrección a P5.4/P5.7)

## Contexto

La misma auditoría multi-agente contra las fuentes de verdad (anteproyecto
de tesis y SRS), corrida el 28 de agosto de 2026, que originó los hallazgos
TASK-183 y TASK-192 sobre el orquestador de conversación, generó también un
hallazgo de prioridad media (TASK-193) sobre `GuardrailService` (P5.4,
TASK-49) en su rol de detección de urgencia. `checkIncomingMessage` compara
el mensaje entrante, sin diacríticos, contra una lista fija de
aproximadamente dieciocho frases (`URGENCY_KEYWORDS`) por coincidencia de
subcadena, antes de invocar al modelo. El hallazgo señala que un mensaje de
crisis parafraseado que no contuviera literalmente una de esas frases — cita
como ejemplos "no aguanto más", "quiero desaparecer", "ya no quiero seguir"
— salteaba por completo ese camino rápido (*fast path*). El prompt del
sistema sí le pide al modelo responder con el texto de urgencia por su
propio criterio cuando reconoce una situación de angustia, pero no existía
ninguna verificación del lado de la respuesta que confirmara que un mensaje
con lenguaje de angustia sostenida efectivamente hubiera recibido
`URGENCY_MESSAGE` — un único nivel de seguridad, precisamente en el caso que
el SRS — Módulo Turnos marca como el más importante ("mensajes
preocupantes").

## Qué se implementó

Se añadieron dos mecanismos complementarios en `src/chatbot/`, ambos
documentados en el propio hallazgo como alternativas "y/o" pero
implementados juntos por tratarse de defensas independientes con costos de
falso positivo distintos:

1. **Ampliación de `URGENCY_KEYWORDS`** (`guardrail.constants.ts`) con
   paráfrasis coloquiales de la ideación de crisis — las tres nombradas
   explícitamente por el hallazgo más variantes cercanas de la misma familia
   semántica ("no aguanto más", "no doy más", "quiero desaparecer", "ya no
   quiero seguir/vivir", "no tengo ganas de vivir", "quiero terminar con
   todo", entre otras). Sigue siendo coincidencia literal de subcadena,
   comparada contra el texto sin diacríticos — el mismo mecanismo que ya
   tenía la regla 4a, simplemente con más entradas.

2. **Regla 4b, nueva**: un chequeo del lado de la respuesta
   (`GuardrailService.checkResponse`, ahora con un segundo parámetro
   `incomingMessage`) respaldado por una nueva constante,
   `DISTRESS_LANGUAGE_PATTERNS` — un conjunto de expresiones regulares con
   conectores flexibles (en lugar de frases literales) que buscan lenguaje
   de angustia o autolesión en el mensaje *entrante*. Si el mensaje
   coincide con alguno de esos patrones y la respuesta final del modelo no
   es ya `URGENCY_MESSAGE`, la regla la reemplaza por el mensaje de urgencia
   y lo reporta bajo el nuevo discriminante `no_missed_urgency`. Se ejecuta
   primero, antes que las cinco reglas preexistentes de `checkResponse`, de
   modo que una crisis no reconocida prevalece incluso si la misma respuesta
   también dispara otra regla (por ejemplo, un rechazo de copago).
   `OrquestadorService.runTurn` pasa el `mensajeEntrante` del turno actual a
   `checkResponse` para habilitar este chequeo.

## Decisiones y por qué

**La regla 4b es una expresión regular flexible, no una segunda lista de
frases literales.** El propio hallazgo diagnostica que el problema de fondo
es depender de una lista fija; ampliar `URGENCY_KEYWORDS` (mecanismo 1)
sigue siendo, por construcción, una lista finita — mitiga el síntoma
concreto que cita el hallazgo, pero no generaliza a paráfrasis no
anticipadas. `DISTRESS_LANGUAGE_PATTERNS` usa conectores con `\s+` y
alternancia entre verbos/objetos (por ejemplo, `/\bno\s+(aguanto|doy|puedo)\s+mas\b/`
captura "no aguanto más", "no doy más" y "no puedo más" con una sola
entrada) precisamente para no tener que anticipar cada variante como una
entrada nueva de la lista. "no puedo más" es, de manera deliberada, un caso
que la regla 4b cubre y `URGENCY_KEYWORDS` no — la prueba de extremo a
extremo de esta corrección usa exactamente ese mensaje para demostrar que
la segunda capa hace un trabajo distinto de la primera, no que lo repite.

**La regla 4b corre en `checkResponse`, no como una segunda regla 4a en
`checkIncomingMessage`.** El hallazgo pide explícitamente una verificación
"del lado de la respuesta", y la razón estructural coincide con la que ya
documenta `CLINICAL_ADVICE_PATTERNS` para la regla 1b: lo que hace sospechosa
a la situación no es el mensaje del paciente por sí solo (una paráfrasis de
angustia puede seguir siendo ambigua sin ver cómo respondió el modelo), sino
que el modelo haya tenido la oportunidad de reconocerla — vía el prompt de
sistema, que sí le pide responder con `URGENCY_MESSAGE` ante angustia — y no
lo haya hecho. Verificar sólo el mensaje entrante, como hace la regla 4a,
habría convertido cualquier ampliación de patrones en un segundo *fast path*
más agresivo, con el mismo riesgo de falsos positivos que motivó no incluir
frases más ambiguas en `URGENCY_KEYWORDS` en primer lugar (por ejemplo, "no
puedo más" dicho sobre un dolor de cabeza). Como regla de respuesta, en
cambio, el falso positivo sólo se paga cuando el modelo *además* falla en
reconocer la situación — un caso mucho más acotado, y exactamente el que el
hallazgo pide cubrir.

**La regla 4b se ejecuta primero, antes que las reglas 1, 1b, 2, 3 y 5.**
Una respuesta a un mensaje de angustia no reconocida podría, en teoría,
disparar también otra regla (por ejemplo, si el modelo intentó cambiar de
tema hacia el copago). El orden importa porque el resultado de
`checkResponse` es de único acierto (*first-match-wins*): si otra regla
corriera primero, el paciente recibiría el mensaje canned de esa regla en
lugar del de urgencia, que es el que el SRS marca como prioritario para
"mensajes preocupantes".

**`incomingMessage` es un parámetro opcional con valor por defecto `''`, no
un parámetro obligatorio nuevo.** Cambiar la firma de `checkResponse` a un
parámetro requerido habría roto en tiempo de compilación cada prueba
unitaria preexistente que la invoca con un solo argumento. Un valor por
defecto de cadena vacía no coincide con ningún patrón de
`DISTRESS_LANGUAGE_PATTERNS`, así que la regla 4b simplemente nunca se
dispara para esas pruebas — se comportan exactamente igual que antes de esta
corrección, sin necesidad de tocarlas.

## Alternativas descartadas

- **Ampliar únicamente `URGENCY_KEYWORDS`, sin la regla 4b**: descartada
  porque no resuelve la causa raíz que el propio hallazgo nombra en su
  título ("depende de una lista fija ... no del criterio del modelo") — sólo
  desplaza el límite de la lista, que sigue siendo finita.
- **Verificar `DISTRESS_LANGUAGE_PATTERNS` contra el mensaje entrante como
  una segunda regla 4a** (short-circuit antes de llamar al modelo), en lugar
  de como una regla 4b del lado de la respuesta: descartada por el
  razonamiento de falsos positivos de la segunda decisión arriba — patrones
  más amplios que los de `URGENCY_KEYWORDS` conviven mejor con el sistema
  cuando el costo de un falso positivo depende también de si el modelo ya
  falló, no sólo del mensaje del paciente.
- **Hacer `incomingMessage` un parámetro requerido de `checkResponse`**:
  descartada por el costo de romper la firma de cada prueba unitaria
  preexistente sin ganancia real — un valor por defecto inerte logra la
  misma compatibilidad hacia atrás.

## Entidades / puertos / adaptadores tocados

- `src/chatbot/guardrail.constants.ts`: `URGENCY_KEYWORDS` ampliada; nueva
  constante `DISTRESS_LANGUAGE_PATTERNS`; `GuardrailRule` gana el
  discriminante `'no_missed_urgency'`.
- `src/chatbot/guardrail.service.ts`: `checkResponse` gana el parámetro
  `incomingMessage` (con valor por defecto) y el método privado
  `missedUrgency`, que implementa la regla 4b y corre primero dentro de
  `checkResponse`.
- `src/chatbot/orquestador.service.ts`: `runTurn` pasa `mensajeEntrante` a
  `checkResponse`.
- Ningún cambio de esquema ni de migración: `GuardrailService` sigue siendo
  íntegramente sin estado y sin base de datos, por el mismo motivo que
  documenta su propio encabezado — las reglas son globales y no
  configurables por tenant.

## Tests y qué validan

- `src/chatbot/guardrail.service.spec.ts`: nuevos casos en la sección de la
  regla 4a probando las paráfrasis agregadas a `URGENCY_KEYWORDS`; nueva
  sección "rule 4b (angustia no reconocida)" con seis pruebas — confirma que
  "no puedo más" no dispara la regla 4a por sí sola (la premisa de que la
  regla 4b cubre un caso distinto); bloquea una respuesta que ignoró el
  lenguaje de angustia con el mensaje de urgencia; deja pasar sin cambios
  una respuesta que el modelo ya acertó como urgencia; no dispara ante un
  mensaje entrante ordinario; nunca se activa cuando no se pasa
  `incomingMessage` (compatibilidad hacia atrás); y tiene prioridad sobre
  otra regla que la misma respuesta también dispara.
- `test/chatbot-flows.e2e-spec.ts`: dos casos nuevos junto a la prueba
  preexistente de la regla 4a — un mensaje parafraseado que la regla 4a no
  detecta, seguido de una respuesta del modelo que tampoco lo reconoce como
  urgencia, termina en `URGENCY_MESSAGE`; el mismo mensaje, cuando el modelo
  sí responde correctamente con `URGENCY_MESSAGE` por su propio criterio, no
  se bloquea ni se altera.
- Ejecución: suite unitaria completa en verde (93 conjuntos, 1010 pruebas,
  `guardrail.service.ts` con 100 % de cobertura de líneas/ramas/funciones);
  suite de extremo a extremo completa en verde con `--runInBand` (146
  conjuntos, 1657 pruebas) contra Postgres local; análisis estático
  (ESLint) sobre el árbol completo sin advertencias. Datos ficticios, sin
  contenido clínico ni datos personales reales.

## Figuras pendientes

Ninguna figura nueva; la corrección no introduce un flujo o diagrama nuevo
respecto del ya registrado para `GuardrailService` (P5.4) — la regla 4b es
un chequeo interno adicional dentro de un componente ya documentado, no un
flujo visible desde el diagrama de arquitectura.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-193-urgency-detection-depth` (creada a
  partir de `main`).
- Ticket: TASK-193 ("[MEDIO] Detección de urgencia depende de una lista fija
  de ~18 frases, no del criterio del modelo"), hallazgo de la misma
  auditoría multi-agente del 28 de agosto de 2026 que originó TASK-183 y
  TASK-192, sobre `src/chatbot/guardrail.constants.ts`
  (`URGENCY_KEYWORDS`) y `src/chatbot/guardrail.service.ts`
  (`checkIncomingMessage`).
