# Fase 5 — Capa conversacional y WhatsApp (backend) — el bot reconoce al paciente por su número de WhatsApp (TASK-223, corrección a P5.3/P5.5)

## Contexto

Hallazgo de uso real reportado por la usuaria (TASK-223, "vuelve a pedir
siempre el dni y parece que no lo valida y pide siempre el
consentimiento"; el ticket no tiene descripción): cada vez que el paciente
inicia una conversación, el bot le vuelve a pedir el DNI y, si es un
paciente nuevo, le vuelve a pedir el consentimiento, aunque ya lo haya
dado. Lo esperado, según la usuaria: una vez que el paciente pasó su DNI y
aceptó el consentimiento, eso debe quedar asociado a su número de celular y
no volver a pedirse.

La inspección del código mostró que los dos datos sí quedaban registrados:

- El alta de un paciente desde el chatbot (`find_or_create_patient`, P5.5,
  TASK-50) guarda como `mobilePhone` el número de WhatsApp desde el que
  escribe, tomado de la conversación y no del modelo.
- El consentimiento es una fila propia (`Consent`, P2.4), y
  `ConsentsService.accept` es idempotente y persiste en la base. En la base
  local de desarrollo, el paciente de prueba creado por el chatbot tenía su
  número y su consentimiento registrados.

El defecto estaba en la memoria de la conversación: lo único que el bot
recordaba de un paciente era el historial de la sesión
(`ConversationSessionStore`), que vive en memoria y vence tras el tiempo de
inactividad configurado por inquilino (30 minutos por defecto) o al
reiniciarse el proceso. Vencido ese historial, el manual de flujos
indicaba, sin excepción, "Pedile el DNI" como primer paso, y el modelo no
tenía forma de saber que ese número ya correspondía a un paciente con
consentimiento aceptado. La validación del DNI en sí (7 u 8 dígitos,
`IsPatientDni`) funcionaba correctamente; la percepción de que "no lo
valida" se explica por el pedido repetido.

## Qué se implementó

- **`CallerIdentityService`** (nuevo, `src/chatbot/caller-identity.service.ts`):
  en cada turno busca los pacientes activos (`deletedAt: null`) del
  inquilino cuyo `mobilePhone` coincide con el número que escribe, con un
  tope de `MAX_PATIENTS_PER_PHONE` (5). `describe()` devuelve una sección de
  prompt titulada "Paciente que te escribe":
  - **Un paciente:** nombre, apellido y DNI; la instrucción de no pedir el
    DNI y llamar directamente a `find_or_create_patient` con ese DNI; la
    prohibición de mostrarle o repetirle el DNI; y, si
    `ConsentsService.verify` indica que ya aceptó, que no se le vuelva a
    pedir ni mencionar el consentimiento. Si la consulta es para otra
    persona, se le pide el DNI de esa persona.
  - **Varios pacientes** (una familia que comparte el teléfono): la lista,
    con la instrucción de preguntar para cuál de ellos es la consulta,
    nombrándolos solo por el nombre de pila, y usar el DNI del elegido sin
    pedirlo.
  - **Ninguno:** no se agrega nada y la identificación sigue como antes.
- **`OrquestadorService.runTurn`** resuelve esa sección junto con el prompt
  del inquilino y la agrega al final del prompt de sistema cuando existe.
- **Manual de flujos** (`conversation-flows.constants.ts`): el paso 1 de la
  identificación pasa de "Pedile el DNI" a "si al final de estas
  instrucciones hay una sección 'Paciente que te escribe', seguí lo que
  dice ahí y no le pidas el DNI; si no la hay, pedile el DNI". El título de
  la sección es una constante compartida
  (`CALLER_IDENTITY_SECTION_TITLE`) entre el manual y el servicio.

## Decisiones y por qué

**El reconocimiento se apoya en la base y no en extender la sesión.** Alargar
el tiempo de inactividad o persistir el historial habría postergado el
problema sin resolverlo: el historial siempre termina vencido, y además
cargaría en cada conversación nueva mensajes viejos ajenos a la consulta
actual. El número y el consentimiento ya son hechos registrados; lo que
faltaba era leerlos al empezar.

**Se consulta en cada turno y no una vez por sesión.** Así, un paciente que
se registra a mitad de la conversación queda reconocido desde el turno
siguiente, y un consentimiento recién aceptado figura como aceptado. El
costo es una consulta indexada por inquilino por mensaje, despreciable a la
escala de la clínica.

**La sección va al final del prompt de sistema y no en la parte
configurable por el inquilino**, porque es un dato de la conversación y no
de la clínica, y se calcula dentro del contexto de conversación que el
orquestador abre por turno (`ConversationContextService`), que es donde se
conoce el número que escribe.

**El modelo sigue llamando a `find_or_create_patient`.** En lugar de inyectar
directamente el id del paciente y sus datos, la sección le da el DNI para
que recorra el mismo camino de siempre, que es el que devuelve
`missingBookingData`, `needsContactUpdate` y los profesionales del paciente.
Un segundo camino de identificación habría tenido que replicar esas reglas.

**El consentimiento se informa solo cuando está aceptado.** Si no lo está,
la sección no dice nada y `check_consent` sigue siendo quien entrega el
texto exacto de la plantilla `SOLICITUD_CONSENTIMIENTO`, que no debe
improvisarse.

**El número se compara tal cual está guardado.** Un paciente dado de alta
por el chatbot tiene exactamente la cadena que entrega el webhook, que es
el caso del ticket. Un número importado de las planillas en otro formato no
coincide, y ese paciente se sigue identificando por DNI.

**Los pacientes dados de baja quedan fuera**, porque
`find_or_create_patient` ya responde por ellos indicando que solo la
clínica puede reactivarlos, y saludarlos como pacientes conocidos
contradiría esa respuesta.

## Alternativas descartadas

- **Persistir el historial de la sesión o alargar su vencimiento.** Ver
  arriba: posterga el síntoma y arrastra contexto viejo.
- **Completar el celular de un paciente existente que se identifica por DNI
  y no lo tiene cargado**, para que también sea reconocido la próxima vez.
  Quedó fuera de esta tarea: `PatientsService.update` sella
  `contactUpdatedAt` al tocar el celular, lo que alteraría la regla de
  actualización de datos de contacto (`needsContactUpdate`), y asociar un
  número a una ficha solo porque alguien escribió su DNI merece una
  decisión explícita de la usuaria.
- **Normalizar los formatos de teléfono** (con o sin `+`, prefijo 9, etc.)
  para reconocer también a pacientes importados. Requiere una regla de
  normalización compartida con el importador y no era necesaria para el
  caso reportado.

## Entidades / puertos / adaptadores tocados

- `src/chatbot/caller-identity.service.ts` (nuevo).
- `src/chatbot/orquestador.service.ts`: agrega la sección al prompt.
- `src/chatbot/chatbot.module.ts`: registra el servicio.
- `src/chatbot/chatbot.constants.ts`: `MAX_PATIENTS_PER_PHONE`.
- `src/chatbot/conversation-flows.constants.ts`:
  `CALLER_IDENTITY_SECTION_TITLE` y paso 1 de la identificación.
- Ningún cambio de esquema, migración, puerto ni ruta HTTP.

## Tests y qué validan

- `src/chatbot/caller-identity.service.spec.ts` (nuevo, 7 pruebas): busca
  por el número de la conversación, solo pacientes activos y con tope; sin
  número no consulta; sin coincidencias no agrega nada; con un paciente
  nombra su DNI e indica no pedirlo; informa el consentimiento aceptado y
  calla cuando no lo está; con varios pacientes los lista y pide elegir sin
  consultar el consentimiento.
- `src/chatbot/orquestador.service.spec.ts`: la sección se agrega al prompt
  y se calcula dentro del contexto de conversación (ve el número que
  escribe); sin sección el prompt queda igual.
- `test/chatbot-conversation-e2e.e2e-spec.ts`: de punta a punta por el
  webhook firmado y contra Postgres real, un número desconocido no recibe
  la sección; el paciente se da de alta (queda con su número como celular)
  y acepta el consentimiento; en el mensaje siguiente el prompt incluye
  "Paciente que te escribe", su nombre y DNI y "Ya aceptó el
  consentimiento".
- Resultado: las tres suites de extremo a extremo del chatbot (54 pruebas)
  en verde; pruebas unitarias en verde salvo la de selección del adaptador
  de cerradura, que falla igual sobre la rama base por la variable local
  `LOCK_ADAPTER="hardcoded"`. En la corrida completa de extremo a extremo
  fallan 4 pruebas de otros módulos (motor de turnos, importación y
  entidades de pacientes), que fallan igual sobre `origin/development` sin
  estos cambios. Análisis estático y tipos sin errores. Datos ficticios,
  sin contenido clínico.
- Límite de la verificación: las pruebas comprueban lo que el prompt le
  dice al modelo, no la conducta del modelo real frente a él; la
  confirmación final es una conversación de prueba por WhatsApp.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-223-remember-patient-identity`
  (creada a partir de `origin/development`).
- Ticket: TASK-223, hallazgo de uso real de la usuaria, corrección al
  orquestador de P5.3 (TASK-48) y al manual de flujos de P5.5 (TASK-50).
