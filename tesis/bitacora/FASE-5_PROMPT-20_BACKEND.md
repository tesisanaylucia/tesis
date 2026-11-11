# Fase 5 — Capa conversacional y WhatsApp (backend) — el bot no ofrece a un paciente nuevo los profesionales que no toman pacientes nuevos (TASK-215, corrección a P5.5/P5.6)

## Contexto

Hallazgo de uso real reportado por la usuaria: siendo paciente nuevo, el
chat le ofrecía la nómina completa de profesionales, incluidos los que no
reciben pacientes nuevos. El ticket (TASK-215, "que no ofrezca de antemano
los que no toman pacientes nuevos si es paciente nuevo") no tiene
descripción; el criterio se tomó del reporte de la usuaria: "me debe ofrecer
turnos solo con los profesionales que acepten pacientes nuevos, si yo soy
paciente nuevo".

La regla de negocio ya se cumplía, pero tarde. La herramienta
`list_professionals` (P5.5, TASK-50) no recibía parámetros y devolvía a
todos los profesionales activos del inquilino; el manual de flujos indicaba
ofrecerlos por nombre, y recién después de que el paciente elegía, el
pre-chequeo `check_booking_eligibility` (P5.6, TASK-51) respondía
`NOT_ACCEPTING_NEW_PATIENTS` y el bot debía decirle que ese profesional no
estaba tomando pacientes nuevos y ofrecerle otro. La reserva misma ya
rechazaba ese caso (compuerta `acepta_nuevos` de
`new-patient-eligibility.rule.ts`), así que el defecto no era de integridad
sino de experiencia: se le ofrecía al paciente una opción que no tenía.

## Qué se implementó

- **`AppointmentsService.findProfessionalIdsClosedToPatient(patientId)`**
  (nuevo): devuelve el conjunto de profesionales del inquilino que no
  aceptan pacientes nuevos y con los que una reserva sería la primera
  sesión del paciente. Una sola consulta: profesionales con
  `acceptsNewPatients = false` que no tengan un vínculo
  `PacienteProfesional` del paciente con `firstSession = false`. Verifica
  antes que el paciente exista y esté activo en el inquilino (404 en caso
  contrario), igual que el resto de las lecturas por paciente del servicio.
- **`list_professionals` recibe un `patientId` opcional.** Si viene, la
  herramienta descarta de la lista a los profesionales de ese conjunto. Si
  no viene, devuelve la nómina completa, como antes.
- **Manual de flujos, paso 3 de "reservar turno"**: llamar a
  `list_professionals` con el id del paciente, ofrecer solo los
  profesionales que devuelve, no nombrar uno que no esté en esa lista
  aunque haya aparecido en una llamada sin paciente, y si viene vacía,
  decir que por el momento ningún profesional está tomando pacientes
  nuevos.
- **Limpieza derivada**: `list_professionals` era la única herramienta sin
  parámetros, de modo que `defineInputlessTool` y `rejectAnyToolInput`
  quedaron sin uso y se eliminaron junto con sus pruebas.

## Decisiones y por qué

**El filtro vive en la herramienta y en el motor de turnos, no solo en el
prompt.** Es la misma división que esta fase sostiene en cada regla que
importa: el prompt es el control primario, pero la herramienta es lo que
no depende de que el modelo obedezca. Con el filtro en la herramienta, el
modelo no tiene en la mano un profesional cerrado para ofrecerlo.

**El parámetro es opcional y no obligatorio**, a diferencia del `patientId`
de `check_availability` (TASK-213). La misma herramienta responde preguntas
que no son sobre un paciente: "¿quiénes atienden?" (consultas generales,
TASK-204) y "¿qué obra social acepta tal profesional?" (P5.6). Ahí la
respuesta correcta es la nómina completa, y exigir un paciente obligaría a
identificar por DNI a alguien que solo pregunta.

**"Paciente nuevo" se lee igual que en la reserva.** El predicado es el
indicador `firstSession` del vínculo, con la ausencia de vínculo contando
como primera sesión, igual que `isFirstSessionWith`, que usan `book` y
`check_booking_eligibility`. No se usa el tipo de paciente: la regla de
inactividad lo devuelve a "nuevo" tras un año sin concurrir, mientras que
`firstSession` sigue en falso, y un paciente de años que vuelve no es a
quien la compuerta `acepta_nuevos` busca rechazar. Así, lo que la lista
oculta y lo que la reserva rechaza no pueden separarse.

**Solo la compuerta `acepta_nuevos`, no la de edad mínima.** El caso
`BIRTH_DATE_UNKNOWN` del pre-chequeo de edad no rechaza: pide la fecha de
nacimiento y vuelve a consultar. Ocultar al profesional por edad haría
desaparecer ese diálogo y le quitaría al paciente la explicación de por qué
no puede atenderse con él. `check_booking_eligibility` sigue siendo el
pre-chequeo para la edad y la red de seguridad para el resto.

**La consulta de exclusión vive en `AppointmentsService`** y no en la
herramienta ni en `ProfessionalsService`, porque la pregunta es sobre el
vínculo paciente/profesional, y ese servicio es donde ya viven
`isFirstSessionWith` y la compuerta. El catálogo de herramientas mantiene
su regla de no contener reglas de agenda ni de reserva.

## Alternativas descartadas

- **Corregir solo el manual de flujos**, pidiéndole al modelo que llame a
  `check_booking_eligibility` para cada profesional antes de ofrecerlos.
  Descartada: multiplica las llamadas a herramientas por la cantidad de
  profesionales y depende de que el modelo lo haga siempre.
- **Exponer `acceptsNewPatients` en `ChatbotProfessional`** y que el modelo
  filtre. Descartada: el modelo no sabe si el paciente es nuevo para cada
  profesional (eso depende del vínculo), y dejarle el dato en la mano es
  justamente lo que permite ofrecerlo por error.
- **Hacer obligatorio el `patientId`.** Descartada por lo dicho arriba
  sobre las consultas generales y de obra social.

## Entidades / puertos / adaptadores tocados

- `src/appointments/appointments.service.ts`: método nuevo
  `findProfessionalIdsClosedToPatient`.
- `src/chatbot/tools/professional.tools.ts`: DTO de entrada nuevo con
  `patientId` opcional (`@OptionalDefined()` + `@IsUUID()`), esquema y
  descripción de la herramienta actualizados, inyección de
  `AppointmentsService` (el módulo del chatbot ya importaba
  `AppointmentsModule`).
- `src/chatbot/conversation-flows.constants.ts`: paso 3 del flujo de
  reserva.
- `src/chatbot/define-tool.ts` y `src/chatbot/validate-tool-input.ts`:
  eliminación de los auxiliares para herramientas sin parámetros.
- Ningún cambio de esquema ni migración. Ninguna ruta HTTP cambió.

## Tests y qué validan

- `src/chatbot/tools/professional.tools.spec.ts`: sin paciente se lista la
  nómina completa y no se consulta la compuerta; con paciente se descarta
  al profesional cerrado; un `patientId` que no es UUID y un parámetro
  inventado (`specialty`) se rechazan con su nombre.
- `test/chatbot-flows.e2e-spec.ts` (PostgreSQL real, con un profesional
  nuevo `acceptsNewPatients = false` en el inquilino de la suite): el
  paciente nuevo no ve al profesional cerrado; tampoco lo ve el paciente
  vinculado pero que todavía no tuvo su primera sesión; sí lo ve un
  paciente al que ese profesional ya atiende; un paciente inexistente
  produce una falla de herramienta y no una lista. La prueba existente de
  la nómina del inquilino ahora incluye al profesional cerrado, que
  confirma que sin paciente la lista sigue completa.
- Resultado: 1164 pruebas unitarias en verde (con la variable local
  `LOCK_ADAPTER` sin definir: el `.env` de la usuaria la tiene en
  `hardcoded`, lo que hace fallar una prueba de selección de adaptador sin
  relación con esta tarea). En extremo a extremo, 668 de 672 en verde; las
  cuatro fallas (en `appointment-engine-integration`, `patients-import` y
  `patients-entities`) se reproducen idénticas sobre la rama base sin estos
  cambios y son preexistentes. Análisis estático y tipos sin errores. Datos
  ficticios, sin contenido clínico ni datos personales reales.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-215-hide-closed-professionals-for-new-patients`
  (creada a partir de `origin/development`).
- Ticket: TASK-215, hallazgo de uso real de la usuaria, corrección a
  `list_professionals` de P5.5 (TASK-50) y al pre-chequeo de P5.6
  (TASK-51).
