# Fase 5 — Capa conversacional y WhatsApp (backend) — "profesionales disponibles" en consultas generales pasa a la lista en vivo (TASK-204, corrección a P5.6)

## Contexto

Hallazgo de prioridad baja de la misma auditoría multi-agente contra las
fuentes de verdad (anteproyecto de tesis y SRS), corrida el 28 de agosto de
2026, que ya había originado TASK-183, TASK-192 y TASK-193 sobre distintas
piezas de la capa conversacional. Esta vez el objeto es el "Flujo 'consultas
generales'" del manual de procedimiento (`CONVERSATION_FLOWS_PROMPT`,
P5.6/TASK-51). El SRS nombra "profesionales disponibles" como uno de los
cuatro temas que debe cubrir consultas generales, junto a dirección,
horarios y obras sociales de la clínica — pero el flujo, tal como quedó
escrito en TASK-51, sólo llamaba a `answer_faq` para cualquier pregunta de
este grupo. La herramienta `list_professionals` (`ChatbotProfessional`,
P5.5/TASK-50) ya devuelve un listado en vivo y siempre actualizado de los
profesionales de la organización, y el propio flujo de reserva la usa como
su primer paso — pero fuera de una reserva, una pregunta como "¿qué
profesionales atienden?" dependía enteramente de que la clínica hubiera
escrito y mantuviera sincronizada una fila de FAQ para esa pregunta
concreta; si no lo había hecho, la respuesta caía en
`FAQ_NO_ANSWER_MESSAGE` aunque el sistema tuviera la respuesta real a una
sola llamada de herramienta de distancia.

## Qué se implementó

Corrección exclusivamente de prompt, sin cambios de código ni de esquema,
en `src/chatbot/conversation-flows.constants.ts`:

- El encabezado del flujo "consultas generales" pasa a nombrar
  explícitamente "profesionales disponibles" entre sus temas, igual que ya
  lo nombra el SRS.
- Se agregó un primer paso que rutea la pregunta puntual por el roster de
  profesionales ("qué profesionales atienden", "quiénes son", "cuántos
  hay") a `list_professionals`, antes de caer en `answer_faq` — con la
  instrucción explícita de no usar `answer_faq` para este subtema. Las
  preguntas sobre la obra social o el importe de un profesional puntual no
  se tocaron: ya tenían su propio flujo ("Información de obra social e
  importes"), anterior en el prompt, que sigue resolviéndolas sin pasar por
  acá.
- El resto del flujo (pasos 2 a 5 renumerados) queda igual: toda otra
  consulta general sigue yendo a `answer_faq`, con el mismo manejo de
  `matched`/`fallbackMessage` que ya tenía.

## Decisiones y por qué

**La corrección es puramente de prompt, no una herramienta ni una regla de
código nuevas.** El propio hallazgo lo plantea como alternativa "o": rutear
la pregunta a `list_professionals` igual que ya hace el flujo de reserva, o
instruir al modelo para que la prefiera sobre la FAQ en este subtema — ambas
formulaciones son un cambio de instrucción, no de herramienta ni de
esquema. `list_professionals` ya existe, ya está acotada al inquilino por
construcción (P5.5) y ya es la fuente que responde la pregunta análoga de
obra social/importe (P5.6/TASK-163) sin una herramienta dedicada — no había
ninguna pieza de datos o de servicio faltante, sólo una instrucción de flujo
que nunca la mencionaba fuera de la reserva.

**Se ruteó dentro del mismo flujo "consultas generales", no se creó un
flujo nuevo.** El SRS agrupa "profesionales disponibles" junto a dirección,
horarios y obras sociales de la clínica como un único tema de consulta
general — separar "quiénes atienden" en su propio flujo habría fragmentado
algo que el documento de requisitos ya trata como una sola pregunta al
paciente, sólo que resuelta por una fuente de datos distinta según el
subtema (herramienta en vivo para el roster, FAQ para todo lo demás), el
mismo patrón que ya usa "Información de obra social e importes" al final de
su propio flujo para remitir las preguntas de alcance clínico a este mismo
flujo de consultas generales.

**El paso nuevo va primero, antes de `answer_faq`, no como una comprobación
posterior al resultado de la FAQ.** Dejar que el modelo llamara primero a
`answer_faq` y sólo recurriera a `list_professionals` ante un
`matched: false` habría seguido dependiendo de que la clínica *no* tuviera
cargada una fila desactualizada que sí matcheara — el hallazgo describe
justamente el caso en que la FAQ existe pero no está sincronizada con el
roster real, no sólo el caso en que falta. Rutear primero por el tema de la
pregunta, no por el resultado de una herramienta ya invocada, es lo que
saca a la FAQ de la ecuación para este subtema en particular.

## Alternativas descartadas

- **Instruir al modelo para que "prefiera" `list_professionals` sobre la
  FAQ sin una regla de ruteo explícita por tema de la pregunta**: la otra
  mitad del "o" que plantea el propio hallazgo. Se descartó en favor de la
  instrucción de ruteo explícita, más específica, por consistencia con el
  resto del manual de flujos, que en ningún otro punto deja a criterio
  implícito del modelo cuál de dos herramientas usar ante una ambigüedad
  previsible — cada bifurcación del prompt (por ejemplo, "Información de
  obra social e importes" paso 5, hacia consultas generales) nombra la
  condición exacta que la dispara.
- **Agregar una herramienta nueva, dedicada, para "quién atiende"**:
  descartada por el mismo motivo que TASK-163 no creó una herramienta nueva
  para importes — `list_professionals` ya carga exactamente los datos que
  esta pregunta necesita (nombre), y una segunda herramienta habría sido un
  segundo viaje de ida y vuelta a leer datos que el modelo ya puede obtener
  de la que existe.

## Entidades / puertos / adaptadores tocados

- `src/chatbot/conversation-flows.constants.ts`: reescritura del flujo
  "consultas generales" dentro de `CONVERSATION_FLOWS_PROMPT`. Ningún otro
  archivo de código cambia — `list_professionals` (`ProfessionalTools`,
  P5.5) y `answer_faq` (`FaqTools`, P5.6) ya existían sin modificación.
- Ningún cambio de esquema ni de migración.

## Tests y qué validan

- `test/chatbot-flows.e2e-spec.ts`: un caso nuevo, junto al ya existente
  que prueba la pregunta de obra social/importe por el mismo mecanismo,
  que guiona una pregunta general ("¿Qué profesionales atienden en la
  clínica?") resuelta con `list_professionals` en lugar de `answer_faq` —
  prueba la plomería del ruteo (la herramienta correcta responde sin fallar
  fuera de una reserva), no la elección del modelo real, ya que el doble de
  `AIPort` sigue un guion y no decide por criterio propio.
- Suite unitaria completa en verde (93 conjuntos, 1020 pruebas); suite de
  extremo a extremo completa en verde con `--runInBand` (53 conjuntos, 653
  pruebas) contra Postgres local; análisis estático (ESLint) sobre el árbol
  completo sin advertencias. Datos ficticios, sin contenido clínico ni
  datos personales reales.

## Figuras pendientes

Ninguna figura nueva; la corrección no cambia el diagrama del catálogo de
herramientas ni introduce un flujo visible distinto del ya documentado para
"consultas generales" (P5.6).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-204-general-inquiries-professionals-roster`
  (creada a partir de `main`).
- Ticket: TASK-204 ("[BAJO] 'Profesionales disponibles' en consultas
  generales depende de la FAQ manual, no de la lista en vivo"), hallazgo de
  la misma auditoría multi-agente del 28 de agosto de 2026 que originó
  TASK-183, TASK-192 y TASK-193, sobre
  `src/chatbot/conversation-flows.constants.ts:119-123` (el propio ticket
  cita ese rango de líneas; el flujo se encontró en la posición actual del
  archivo al momento de esta corrección).
