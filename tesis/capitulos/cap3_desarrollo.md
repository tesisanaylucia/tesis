# Capítulo 3: Solución PSIQUE

> Documento en construcción. Las subsecciones se completan de forma
> incremental a medida que la skill `documentacion-tesis` integra el
> trabajo validado de cada fase, en ambos repos (backend y móvil). El
> contenido de este archivo corresponde a 3.2 Proceso de desarrollo; 3.1 y
> 3.3 se redactan aparte cuando corresponda.

## 3.2 Proceso de desarrollo

### 3.2.0 Fundaciones

El desarrollo del backend de PSIQUE comenzó por establecer una base
arquitectónica capaz de sostener, desde el primer módulo de negocio, los
requisitos transversales del sistema: aislamiento estricto entre
organizaciones, trazabilidad de las acciones sobre datos sensibles, y una
frontera clara entre la lógica de negocio y los proveedores externos que
el sistema eventualmente integraría. En lugar de posponer estos requisitos
a una etapa de "endurecimiento" posterior, se los incorporó en la fase
inicial del proyecto, dado que introducirlos una vez que existen múltiples
módulos de negocio construidos sobre supuestos distintos resulta
significativamente más costoso que diseñarlos desde el inicio.

El aislamiento multi-tenant se resolvió a nivel del cliente de acceso a
datos en lugar de delegarlo a la disciplina de cada desarrollador al
escribir una consulta. Se extendió el cliente de Prisma para que, ante
cualquier operación sobre un modelo que incluye `organizationId` en su
esquema, inyecte automáticamente el identificador de la organización activa
—propagado mediante un almacenamiento de contexto asincrónico
(`AsyncLocalStorage`) poblado al inicio de cada solicitud— y rechace la
operación si no hay una organización en contexto. Esta decisión traslada el
riesgo de omitir el acotamiento por tenant desde una fuga silenciosa de
datos entre organizaciones hacia un error explícito y detectable en tiempo
de ejecución, lo cual se consideró preferible dado que una fuga de datos
entre organizaciones —en un sistema que maneja información de pacientes de
salud mental— constituye una falla de cumplimiento normativo (Ley 25.326),
no solamente un defecto funcional.

Sobre esa base se incorporaron, también como parte de las fundaciones, la
autenticación basada en JWT con autorización por rol aplicada de forma
global (en lugar de ruta por ruta, para que una ruta nueva quede protegida
por omisión), un registro de auditoría desacoplado de las entidades de
negocio para poder reconstruir quién hizo qué acción sobre qué entidad y
cuándo, y tres puertos de integración (mensajería, cerradura inteligente y
procesamiento de IA conversacional) resueltos por inyección de dependencias
contra adaptadores *stub*. Definir estos puertos antes de contar con
credenciales o acceso a los proveedores reales permitió avanzar el
desarrollo del resto del sistema de forma independiente de esas
integraciones, y deja como trabajo de una fase posterior únicamente la
tarea de reemplazar cada adaptador *stub* por su contraparte real, sin
tocar la lógica de negocio que los consume.

La autorización por rol se revisó posteriormente contra el diagrama
entidad-relación de la base de datos, que define tres valores posibles
para el rol de un usuario —`profesional`, `admin` y `sistema`— mientras
que la implementación inicial solo contemplaba los dos primeros. El tercer
valor, incorporado al enum como `SYSTEM`, se reservó para procesos
automatizados que necesitan actuar sobre la base de datos sin que exista
un usuario humano en el contexto de la solicitud: los trabajos programados
de recordatorio y expiración (confirmación a 24 horas, cancelación
automática a 4 horas, vencimiento de códigos de acceso temporal) y, más
adelante, el orquestador de la capa conversacional. Sin un rol dedicado
para estos actores, cualquier acción automatizada sobre datos de pacientes
—incluida su propia traza de auditoría— carecería de un identificador de
usuario válido y consistente para atribuirla. Al mismo tiempo, se decidió
excluir explícitamente a `SYSTEM` de la vía de autenticación manual: el
endpoint de inicio de sesión rechaza con un código de prohibido cualquier
intento de ingresar con una cuenta de ese rol, verificación que se realiza
después de validar la contraseña para no exponer, a través del código de
respuesta, si una cuenta con esas credenciales existe o qué rol tiene antes
de que la autenticación sea exitosa.

El registro de auditoría se revisó, a su vez, contra el mismo diagrama
entidad-relación que había motivado la corrección del rol `SYSTEM`. Ese
diagrama define, para la entidad de auditoría, un campo propio que la
vincula directamente con la entidad turno, además de los campos genéricos
de entidad y acción con los que se había implementado originalmente. Sin
ese vínculo directo, reconstruir el historial de cambios sobre un turno
puntual requería filtrar por el identificador genérico de entidad, en
lugar de contar con una columna dedicada para esa consulta. Se incorporó
entonces un campo opcional que referencia al turno afectado, sin agregarle
todavía una relación formal de clave foránea a nivel de esquema: la
entidad turno en sí pertenece a una fase posterior del desarrollo (el
motor de turnos), de modo que el campo queda por ahora como una columna
simple, a la espera de que esa fase incorpore la entidad y permita
declarar la relación completa. Esta secuencia —introducir un campo que
anticipa una relación futura sin bloquear el trabajo actual a que esa
relación exista— se adoptó de forma análoga a como los puertos de
integración se definieron antes de contar con los adaptadores reales:
en ambos casos se prioriza dejar preparado el punto de extensión sin
imponerle una dependencia dura a una pieza que todavía no se construyó.

Una revisión posterior del mismo registro de auditoría, contra el
documento de requisitos y el diagrama entidad-relación en conjunto,
identificó que la tabla combinaba, en la práctica, tres categorías de
eventos de naturaleza distinta: acciones humanas sobre un turno (ya
resueltas mediante el campo dedicado descripto en el párrafo anterior),
acciones humanas sobre un paciente sin turno asociado —como la
exportación o la supresión de sus datos, propias del módulo de
cumplimiento normativo— y eventos operativos generados por el adaptador
de la cerradura inteligente, que no son decisiones de una persona sino
registros de una integración de hardware. Para la segunda categoría se
incorporó un campo opcional que referencia al paciente afectado, siguiendo
el mismo tratamiento que el campo de turno: sin relación de clave foránea
todavía, a la espera de que la fase de pacientes incorpore esa entidad al
esquema. Se aprovechó además la aparición de esta segunda categoría para
corregir el campo de acción del registro de auditoría, que hasta entonces
estaba restringido a un conjunto cerrado de tres valores (creación,
modificación, eliminación): dado que las acciones de cumplimiento sobre
pacientes no se conocen de antemano como un conjunto fijo, se convirtió
ese campo a texto libre, evitando así tener que sostener dos columnas de
acción distintas —una por conjunto cerrado y otra por texto libre— en el
mismo registro.

Para la tercera categoría, en cambio, se optó por una tabla y un servicio
completamente separados del registro de auditoría, en lugar de distinguir
el origen del evento dentro de la misma tabla mediante una columna
adicional. Mezclar en una sola tabla la trazabilidad de decisiones humanas
sobre datos de pacientes con el volumen operativo de una integración de
hardware —reintentos, errores de comunicación, expiraciones de código—
contaminaría una traza pensada para sustentar responsabilidad sobre datos
sensibles con registros de diagnóstico técnico, de cardinalidad y
propósito distintos. La tabla resultante replica, para el alcance
multi-tenant, el mismo mecanismo de acotamiento automático por
organización que ya usaban el registro de auditoría y la configuración por
tenant, y se implementó como módulo independiente, sin que el módulo de
auditoría dependa de él ni viceversa. Ni este servicio de eventos de
cerradura ni el campo de paciente del registro de auditoría se conectaron
todavía a ningún llamador real: el adaptador de la cerradura inteligente y
los endpoints administrativos sobre pacientes pertenecen a fases
posteriores del desarrollo que, al momento de esta corrección, no existían
en el código.

Una revisión posterior del esquema, esta vez centrada en los catálogos
auxiliares incorporados junto con la configuración por tenant, encontró un
modelo que no correspondía a ninguna fuente de verdad vigente del
proyecto: un catálogo de diagnósticos con códigos CIE-10, agregado a
partir de una mención aislada del documento de requisitos a "tablas
auxiliares" pero ausente del diagrama entidad-relación de la base de
datos. Ninguna entidad de dominio implementada o planificada declaraba una
clave foránea hacia ese catálogo, y la capa conversacional excluye
explícitamente los diagnósticos de su alcance. Se optó por eliminar el
modelo por completo en lugar de conservarlo sin uso: una tabla en el
esquema que no responde a ningún requisito ni relación real introduce una
discrepancia entre el código y su especificación que resulta más costosa
de sostener que el trabajo ya invertido en crearla. Si un módulo de
diagnósticos resulta necesario más adelante, queda documentado que deberá
diseñarse desde cero con su propio diagrama entidad-relación, en lugar de
recuperar este modelo descartado.

### 3.2.1 Profesionales

El primer módulo de negocio construido sobre las fundaciones fue el de
Profesionales, y su punto de partida fue el modelado de las entidades de
dominio a partir del diagrama entidad-relación que actúa como fuente de
verdad de la base de datos. Se incorporaron al esquema cinco entidades: el
profesional, su especialidad, sus matrículas, sus horarios de atención y
sus ausencias. Esta primera tarea se acotó deliberadamente al esquema y su
migración, sin endpoints ni servicios, de modo que el modelo de datos
quedara estabilizado y verificado antes de construir sobre él la lógica de
gestión, la configuración de agenda y los filtros de asignación que
abordan las tareas siguientes de la fase.

El modelado reafirmó dos convenciones ya adoptadas en las fundaciones. La
primera es que las entidades y sus columnas se nombran en inglés, pese a
que el vocabulario de dominio del proyecto es el castellano de la clínica:
la traza de esa decisión es la migración temprana que renombró al inglés
los catálogos originalmente escritos en español, y las nuevas entidades se
sumaron a esa convención (`Specialty`, `Professional`, `License`,
`WorkingHour`, `Absence`). La segunda es el acotamiento por organización,
que aquí se aplicó de forma matizada: cuatro de las cinco entidades
declaran `organizationId` y quedan, por ese solo hecho, sujetas al
acotamiento automático de la extensión de Prisma; la matrícula, en cambio,
se dejó deliberadamente sin ese campo. La razón es que una matrícula nunca
se consulta de manera independiente sino como colección de un profesional
que ya está acotado por tenant, de modo que agregarle un `organizationId`
propio sería redundante y podría incluso introducir la posibilidad de una
inconsistencia entre el tenant de la matrícula y el de su profesional. El
aislamiento de la matrícula queda así garantizado por la relación, no por
un campo replicado.

Varias reglas del dominio se ubicaron conscientemente fuera del esquema.
El máximo de tres matrículas por profesional que fija el requisito se dejó
como validación de la capa de servicio y no como restricción de base de
datos, tanto porque el propio ticket lo sitúa allí como porque ese tope no
se deriva naturalmente de la combinación de tipos de matrícula posibles y
forzarlo en el esquema habría expresado la regla de forma artificiosa. Del
mismo modo, los atributos que otras tareas de la fase configuran más
adelante —la duración de la consulta y la franja horaria extra para
pacientes nuevos— se declararon opcionales, para que el alta de un
profesional no dependa de valores que todavía no se definen en esta etapa,
mientras que los indicadores de comportamiento recibieron valores por
defecto razonables. La baja de un profesional se modeló como lógica, a
través de un indicador de actividad, nunca como borrado físico, en línea
con el diagrama y con la necesidad de preservar la trazabilidad histórica.

La especialidad se modeló como un catálogo acotado por tenant, con nombre
de texto libre y unicidad por organización, replicando el patrón ya usado
para el catálogo de obras sociales. Se descartó representarla como un
enumerado cerrado —pese a que el diagrama menciona valores concretos—
porque ello fijaría en el esquema la nomenclatura de una organización
particular, en conflicto con el diseño para marca blanca que persigue el
sistema. Los horarios de atención, por su parte, guardan las horas de
inicio y fin como texto en formato de reloj de pared de veinticuatro
horas, sin fecha ni huso asociados; se prefirió esta representación al tipo
temporal nativo porque este último se expone como una fecha completa con un
día artificial, semántica engañosa para un horario recurrente semanal que
carece de fecha.

La verificación se concentró en los criterios de aceptación del modelo: que
la migración se aplicara sin dejar pendientes y que las relaciones fueran
utilizables de extremo a extremo. Una prueba de integración inserta un
profesional junto con su especialidad y sus matrículas y lo vuelve a leer,
comprobando que la clave foránea a la especialidad resuelve, que la
relación uno-a-muchos con las matrículas devuelve las filas esperadas y que
los valores por defecto se aplican. Otras dos pruebas confirman que el
acotamiento por tenant se extiende a las nuevas entidades: una consulta sin
organización en contexto es rechazada, y una organización no accede a los
profesionales de otra a través del cliente acotado.

Estabilizado el modelo de datos, se construyó sobre él la capa de negocio
del módulo: los servicios y los endpoints REST para el alta, la consulta, la
modificación y la baja lógica de profesionales, junto con la gestión de sus
matrículas. El módulo no reintroduce infraestructura propia sino que se
apoya en las fundaciones ya descritas: el cliente de datos acotado por
tenant, el registro de auditoría, los guards de autenticación y rol, y el
interceptor que propaga la organización del request. Cada mutación —alta,
edición, baja, y las operaciones sobre matrículas— deja su rastro en la
traza de auditoría, reutilizando el servicio existente sin acoplarse a su
implementación.

La decisión de diseño más relevante de esta etapa fue cómo expresar los tres
niveles de acceso que exige el módulo. Algunas operaciones son exclusivas
del rol administrativo (crear un profesional, darlo de baja); otras puede
ejercerlas también el propio profesional sobre su registro (editar sus
datos y sus matrículas); y las consultas quedan disponibles para cualquier
usuario autenticado del tenant, dado que el listado alimenta al chatbot. Los
dos primeros niveles se resolvieron de forma declarativa y complementaria:
el decorador de rol ya existente para las rutas exclusivas del administrador,
y un guard de propiedad nuevo para las rutas de tipo "administrador o
dueño", que compara el profesional asociado al usuario autenticado con el
profesional referido en la ruta y admite incondicionalmente al
administrador. Se prefirió un guard reutilizable antes que incrustar la
comprobación en cada método de servicio, porque la propiedad es una
preocupación de autorización transversal a varias rutas; centralizarla evita
repetir la lógica, la mantiene junto a la ruta que protege y la deja
verificable de forma aislada. Para que un mismo guard sirviera a la edición
del profesional y a las tres operaciones de matrículas, estas últimas se
modelaron como sub-recurso anidado bajo el profesional, exponiendo su
identificador con el mismo nombre de parámetro.

El tratamiento de las matrículas heredó la consecuencia de haberlas dejado,
en P1.1, sin identificador de organización propio: al no estar sujetas al
acotamiento automático, el servicio nunca las resuelve de forma aislada por
su identificador, sino que primero verifica —a través del servicio de
profesionales— que el profesional padre pertenece al tenant del request, y
solo entonces opera sobre la matrícula filtrando además por ese profesional.
El aislamiento se deriva así del profesional ya acotado, en coherencia con
el modelo. El tope de tres matrículas por profesional se validó en dos
planos que comparten una única constante: el DTO de alta, para las cargadas
en línea junto con el profesional, y el servicio, para el alta incremental,
que cuenta las existentes antes de insertar; superar el límite produce un
error de validación. Finalmente, las respuestas no devuelven la entidad de
persistencia en crudo sino que pasan por una función de presentación que
fija su forma y omite el identificador de organización, para que ese dato
interno no cruce la frontera de la interfaz; la definición de qué relaciones
se cargan se comparte entre las consultas y el tipo de la respuesta, de modo
que ambas no puedan divergir.

La verificación de esta capa se realizó atravesando la interfaz HTTP con
credenciales reales. Las pruebas end-to-end ejercitan el alta por el
administrador con matrículas en línea y su registro de auditoría, el rechazo
del alta a un rol no administrativo, el listado con nombre y matrículas, el
aislamiento por tenant —una organización recibe una respuesta de no
encontrado ante un profesional de otra—, la edición del propio registro por
su dueño y el rechazo al editar el de un tercero, el tope de matrículas y la
baja lógica exclusiva del administrador, que retira al profesional del
listado activo pero lo conserva en la base con su indicador de actividad en
falso. Una prueba unitaria adicional aísla la lógica del guard de propiedad.

Estabilizado el módulo de profesionales, se lo completó con dos capacidades
de su agenda: la grilla semanal de horarios de atención y la gestión de
ausencias. Ambas se modelaron como sub-recursos anidados bajo el profesional,
en coherencia con las matrículas, y reutilizan sin cambios el guard de
propiedad, el registro de auditoría y el anclaje en el profesional padre ya
descritos. Como las tablas de horario y ausencia se habían creado en el
modelado inicial, esta etapa aportó únicamente la capa de negocio sobre
ellas.

La grilla de horarios se gestiona mediante un reemplazo total e idempotente:
un único endpoint recibe el conjunto completo de bloques y sustituye con él
toda la grilla del profesional, borrando los bloques previos e insertando los
nuevos dentro de una misma transacción, de modo que un fallo no deje un
estado parcial y que reenviar el mismo contenido produzca siempre el mismo
resultado. Se prefirió este esquema de reemplazo frente a un alta y baja
individual de bloques porque coincide con la forma en que una interfaz de
agenda edita una grilla semanal y porque simplifica la validación de
solapamientos, que puede realizarse sobre el conjunto entrante completo sin
contrastarlo contra el estado ya persistido. Esa validación es una regla de
negocio y se ubicó en el servicio, no en el DTO: aprovechando que las horas
se guardan como texto de reloj de pared con cero a la izquierda —decisión
tomada en el modelado—, los bloques se comparan por orden de cadena sin
necesidad de convertirlos a un tipo temporal; se verifica que en cada bloque
el inicio preceda al fin y que, agrupados por día y ordenados por hora, dos
bloques del mismo día no se superpongan, admitiéndose como válidos los que
apenas se tocan para permitir modelar mañana y tarde contiguas. El DTO, por
su parte, solo valida el formato de cada hora. Una violación de cualquiera de
estas reglas produce un error de validación.

La gestión de ausencias —registro, consulta y cancelación— introdujo el
requisito de dejar preparado un punto de extensión: al registrarse una
ausencia debe emitirse un evento para que un módulo posterior dispare la
reasignación de los turnos afectados, sin que esa lógica de reasignación se
implemente todavía aquí. Para no acoplar el dominio de profesionales a ese
futuro consumidor, el evento se publica a través de un puerto de dominio
propio, análogo a los puertos de integración externa ya presentes: el
servicio de ausencias depende de una abstracción de publicación por
inyección y, tras persistir y auditar la ausencia, emite el evento con los
identificadores del profesional, del tenant y de la ausencia, y el rango de
fechas. El adaptador por defecto se limita a registrar la emisión, sin
exponer datos sensibles, y será sustituido más adelante por un suscriptor
real sin tocar este módulo. Se prefirió este puerto enfocado a incorporar un
bus de eventos genérico, porque el único evento que la fase necesita es el de
ausencia registrada y un contrato explícito y tipado resulta más claro, evita
sumar una dependencia y es coherente con el mecanismo de extensión que el
resto del sistema ya emplea. En línea con el requisito, registrar una
ausencia bloquea el período pero no altera la grilla de horarios habituales:
el servicio de ausencias nunca toca los horarios. La emisión del evento
representa el punto de extensión hacia la reasignación de turnos que
abordará un módulo posterior, y se dejó registrada como figura pendiente.

La autorización de estas dos capacidades reprodujo el criterio del ABM: la
consulta de la grilla y de las ausencias queda abierta a cualquier usuario
autenticado del tenant, porque alimentará a las capas de agenda y de chatbot,
mientras que el reemplazo de la grilla y el alta y la cancelación de
ausencias se restringen, con el mismo guard de propiedad reutilizado, al
propio profesional o al administrador. La verificación volvió a realizarse de
extremo a extremo sobre la interfaz HTTP, sustituyendo el puerto de eventos
por un doble que captura las publicaciones para comprobar su contenido. Las
pruebas ejercitan el reemplazo idempotente de la grilla, el rechazo de
bloques solapados, de un bloque con inicio no anterior al fin y de una hora
mal formada, la aceptación de bloques contiguos, el aislamiento por tenant y
el rechazo al modificar la grilla ajena; y, para las ausencias, el registro
con emisión y auditoría del evento, el rechazo de un rango de fechas
invertido sin emitir evento, la conservación de la grilla al registrar una
ausencia, el listado y la cancelación, y el rechazo al operar sobre otro
profesional.

El módulo se completó, por último, con la configuración que cada profesional
define para sí mismo, abordada en dos etapas sucesivas. La primera de ellas
introdujo la configuración de agenda: la duración de sus turnos y la franja
horaria extra destinada a la primera sesión de un paciente nuevo, que por ser
más larga ocupa dos turnos consecutivos. Dos de los campos involucrados —la duración de la consulta, en
minutos, y la franja extra, en horas— ya se habían declarado opcionales en el
modelado inicial, difiriendo su configuración a esta etapa; a ellos se sumó un
tercer atributo, ausente del diagrama entidad-relación original, que registra
dónde se ubica esa franja extra respecto de la agenda habitual. Este último se
incorporó como un enumerado nuevo con tres valores tomados literalmente de la
fuente de requisitos: la franja extra como primer turno del día, agregada antes
de la franja habitual; como último turno del día, agregada después; o dentro de
la franja habitual, ocupando dos turnos consecutivos de la agenda. Se lo modeló
en inglés y nullable, en coherencia con los demás enumerados de persistencia y
con los otros dos campos de configuración, dejando en comentarios del esquema el
mapeo de cada valor a su semántica en castellano. Es importante subrayar el
límite del alcance: esta etapa únicamente almacena la configuración; la lógica
que la aplica al generar los turnos de la agenda, el cupo de un paciente nuevo
por día y el aviso de la duración de la sesión al paciente corresponden a
módulos posteriores.

La edición de esta configuración se resolvió con un endpoint dedicado, separado
del que modifica los datos generales del profesional. Se prefirió esa
separación por cohesión —los datos identitarios y la configuración de agenda son
preocupaciones distintas, con validaciones propias—, pero, a diferencia de los
horarios y las ausencias, no se introdujo un controlador ni un servicio nuevos:
como la configuración son campos de la propia entidad profesional y no una
entidad aparte, el método se sumó al controlador y al servicio ya existentes,
reutilizando la verificación de pertenencia al tenant, el registro de auditoría
y el guard de propiedad. La operación respeta la semántica de una modificación
parcial: solo se escriben los campos presentes en la petición y los omitidos se
dejan intactos, de modo que un profesional pueda ajustar un único parámetro sin
reenviar el resto de su configuración. Las validaciones de los criterios de
aceptación —duración estrictamente positiva y franja extra no negativa, con el
cero admitido como ausencia de tiempo extra, y modalidad restringida a los tres
valores del enumerado— se ubicaron en el DTO por tratarse de restricciones de
forma sobre campos aislados, en coherencia con el criterio ya adoptado para
distinguir la validación de forma de la regla de negocio. La verificación
combinó una prueba unitaria del servicio, que aísla la persistencia de los tres
valores de modalidad y la semántica de modificación parcial con el cliente de
datos simulado, con pruebas end-to-end que ejercitan sobre la interfaz HTTP la
configuración y persistencia de las tres modalidades, el rechazo de una
duración no positiva, de una franja negativa, de una duración no entera y de una
modalidad desconocida, y las reglas de autorización y aislamiento por tenant ya
habituales en el módulo.

La segunda etapa incorporó los tres atributos de política que gobiernan a qué
pacientes atiende cada profesional y qué ocurre con sus turnos liberados: el
filtro de edad, que restringe la atención a personas adultas; el indicador de
apertura o cierre de la admisión de pacientes nuevos; y la modalidad de
reasignación de un turno cancelado, que puede ser automática —el sistema ofrece
la franja liberada a los pacientes en lista de espera según un orden de
prioridad— o manual —la franja queda reservada para que el profesional la
asigne—. A diferencia de la etapa anterior, esta no requirió modificar el
esquema de datos: los tres campos, junto con el enumerado de modalidad de
reasignación, se habían incorporado ya en el modelado inicial del módulo con sus
valores por defecto —sin filtro de edad, admisión abierta y reasignación
automática—, de modo que el modelo de persistencia ya reflejaba la fuente de
verdad y lo que restaba era habilitar su edición. El trabajo se concentró, por
tanto, íntegramente en la capa de entrada.

La decisión de diseño de esta etapa fue consolidar esos atributos en el endpoint
de configuración ya existente en lugar de exponer uno nuevo para la política de
admisión. Se prefirió la consolidación porque los tres campos son, igual que los
de agenda, atributos de la propia entidad profesional que su titular ajusta
desde la aplicación y que se rigen por el mismo criterio de acceso; abrir una
ruta por subconjunto temático de campos habría fragmentado una operación única
—el profesional ajusta su configuración— y obligado a duplicar el guard de
propiedad, el acotamiento por tenant y el registro de auditoría sin ganancia de
cohesión. Se mantuvo, en cambio, la separación ya establecida entre los datos
identitarios del profesional y su configuración, que continúan en endpoints
distintos. Las validaciones siguieron el criterio del módulo: los dos
indicadores se validan como booleanos y la modalidad como uno de los dos valores
de su enumerado, todo en el DTO por tratarse de restricciones de forma sobre
campos aislados.

La semántica de modificación parcial, ya adoptada en la etapa anterior, adquirió
aquí una relevancia adicional. Como el indicador de admisión de pacientes nuevos
tiene por defecto el valor verdadero, resulta imprescindible que un valor falso
explícito se distinga de la ausencia del campo en la petición: los campos no
enviados se propagan a la capa de persistencia como indefinidos, interpretados
como ausencia de cambio, mientras que los presentes se escriben con su valor. De
ese modo, cerrar la admisión de pacientes nuevos no obliga a reenviar la
duración de la consulta ni la franja extra, y no puede confundirse con no haber
enviado nada.

Cabe subrayar, como en la etapa precedente, el límite del alcance: esta etapa
únicamente persiste y expone la configuración. La validación de la edad del
paciente durante la conversación —que solicita documento y fecha de nacimiento
solo a quienes no están registrados y reutiliza la fecha ya almacenada para los
demás— y la lógica de reasignación que da sentido a cada modalidad, con las
ventanas de espera que cada una implica, corresponden a módulos posteriores.

La verificación incorporó, junto a las pruebas unitarias de persistencia de cada
modalidad y de los dos indicadores, y a las pruebas end-to-end de los criterios
de aceptación —modificación con filtro de edad activo, admisión cerrada y cada
modalidad de reasignación, rechazo de una modalidad desconocida y de valores no
booleanos, y las reglas de autorización y aislamiento ya habituales—, una prueba
específica para el requisito de que el cierre de la admisión surta efecto de
forma inmediata. Dado que la capa conversacional obtiene los profesionales
disponibles a través del listado del módulo antes de ofrecer turnos, la prueba
modifica el indicador y verifica que la lectura siguiente del listado ya lo
refleja, comprobando lo propio al reabrir la admisión; se traduce así un
requisito enunciado sobre el comportamiento de un módulo futuro en una garantía
verificable sobre la interfaz que ese módulo consumirá.

El módulo se cerró con dos tareas de consolidación: la carga de datos de
desarrollo que reproduce el plantel del piloto y el completamiento de la
cobertura de pruebas. La fuente de requisitos indica que la clínica cuenta
actualmente con cuatro psiquiatras y una psicóloga, y que no incorpora nuevos
profesionales con frecuencia; ese plantel es el que reproduce el sembrado de
datos, con cada profesional acompañado de su especialidad, sus dos matrículas
—provincial y profesional, tal como el chatbot las exhibe—, una grilla semanal
de horarios de atención y la configuración base completa de las dos etapas
anteriores. Todos los datos son ficticios y deliberadamente reconocibles como
tales, a la espera de que el cliente provea los reales. La configuración se
distribuyó de forma heterogénea a lo largo del plantel —duraciones de consulta
que reproducen los dos ejemplos de la fuente de requisitos, franjas extra de una
y dos horas, las tres modalidades de ubicación de esa franja, las dos
modalidades de reasignación, y profesionales con la admisión cerrada y con el
filtro de edad activo—, de modo que una base recién sembrada ejercite todas las
ramas que los módulos de agenda y de conversación deberán manejar, en lugar de
un único caso homogéneo que ocultaría errores en las ramas no representadas.

El requisito de idempotencia del sembrado se satisfizo con una garantía más
fuerte que la enunciada. No basta con que una segunda ejecución no duplique
filas: como ninguna de las entidades involucradas posee una clave de negocio
natural sobre la que insertar o actualizar, los identificadores deben fijarse
explícitamente, y al cambiar ese esquema de identificadores —situación que se
produjo en esta misma tarea— las filas de la versión anterior sobrevivirían
junto a las nuevas, duplicando la colección de cada profesional. Se optó por un
sembrado convergente: además de insertar o actualizar cada fila sobre un
identificador estable, las colecciones hijas de cada profesional se reconcilian,
eliminando toda fila que ya no figure en la declaración. El sembrado converge
así siempre al mismo estado, incluso después de que sus propios datos cambien.
Los identificadores, por su parte, se derivan de un espacio de nombres por
entidad y un número de secuencia en lugar de enumerarse literalmente, decisión
que resultó necesaria al incorporar una grilla semanal de horarios por
profesional, cuyo volumen vuelve impracticable la enumeración explícita.

El completamiento de la cobertura se abordó relevando primero lo ya verificado.
Ese relevamiento mostró que la mayor parte de los casos enunciados —alta,
consulta, edición y baja lógica de profesionales, tope de matrículas,
aislamiento por tenant en la consulta individual y reglas de rol y propiedad— ya
estaba ejercitada desde la construcción de la capa de negocio del módulo, de
modo que se prefirió no duplicar pruebas equivalentes y concentrar el esfuerzo
en los huecos reales. Estos resultaron ser la edición y la eliminación de una
matrícula, el rechazo por falta de propiedad sobre las tres rutas de matrículas,
y el aislamiento por tenant del listado, complemento de la garantía ya
verificada sobre la consulta individual. La idempotencia del sembrado, en
cambio, se verificó manualmente y no mediante una prueba automatizada, dado que
ejecutarlo desde el conjunto de pruebas introduciría un efecto colateral sobre
la base de datos compartida que las demás pruebas —que crean sus propias
organizaciones aisladas— no controlan.

Una revisión integral del módulo, realizada una vez completadas sus tareas,
puso de manifiesto una decisión de diseño que convenía revisar. La consulta de
profesionales devolvía la especialidad y las matrículas, pero no la grilla de
horarios de atención, accesible únicamente a través de su sub-recurso dedicado.
Dado que el motor de agenda y la capa conversacional necesitan esa grilla para
determinar qué turnos ofrecer, la separación los habría obligado a consultar el
listado y luego, por cada profesional, su grilla, incurriendo en el patrón de
consulta conocido como N+1. Se optó por incorporar la grilla a la definición
compartida de las relaciones que acompañan al profesional, ordenada por día de
la semana y hora de inicio para que los consumidores puedan confiar en ese
orden. El criterio que sostiene la decisión es el volumen: la grilla es acotada
—unos pocos bloques semanales—, de modo que transportarla resulta más económico
que la consulta adicional que evita; el sub-recurso dedicado se conserva, pues
sigue siendo la vía para reemplazarla.

Esa corrección puso al descubierto una segunda, de naturaleza distinta. Los
servicios de los recursos anidados anclaban cada operación en el profesional
padre invocando el método que lo carga junto con todas sus relaciones, cuando
únicamente necesitaban la garantía de que existe y pertenece al tenant del
solicitante; las relaciones cargadas se descartaban. La ineficiencia era
tolerable mientras se limitaba a la especialidad y las matrículas, pero al
sumarse la grilla de horarios toda operación sobre una matrícula o una ausencia
habría pasado a cargarla sin usarla. Se introdujo entonces una verificación de
pertenencia liviana, con idéntica semántica de no encontrado pero sin cargar
relación alguna, y se la adoptó en los puntos donde la entidad completa no se
utilizaba. El episodio ilustra un riesgo propio del trabajo incremental: una
mejora local puede convertir una ineficiencia latente en un problema efectivo si
no se revisa el efecto conjunto sobre el resto del módulo.
