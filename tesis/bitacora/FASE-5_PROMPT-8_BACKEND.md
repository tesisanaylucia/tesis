# Fase 5 — Capa conversacional y WhatsApp (backend) — adaptador de WhatsApp Cloud API (TASK-53)

## Qué se implementó

Se implementó el adaptador real del `MessagingPort` contra la WhatsApp
Business Cloud API de Meta —reemplazando al adaptador *stub* que existía
desde la fase de Fundaciones— y el webhook HTTP que recibe los mensajes
entrantes de los pacientes: verificación del webhook ante Meta, validación
de la firma de cada mensaje recibido, resolución del inquilino a partir del
número que lo recibió, y el llamado al orquestador de conversación que
todas las tareas anteriores del Módulo 5 dejaron listo pero sin ningún
punto de entrada HTTP real.

## Decisiones y por qué

**El webhook vive en un módulo propio, no dentro de `ChatbotModule`.**
`OrquestadorService.procesar` es el único punto de entrada que el
orquestador necesita, y todas las tareas anteriores del módulo lo dejaron
comentado como "sin controlador propio, hasta que TASK-53 lo agregue". Se
optó por no agregarle un controlador directamente sino crear
`WhatsappModule`, que importa `ChatbotModule` (por el orquestador) e
`IntegrationsModule` (por el puerto de mensajería), en vez de invertir la
dependencia. La razón es que `ChatbotModule` es, en principio, agnóstico
del canal —el orquestador no sabe que el mensaje vino de WhatsApp— y un
canal futuro (o una prueba de conversación extremo a extremo, como las que
ya existían) no debería arrastrar un servidor HTTP para poder ejercitarlo.

**Las credenciales de envío son configuración global y no por inquilino,**
pese a que el propio ticket describe el mapeo número→organización como "la
clave del multi-tenant en el canal de WhatsApp". La razón es que la firma
de `MessagingPort.sendMessage(destinatario, mensaje)` —ya fijada por
TASK-46 y usada sin cambios por cinco llamadores distintos— no recibe
ningún identificador de número de origen, y agregarle uno habría roto el
contrato que consume media base de código. El mismo criterio ya regía para
`OpenAiAdapter`: modelo y clave son configuración global, no por
inquilino, porque la entrega inicial es de una sola clínica. La tabla de
mapeo que sí exige el ticket cumple entonces una única función, asimétrica
de la anterior: no decide con qué credencial se envía, sino a qué
organización pertenece un mensaje *entrante*, que es el problema real que
un número compartido por varios inquilinos plantearía.

**La tabla de mapeo se lee con el cliente de Prisma sin acotar por
inquilino, deliberadamente.** Es el mismo caso que ya documentaba
`UsersService.findByEmail`: la propia fila que hay que leer es la que
determina qué organización abrir, así que no puede exigirse un contexto de
inquilino que todavía no existe. Es seguro sin acotar porque
`phoneNumberId` es única a nivel de toda la base, no por organización —no
existe ninguna otra fila de otro inquilino con la que pudiera confundirse.

**La firma del webhook se valida contra los bytes crudos del cuerpo, no
contra el cuerpo ya interpretado como JSON.** Meta firma exactamente los
bytes que envía; volver a serializar el cuerpo que Nest ya parseó
(`JSON.stringify(req.body)`) podría producir una cadena distinta byte a
byte —orden de claves, espacios, escape de acentos— y aceptar una firma
que Meta nunca calculó sobre ese contenido. Por eso se habilitó la opción
`rawBody` de Nest a nivel de toda la aplicación, que conserva el cuerpo
crudo junto al ya parseado sin afectar ninguna otra ruta, y la comparación
de la firma se hace con `timingSafeEqual` en lugar de una comparación de
cadenas común, para no filtrar el resumen verdadero un byte a la vez por
el tiempo de respuesta.

**Un mensaje válidamente firmado para un número sin mapeo no es un error
HTTP.** Se responde 200 igual que un mensaje procesado con éxito, en lugar
de 404 o 500. La razón es doble: por un lado es una falta de configuración
del operador (un número todavía no vinculado a ningún inquilino) y no un
intento malicioso ni un error del cliente; por otro, Meta reintenta un
webhook que no responde 200, y reintentar el mismo mensaje no lo va a
volver mapeable. Lo mismo aplica a las notificaciones de estado de entrega
(`value.statuses` en lugar de `value.messages`), que Meta envía por el
mismo webhook: se reconocen y se descartan sin invocar al orquestador,
también con 200.

**El adaptador de envío no oculta sus propias fallas detrás de las de sus
llamadores.** Los cinco lugares que ya llamaban a `MessagingPort.sendMessage`
—los dos recordatorios programados, dos flujos de `AppointmentsService` y
la oferta de lista de espera— nunca envolvían el llamado en un `try/catch`,
porque contra el adaptador *stub* anterior no había nada que fallara. Al
conectar un adaptador real, uno de esos llamadores —el pedido de
confirmación de reprogramación dentro de una prueba de conversación
extremo a extremo ya existente— pasó a fallar de verdad, porque en el
entorno de prueba la credencial configurada no es una válida. La prueba
en cuestión (`chatbot-flows.e2e-spec.ts`) no ejercita el envío en sí, así
que se corrigió reemplazando el puerto de mensajería real por uno simulado
en esa prueba puntual, con el mismo mecanismo que ya usaban las pruebas de
reprogramación y de reasignación por lista de espera. Quedó registrado,
sin corregirse en esta tarea por quedar fuera de su alcance declarado, que
ninguno de los cinco llamadores atrapa hoy una falla real de entrega —una
caída de la API de WhatsApp interrumpiría, por ejemplo, un recordatorio
programado completo en lugar de continuar con el resto de los turnos de la
organización.

**Se corrigió, al validar, un defecto real y anterior a esta tarea en una
migración de datos ya fusionada a `main`.** La migración que siembra el
*system prompt* y el tiempo de espera del orquestador
(`20260821050000_seed_chatbot_config`, de TASK-48) usa
`to_jsonb(E'texto largo')` sin convertir antes el literal a `text`;
PostgreSQL no puede resolver el tipo polimórfico de un literal sin tipo en
ese contexto y la migración fallaba siempre, en cualquier base nueva,
desde que se escribió. No se detectó antes porque ninguna sesión anterior
había corrido las migraciones contra una base completamente vacía. Se
corrigió agregando el *cast* explícito (`::text`) al literal, sin cambiar
el contenido sembrado.

## Entidades, puertos y adaptadores tocados

- Modelo nuevo `WhatsappPhoneNumber` (acotado por organización, patrón 1
  del esquema): mapea el `phoneNumberId` de Meta —único en toda la base—
  a la organización que lo recibe. Migración aditiva, sin tocar ningún
  modelo existente.
- `WhatsAppCloudAdapter` (`src/infrastructure/adapters/`), implementación
  real de `MessagingPort` contra `POST /v19.0/{phoneNumberId}/messages` de
  la API Graph de Meta, reemplazando a `StubMessagingAdapter` (eliminado,
  mismo tratamiento que recibió el *stub* de `AIPort` al conectarse
  `OpenAiAdapter`). `sendMessage` envía un mensaje de texto; `sendTemplate`
  arma una plantilla de utilidad con parámetros posicionales.
- `WhatsappModule` nuevo (`src/whatsapp/`): `WhatsappWebhookController`
  (`GET`/`POST /webhook`, ambas rutas públicas) y
  `WhatsappPhoneNumbersService`, el servicio que resuelve la organización a
  partir del número receptor sin pasar por el cliente acotado por
  inquilino.
- `IntegrationsModule`: vinculación de `MESSAGING_PORT` al adaptador real,
  más un proveedor nuevo para el `fetch` inyectable con el que se prueba el
  adaptador sin red real.
- `main.ts`: habilitación de `rawBody` a nivel de toda la aplicación.

## Tests

- Pruebas unitarias del adaptador: formato exacto del cuerpo enviado para
  un mensaje de texto y para una plantilla con y sin parámetros, y que una
  respuesta de error de la API se traduce en una excepción propia sin que
  el token de acceso aparezca en ningún mensaje de error.
- Pruebas unitarias de la validación de firma: acepta la firma calculada
  con la misma clave sobre los mismos bytes, rechaza clave distinta,
  cuerpo alterado, encabezado ausente y encabezado mal formado.
- Pruebas unitarias de la extracción del mensaje entrante del payload de
  Meta: mensaje de texto válido, notificación de estado sin mensajes,
  mensaje no textual y payload incompleto.
- Pruebas unitarias del servicio de resolución de organización por número.
- Pruebas de integración extremo a extremo del webhook contra HTTP real,
  con el orquestador y el resto de la aplicación reales y sólo el puerto
  de IA simulado (mismo criterio que las demás pruebas de conversación) y
  el puerto de mensajería simulado para no depender de la red: los seis
  criterios de aceptación del ticket (verificación con token correcto y
  con token incorrecto, mensaje con firma inválida, mensaje válido que
  efectivamente invoca al orquestador y responde por `MessagingPort`,
  número sin mapeo y notificación de estado, ambos reconocidos sin invocar
  al orquestador).

Suite completa en verde al cierre: 669 pruebas unitarias y 491 de
integración.

## Figuras pendientes

- Diagrama de secuencia del webhook de mensajes entrantes (mensaje de
  WhatsApp → validación de la firma HMAC-SHA256 contra el cuerpo crudo →
  extracción del mensaje de texto → resolución de la organización por el
  número receptor → `OrquestadorService.procesar` → respuesta enviada por
  `MessagingPort.sendMessage`), con la rama de firma inválida y la rama de
  número sin mapeo. Sección 4.6 Capa conversacional y WhatsApp.
- Diagrama entidad-relación de `WhatsappPhoneNumber`, señalando que el
  campo que se busca (`phoneNumberId`) es único en toda la base y no por
  organización, a diferencia del resto de los modelos acotados por
  inquilino del proyecto. Sección 4.6 Capa conversacional y WhatsApp.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-53-whatsapp-cloud-adapter`, creada
  desde `origin/main` como pide la usuaria.
- Ticket: TASK-53 (Jira), "P5.8 – Adaptador de WhatsApp Cloud API", bajo el
  épico TASK-8 (Módulo 5).
- Dependencias declaradas: TASK-48 (P5.3, orquestador) y TASK-46 (P5.1,
  interfaz `MessagingPort`), ambas ya fusionadas antes de esta sesión.
- Fuera de alcance, declarado en el propio ticket y respetado: no se envía
  el código de acceso TTLock en el recordatorio, que se agrega recién en
  M6/P6.3 (TASK-57) cuando la entidad de código de acceso exista.
- Corrección de un defecto anterior encontrado al validar, fuera del
  alcance literal del ticket pero necesaria para poder correr las
  migraciones contra una base nueva: el *cast* de tipo faltante en la
  migración de siembra del *system prompt* de TASK-48
  (`20260821050000_seed_chatbot_config`).
