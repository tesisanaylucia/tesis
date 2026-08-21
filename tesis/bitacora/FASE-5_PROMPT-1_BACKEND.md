# Fase 5 — Capa conversacional y WhatsApp (backend) — adaptador de IA con GPT-4o mini (TASK-46)

## Qué se implementó

Se implementó el adaptador del `AIPort` con GPT-4o mini vía el SDK oficial
`openai`: soporte de function calling (tools), historial de conversación
multi-turno y manejo de errores con reintento. Es la primera tarea del
Módulo 5 (Capa conversacional y WhatsApp) y la primera vez que `AIPort`
—declarado desde Fundaciones como puerto con un único adaptador *stub*—
tiene una implementación real.

Una primera versión de esta tarea se implementó por error contra Claude
Sonnet (`@anthropic-ai/sdk`), interpretando el nombre de la tarea del
tablero ("Adaptador de IA con Sonnet") como el proveedor definitivo. La
usuaria corrigió esto durante la misma sesión: la decisión real, tomada a
partir del documento de evaluación de tecnologías de IA conversacional
(insumo del Objetivo Específico 2 del anteproyecto) y de una conversación
previa no accesible para el agente, es GPT-4o mini sobre el SDK de OpenAI.
La implementación se rehizo por completo sobre ese proveedor antes de
cerrar la tarea; esta entrada documenta únicamente la versión final.

El contrato del puerto (`src/domain/ports/ai.port.ts`) se amplió respecto
de la firma mínima sugerida por el ticket
(`AIPort.procesarMensaje(mensajes, tools)`): `processMessage` pasó a
aceptar `messages: AIMessage[]`, `tools?: AITool[]` y, además, un tercer
parámetro opcional `system?: string` para la instrucción de sistema, con
tipos de dominio propios (`AIMessage`, `AIContentBlock`
—texto/tool_use/tool_result—, `AITool`, `AIResponse`, `AIStopReason`) que
no reexportan los de ningún SDK de proveedor concreto. La respuesta
(`AIResponse`) trae `text: string | null`, `toolUses: AIToolUseBlock[]` y
`stopReason`, para representar tanto una respuesta de sólo texto como una
de sólo `tool_use`, o una mixta. Como el contrato del puerto ya se había
diseñado sin filtrar tipos del SDK a la capa de dominio, el cambio de
proveedor a mitad de la tarea no tocó `ai.port.ts` en absoluto —sólo sus
comentarios, que nombraban al adaptador anterior—, lo cual valida en la
práctica la razón de ser de esa separación.

El adaptador real, `OpenAiAdapter`
(`src/infrastructure/adapters/openai.adapter.ts`), construye la petición a
partir de esas formas de dominio, traducidas por un módulo de funciones
puras separado (`openai.mappers.ts`), llama a
`client.chat.completions.create` con el modelo configurado, y traduce la
`ChatCompletion` de vuelta a `AIResponse`. El cliente del SDK se construye
en un *provider* de fábrica aparte (`openai-client.provider.ts`), inyectado
por un token de DI dedicado (`OPENAI_CLIENT`), en vez de construirse dentro
del propio adaptador.

## Decisiones y por qué

**GPT-4o mini y no Claude Sonnet.** El documento de evaluación de
tecnologías ubica la tarea del modelo de lenguaje en este chatbot como de
"complejidad conversacional media" —comprensión de intención, conducción
del diálogo, function calling y redacción de respuestas breves—, sin
requerir las capacidades de razonamiento de frontera de los modelos más
costosos, y prioriza por eso el equilibrio costo–calidad entre los tiers
económicos de cada proveedor. En ese comparativo, GPT-4o mini
(USD 0,15/0,60 por millón de tokens de entrada/salida) es sensiblemente
más económico que el tier equivalente de Anthropic considerado en el mismo
documento (Claude Haiku 4.5, USD 1/5) y el propio documento señala la
madurez de su function calling —clave porque el bot debe invocar de forma
confiable las funciones del backend: consultar grilla, reservar, generar
PIN— como una fortaleza distintiva frente a los demás proveedores
evaluados. Es, además, el criterio que efectivamente se usó: la elección
del equipo, informada por ese documento y por una discusión previa con la
propia herramienta de IA, fue explícitamente GPT-4o mini.

**Tipos de dominio propios en vez de los tipos de un SDK concreto.** El
puerto podría haber usado directamente los tipos del SDK del proveedor
elegido en cada momento. Se descartó porque `AIPort` vive en
`src/domain/ports/` y CLAUDE.md establece que un puerto de integración
externa no debe filtrar el proveedor concreto a la capa de dominio —el
mismo principio que ya aplican `MessagingPort` y `LockPort`—; el futuro
orquestador (TASK-48) y sus tests dependerían si no de la forma exacta que
un proveedor le da a un bloque de contenido, en lugar de depender de una
abstracción estable. El costo es un módulo de mapeo (`openai.mappers.ts`)
que traduce en ambos sentidos, compensado por poder testear esa traducción
de forma aislada (`openai.mappers.spec.ts`), sin el cliente HTTP ni la
política de reintentos de por medio, y —según quedó demostrado durante
esta misma tarea— por poder cambiar de proveedor sin tocar el contrato que
el resto del sistema conoce.

**El tercer parámetro `system` no estaba en la firma literal del ticket,
pero se agregó igual.** El ticket dice explícitamente que el prompt de
sistema del orquestador (TASK-48) es donde se incorpora el contexto de
inquilino. Sin un canal para pasarlo, el puerto habría quedado sin forma
de transmitir esa instrucción cuando TASK-48 exista, obligando a romper el
contrato de nuevo en esa tarea. Se prefirió declararlo ahora, opcional,
como una extensión mínima y de bajo riesgo en lugar de posponer un cambio
de contrato ya previsible. A nivel de adaptador, `OpenAiAdapter` lo
antepone como un mensaje `{role: "system", content: system}` a la
conversación, ya que la API de Chat Completions de OpenAI —a diferencia de
la de Anthropic, que sí tiene un campo `system` separado— no distingue el
prompt de sistema del resto de los mensajes salvo por su rol.

**Un turno del usuario con varios resultados de herramienta se parte en
varios mensajes `role: "tool"`.** Es una diferencia real entre las dos
APIs y no sólo cosmética: Anthropic admite un único turno de usuario con
varios bloques `tool_result`, mientras que la API de Chat Completions de
OpenAI exige un mensaje `tool` independiente por cada `tool_call_id` que
se responde. El módulo de mapeo resuelve esto con un `flatMap` —un
`AIMessage` de dominio puede convertirse en cero, uno o varios mensajes de
OpenAI—, en vez de forzar el resultado a un único mensaje con contenido
compuesto, que la API rechazaría.

**Política de reintento propia en el adaptador, con el reintento del SDK
apagado (`maxRetries: 0`).** El SDK `openai` ya reintenta automáticamente
429/5xx con backoff exponencial por su cuenta, pero esa política vive
dentro del transporte HTTP del cliente: no es observable desde el código
que lo usa y, sobre todo, no es ejercitable de forma determinista con un
doble de prueba que sustituye directamente
`client.chat.completions.create` (mockear ese método salta por completo la
capa de transporte donde el SDK reintenta, así que un test así nunca vería
un reintento real ocurrir). Se implementó entonces el reintento
explícitamente en `OpenAiAdapter.withRetry`, distinguiendo por clase de
error (`OpenAI.RateLimitError`, `OpenAI.InternalServerError` son
reintentables; cualquier otro error no lo es), con backoff exponencial
(500 ms · 2^(intento-1)) y tope de tres intentos totales, tal como pide el
ticket.

**Los argumentos de una llamada a herramienta se parsean de forma
defensiva.** A diferencia del SDK de Anthropic, que entrega el `input` de
un `tool_use` ya parseado como objeto, el SDK de OpenAI entrega
`function.arguments` como una cadena JSON cruda que el propio proveedor
advierte que el modelo no siempre genera válida. Un `JSON.parse` sin
protección habría propagado un `SyntaxError` opaco en vez de un error
descriptivo; se agregó entonces un `parseToolArguments` que envuelve la
falla en `AIProcessingError`, nombrando la herramienta afectada.

**La API key nunca llega al adaptador.** `OpenAiAdapter` recibe el cliente
del SDK ya construido (vía el token `OPENAI_CLIENT`), nunca la clave en sí
— sólo `openai-client.provider.ts`, un archivo aparte, la lee de
`ConfigService` para construir el cliente. Es una garantía más fuerte que
"no loguear la clave a propósito": estructuralmente el adaptador no tiene
forma de loguearla porque no la posee. Lo único que llega al logger, en
cualquier punto del ciclo de reintento, es `status`/`name`/`message` del
error tipado del SDK — nunca el cliente completo ni el cuerpo crudo de la
petición o la respuesta.

**Falla rápida en el arranque si falta `OPENAI_API_KEY`.** Se usó
`configService.getOrThrow`, el mismo criterio que ya aplica `JWT_SECRET` en
`AuthModule`: si la clave falta, la aplicación no arranca, en vez de fallar
recién ante el primer mensaje real de un paciente. Efecto colateral
aceptado: el archivo `.env` local (no versionado) necesitó una entrada
`OPENAI_API_KEY` — un valor de relleno, ya que ninguna prueba existente hoy
invoca `AIPort` de verdad — para que la aplicación y la batería de pruebas
e2e, que arrancan `AppModule` completo, siguieran levantando.

**El adaptador stub (`stub-ai.adapter.ts`) se eliminó, no se dejó como
alternativa.** Mismo criterio que P4.5 aplicó a
`StubWaitlistResponseAdapter` al reemplazarlo por el adaptador real: una
vez que existe una implementación real y funcional del puerto, mantener el
stub sin ningún punto que lo siga usando es código muerto.

## Alternativas descartadas

- **Claude Sonnet vía `@anthropic-ai/sdk`**: primera lectura de la tarea,
  descartada por la usuaria a mitad de la implementación en favor de
  GPT-4o mini, la decisión efectivamente tomada por el equipo según el
  documento de evaluación y una discusión previa no accesible para el
  agente. Ver más arriba.
- **Dejar el reintento en manos del `maxRetries` por defecto del SDK**:
  descartada por las razones de observabilidad y testeabilidad detalladas
  arriba.
- **Reexportar los tipos de un SDK de proveedor concreto en la firma de
  `AIPort`**: descartada por acoplar el dominio al proveedor concreto,
  contra el principio ya establecido para el resto de los puertos — y
  validada por el propio cambio de proveedor a mitad de tarea.
- **Exponer la política de reintento (tope de intentos, backoff) como
  variables de entorno**: descartada porque el propio ticket fija esa
  política de forma explícita ("máximo 3 intentos"), a diferencia del
  modelo o el timeout, que sí declara configurables.
- **Resolver `OPENAI_API_KEY` de forma perezosa, recién en el primer
  llamado**: descartada en favor de *fail-fast* en el arranque, mismo
  criterio que `JWT_SECRET`.

## Entidades / puertos / adaptadores tocados

- `AIPort` (`src/domain/ports/ai.port.ts`): contrato ampliado —
  `processMessage(messages, tools?, system?)` → `AIResponse` con texto
  y/o `tool_use`.
- `AIProcessingError` (`src/domain/ports/ai.errors.ts`): error de dominio
  nuevo, no HTTP.
- `OpenAiAdapter` (`src/infrastructure/adapters/openai.adapter.ts`):
  implementación real del puerto.
- `openai-client.provider.ts`, `openai.constants.ts`, `openai.mappers.ts`
  (mismo directorio): construcción del cliente del SDK, constantes de
  configuración por defecto, y traducción de tipos.
- `stub-ai.adapter.ts`: eliminado.
- `IntegrationsModule`: `AI_PORT` pasa de `StubAiAdapter` a `OpenAiAdapter`.
- Sin cambios de esquema Prisma — este puerto no persiste nada.

## Tests y qué validan

- `openai.adapter.spec.ts` (nuevo): respuesta de sólo texto; lectura del
  modelo desde `OPENAI_MODEL`; el prompt de sistema antepuesto como mensaje
  `role: "system"`; respuesta de sólo `tool_use` y respuesta mixta texto +
  `tool_use`; error descriptivo ante argumentos de herramienta con JSON
  inválido; reintento ante un único `RateLimitError` (429) seguido de
  éxito; reintento con backoff exponencial ante dos `InternalServerError`
  (5xx) seguidos de éxito; agotamiento de los tres intentos ante 429
  persistente, con `AIProcessingError` descriptivo; ausencia de reintento
  ante un error no reintentable (400); y que ningún mensaje de log
  contiene la clave de API ni encabezados de autorización — sólo el patrón
  `status`/`name`/`message` del error. Usa temporizadores simulados de
  Jest para no esperar los backoffs reales.
- `openai.mappers.spec.ts` (nuevo): traducción de mensajes de texto plano;
  un turno de asistente con texto + `tool_use` queda en un único mensaje;
  un turno de usuario con varios `tool_result` se separa en varios mensajes
  `role: "tool"`, incluyendo uno marcado como error; error si un
  `tool_use` aparece indebidamente en un mensaje de usuario; traducción de
  herramientas al formato de función del SDK; traducción de la respuesta
  (texto, `text: null` ante una respuesta de sólo herramienta, los
  distintos `finish_reason`, incluyendo el caso por defecto para uno que
  este adaptador no debería ver nunca, y el error descriptivo ante
  argumentos con JSON inválido).
- `integrations.module.spec.ts` (modificado): el test de `AIPort` pasa a
  invocar `processMessage` con la nueva firma y a sustituir el cliente del
  SDK real (`OPENAI_CLIENT`) por un doble, en vez de resolver contra el
  adaptador stub — se agregó además `ConfigModule.forRoot({isGlobal:
  true})` a las importaciones del módulo de prueba, necesario porque
  `OpenAiAdapter` depende de `ConfigService`.
- Suite unitaria completa del repo en verde (42 suites, 489 tests) y suite
  e2e completa en verde (38 suites, 458 tests, corrida en serie: en
  paralelo aparecen conflictos de serialización 409 preexistentes, ajenos
  a este cambio, por condiciones de carrera entre archivos de prueba que
  comparten la misma base de Postgres), más `tsc --noEmit` y ESLint sin
  advertencias sobre los archivos tocados.

## Figuras pendientes

- Diagrama conceptual del ciclo de reintento del adaptador de IA (llamada
  → error 429/5xx → backoff exponencial → reintento, hasta tres intentos
  → error descriptivo final; con la rama de error no reintentable saliendo
  directo). Sección 3.2.5 Capa conversacional y WhatsApp.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-46-openai-ai-adapter`, creada desde
  `main` (renombrada desde `feature/TASK-46-anthropic-ai-adapter`, su
  nombre durante el primer intento sobre Claude; sin commits todavía en
  ese momento, así que renombrarla no perdía ningún historial). Sin commit
  ni push al momento de escribir esta entrada: pendiente de autorización
  explícita de la usuaria.
- Ticket: TASK-46 (Jira), "P5.1 – Adaptador de IA con Sonnet" — el título
  del ticket en el tablero sigue mencionando Sonnet; la usuaria fue
  consultada sobre actualizarlo para reflejar la decisión real (GPT-4o
  mini). Bajo el épico TASK-8 (Módulo 5 – Chatbot e IA conversacional,
  WhatsApp). Depende de TASK-13 (estructura hexagonal del proyecto,
  Fundaciones). Fuentes de la decisión de proveedor: documento "Evaluación
  de tecnologías de inteligencia artificial conversacional para el chatbot
  de la Secretaría Virtual" (insumo del Objetivo Específico 2) y una
  conversación previa del equipo con la herramienta de IA, no accesible
  para el agente en el momento de escribir esta entrada. Fuera de alcance
  y reservado a tareas futuras: la definición de las herramientas
  concretas del chatbot (P5.2, TASK-47) y el orquestador que arma el
  historial, el prompt de sistema por inquilino y ejecuta el ciclo de
  herramientas (P5.3, TASK-48).
