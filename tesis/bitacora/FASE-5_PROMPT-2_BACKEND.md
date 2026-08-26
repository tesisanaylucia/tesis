# Fase 5 — Capa conversacional y WhatsApp (backend) — catálogo de herramientas del motor de turnos (TASK-47)

## Qué se implementó

Se definió el catálogo completo de herramientas (tools) que el modelo de
lenguaje puede invocar: los once handlers que el ticket nombra —
disponibilidad, reserva, confirmación, reprogramación, cancelación y
consulta de turnos; búsqueda/alta de paciente; registro y verificación de
consentimiento; solicitud de receta; y FAQ — cada uno como un adaptador
delgado sobre el servicio de dominio ya existente de M2/M3, sin lógica de
negocio propia.

Dos piezas de dominio no existían todavía y se agregaron como parte de esta
tarea, porque las herramientas no tenían a qué delegar sin ellas: el modelo
`Faq` (el ER lo dibuja — `id_faq`, `pregunta`, `respuesta` — pero nada antes
del chatbot lo necesitaba) con su propio `FaqService`, y
`PrescriptionRequestsService`, ya que `PrescriptionRequest` existía solo
como entidad de esquema desde TASK-27 sin ningún servicio que la escribiera.

## Decisiones y por qué

**La rama de esta tarea se creó sobre la rama de TASK-46, no sobre `main`.**
El catálogo de herramientas usa los tipos de dominio que `AIPort` expone
—`AITool`, en particular— y esos tipos no existen todavía en `main`: los
introdujo TASK-46, cuya rama (`feature/TASK-46-openai-ai-adapter`) sigue sin
mergear. El propio ticket declara esa dependencia ("Dependencias: TASK-46").
Ramificar desde `main` a secas habría dejado el código sin compilar o
forzado a reintroducir esos tipos por duplicado, así que la rama de esta
tarea (`feature/TASK-47-chatbot-tools`) parte de la de TASK-46 en su lugar
—la práctica habitual cuando una tarea depende del trabajo, todavía no
integrado, de otra— y quedará para rebasar sobre `main` una vez que esa
rama se mergee.

**Cada tool se ensambla con una única función `defineTool`, no a mano once
veces.** El ticket pide, para cada una: validar la entrada, resolver quién
ejecuta la escritura, ejecutar el servicio de dominio y devolver
`{success, data, error}` sin nunca lanzar una excepción al modelo. Esas
cuatro partes son idénticas en las once — sólo cambian el DTO de entrada y
la llamada final —, así que se extrajeron a `defineTool`
(`src/chatbot/define-tool.ts`), que a su vez compone tres piezas más chicas
y también reutilizables: `validateToolInput` (el mismo pase de
class-validator que ya usan los DTO HTTP y, a mano, `validateImportRow` del
importador de pacientes), `resolveSystemActor` (ver abajo) y
`runToolHandler` (que traduce cualquier excepción en un resultado
descriptivo). Cada tool queda entonces en unas pocas líneas: su nombre,
descripción, `input_schema`, el DTO y la llamada al servicio.

**Los errores de las tools reutilizan las excepciones HTTP de Nest en lugar
de un esquema de error propio.** `runToolHandler` atrapa cualquier
excepción que la llamada al servicio lance y, si es una `HttpException`
(`NotFoundException`, `BadRequestException`, `ConflictException`,
`ForbiddenException`, las mismas que ya usan `AppointmentsService`,
`PatientsService`, etc.), extrae su mensaje ya pensado para mostrarse a un
llamador — el mismo texto que de otro modo formaría el cuerpo de una
respuesta HTTP 4xx — y lo devuelve como el `error` de la tool. No hizo falta
inventar una segunda taxonomía de errores de dominio: la que ya existe
alcanza.

**Cada tool que escribe corre como el usuario SYSTEM del tenant, resuelto
desde el contexto ya abierto, nunca desde un parámetro.** Varios métodos de
`AppointmentsService` (`book`, `confirm`, `cancel`, `reschedule`) piden un
`actorId` y, los tres últimos, un `AuthenticatedUser` completo para
auditoría y para las comprobaciones de permiso — comprobaciones que ya
tratan a SYSTEM igual que a ADMIN, y que en el caso de `reschedule`
distinguen explícitamente el camino del bot (SYSTEM) del de un profesional
reprogramando por su cuenta. No existía todavía ningún llamador no-HTTP que
necesitara ese actor fuera de los crons (que resuelven el SYSTEM de *cada*
organización, recorriéndolas desde fuera de cualquier contexto de
inquilino). Se agregó `resolveSystemActor`
(`src/common/actors/system-actor.ts`) como el caso simétrico: asume que el
contexto de inquilino ya está abierto —el futuro orquestador (TASK-48) lo
abre una vez por turno de conversación— y lee el usuario SYSTEM a través
del cliente ya acotado, sin recibir ni aceptar un `organizationId` por
parámetro. Es, a la vez, la respuesta concreta al requisito de marca blanca
del ticket ("cada tool recibe o infiere el organizationId del contexto de
la sesión, no del input del usuario"): ningún `input_schema` declara ese
campo, y no hay forma de que una tool actúe para un tenant que no sea el
que ya está en contexto.

**FAQ usa una heurística de superposición de palabras, no embeddings ni
`pg_trgm`.** El ticket describe la tool como "búsqueda por similitud", pero
no dice con qué mecanismo. El conjunto de FAQ de un tenant es a lo sumo unas
pocas decenas de filas curadas por la clínica —no el volumen para el que
existen esas herramientas—, y ninguna de las dos está integrada en el
proyecto en ningún otro punto. Se implementó entonces una similitud de
Jaccard sobre el conjunto de palabras normalizadas (minúsculas, sin tildes,
sin puntuación) de la pregunta y de cada `pregunta` de la FAQ, con un umbral
mínimo por debajo del cual la tool devuelve `matched: false` en lugar de
forzar una respuesta — que el modelo, no la tool, decida qué decirle al
paciente cuando no hay nada que responder está en línea con el resto del
diseño (una tool nunca decide qué se le dice al paciente, sólo entrega
datos).

**`buscarOCrearPaciente` reutiliza `CreatePatientDto` completo, no la firma
abreviada del ticket.** El ticket describe la tool como
`buscarOCrearPaciente(dni, nombre?, apellido?, celular?)`, pero
`CreatePatientDto` —ya construido en P2.1 y con el comentario explícito de
que lo usan tanto un admin como el chatbot— exige además fecha de
nacimiento y contacto de emergencia para dar de alta un paciente, porque el
SRS los pide como datos obligatorios para poder reservar. Aceptar la firma
abreviada del ticket habría creado pacientes no reservables, pateando el
mismo error hacia adelante, al momento de la reserva. Se amplió entonces el
`input_schema` de la tool a los cinco campos que `CreatePatientDto`
realmente exige más los dos opcionales, documentando el porqué en el propio
archivo; el chatbot puede llamarla primero sólo con el DNI para verificar
si el paciente ya existe, y una segunda vez con el resto si tiene que
crearlo.

**`registrarSolicitudReceta` no dispara ninguna notificación.** El
`NotificationType.PRESCRIPTION_REQUESTED` ya existe en el esquema desde
TASK-90, con un comentario explícito de que no tiene todavía ningún
llamador. Wire-ar la notificación aquí habría sido alcance no pedido por
este ticket —cuya única exigencia es que el pedido quede con
`estado=pendiente` en base y que no se genere ninguna receta— y hubiera
adelantado una decisión (a quién y cuándo notificar) que no le corresponde
a esta tarea.

## Alternativas descartadas

- **Devolver todo el listado de FAQ al modelo y dejar que él elija**:
  hubiera aprovechado mejor la capacidad semántica real del LLM que
  cualquier heurística de palabras, pero el ticket pide explícitamente que
  el servicio haga la búsqueda por similitud y devuelva una respuesta
  puntual, no la lista completa en cada turno — un diseño consciente,
  probablemente para no inflar el prompt de cada turno con todo el
  contenido de la FAQ.
- **Aceptar la firma abreviada de `buscarOCrearPaciente` del ticket**:
  descartada por las razones dadas arriba — crearía pacientes no
  reservables.
- **Escribir la validación y el manejo de errores a mano en cada una de las
  once tools**: descartada en favor de `defineTool`, que las centraliza sin
  perder la posibilidad de que cada tool declare su propio DTO y su propia
  llamada.
- **Notificar al profesional al crear una `PrescriptionRequest`**:
  descartada por las razones dadas arriba — alcance no pedido, y el propio
  esquema ya documenta que esa integración queda para más adelante.

## Entidades / puertos / adaptadores tocados

- `Faq` (modelo Prisma nuevo, tenant-scoped patrón 1): `id`, `organizationId`,
  `question`, `answer`. Migración `20260821023826_add_faq`. Sin endpoint
  administrativo todavía — sólo la lectura que `FaqService` necesita.
- `FaqService` + `faq-similarity.ts` (`src/faq/`).
- `PrescriptionRequestsService` (`src/patients/`), exportado desde
  `PatientsModule`.
- `src/chatbot/` (módulo nuevo): `ChatbotToolsService` (catálogo agregado),
  `defineTool`/`validateToolInput`/`runToolHandler`/`chatbot-tool.types.ts`
  (plumbing compartido), y `tools/appointment.tools.ts` (6),
  `tools/patient.tools.ts` (1), `tools/consent.tools.ts` (2),
  `tools/prescription-request.tools.ts` (1), `tools/faq.tools.ts` (1).
- `src/common/actors/system-actor.ts` (nuevo, compartido).
- Ningún puerto de dominio tocado — estas tools llaman directamente a
  servicios de dominio existentes (`AppointmentsService`,
  `AvailabilityService`, `PatientsService`, `ConsentsService`), no a un
  puerto: el ticket lo pide así ("las tools no incluyen lógica de negocio;
  solo adaptan parámetros y delegan al servicio").

## Tests y qué validan

- `faq-similarity.spec.ts`: normalización (tildes, puntuación, mayúsculas) y
  puntaje de Jaccard — 1 para el mismo conjunto de palabras en cualquier
  orden, 0 sin superposición, parcial en el medio.
- `faq.service.spec.ts`: devuelve la FAQ de mayor puntaje por sobre el
  umbral; `null` si nada lo alcanza o si el tenant no tiene FAQ.
- `prescription-requests.service.spec.ts`: crea la solicitud con
  `estado=pendiente` y su entrada de auditoría; el link paciente-profesional
  se verifica antes de escribir (404 si no existe, sin llegar a `create`).
- `validate-tool-input.spec.ts`, `run-tool-handler.spec.ts`,
  `system-actor.spec.ts`, `define-tool.spec.ts`: cada pieza del plumbing
  compartido por separado — validación con reporte descriptivo, conversión
  de excepción a resultado, resolución del actor SYSTEM, y la composición
  de las tres en `defineTool` (entrada inválida no llega a tocar el actor
  ni el servicio; el actor se resuelve y se pasa al `execute`; un error del
  servicio se convierte en fallo descriptivo, nunca en una excepción que
  escape).
- `appointment.tools.spec.ts`, `patient.tools.spec.ts` (incluye
  explícitamente "devuelve el paciente existente si el DNI ya está
  registrado, sin crear"), `consent.tools.spec.ts`,
  `prescription-request.tools.spec.ts` (incluye "crea con estado=pendiente,
  nunca genera receta"), `faq.tools.spec.ts`: cada tool, con el servicio de
  dominio simulado — parámetros válidos llaman al servicio correcto con los
  argumentos correctos (incluido el actor SYSTEM cuando corresponde);
  parámetros faltantes o inválidos devuelven un error descriptivo sin
  llegar a llamar al servicio.
- `chatbot-tools.service.spec.ts`: agrega las cinco listas de tools en un
  catálogo único; ejecuta por nombre; nombre desconocido → fallo
  descriptivo, no una excepción.
- `test/chatbot-tools.e2e-spec.ts` (nuevo): arranca la aplicación completa y
  resuelve `ChatbotToolsService` real, para probar el cableado de
  inyección de dependencias de punta a punta —algo que los dobles de
  prueba de los tests unitarios, por construcción, no pueden probar—;
  confirma las once tools con nombres únicos y un `input_schema` bien
  formado, y que un nombre de tool desconocido no rompe la ejecución.
- Suite unitaria completa del repo en verde (55 suites, 541 tests) y suite
  e2e completa en verde (39 suites, 460 tests, corrida en serie por las
  mismas condiciones de carrera entre archivos preexistentes que ya afectan
  la corrida en paralelo, documentadas en la entrada de TASK-46), más
  `tsc --noEmit` y ESLint sin advertencias sobre los archivos tocados.

## Figuras pendientes

- Diagrama de la composición de una tool (`defineTool`): entrada cruda del
  modelo → `validateToolInput` → `resolveSystemActor` → llamada al servicio
  de dominio → `runToolHandler` → `{success, data}` o `{success, error}`,
  nunca una excepción. Sección 4.6 Capa conversacional y WhatsApp.
- Diagrama entidad-relación de `Faq` y `PrescriptionRequest` dentro del
  esquema, junto al resto del subdominio de Pacientes/Turnos. Sección 4.6.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-47-chatbot-tools`, creada desde
  `feature/TASK-46-openai-ai-adapter` (no desde `main`) por la dependencia
  real de tipos explicada arriba — quedará para rebasar sobre `main` una
  vez que esa rama se mergee. Sin commit ni push al momento de escribir
  esta entrada: pendiente de autorización explícita de la usuaria.
- Ticket: TASK-47 (Jira), "P5.2 – Definir las herramientas (tools) del
  motor de turnos", bajo el épico TASK-8 (Módulo 5). Fuentes de verdad
  consultadas en Drive: el documento de Especificación de Requisitos de
  Software (sección "Anexo: Módulo turnos" para los flujos conversacionales
  del SRS) y `modelo_base_de_datos.png` (forma de `FAQ` y
  `SOLICITUD_RECETA`). Dependencias: TASK-46 (P5.1, AIPort — ver nota sobre
  la rama), TASK-35/36/38 (disponibilidad/reserva/estados de turno),
  TASK-30 (consentimiento). Fuera de alcance y reservado a tareas futuras:
  el orquestador que decide cuándo invocar cada tool (P5.3, TASK-48); el
  flujo conversacional completo de solicitud de receta —desambiguación
  entre profesionales, mensajes de fuera de alcance— (P5.7, TASK-52); la
  visualización de solicitudes de receta desde la app del profesional (M7).
