# Fase 5 — Capa conversacional y WhatsApp (backend) — el "si" suelto como respuesta al pedido de confirmación (TASK-225, corrección a P5.5/P4.2)

## Contexto

TASK-225 ("Si respondo si en minuscula no lo entiende"; el ticket no tiene
descripción). La usuaria informó, con una captura de WhatsApp, que al
contestar "Si" al pedido de confirmación de un turno el bot no lo
interpretaba.

La captura mostró que las mayúsculas no eran la causa: el "SI" que el propio
mensaje pide tampoco confirmó el turno, y recién "Confirmo" lo hizo. La
inspección del código encontró el motivo: el pedido lo envía
`AppointmentConfirmationService` por fuera de toda conversación, de modo que
nunca entraba al historial que lee el modelo. La respuesta del paciente le
llegaba como primer mensaje de una conversación vacía.

## Qué se implementó

- **`PendingConfirmationService`.** Al comienzo de cada turno de
  conversación busca los turnos futuros en estado RESERVADO, con
  `confirmationRequestedAt` definido, de los pacientes activos cuyo celular
  es el número que escribe. Si hay alguno, el orquestador agrega al final del
  prompt de sistema la sección "Confirmación de turno pendiente de
  respuesta", con paciente, profesional, fecha y hora e identificador del
  turno.
- **Qué se le indica al modelo.** Un "sí" en cualquier forma responde esa
  pregunta, salvo que el último mensaje del bot haya preguntado otra cosa.
  Con un solo turno pendiente lo confirma directamente; con varios pregunta
  cuál. Ante un "no" no confirma ni cancela: pregunta si se quiere cancelar o
  reprogramar.
- **Flujo "confirmar turno" del manual.** Su primer paso remite ahora a esa
  sección; sin ella, los pasos son los de antes.
- **`declined-confirmation.ts`.** El orquestador rechaza
  `confirm_appointment`, antes de ejecutarla, cuando el mensaje del paciente
  es una negativa simple ("no", "No, gracias", "no puedo ir"), y le devuelve
  al modelo un error con la indicación de preguntar.

## Decisiones y su porqué

- **Leer de la base y no guardar el mensaje en el historial.** El historial
  está en memoria y vence por inactividad, mientras que el paciente dispone
  de horas para contestar. Leído del turno, el dato sobrevive a un reinicio y
  desaparece solo al confirmarse o cancelarse el turno.
- **Control en código para la negativa.** En una prueba con el modelo real,
  un "no" terminó confirmando el turno, acción que además envía un código de
  acceso. Es la misma división de trabajo que los guardrails: el prompt es el
  control primario y el código el que no depende de la obediencia del modelo.
- **Patrón de negativa anclado en ambos extremos.** "No hay problema,
  confirmo" empieza con "no" y es una afirmación: solo se rechaza el mensaje
  que es una negativa completa.
- **Un "no" no cancela.** Se pregunta antes, porque cancelar libera el turno.
- **Una entrada por sesión.** Las dos mitades de una primera sesión se
  colapsan con `toLogicalSessions`, agrupando antes por paciente, porque una
  familia puede compartir el número.

## Alternativas descartadas

- **Normalizar mayúsculas y tildes del mensaje entrante.** Es lo que sugería
  el título del ticket, pero no resolvía la causa.
- **Escribir el mensaje enviado en `ConversationSessionStore`.** Vence antes
  de que el paciente responda, por lo indicado arriba.

## Entidades, puertos y adaptadores tocados

- Nuevos: `src/chatbot/pending-confirmation.service.ts` y
  `src/chatbot/declined-confirmation.ts`.
- Modificados: `orquestador.service.ts`, `chatbot.module.ts`,
  `chatbot.constants.ts` (`MAX_PENDING_CONFIRMATIONS`) y
  `conversation-flows.constants.ts`.
- Ningún cambio de esquema, migración, puerto ni ruta HTTP.

## Tests y qué validan

- `pending-confirmation.service.spec.ts`: el filtro de la consulta, la
  ausencia de sección sin número o sin pendientes, un turno, varios turnos y
  una primera sesión nombrada una sola vez.
- `declined-confirmation.spec.ts`: negativas reconocidas, mensajes que quedan
  a criterio del modelo y que solo se rechaza la confirmación.
- `orquestador.service.spec.ts`: la sección se agrega al prompt, y una
  confirmación pedida tras un "no" no se ejecuta.
- `chatbot-conversation-e2e.e2e-spec.ts`: un "si" en minúsculas por el
  webhook confirma el turno y la sección no aparece en el mensaje siguiente.
- Resultado: 1249 pruebas unitarias en verde con `LOCK_ADAPTER` vacío (con el
  valor local "hardcoded" falla una ajena al cambio, TASK-229) y 66 e2e de
  chatbot y webhook en verde.
- Con el modelo real, paciente ficticio y WhatsApp simulado: sin la sección,
  0 de 7 afirmaciones confirmaron el turno; con ella, 11 de 11. En la versión
  final, 6 de 6 negativas no lo confirmaron.
- Límite de la verificación: antes de incorporar el control de la negativa,
  la prueba nueva falló 1 de 9 ejecuciones con el modelo real. La causa no se
  determinó; la hipótesis es el límite de cinco segundos de Jest (TASK-230).

## Hallazgos derivados

TASK-226 y TASK-227 (el chatbot no puede responder una oferta de lista de
espera ni una reprogramación propuesta), TASK-228 (celulares guardados en
otro formato), TASK-229 y TASK-230 (pruebas).

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-225-confirmation-reply` (creada a
  partir de `origin/development`), commit `a073980`.
- Ticket: TASK-225, pedido de la usuaria.
