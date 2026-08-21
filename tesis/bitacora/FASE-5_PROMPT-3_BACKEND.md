# Fase 5 — Capa conversacional y WhatsApp (backend) — orquestador de conversación y system prompt (TASK-48)

## Qué se implementó

Se implementó `OrquestadorService.procesar(sessionId, mensajeEntrante,
organizationId)`, la pieza que conduce un turno completo de la conversación
del chatbot: recupera el historial de la sesión, arma la instrucción de
sistema del inquilino, llama a `AIPort` con el catálogo completo de
herramientas (P5.2, TASK-47), ejecuta cada herramienta que el modelo pida,
le devuelve el resultado y repite el ciclo hasta que el modelo responde con
texto plano o hasta un tope de diez iteraciones. Incluye el system prompt
—rol, tono y límites del bot— y su mecanismo de personalización por
inquilino, el historial de conversación en memoria con expiración por
inactividad, y la composición del identificador de sesión a partir del
celular del paciente y el inquilino.

## Decisiones y por qué

**El orquestador abre el contexto de inquilino una única vez por turno,
igual que un request HTTP lo abre una vez por petición.** `procesar` recibe
`organizationId` como parámetro explícito —tal como pide la firma del
ticket— y lo primero que hace es envolver todo el turno en
`TenantContextService.run(organizationId, …)`, exactamente el mismo
mecanismo que `TenantContextInterceptor` usa para una petición HTTP común.
Esto no es una decisión nueva de esta tarea: el comentario de
`resolveSystemActor`, escrito durante TASK-47, ya documentaba que "el futuro
orquestador (TASK-48) lo abre una vez por turno de conversación" como el
caso simétrico de un cron que recorre organizaciones desde afuera de
cualquier contexto. Esta tarea sólo cumplió esa promesa: con el contexto
abierto una vez al principio, tanto la resolución de configuración del
inquilino (`ChatbotConfigService`) como cada herramienta que el modelo
invoca durante el resto del turno leen a través del cliente de Prisma ya
acotado, sin que ninguna de las dos reciba ni necesite un identificador de
organización propio.

**El ciclo de llamado a herramientas pasa una copia del historial a
`AIPort` en cada vuelta, no el arreglo mutable en construcción.** El
historial de la conversación se arma como un único arreglo que crece con
cada vuelta del ciclo (el mensaje del usuario, el `tool_use` del asistente,
el `tool_result` de la herramienta, y así sucesivamente). Pasar ese mismo
arreglo por referencia a `AIPort.processMessage` funciona en la práctica
—el adaptador real serializa el pedido antes de devolver el control—, pero
deja una trampa para cualquier adaptador futuro que retenga la referencia
más allá de la llamada (por ejemplo, para loguear el pedido de forma
asíncrona): vería un historial que siguió cambiando después de la llamada
que en teoría ya se resolvió. Se optó por pasar una copia superficial
(`[...messages]`) en cada llamada, un costo mínimo que elimina esa clase de
error por completo. La necesidad de esto se descubrió al escribir el test
de reserva completa: las aserciones sobre los argumentos de la segunda
llamada al mock de `AIPort` fallaban con datos de vueltas posteriores del
ciclo, hasta que se advirtió que Jest conserva la *referencia* al arreglo
pasado en cada llamada, no una copia congelada en ese momento — el mismo
comportamiento que tendría cualquier otro consumidor externo del arreglo.

**El historial de conversación vive en memoria, acotado por sesión, con
expiración perezosa en la lectura en vez de un trabajo programado.**
`ConversationSessionStore` es un `Map` en memoria de alcance de proceso — el
stack del proyecto no declara ninguna infraestructura de caché además de
Postgres, y el despliegue previsto para el piloto es de una única
instancia. La expiración no corre como un cron que recorra las sesiones
periódicamente: cada lectura del historial compara la marca de tiempo de la
última actividad contra el umbral vigente del inquilino y, si ya venció,
borra la entrada ahí mismo antes de devolver un historial vacío. Es el mismo
criterio que ya aplica `PatientInactivityService.threshold()` (documentado
en `CLAUDE.md`: "no hay cron, la evaluación de la regla se hace como
consulta al leer") extendido a un caso nuevo: una sesión que nadie vuelve a
leer después de quedar inactiva simplemente no se libera hasta que el
proceso se reinicia, un costo aceptado a la escala del piloto y coherente
con la razón ya dada para el resto del sistema.

**El umbral de inactividad de la sesión y el texto base del system prompt
son configurables por inquilino, con el mismo mecanismo ya usado por el
resto del sistema para reglas de negocio.** El ticket pide explícitamente
que el system prompt sea "configurable por tenant" y da el umbral de
inactividad como "configurable, default 30 min" sin especificar si es una
configuración global o por inquilino; se resolvió tratarlo también como
dato por inquilino, siguiendo la instrucción general de CLAUDE.md de que las
reglas de negocio (incluidos plazos y ventanas de tiempo) viven en
`OrganizationConfig`, no en condicionales del código — el mismo criterio ya
aplicado a `patient_inactivity_months` y a las ventanas de recordatorio y
cancelación de turno. `ChatbotConfigService` agrupa la resolución de ambos
valores: `sessionTimeoutMinutes()` reutiliza el nuevo método genérico
`ConfigTenantService.getPositiveInteger` (ver más abajo), y `systemPrompt()`
sigue exactamente el mismo patrón de reemplazo que ya usa
`NotificationTemplateService.resolveTemplate` para las plantillas de
notificación: el texto propio del inquilino si lo personalizó, el texto base
del módulo si no. El nombre de la organización se interpola en el texto
resultante con el mismo mecanismo `{placeholder}` que ya usan las plantillas
de notificación para `{patientName}`/`{professionalName}`/etc., de modo que
un inquilino que sobrescribe el texto puede reusar `{organizationName}` en
su propia versión. Ambos valores se sembraron además en una migración de
datos —no sólo en el seed— para toda organización existente, replicando
exactamente la razón ya documentada para `patient_inactivity_months` y las
plantillas de notificación: el seed es dato de desarrollo y no corre en un
entorno real, así que sin la migración la fila editable que el requisito de
"configurable por tenant" promete no existiría todavía en ninguna
organización productiva.

**Se extrajo `ConfigTenantService.getPositiveInteger` en vez de repetir por
quinta vez el mismo patrón de lectura de un umbral numérico.** Antes de esta
tarea, cuatro lugares del código (`PatientInactivityService.threshold`,
`AppointmentsService.cancellationMinHours`/`.minimumAge`,
`AppointmentReminderCron.reminderWindowHours`) repetían, con comentarios
casi idénticos, la misma comprobación: leer la clave, aceptar el valor sólo
si es un entero positivo, devolver una constante de reserva en cualquier
otro caso. `sessionTimeoutMinutes()` necesitaba exactamente esa misma
lógica, así que en vez de sumar una quinta copia se generalizó a un método
en `ConfigTenantService` — el lugar donde ya vive el resto de la lectura de
configuración por inquilino —, dejando a los cuatro sitios preexistentes sin
tocar (una migración de esos cuatro a reusar el método nuevo es un cambio de
otro alcance, no de esta tarea) pero disponible para cualquier regla nueva
de esta forma.

**El texto base del system prompt y las utilidades de interpolación de
plantillas se extrajeron a `src/common/templates/text-template.ts`,
compartidas con `NotificationTemplateService`.** El mecanismo de
`{placeholder}` que las plantillas de notificación ya tenían
(`notification-template.renderer.ts`) es exactamente el que el system prompt
necesitaba para `{organizationName}`; en vez de duplicar la misma expresión
regular y las mismas dos funciones puras en el módulo del chatbot, se
trasladaron a una ubicación compartida bajo `src/common/`, y
`notification-template.service.ts` pasó a importarlas desde ahí. El cambio
es mecánico —mismas funciones, mismos tests, sólo de ubicación— y no altera
ningún comportamiento de P4.1.

**El límite de diez iteraciones del ciclo de herramientas se implementa
como un tope estricto del propio orquestador, no como configuración por
inquilino.** A diferencia del umbral de inactividad de sesión, el ticket da
este número sin calificarlo de "configurable": es un resguardo contra un
modelo que entra en un ciclo de invocación de herramientas sin fin, no una
regla de negocio que un inquilino de marca blanca tuviera razón legítima
para querer distinta — mismo criterio que ya separa los topes anti-abuso
(constantes) de las reglas de negocio (`OrganizationConfig`) en el resto del
código. Superado el tope, el orquestador lanza `ToolCallLimitExceededError`
—un `Error` de dominio, no una `HttpException`, mismo criterio que
`AIProcessingError`— en vez de devolver algún texto de disculpa armado a
mano: este servicio no está detrás de un controlador propio todavía (lo
estará recién con el webhook de WhatsApp de TASK-53), así que decidir qué
lee el paciente ante este error es una decisión de esa tarea futura, no de
ésta. El historial de la sesión deliberadamente no se guarda cuando esto
ocurre: el turno atascado no produjo nada que un paciente debiera leer, así
que el próximo mensaje arranca desde el último turno exitoso en vez de
repetir el ciclo trabado.

## Alternativas descartadas

- **Pasar el arreglo `messages` en construcción directamente a `AIPort` en
  cada vuelta del ciclo**: funciona con el adaptador real actual, pero deja
  una dependencia implícita de que ningún adaptador retenga la referencia
  más allá de la llamada; descartada en favor de una copia superficial por
  vuelta, según lo explicado arriba.
- **Umbral de inactividad de sesión como variable de entorno global, no por
  inquilino**: el ticket no lo pide explícitamente de una forma u otra;
  descartada en favor de tratarlo como dato por inquilino, siguiendo la
  instrucción general de CLAUDE.md sobre reglas de negocio como
  `OrganizationConfig`, no como constante de código.
- **Expirar el historial de conversación con un trabajo programado
  periódico**: descartada en favor de la expiración perezosa en la lectura,
  mismo criterio que `PatientInactivityService.threshold()`.
- **Repetir la lectura de umbral numérico con reserva a mano en
  `ChatbotConfigService`, como ya la repiten otros cuatro lugares del
  código**: descartada en favor de extraer `ConfigTenantService.
  getPositiveInteger`, generalizando el patrón en el lugar que ya concentra
  la lectura de configuración por inquilino.
- **Devolver algún texto de disculpa al paciente cuando se agota el tope de
  iteraciones, en vez de lanzar una excepción**: descartada porque decidir
  qué lee el paciente ante un error es una decisión del futuro controlador
  (TASK-53, fuera de alcance de esta tarea), no de un servicio que hoy no
  está detrás de ningún endpoint.

## Entidades / puertos / adaptadores tocados

- `src/chatbot/orquestador.service.ts` (nuevo): `OrquestadorService`, el
  ciclo de llamado a herramientas.
- `src/chatbot/chatbot-config.service.ts` (nuevo): `ChatbotConfigService`,
  resolución del umbral de inactividad y del system prompt por inquilino.
- `src/chatbot/conversation-session.store.ts` (nuevo):
  `ConversationSessionStore`, historial en memoria con expiración perezosa.
- `src/chatbot/session-id.ts` (nuevo): `buildSessionId(organizationId,
  mobilePhone)`, la composición de la llave de sesión que usará el futuro
  webhook de WhatsApp (TASK-53).
- `src/chatbot/chatbot.constants.ts` (nuevo): claves de configuración,
  valores por defecto y texto base del system prompt.
- `src/chatbot/chatbot.errors.ts` (nuevo): `ToolCallLimitExceededError`,
  `UnknownOrganizationError`.
- `src/chatbot/chatbot.module.ts` (modificado): agrega los cuatro
  proveedores nuevos, importa `IntegrationsModule` (por `AI_PORT`), exporta
  `OrquestadorService`.
- `src/config-tenant/config-tenant.service.ts` (modificado): nuevo método
  `getPositiveInteger(key, fallback)`.
- `src/common/templates/text-template.ts` (nuevo, trasladado desde
  `src/notifications/notification-template.renderer.ts`): `interpolate`,
  `placeholdersIn`.
- `src/notifications/notification-template.service.ts`,
  `notification-template.constants.ts` (modificados sólo en el import, tras
  el traslado anterior).
- `prisma/migrations/20260821050000_seed_chatbot_config/` (nueva): siembra
  `chatbot_session_timeout_minutes` (30) y `chatbot_system_prompt` (texto
  base con `{organizationName}` literal) para toda organización existente.
- `prisma/seed.ts` (modificado): agrega las dos mismas claves a
  `TENANT_CONFIG`, para que una organización creada después de la migración
  las reciba igual.
- Ningún puerto de dominio nuevo ni cambio de `AIPort` — el orquestador
  consume el puerto tal como TASK-46 lo dejó, incluido el parámetro `system`
  agregado en esa tarea previendo exactamente este uso.

## Tests y qué validan

- `orquestador.service.spec.ts` (nuevo, el foco de los cuatro criterios de
  aceptación del ticket): reserva completa resuelta en tres vueltas del
  ciclo (`check_availability` → `book_appointment` → texto final), con
  aserciones sobre el orden y los argumentos exactos de cada llamada a la
  herramienta y sobre la forma completa del historial que la segunda
  llamada a `AIPort` recibe (el `tool_use` y su `tool_result`
  correspondiente, ya interpolado a JSON); una herramienta fallida se
  traduce en un `tool_result` marcado como error, nunca en una excepción
  que corte el turno; el segundo mensaje de una misma sesión recibe en su
  historial el turno anterior completo; el historial se descarta una vez
  vencido el umbral de inactividad simulado con temporizadores falsos de
  Jest, tanto con el umbral por defecto como con uno propio del inquilino;
  agotadas las diez vueltas sin una respuesta de sólo texto, el orquestador
  lanza `ToolCallLimitExceededError` y no persiste nada de ese turno; el
  contexto de inquilino está abierto tanto durante la resolución de
  configuración como durante cada llamada a herramienta, y no queda abierto
  una vez terminado el turno.
- `chatbot-config.service.spec.ts` (nuevo): el umbral delega en
  `ConfigTenantService.getPositiveInteger` con la clave y el valor por
  defecto correctos; el system prompt interpola el nombre de la
  organización sobre el texto base cuando el inquilino no lo personalizó,
  usa el texto propio del inquilino cuando sí lo hizo, y vuelve al texto
  base ante un valor en blanco; lanza `UnknownOrganizationError` si no hay
  contexto de inquilino abierto o si el identificador no resuelve ninguna
  fila.
- `conversation-session.store.spec.ts` (nuevo): sin historial para una
  sesión nunca guardada; historial intacto dentro del umbral; historial
  vacío y la entrada efectivamente eliminada del mapa una vez vencido el
  umbral simulado; dos sesiones no se pisan entre sí; cada `save` refresca
  el reloj de actividad de esa sesión.
- `session-id.spec.ts` (nuevo): combina organización y celular con un
  separador; dos organizaciones con el mismo celular, o un mismo
  organizationId con dos celulares, producen identificadores distintos.
- `config-tenant.service.spec.ts` (ampliado): `getPositiveInteger` devuelve
  el valor configurado cuando es un entero positivo, y el valor por defecto
  ante clave ausente, cero, negativo, no entero o no numérico
  (`it.each`).
- `text-template.spec.ts` (trasladado sin cambios desde
  `notification-template.renderer.spec.ts`): mismas pruebas de
  `placeholdersIn`/`interpolate` que ya existían para P4.1, ahora sobre la
  ubicación compartida.
- Suite unitaria completa del repo en verde (59 suites, 569 tests), más
  `tsc --noEmit` y ESLint sin advertencias sobre el árbol completo
  (`src/`, `prisma/`) — no sólo sobre los archivos tocados. No se corrió la
  suite e2e ni se aplicó la migración nueva contra una base real: Docker
  Desktop no estaba disponible en el entorno de esta sesión (mismo
  impedimento que ya registra la nota de memoria del proyecto), así que la
  migración se verificó por inspección — comparación programática de que el
  texto embebido en el SQL coincide carácter por carácter con la constante
  de TypeScript, y el mismo patrón ya ejecutado con éxito por las dos
  migraciones de siembra anteriores (`patient_inactivity_months`,
  plantillas de notificación) — en vez de ejecución real. Queda pendiente
  correr `npx prisma migrate deploy` (o `dev`) contra una base local la
  próxima vez que Docker esté disponible.

## Figuras pendientes

- Diagrama de secuencia del ciclo de llamado a herramientas de un turno de
  conversación (mensaje del paciente → apertura del contexto de inquilino →
  historial + system prompt del inquilino → `AIPort.processMessage` →
  ¿`tool_use`? → ejecución de la herramienta → `tool_result` agregado al
  historial → nueva llamada a `AIPort`, repetido hasta texto plano o hasta
  el tope de diez vueltas). Sección 3.2.5 Capa conversacional y WhatsApp.
- Diagrama de estados de una sesión de conversación en
  `ConversationSessionStore` (sin historial → historial activo tras el
  primer mensaje → refrescada en cada mensaje nuevo → expirada y eliminada
  en la primera lectura posterior al umbral de inactividad del inquilino).
  Sección 3.2.5.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-48-orquestador-conversacion`, creada
  desde `main` (TASK-46 y TASK-47 ya estaban mergeadas a `main` al momento
  de iniciar esta tarea, así que no hizo falta ramificar sobre ninguna otra
  rama de trabajo en curso, a diferencia de TASK-47). Sin commit ni push al
  momento de escribir esta entrada: pendiente de autorización explícita de
  la usuaria. Nota de proceso: la rama se creó recién a mitad de la
  implementación en esta sesión —el trabajo había arrancado por error sobre
  `main`—, corregido en cuanto se advirtió, antes de cualquier commit.
- Ticket: TASK-48 (Jira), "P5.3 – Orquestador de conversación + system
  prompt", bajo el épico TASK-8 (Módulo 5 – Chatbot e IA conversacional,
  WhatsApp). Fuentes de verdad consultadas en Drive: `arquitectura_backend.png`
  ("Capa conversacional (IA) → Motor de IA conversacional / PLN / Orquestador
  de diálogo — conduce la conversación") y el documento de Especificación de
  Requisitos de Software (sección del módulo Chatbot, "el chatbot gestiona
  la conversación con el paciente"); ninguno de los dos agregó un requisito
  no ya presente en el propio texto del ticket. Dependencias: TASK-46 (P5.1,
  AIPort), TASK-47 (P5.2, catálogo de herramientas). Fuera de alcance y
  reservado a tareas futuras: los guardrails sobre la respuesta final del
  modelo (P5.4, TASK-49); los flujos de negocio específicos por sobre el
  ciclo genérico de herramientas (P5.5, TASK-50); la integración real con
  WhatsApp, incluido quién construye el `sessionId` con `buildSessionId` y
  quién atrapa `ToolCallLimitExceededError` (P5.8, TASK-53).
