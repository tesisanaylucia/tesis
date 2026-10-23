# Fase 5 — Capa conversacional y WhatsApp (backend) — los mensajes que inicia el sistema se envían como plantillas de utilidad aprobadas (TASK-219, corrección a P5.8/P4.2)

## Contexto

Hallazgo de uso real sobre el entorno de desarrollo, reportado por la
usuaria: un turno reservado para las nueve de la mañana, correspondiente a
un paciente nuevo, nunca recibió el pedido de confirmación que debe salir
veinticuatro horas antes. Como el paciente no recibió nada, tampoco pudo
responder "SI", y en consecuencia el código de acceso —que se genera y se
envía recién cuando el turno pasa a confirmado (4.7)— no se generó ni se
envió. Los dos síntomas que la usuaria observó por separado resultaron ser
efectos de una sola causa.

La causa no estaba en los procesos programados ni en la condición de
paciente nuevo, que fueron las dos primeras hipótesis descartadas durante
el diagnóstico. La Cloud API de WhatsApp solo entrega un mensaje de texto
libre dentro de la llamada ventana de atención al cliente de veinticuatro
horas: las veinticuatro horas posteriores al último mensaje *entrante* del
paciente. Pasada esa ventana, la plataforma rechaza el envío y la única
forma de reabrir la conversación es una plantilla de utilidad previamente
aprobada. Todo mensaje que el sistema inicia por su cuenta nace fuera de esa
ventana por definición: el pedido de confirmación sale veinticuatro horas
antes del turno, cuando el paciente reservó días atrás y la conversación
quedó inactiva hacía mucho. Enviarlos como texto libre era, en los hechos,
garantizar que no llegaran.

El diagnóstico se apoyó en una evidencia que estaba en el propio código: la
operación de envío por plantilla del puerto de mensajería, y la constante
que fija el idioma con que esas plantillas se registran, existían desde la
integración inicial de la Cloud API (P5.8) —el comentario de esa constante
incluso nombra la plantilla de confirmación de turno como ejemplo— pero no
tenían un solo llamador en todo el proyecto. La capacidad se había
construido y nunca se había conectado; los seis flujos que la necesitaban
usaban, todos, la operación de texto libre. El modo de falla era además
silencioso por diseño previo: el rechazo de la plataforma se registraba
como advertencia y revertía la marca de idempotencia (TUR-10), de manera
que el turno quedaba reservado y el envío se reintentaba, lo cual evitaba
la consecuencia destructiva —la cancelación automática por falta de
respuesta— pero no hacía que el mensaje llegara nunca.

## Qué se implementó

El envío de todo mensaje iniciado por el sistema pasa a intentar primero la
plantilla aprobada y a recurrir al texto libre solo si esa vía falla.

- **Catálogo de plantillas** (`src/notifications/whatsapp-template.constants.ts`,
  nuevo): declara, para cada clave de notificación que viaja al paciente, el
  nombre con el que la plantilla queda registrada en la plataforma y el
  orden posicional que toman sus parámetros. Incluye los auxiliares que
  derivan la fila de configuración por inquilino y la variable de entorno
  correspondientes a cada clave, y el tipo de las claves entregables, que se
  deriva del propio catálogo en lugar de mantenerse como una segunda lista.
- **Servicio de entrega** (`src/notifications/notification-delivery.service.ts`,
  nuevo): único punto de salida de un mensaje iniciado por el sistema.
  Resuelve el nombre de la plantilla, valida que estén presentes los
  parámetros que esa plantilla declara, los reordena a las posiciones
  esperadas, envía la plantilla y, si eso falla, renderiza el texto libre
  con el motor de plantillas de mensajes (4.5) y lo envía por la vía
  anterior.
- **Los seis flujos proactivos** pasan por ese servicio: el pedido de
  confirmación de veinticuatro horas, el recordatorio configurable, el
  envío del PIN del código de acceso, el aviso de cancelación, el aviso de
  reprogramación junto con su pedido de confirmación, la oferta de la lista
  de espera y el envío del PIN ad-hoc.
- **Cambio de contrato interno**: el método que notifica al paciente desde
  el motor de turnos pasa a recibir la clave de plantilla y sus parámetros
  en lugar de un texto ya renderizado.
- **Exclusiones deliberadas**: la solicitud de consentimiento se mantiene
  como texto libre, y los tres avisos in-app del profesional no pasan por
  WhatsApp en absoluto.
- **Documentación operativa**: la aprobación de las ocho plantillas en la
  plataforma pasa a ser un prerrequisito de la marcha blanca, con su propio
  ítem de checklist y una prueba de aceptación que reproduce el caso del
  ticket.

## Decisiones y por qué

**Se intenta la plantilla primero y el texto libre queda como respaldo, en
lugar de reemplazar uno por el otro.** La plantilla es el único transporte
que entrega fuera de la ventana de veinticuatro horas, de modo que tiene
que ir primera. El texto libre, sin embargo, sigue siendo el que
efectivamente funciona mientras el paciente está dentro de la ventana —por
ejemplo el código de acceso que sale inmediatamente después de que el
paciente respondió— y es también el único que funciona en un entorno donde
las plantillas todavía no fueron aprobadas por la plataforma. Conectar las
plantillas sin ese respaldo habría dejado de funcionar todo lo que hasta
ahora funcionaba, incluidas las pruebas de la usuaria en desarrollo, hasta
que cada plantilla completara su aprobación.

**Un fallo de la plantilla se registra como advertencia y no se propaga,
pero un fallo del texto libre sí.** Que el nombre esté mal configurado o la
plantilla no esté aprobada es un problema operativo que el operador debe
ver en el registro de la aplicación, no una razón para no intentar el otro
transporte. Si ambos fallan, el error del segundo se propaga tal como lo
hacía antes de esta corrección, de modo que cada flujo conserva intacta la
decisión que ya tomaba al respecto: el pedido de confirmación revierte su
marca de idempotencia y el resto registra y continúa.

**El nombre de la plantilla se resuelve en tres niveles y no queda fijo en
el código**: la fila de configuración del inquilino, luego una variable de
entorno por mensaje, y finalmente el nombre por defecto que declara el
catálogo. Esa precedencia replica la que ya usa el motor de plantillas de
mensajes —lo propio del inquilino gana sobre lo del sistema— con un escalón
intermedio nuevo, cuya razón es que el nombre no lo elige este sistema sino
quien da de alta la plantilla en la plataforma: en una instalación de un
solo inquilino, como la de la Clínica, configurarlo por entorno es
notablemente menos ceremonia que escribir una fila de configuración por
cada mensaje, y en desarrollo permite apuntar a plantillas de prueba sin
alterar la configuración persistida. Un valor configurado que no sea una
cadena no vacía se ignora en lugar de confiarse, por el mismo motivo que ya
regía en el resto de las lecturas de configuración: el almacén guarda
documentos JSON arbitrarios y esa verificación es lo único que separa una
fila mal tipeada de un envío contra un nombre de plantilla inexistente.

**El orden posicional de los parámetros se declara explícitamente en vez de
derivarse del texto por defecto, aunque hoy ambos coincidan.** Las
plantillas de la plataforma toman parámetros posicionales, no placeholders
con nombre como los que usa el motor de plantillas de mensajes, y la
traducción entre los dos mundos es precisamente ese orden. Derivarlo del
texto en tiempo de ejecución sería incorrecto porque un inquilino puede
personalizar el texto libre y reordenar sus placeholders sin tocar la
plantilla aprobada, que es un objeto aparte, de forma fija y que vive en la
plataforma. Que hoy coincidan no es casualidad sino el contrato que debe
respetar quien registra la plantilla, y por eso se agregó una prueba que lo
fija: es el único mecanismo capaz de detectar esa divergencia, ya que un
orden desalineado no produce ningún error sino un mensaje con los valores
cruzados.

**El objeto de parámetros se reconstruye en el orden declarado antes de
entregarlo al adaptador.** El adaptador de la Cloud API posiciona los
parámetros según el orden de inserción del objeto que recibe, de modo que
depender del orden en que cada flujo armó el suyo habría sido un contrato
invisible y trivialmente fácil de romper al agregar un campo.

**La decisión de transporte se centralizó en un único servicio en lugar de
repetir el intento y el respaldo en cada flujo.** Son seis flujos
distribuidos en cuatro módulos; replicar la secuencia en cada uno los
habría dejado libres de divergir, el mismo motivo por el cual el envío del
PIN ya era un único lugar compartido entre la confirmación y la
reprogramación. Como consecuencia, el método que notifica al paciente dejó
de recibir texto renderizado: renderizar antes de la llamada era
exactamente lo que dejaba la decisión de transporte sin un lugar donde
vivir.

**La solicitud de consentimiento se mantiene como texto libre.** No es un
mensaje que el sistema inicie: se renderiza y se le devuelve al modelo como
texto, que sale después en la respuesta del bot a un mensaje entrante y por
lo tanto siempre dentro de la ventana de veinticuatro horas. Darle una
plantilla habría sugerido, falsamente, que puede entregarse de manera
proactiva.

## Alternativas descartadas

- **Enviar únicamente por plantilla, sin respaldo de texto libre.** Es la
  opción más fiel al modelo de la plataforma y la de código más simple, pero
  deja sin funcionar todos los mensajes proactivos hasta que las ocho
  plantillas estén aprobadas, incluido lo que la usuaria prueba en
  desarrollo. Se descartó por eso, y el respaldo quedó explícitamente
  documentado como transitorio en su propósito aunque permanente en el
  código, ya que sigue siendo el transporte correcto dentro de la ventana.
- **Corregir solo el pedido de confirmación**, que es el mensaje que el
  ticket nombra. Se descartó porque el código de acceso —la segunda mitad
  del síntoma que la usuaria reportó— seguiría sin llegar, de modo que el
  caso de prueba del ticket no pasaría de punta a punta, y porque los otros
  cinco flujos arrastran el mismo defecto: arreglar uno habría dejado la
  misma falla silenciosa en cinco lugares.
- **Derivar el orden de los parámetros del texto por defecto en tiempo de
  ejecución.** Descartada por lo dicho más arriba: un texto personalizado
  por el inquilino redefiniría en silencio el contrato posicional de una
  plantilla que no cambió.
- **Consultar el último mensaje entrante del paciente para decidir el
  transporte** y enviar texto libre cuando la ventana está abierta,
  reservando la plantilla para cuando está cerrada. Es la optimización
  correcta en términos de costo —una plantilla de utilidad se tarifa y un
  mensaje dentro de la ventana no— pero exige persistir la marca temporal
  del último mensaje entrante por paciente, que hoy no se registra, y el
  objeto de esta corrección era que los mensajes llegaran. Quedó registrada
  como trabajo pendiente.

## Entidades / puertos / adaptadores tocados

- `src/notifications/whatsapp-template.constants.ts` (nuevo): catálogo de
  datos puro, sin estado ni entrada/salida.
- `src/notifications/notification-delivery.service.ts` (nuevo): el servicio
  de entrega, ubicado junto al motor de plantillas de mensajes porque
  decide *cómo se entrega* un mensaje cuyo contenido define ese motor.
- `src/notifications/notifications.module.ts`: el módulo pasa a importar el
  de integraciones para poder resolver el puerto de mensajería, ya que dejó
  de solo decir qué dice un mensaje y ahora también decide cómo sale. No
  introduce ciclo: el módulo de integraciones no importa ningún módulo de
  funcionalidad.
- `src/appointments/appointment-confirmation.service.ts`,
  `src/appointments/appointment-reminder.cron.ts` y
  `src/appointments/appointments.service.ts`: los cuatro mensajes del motor
  de turnos y del planificador.
- `src/access-codes/access-code.service.ts` y
  `src/waitlist/waitlist-reassignment.service.ts`: el PIN ad-hoc y la oferta
  de la lista de espera.
- **El puerto de mensajería no cambió**: la operación de envío por
  plantilla ya formaba parte de su contrato. El adaptador de la Cloud API
  tampoco: su implementación de esa operación quedó tal como estaba desde
  P5.8, y esta tarea solo le dio llamadores.
- Ningún cambio de esquema ni migración. Ninguna ruta HTTP cambió de
  contrato.
- `docs/marcha-blanca-checklist.md` y el archivo de ejemplo de variables de
  entorno del repositorio: documentación operativa del prerrequisito.

## Tests y qué validan

- `src/notifications/notification-delivery.service.spec.ts` (nuevo): la
  decisión de transporte. Que la plantilla se envía y el texto libre no
  llega siquiera a renderizarse en el camino normal; que los parámetros se
  reordenan a las posiciones declaradas sin importar el orden en que los
  armó el llamador y que uno de más se descarta; los tres niveles de
  resolución del nombre, con sus casos de valor mal tipeado ignorado y de
  espacios recortados; el respaldo cuando la plataforma rechaza la
  plantilla; que una plantilla con un parámetro ausente no se envía; que el
  valor "0" cuenta como presente, por la misma razón que ya regía en el
  motor de plantillas de mensajes; y la propagación del error cuando ambos
  transportes fallan.
- `src/notifications/whatsapp-template.constants.spec.ts` (nuevo): los
  invariantes del catálogo. El central es que el orden posicional declarado
  coincide con el orden en que el texto por defecto nombra sus placeholders,
  para cada una de las ocho claves. Además, que cada nombre por defecto
  tiene la forma que la plataforma admite, que no hay nombres repetidos, que
  las ocho claves iniciadas por el sistema tienen entrada en el catálogo
  —con la lista escrita a mano a propósito, ya que derivarla del catálogo
  haría que la prueba no pudiera fallar— y que las cuatro claves excluidas
  siguen sin entrada.
- `src/appointments/appointment-confirmation.service.spec.ts`: reescrito
  alrededor de un servicio de entrega real, con sus dependencias simuladas,
  en lugar de un doble del envío. Verifica que el mensaje del ticket sale
  por la plantilla y no por texto libre, el respaldo cuando la plantilla
  falla, el criterio de P4.1 sobre informar la duración de la sesión en
  ambos transportes, y —el escenario propio del ticket— que cuando los dos
  transportes fallan la marca de idempotencia se revierte y no se escribe
  asiento de auditoría, de modo que las garantías de TUR-10 siguen en pie
  bajo el modo de falla nuevo.
- Las suites de los flujos llamadores se adaptaron al contrato nuevo. En
  los dos casos donde la suite ya verificaba qué plantilla se pedía y con
  qué parámetros, el doble del servicio de entrega delega en los mismos
  dobles que esas pruebas ya afirmaban, que es exactamente lo que hace el
  camino de respaldo real, de manera que las aserciones existentes
  conservan su intención.
- Suites de extremo a extremo con PostgreSQL real: las aserciones que
  verificaban el *contenido* del texto enviado pasaron a verificar el nombre
  de la plantilla y sus parámetros. El cambio no es cosmético sino
  conceptual: el texto que lee el paciente dejó de vivir en este sistema y
  pasó a vivir en la plantilla registrada en la plataforma, de modo que
  afirmar sobre él ya no verificaba nada de este backend.
- Resultado de la corrida completa: 1813 de 1817 pruebas en verde, con 39
  pruebas nuevas. Las cuatro fallas restantes son preexistentes y ajenas a
  esta corrección, verificadas como tales ejecutándolas sobre la rama base
  sin estos cambios. Corresponde señalar que la suite de extremo a extremo
  es inestable por compartir una única base de datos: el conjunto de fallas
  varía entre corridas y una de ellas depende de la hora del día —pasaba al
  comienzo de la sesión de trabajo y fallaba al final, también sin estos
  cambios—, por lo que la comparación se hizo contra la rama base y no
  contra una corrida anterior. Análisis estático y formato sin advertencias.
  Datos ficticios, sin contenido clínico ni datos personales reales.

## Figuras pendientes

Se registra una figura nueva (N° 59 tentativo) con el diagrama de secuencia
de la decisión de transporte. La figura conceptual del modelo de mensajería
de la plataforma —la ventana de veinticuatro horas y el disparo de
plantillas fuera de ella— ya estaba registrada para el Marco Teórico (N° 4)
desde antes de esta tarea y no se duplica: el texto de 4.6 la referencia.

## Trabajo pendiente que esta tarea deja registrado

Primero, y de orden operativo antes que de desarrollo: **la corrección no
surte efecto hasta que las ocho plantillas estén aprobadas en la
plataforma**, con el idioma correspondiente y con sus parámetros
posicionales en el orden que declara el catálogo. Mientras eso no ocurra, el
respaldo mantiene el comportamiento anterior, que solo entrega dentro de la
ventana de veinticuatro horas. El ítem de checklist agregado a la marcha
blanca incluye la prueba de aceptación que lo verifica.

Segundo, el transporte se elige sin consultar si la ventana está abierta, de
modo que se envía una plantilla incluso cuando un mensaje de texto libre
habría alcanzado. Es correcto en cuanto a la entrega pero no en cuanto al
costo, ya que una plantilla de utilidad se tarifa; optimizarlo exige
persistir la marca temporal del último mensaje entrante por paciente, que
hoy no se registra.

Tercero, sigue pendiente lo que ya había registrado la corrección anterior
de esta fase (TASK-214) y que esta tarea no aborda: los avisos automáticos
se envían **por fila** y no por sesión, de modo que la primera sesión de un
paciente nuevo todavía produce dos mensajes nombrando las nueve y las nueve
y media. Esta corrección cambia el transporte de esos mensajes, no su
cantidad, y corresponde a un ticket aparte.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-219-proactive-whatsapp-templates`
  (creada a partir de `development`).
- Ticket: TASK-219, hallazgo de uso real de la usuaria sobre el entorno de
  desarrollo ("tenía un turno para hoy a las 9 de paciente nuevo, y 24 h
  antes no llegó el recordatorio para confirmar el turno y que luego llegue
  el código"), corrección a la integración de la Cloud API de P5.8
  (TASK-53) tal como la consumen el pedido de confirmación de P4.2
  (TASK-43) y los demás avisos de la fase 4.
- Nota de proceso: el servidor de integración con el gestor de incidencias
  no estuvo disponible durante esta sesión, de modo que el enunciado del
  ticket no pudo leerse y el trabajo se guió por el reporte de la usuaria y
  por el diagnóstico sobre el código.
