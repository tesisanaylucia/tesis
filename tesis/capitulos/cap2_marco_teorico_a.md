# Capítulo 2: Marco Teórico

> Borrador. Capítulo conceptual: cada tecnología o concepto se describe en
> términos generales, independientemente de cómo se haya utilizado en
> PSIQUE. Las decisiones concretas de diseño e implementación se abordan en
> el Capítulo 3 (Solución PSIQUE), que se apoya en los conceptos
> introducidos aquí sin repetir su explicación. Contenido dividido en dos
> archivos: este cubre 2.1 a 2.4; `cap2_marco_teorico_b.md` cubre 2.5 a 2.8.

Este capítulo introduce el conjunto de conceptos, tecnologías y marco
normativo sobre los que se apoya la solución descripta en el Capítulo 3.
Se organiza en dos bloques: un primer bloque (2.1 a 2.4) referido al
dominio de aplicación —secretarías virtuales en salud, inteligencia
artificial conversacional, la plataforma de mensajería de WhatsApp y los
principios de arquitectura de software adoptados— y un segundo bloque
(2.5 a 2.8, en el archivo complementario) referido a las tecnologías de
implementación concretas y al marco normativo argentino aplicable.

## 2.1 Secretarías virtuales y asistentes conversacionales en salud

Una secretaría virtual es un sistema de software que automatiza tareas
administrativas típicamente asociadas a la recepción y secretaría de una
institución —agendar turnos, responder consultas frecuentes, enviar
recordatorios, gestionar cancelaciones— sin intervención humana directa en
cada interacción. A diferencia de un sistema de gestión administrativa
tradicional, operado por el personal de la institución a través de una
interfaz propia, una secretaría virtual está pensada para ser utilizada
directamente por el usuario final (paciente o profesional) a través de un
canal conversacional, lo que desplaza parte de la carga operativa desde el
personal administrativo hacia el propio sistema.

La automatización de tareas administrativas en el ámbito de la salud
mediante canales conversacionales no es un desarrollo reciente: los
primeros sistemas de respuesta de voz interactiva (IVR) permitían ya
confirmar o cancelar turnos por teléfono mediante menús de tonos. La
aparición de plataformas de mensajería masiva y, más recientemente, de
modelos de lenguaje capaces de sostener una conversación en lenguaje
natural, ampliaron sustancialmente el rango de tareas que un sistema de
este tipo puede resolver sin intervención humana, permitiendo pasar de
menús rígidos a interacciones en lenguaje libre [CITA: evolución de los
sistemas conversacionales en salud, de IVR a asistentes basados en LLM].

En el dominio específico de la salud mental, la literatura distingue con
particular énfasis entre asistentes conversacionales de carácter
administrativo —cuyo alcance se limita a agenda, recordatorios y preguntas
frecuentes— y asistentes con pretensión clínica o terapéutica, que
interactúan con el paciente sobre su estado de salud o intentan ofrecer
algún tipo de apoyo terapéutico automatizado. Esta distinción es relevante
porque ambos tipos de sistema están sujetos a exigencias muy distintas en
materia de seguridad, supervisión profesional y marco normativo: un
asistente puramente administrativo no requiere la validación clínica que
sí exige un asistente que interviene, aunque sea tangencialmente, en el
proceso terapéutico [CITA: distinción entre chatbots administrativos y
chatbots clínicos/terapéuticos en salud mental]. La adopción de asistentes
conversacionales en instituciones de salud reporta beneficios asociados
principalmente a la reducción de la carga administrativa del personal y a
la disminución de inasistencias a turnos mediante recordatorios
automatizados, aunque la magnitud de ese efecto varía según el estudio y el
contexto de implementación [CITA: efecto de los recordatorios
automatizados sobre la tasa de inasistencia a turnos en salud].

Un asistente conversacional de este tipo introduce también riesgos
específicos que la literatura señala de forma recurrente: la posibilidad
de que el usuario le atribuya al sistema una capacidad de comprensión
clínica que este no posee, la necesidad de establecer límites claros y
comunicados sobre qué tipo de consultas puede resolver, y la obligación de
derivar a un humano ante cualquier consulta que exceda ese alcance
administrativo. Estos riesgos son especialmente sensibles en el ámbito de
la salud mental, donde una respuesta automatizada percibida como
inadecuada puede tener un costo mayor que en otros dominios administrativos
[CITA: riesgos de la automatización conversacional en contextos de salud
mental].

## 2.2 Modelos de lenguaje grandes (LLM) e IA conversacional

Un modelo de lenguaje grande (*large language model*, LLM) es un modelo
estadístico entrenado sobre grandes volúmenes de texto para estimar, dado
un fragmento de texto de entrada, la distribución de probabilidad del texto
que razonablemente lo continúa. La arquitectura predominante detrás de los
LLM contemporáneos es la arquitectura *transformer*, que procesa una
secuencia de texto mediante mecanismos de atención capaces de ponderar la
relevancia de cada elemento de la secuencia respecto de los demás, en
lugar de procesarla estrictamente en orden como hacían las arquitecturas
recurrentes anteriores [CITA: arquitectura transformer y mecanismos de
atención]. Entrenado sobre corpus de texto de gran escala y ajustado
posteriormente mediante técnicas de afinamiento orientadas a seguir
instrucciones y sostener diálogo, un modelo de este tipo es capaz de
generar texto coherente en respuesta a una instrucción o a una conversación
en curso, sin haber sido programado explícitamente con reglas para cada
posible intercambio.

Desde la perspectiva de quien integra un LLM en un sistema de software, el
modelo se consume típicamente como un servicio: la aplicación envía una
secuencia de texto (el historial de la conversación más una instrucción o
*prompt* que fija el rol y las restricciones del asistente) y recibe como
respuesta texto generado por el modelo. La calidad y pertinencia de esa
respuesta depende fuertemente de cómo se construye ese *prompt*, disciplina
conocida como ingeniería de *prompts*, y del contexto que se le provee al
modelo en cada solicitud, dado que estos modelos no retienen memoria entre
solicitudes salvo que la aplicación se la reenvíe explícitamente en cada
turno.

Un LLM, por sí solo, únicamente genera texto: no puede consultar una base
de datos, verificar disponibilidad de un recurso externo, ni ejecutar una
acción sobre otro sistema. La técnica conocida como *function calling* (o,
más genéricamente, uso de herramientas) extiende esta capacidad
permitiéndole al modelo, durante la generación de una respuesta, solicitar
la ejecución de una función externa —descripta de antemano a el modelo
mediante un esquema con su nombre, propósito y parámetros— y recibir el
resultado de esa ejecución para incorporarlo a su respuesta siguiente. De
este modo, un asistente conversacional construido sobre un LLM con
*function calling* puede, por ejemplo, consultar la disponibilidad real de
un recurso antes de proponer una fecha, en lugar de generar una respuesta
basada únicamente en lo que el modelo "cree" plausible a partir de su
entrenamiento [CITA: function calling / tool use en modelos de lenguaje].
Esta capacidad es la que habilita, en términos generales, la
transformación de un LLM de generador de texto a componente de un sistema
agéntico, capaz de tomar decisiones sobre qué acción externa ejecutar en
función del estado de la conversación.

Entre los proveedores que ofrecen LLM como servicio se encuentra Anthropic,
con su familia de modelos Claude, orientada explícitamente al desarrollo
de asistentes conversacionales seguros y con soporte nativo para uso de
herramientas (*tool use*) como el descripto en el párrafo anterior
[VERIFICAR: características y versión de modelo de la familia Claude
vigentes al momento de redacción]. La elección de un proveedor de LLM
concreto para un sistema determinado depende de factores como el costo por
uso, los límites de la ventana de contexto, el soporte de *function
calling*, y las garantías de manejo de datos que ofrece el proveedor,
factores que resultan particularmente relevantes cuando la conversación
puede involucrar datos personales de pacientes.

## 2.3 WhatsApp Business Platform (Cloud API)

WhatsApp Business Platform es la oferta de Meta para que una organización
integre WhatsApp como canal de comunicación con sus clientes o usuarios de
forma programática, a diferencia de la aplicación WhatsApp Business
orientada a un uso manual por parte de una persona. La modalidad
actualmente promovida por Meta para esta integración es la Cloud API, en
la que los servidores que procesan los mensajes son operados por Meta y la
organización se conecta a ellos mediante llamadas HTTP autenticadas, sin
necesidad de operar su propia infraestructura de mensajería —a diferencia
del modelo On-Premises API previo, discontinuado en favor de esta
modalidad alojada [VERIFICAR: estado y fecha de discontinuación de la
On-Premises API].

La recepción de mensajes entrantes se resuelve mediante *webhooks*: la
organización expone un extremo HTTP propio y lo registra ante la
plataforma, de modo que, cada vez que un usuario envía un mensaje al
número de WhatsApp de la organización, la plataforma realiza una solicitud
HTTP a ese extremo con el contenido del mensaje. Este modelo, basado en
notificación por eventos en lugar de sondeo periódico (*polling*), exige
que el sistema receptor esté disponible de forma continua y valide la
autenticidad de cada solicitud entrante, dado que el extremo del *webhook*
es, por diseño, de acceso público [CITA: modelo de webhooks de WhatsApp
Business Platform].

El envío de mensajes desde la organización hacia el usuario está sujeto a
una restricción central del modelo de mensajería: la ventana de servicio al
cliente de 24 horas. Dentro de esa ventana, contada desde el último mensaje
recibido del usuario, la organización puede enviarle mensajes de texto
libre. Fuera de esa ventana, solo puede iniciar una conversación mediante
una plantilla de mensaje (*message template*) previamente redactada y
aprobada por Meta, que admite variables pero no contenido arbitrario. Las
plantillas se clasifican en categorías —de utilidad, de marketing y de
autenticación, entre otras— y cada categoría está sujeta a su propio
proceso de revisión y a distintas condiciones de costo y de frecuencia de
envío [VERIFICAR: categorías de plantillas y modelo de costo vigentes].
Este mecanismo de plantillas existe para prevenir el uso de la plataforma
como canal de mensajería masiva no solicitada, y tiene implicancias
directas de diseño para cualquier sistema que necesite iniciar una
conversación —por ejemplo, para enviar un recordatorio de turno— sin que el
usuario haya escrito primero.

La plataforma impone además límites de volumen de mensajes salientes por
número de teléfono de la organización, organizados en niveles (*tiers*) que
se incrementan a medida que el número acumula un historial de calidad de
mensajería aceptable, y provee mecanismos de verificación de la cuenta y
del número de teléfono asociado [VERIFICAR: niveles de tier y límites de
mensajería vigentes]. El conjunto de estas restricciones —ventana de 24
horas, plantillas aprobadas y límites por nivel— condiciona de forma
directa el diseño de cualquier asistente conversacional construido sobre
esta plataforma, que debe planificar con anticipación cuándo puede iniciar
una conversación y cuándo debe esperar a que la inicie el usuario.

## 2.4 Arquitectura de software: monolito modular, arquitectura hexagonal, multi-tenancy, reglas de negocio como datos y ejecución de tareas programadas

Un monolito modular es un estilo arquitectónico en el que la aplicación se
despliega como una única unidad ejecutable, pero se organiza internamente
en módulos con límites explícitos y responsabilidades bien definidas, de
forma análoga a como se organizaría un sistema de microservicios, sin
incurrir en el costo operativo de desplegar, versionar y coordinar
múltiples servicios independientes. Este estilo se contrapone tanto al
monolito no modular —donde no existen límites internos claros y cualquier
parte del código puede depender de cualquier otra— como a la arquitectura
de microservicios, que gana en despliegue y escalado independientes por
servicio a costa de introducir complejidad de comunicación entre procesos,
consistencia distribuida y observabilidad [CITA: comparación entre
monolito modular y arquitectura de microservicios]. La elección entre estos
estilos suele depender del tamaño del equipo, de la madurez del dominio y
de si existen partes del sistema con requisitos de escalado
significativamente distintos entre sí.

La arquitectura hexagonal, también conocida como arquitectura de puertos y
adaptadores, es un patrón que organiza el código de un módulo en capas
concéntricas: un núcleo de dominio que contiene las reglas de negocio y no
depende de ningún framework ni tecnología de infraestructura concreta; una
capa de aplicación que orquesta esas reglas de dominio; y una capa externa
de infraestructura que implementa los detalles técnicos (bases de datos,
proveedores externos, interfaces de usuario). La comunicación entre el
núcleo y el exterior se produce exclusivamente a través de interfaces
definidas por el propio dominio, llamadas puertos, cuya implementación
concreta —el adaptador— se resuelve por fuera del núcleo, típicamente
mediante inyección de dependencias [CITA: arquitectura hexagonal / puertos
y adaptadores, Cockburn]. La ventaja central de este patrón es que permite
sustituir un proveedor externo, o reemplazarlo por una implementación de
prueba, sin modificar la lógica de negocio que lo utiliza, y facilita
testear esa lógica de forma aislada de la infraestructura.

La multi-tenancy (multi-inquilinato) es la capacidad de un mismo sistema de
servir a múltiples organizaciones o clientes (*tenants*) independientes
entre sí, garantizando que los datos y la configuración de cada uno
permanezcan aislados de los demás. Los modelos habituales de multi-tenancy
van desde el aislamiento completo —una base de datos o incluso una
instancia de la aplicación por cada tenant— hasta el modelo agrupado
(*pooled*), en el que todos los tenants comparten la misma base de datos y
el aislamiento se garantiza a nivel de fila mediante un identificador de
tenant presente en cada registro, pasando por modelos híbridos [CITA:
modelos de aislamiento en arquitecturas multi-tenant]. El modelo agrupado
resulta más económico de operar a medida que crece la cantidad de tenants,
pero traslada la responsabilidad de garantizar el aislamiento del nivel de
infraestructura al nivel de la aplicación, lo que exige mecanismos que
impidan de forma sistemática, y no solo por convención de código, que una
consulta de un tenant pueda alcanzar datos de otro.

Finalmente, el principio de tratar las reglas de negocio como datos
propone externalizar como configuración —registros en una base de datos o
en un archivo de configuración— aquellos comportamientos del sistema que
varían de una instalación u organización a otra, en lugar de codificarlos
como condicionales fijos en el código fuente. Bajo este principio, un
cambio en una regla de negocio (por ejemplo, un plazo, un texto o un
umbral) se resuelve modificando un registro de datos, sin requerir una
nueva versión desplegada de la aplicación. Este enfoque es particularmente
relevante en sistemas pensados para operar sobre múltiples organizaciones
con reglas propias, dado que evita que cada variante de negocio se traduzca
en una rama de código distinta [CITA: patrones de configuración dirigida
por datos / motores de reglas de negocio].

Un cuarto elemento arquitectónico, complementario a los anteriores, es la
ejecución de tareas programadas (*scheduled tasks* o *cron jobs*) dentro de
la propia aplicación: procesos que se disparan a intervalos regulares o en
instantes predeterminados, sin que medie una petición externa que los
inicie, para detectar y actuar sobre estados del sistema que de otro modo
solo cambiarían por la iniciativa de un usuario. Un trabajo programado
típico recorre periódicamente el conjunto de registros que cumple una
condición temporal —por ejemplo, "vencido hace más de N horas" o "dentro de
una ventana de aviso previo a un evento futuro"— y aplica sobre cada uno la
transición o el efecto colateral correspondiente [CITA: patrones de
ejecución de tareas programadas en aplicaciones backend]. Esta forma de
disparo se diferencia de la respuesta síncrona a una petición HTTP en que
ninguna parte externa espera su resultado de manera inmediata, lo que la
vuelve el mecanismo natural para reglas de negocio que dependen únicamente
del paso del tiempo —una confirmación no respondida, un recordatorio previo
a una cita, la expiración de una credencial temporal— en lugar de un acto
deliberado de un usuario.

Dos propiedades resultan centrales al diseñar un trabajo programado que
opera sobre un conjunto de registros compartido. La primera es la
idempotencia: dado que el mismo trabajo vuelve a ejecutarse en cada
intervalo, debe poder distinguir un registro ya procesado de uno pendiente,
habitualmente mediante una marca de tiempo o un estado dedicado que
registre el efecto ya aplicado, para no repetir una acción —como el envío
de un mensaje— sobre el mismo registro en corridas sucesivas [CITA:
idempotencia en el procesamiento periódico de datos]. La segunda es la
seguridad frente a la concurrencia entre el trabajo programado y una acción
manual simultánea sobre el mismo registro: una escritura condicionada al
estado que el trabajo espera encontrar —de forma que la actualización
simplemente no tenga efecto si un tercero ya modificó el registro entre la
lectura y la escritura del trabajo— evita que el resultado de una acción
humana reciente sea sobrescrito o entre en conflicto con el de la corrida
programada.

En un sistema multi-tenant, un trabajo programado agrega una dimensión más:
a diferencia de una petición HTTP, que trae consigo la identidad del tenant
que la origina, un trabajo programado se dispara sin ningún contexto de
tenant asociado, por lo que debe iterar explícitamente sobre cada
organización y establecer, para cada una, el contexto de aislamiento bajo
el cual va a leer y escribir datos, antes de aplicar la misma lógica de
negocio de forma independiente por organización [CITA: ejecución de tareas
programadas en aplicaciones multi-tenant].
