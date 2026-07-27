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

Cerrado el primer módulo de negocio y antes de abordar los siguientes, se
revisó el esquema completo contra tres criterios explícitos —integridad,
normalización y relaciones lógicas— con el objeto de asentar sobre una base
correcta el trabajo restante. La revisión encontró que el esquema satisfacía
el aislamiento entre organizaciones en tiempo de ejecución pero no lo
sostenía en la base de datos misma: ninguna de las referencias a la
organización era una clave foránea real, sino un identificador universal sin
restricción alguna. La base aceptaba, en consecuencia, filas que apuntaban a
organizaciones inexistentes. Que la carencia no fuera hipotética se comprobó
al declarar las restricciones: una suite de pruebas existente creaba usuarios
contra un identificador de organización inventado, sin fila que lo
respaldara, y venía pasando en verde.

La segunda observación afectó a una decisión de diseño previa. El esquema
replicaba deliberadamente el identificador de organización en las entidades
hijas —horarios de atención y ausencias, que pertenecen a un profesional que
ya lo lleva— con el argumento de que así toda consulta se acota sin unir
tablas. Se revirtió ese criterio. El costo de la réplica es un valor capaz de
discrepar del de su padre, y esa discrepancia constituye una corrupción que
ninguna consulta puede detectar, puesto que la fila pertenece a una
organización según su propia columna y a otra según su padre. El beneficio,
en contrapartida, es una unión evitada sobre una tabla indexada, en un
sistema cuyo volumen previsto es el de una clínica. Se adoptó, por tanto, el
criterio de que el identificador de organización reside en las entidades que
pertenecen al inquilino de forma directa y carecen de padre por el cual
alcanzarse, y nunca en aquellas que sí lo tienen, las cuales se consultan
anclando la operación en la verificación de pertenencia de su padre. Queda
consignado que una desnormalización controlada podrá evaluarse ante un
problema de rendimiento medido, y no de forma preventiva.

El caso en que el identificador de organización debe permanecer se resolvió
mediante claves foráneas compuestas: cuando una entidad lo conserva y además
referencia a otra entidad acotada por inquilino, la clave se declara sobre el
par formado por la organización y el identificador destino, contra una
restricción de unicidad equivalente en el destino. La base rechaza entonces
por sí misma todo vínculo entre organizaciones distintas —un profesional con
una especialidad ajena, una cuenta vinculada a un profesional ajeno, una
entrada de auditoría atribuida a un usuario ajeno— en lugar de confiar en que
cada camino de servicio recuerde comprobarlo. Esta construcción es lo que
vuelve segura, y no meramente cómoda, la permanencia del identificador donde
resulta necesaria: el registro de auditoría, cuya referencia a la entidad
auditada es polimórfica y por ello irreconstruible mediante uniones.

La revisión corrigió, por último, un error de modelado en el catálogo de
obras sociales, que estaba acotado por organización. Una organización no
tiene obras sociales: estas existen con independencia de ella, y lo propio
del dominio es qué obras sociales acepta cada profesional. El catálogo
almacenaba así la misma entidad una vez por organización y dejaba, aun así,
sin responder la pregunta relevante. Se lo convirtió en catálogo global con
nombre único y se incorporó una relación opcional de muchos a muchos entre
profesional y obra social. El catálogo de especialidades, en cambio, se
mantuvo acotado por organización, dado que la nomenclatura de especialidades
sí es propia de cada clínica.

La corrección del esquema dejó, sin embargo, esa relación sin ninguna vía de
acceso desde la API. Se cerró el hueco en la misma intervención, exponiendo el
catálogo global mediante un endpoint de solo lectura independiente del módulo
de Profesionales, y la aceptación por profesional mediante un recurso anidado
que sigue el mismo patrón de reemplazo completo ya usado para la grilla de
horarios: un `PUT` idempotente que declara el conjunto entero de obras
sociales aceptadas, con el arreglo vacío como valor válido para un profesional
que solo atiende de forma particular. A diferencia de la grilla de horarios,
la validez del conjunto nuevo no depende del estado anterior, por lo que el
reemplazo no requiere aislamiento serializable. El conjunto aceptado se
incorporó además a la respuesta del profesional, por la misma razón que ya
llevó a incluir la grilla de horarios: evitar que la capa conversacional, al
filtrar profesionales por obra social, deba resolver una consulta N+1.

Una segunda revisión sistemática de código, realizada al cerrar la
implementación de los módulos de Profesionales y Pacientes, reforzó las
fundaciones en cuatro puntos. El más relevante afecta a la extensión que aplica
el filtrado por organización a cada consulta: no contemplaba dos operaciones que
la biblioteca de acceso a datos ofrece —la actualización y la creación múltiples
con retorno de filas—, y la primera no falla de forma segura ante la ausencia de
filtro, de modo que una consulta de ese tipo habría podido leer y modificar filas
de todas las organizaciones. Como la extensión es la barrera única sobre la que
descansa la garantía de aislamiento, la corrección se aplicó allí y no en cada
consumidor, cubriendo además el caso de una consulta sin argumentos que provocaba
un error en tiempo de ejecución en vez de devolver el resultado ya filtrado. En
el inicio de sesión se cerró un canal lateral de tiempo que permitía distinguir un
correo registrado de uno inexistente por la latencia de la respuesta, igualando el
trabajo de ambos caminos aun cuando el mensaje de error ya era idéntico. Se
rediseñó el contrato del puerto que abstrae la cerradura electrónica: identificaba
cada código por el valor del PIN y no permitía indicar sobre qué cerradura operar,
por lo que no habría permitido invalidar el código anterior al reprogramar o
cancelar un turno —lo que la especificación del control de acceso exige—; el
contrato revisado devuelve un identificador opaco junto con el PIN y recibe el
identificador de la cerradura en cada operación, corregido mientras el único
implementador es todavía el adaptador de prueba y antes de construir la
integración real. Por último, se conciliaron la convención de nomenclatura del
repositorio y el código: los métodos de los puertos de integración se nombran en
inglés, como el resto de los identificadores, y el término del glosario en español
se conserva en un comentario sobre cada puerto.

Un repaso posterior de tres campos del esquema, motivado por preguntas sobre su
sentido, consolidó dos convenciones transversales del modelo de datos. La primera
atañe a la tabla de auditoría: identifica el registro afectado por cada acción con
un puntero genérico —el nombre del tipo de entidad como texto y el identificador de
la fila—, porque una sola tabla registra acciones sobre entidades de tipos distintos
y no puede sostener una clave foránea real hacia cada una; además de ese puntero,
cuando la acción concierne a un paciente se guarda también una clave foránea real
hacia el paciente. Esa duplicación es deliberada y quedó documentada como tal: el
puntero genérico obligaría a la consulta central de cumplimiento —reconstruir todo lo
actuado sobre un paciente, exigido por la protección de datos personales— a
interpretar cadenas de texto a través de todos los tipos de entidad, mientras que la
clave foránea real la resuelve como una unión indexada que la base valida y acota a la
misma organización. La segunda convención unifica la forma de la baja lógica: en
lugar de un indicador booleano de actividad, profesionales y pacientes llevan una
marca temporal anulable cuya ausencia significa activo y cuyo valor registra el
instante de la baja, de modo que la traza conserva no sólo que un registro fue dado de
baja sino cuándo, y la reactivación se expresa como el borrado de esa marca; se
descartó el par booleano-más-fecha por requerir una coherencia entre dos columnas que
una sola marca temporal vuelve imposible de violar. La migración correspondiente
preserva el estado previo, convirtiendo cada fila inactiva en una marca temporal para
no perder ninguna baja registrada.

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
identificador estable, se elimina todo lo que el sembrado creó antes y ya no
declara. El sembrado converge así siempre al mismo estado, incluso después de
que sus propios datos cambien.

Conviene dejar constancia de que esa convergencia se alcanzó en dos pasos, y que
el primero fue incompleto. La reconciliación se aplicó inicialmente solo a las
colecciones hijas de cada profesional, de modo que retirar a un profesional del
plantel declarado dejaba su registro activo en la base mientras sus matrículas y
horarios sí desaparecían: un profesional que la capa conversacional ofrecería
sin disponibilidad, resultado peor que el de no reconciliar nada. La revisión
posterior del módulo lo detectó y la corrección extendió la reconciliación a los
profesionales, acotándola al espacio de identificadores y al rango de secuencia
que el propio sembrado utiliza, para que un profesional creado a través de la
interfaz durante el desarrollo no resulte nunca eliminado. El episodio ilustra
un riesgo específico de la documentación técnica escrita junto con el código: la
afirmación de convergencia se redactó a partir de la intención del diseño y no
de su alcance efectivo, y sobrevivió sin ser cuestionada hasta que una revisión
independiente contrastó el texto con la implementación.
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

Cerrado el módulo, se lo sometió a una revisión más exhaustiva, conducida por
varios revisores independientes con enfoques deliberadamente distintos: el
aislamiento entre organizaciones y la autorización, la conformidad con las
convenciones arquitectónicas declaradas, la correctitud funcional y la calidad de
las pruebas. El procedimiento resultó productivo precisamente por la diversidad
de enfoques: los defectos hallados no se solapan entre sí, y ninguno de ellos era
detectable desde la perspectiva de los demás.

El más grave era de validación de entrada. La biblioteca empleada para validar
los datos recibidos omite todas las restricciones de un campo cuando su valor es
nulo, y no únicamente cuando el campo está ausente; combinado con una tubería de
validación que descarta solo las propiedades no declaradas, ello permitía que un
campo enviado explícitamente en nulo atravesara la validación intacto y alcanzara
la capa de persistencia, que lo rechazaba contra una columna no nulable con un
error no controlado. Lo notable del caso es que la corrección no podía ser
uniforme: ese mismo comportamiento constituía la única vía por la que podía
restablecerse a nulo un parámetro de agenda ya configurado, capacidad que
funcionaba de manera accidental y no declarada. Suprimir el nulo de forma global
la habría eliminado sin que nada lo advirtiera. Se optó por obligar a cada campo
opcional a declarar explícitamente cuál de los dos casos representa —si admite el
nulo como instrucción de borrado o si debe rechazarlo—, con lo que una ambigüedad
del comportamiento por defecto se convirtió en una decisión visible en el código.

Dos invariantes que abarcan una lectura seguida de una escritura resultaron
además desprotegidos frente al acceso concurrente. El nivel de aislamiento
transaccional por omisión admite que dos peticiones simultáneas lean un estado
que autoriza la escritura y ambas escriban, produciendo un estado que ninguna de
las dos habría permitido individualmente. En el reemplazo de la grilla semanal
ello persistía la unión de dos grillas, solapada, que es justamente lo que la
validación de solapamientos existe para impedir; en el tope de matrículas,
permitía superar el máximo. Ambos se resolvieron elevando el aislamiento de la
transacción a serializable, de modo que la operación perdedora se aborta y el
conflicto se informa explícitamente en lugar de corromper los datos en silencio.
En la misma línea, las entradas de la traza de auditoría se escribían fuera de la
transacción de la mutación que documentan, de manera que un fallo posterior a la
confirmación del cambio dejaba ese cambio sin registrar —y, tratándose de
eliminaciones, sin posibilidad de reconstruir su autoría—. Dado que la traza
responde a una obligación legal, se incorporó el registro a la misma transacción
que la mutación.

La revisión alcanzó también a la documentación. Se constató que la separación en
capas de dominio, aplicación e infraestructura que declaraban las convenciones
del repositorio no la implementa ningún módulo del sistema: el directorio
destinado al dominio contiene únicamente puertos de integración y el de
infraestructura únicamente sus adaptadores. La porción del patrón que sí se
aplica de manera consistente es la de puertos y adaptadores en los límites de
integración externa, que es donde aporta valor efectivo. Se resolvió corregir el
documento antes que reestructurar el código: con una única tecnología de
persistencia y sin previsión de sustituirla, introducir entidades de dominio y
sus conversores constituiría costo sin beneficio, mientras que un documento de
convenciones que describe una realidad inexistente induce a desatenderlo por
completo, incluidas las prescripciones que sí resultan críticas, como el
acotamiento por organización de toda consulta.

El hallazgo funcionalmente más grave, sin embargo, afectaba a las ausencias, y
resulta ilustrativo de cómo una decisión de modelado aparentemente inocua puede
anular una funcionalidad completa. Las fechas de la ausencia se habían modelado
como marcas temporales, de modo que una ausencia de un solo día se almacenaba
con el mismo instante como inicio y como fin: un intervalo de longitud nula. Un
consumidor que preguntara si un turno cae dentro de ese intervalo no bloquearía
ninguno, con lo que el profesional declaraba un día libre y su agenda permanecía
intacta. A ello se sumaba que, por corresponder la clínica a la zona horaria
UTC-3, toda fecha se representaba un día antes de la declarada.

La corrección consistió en aplicar a las ausencias el mismo criterio que el
propio sistema ya había adoptado para los horarios de atención, y por idéntica
razón: una ausencia es un hecho de calendario, no un instante. Las fechas pasaron
a expresarse como días de calendario inclusivos, almacenados sin hora ni zona
horaria, de manera que una ausencia de un día bloquea efectivamente ese día. Se
descartó la alternativa de normalizar las marcas temporales al comienzo y al fin
del día en la zona horaria de la clínica, por requerir aritmética de husos
horarios y dejar el valor expuesto a la misma clase de error. La coherencia con
la convención previa no es casual: ambos casos son valores de reloj o de
calendario tal como los lee la institución, y el proyecto ya había aprendido, al
modelar los horarios, que atarles un instante introduce una precisión que el
dominio no posee y una ambigüedad que sí padece.

Al redactar las pruebas de esa corrección se descubrió un defecto adicional que
ninguna de las revisiones había anticipado: la validación del formato de fecha
verificaba la forma de la cadena pero no la existencia del día, de modo que una
fecha imposible como el 31 de febrero superaba la validación y la conversión
posterior la desplazaba en silencio al 3 de marzo, registrando un día que el
solicitante nunca había pedido. La observación es metodológicamente relevante:
la prueba escrita para verificar una corrección terminó revelando un defecto
distinto y preexistente, lo que refuerza el valor de acompañar cada corrección
con su verificación automatizada antes que confiar en la inspección del código.

Antes de avanzar hacia los módulos posteriores se sometió el código ya construido
a una revisión sistemática, contrastándolo desde varios ángulos independientes
—el modelo de datos, la conformidad funcional con la especificación, y la calidad,
la seguridad y la concurrencia del código— contra las fuentes de verdad del proyecto
y contra los tres pilares del modelo de datos. La revisión confirmó que el esquema
satisface la integridad, la normalización y las relaciones lógicas sin cambios
estructurales, y produjo un conjunto acotado de correcciones de comportamiento y de
contrato en el módulo de Profesionales, cada una acompañada de su verificación
automatizada. La primera unifica el tratamiento de la fecha de confirmación de la
incorporación del profesional con el del resto de las fechas de calendario del
sistema: se la había modelado como un instante con marca temporal, lo que reintroduce
el desplazamiento de un día en la zona horaria de la clínica que la convención de
fechas del proyecto evita, de modo que se la llevó al tipo fecha sin hora, con la misma
validación de día existente y la misma conversión canalizada por el único módulo
autorizado a cruzar esa frontera. La conversión de una fecha de calendario opcional y
anulable se encapsuló, además, en una función que preserva la distinción entre el valor
ausente —que no modifica la columna— y el nulo —que la limpia—, una diferencia que la
expresión ingenua de conversión colapsaría, dejando de ser posible borrar la fecha.

La segunda corrección extiende al alta de ausencias la misma disciplina de concurrencia
ya adoptada para el reemplazo de la grilla de horarios. La verificación de que una
ausencia no se solapa con otra ya registrada es un invariante de lectura seguida de
escritura que corría bajo el nivel de aislamiento por defecto, de modo que dos altas
concurrentes con rangos solapados podían verificar cada una la ausencia de conflicto
antes de que la otra confirmara, y ambas persistir; se elevó la operación al nivel de
aislamiento serializable, con la verificación de solapamiento dentro de la misma
transacción que la inserción, de manera que la transacción perdedora de la carrera aborta
y se traduce en una respuesta de conflicto en lugar de una corrupción silenciosa. La
misma revisión motivó una tercera corrección menor —acotar la franja horaria extra para
pacientes nuevos a un máximo de dos horas, en correspondencia con los valores que la
especificación ejemplifica— y una cuarta, transversal al almacén de configuración por
inquilino, donde la secuencia de buscar y luego crear un parámetro se reemplazó por una
única operación atómica de alta-o-actualización sobre la restricción de unicidad de la
clave, eliminando un fallo interno que dos escrituras concurrentes de una misma clave
nueva podían provocar. Estas correcciones se acompañaron de pruebas de concurrencia que
someten cada invariante a una carrera real y afirman su estado final consistente —una
sola grilla, una sola ausencia por período, nunca más de tres matrículas— antes que un
desenlace temporalmente determinista, dado que cuál transacción gana la carrera depende
del planificador y no del comportamiento que se prueba.

Una revisión posterior, ya al cierre de la etapa, completó la configuración de agenda
del profesional con un dato que hasta entonces faltaba y reconsideró la cota fijada más
arriba. La especificación distingue dos parámetros que la configuración inicial colapsaba
en uno solo: la cadencia con que se abren los turnos en la agenda —cada cuánto tiempo hay
un horario ofrecible— y la duración de la sesión, que es la información que el asistente
comunica al paciente al confirmar el turno. Colapsarlos impedía expresar el propio ejemplo
de la fuente de verdad —atención cada una hora con sesiones de cuarenta y cinco minutos—,
por lo que se agregó la cadencia como un dato propio de cada profesional, distinto de la
duración, nulo hasta que se configura y destinado a alimentar la futura generación de
agenda. En cuanto a la franja horaria extra para pacientes nuevos, la cota de dos horas
introducida por la revisión anterior se revisó a la baja de exigencia: la especificación
enumera esos valores a modo de ejemplo y deja la magnitud abierta, de modo que el límite
dejó de interpretarse como una regla de negocio y pasó a ser una cota amplia de contención.
La misma revisión unificó una definición duplicada de la representación de una obra social,
que existía por partida doble en la capa de presentación de Profesionales y en la de Obras
Sociales, importándola desde su lugar canónico para que ambas vistas no puedan divergir.

Un ajuste posterior retiró por completo la fecha de confirmación de la incorporación del
profesional, cuyo tratamiento como día de calendario se describió más arriba. El campo se
almacenaba y se devolvía en la respuesta, pero ninguna regla de negocio llegó a consumirlo,
de modo que se lo eliminó del esquema, de los objetos de transferencia y de la respuesta,
por no aportar función alguna en el estado actual del sistema; la disciplina de fechas de
calendario que aquel tratamiento ilustra sigue vigente para el resto de las fechas del
dominio —ausencias, fecha de nacimiento del paciente—, sólo que ya no se aplica a un campo
que dejó de existir. La representación de la baja lógica del profesional, por su parte, pasó
a la marca temporal anulable descripta en 3.2.0, común a profesionales y pacientes.

### 3.2.2 Pacientes

El segundo módulo de negocio siguió el mismo criterio de apertura que el
anterior: antes de construir la gestión, se estabilizó el modelo de datos. Se
incorporaron al esquema cuatro entidades tomadas del diagrama entidad-relación
y del documento de especificación de requisitos: el paciente, el vínculo entre
un paciente y cada profesional que lo atiende, el consentimiento de tratamiento
de datos y la solicitud de receta. La tarea se acotó nuevamente al esquema y su
migración, sin endpoints ni servicios, que corresponden a las tareas siguientes
de la fase.

La primera decisión de modelado atiende a un requisito explícito: el sistema
identifica al paciente por su número de documento. La unicidad se declaró, sin
embargo, sobre el par formado por la organización y el documento, y no sobre el
documento solo. En un sistema diseñado para marca blanca la misma persona puede
ser paciente de dos organizaciones distintas, y cada una debe poder mantener su
propio registro; una unicidad global habría hecho que el alta en una impidiera
el alta en la otra, convirtiendo una coincidencia legítima en un conflicto.

La segunda atiende a la ubicación de los datos que varían según quién observe
al paciente. El diagrama sitúa en la tabla de vínculo la prioridad, las
observaciones de uso interno, el correo, el contacto de emergencia, el
indicador de primera sesión, el tipo de paciente y la fecha de última consulta,
y esa ubicación se respetó porque expresa un hecho del dominio y no una
comodidad de implementación: una misma persona puede ser paciente nuevo para un
profesional y recurrente para otro, con prioridad y observaciones distintas en
cada caso. Alojar esos campos en el paciente habría obligado a elegir
arbitrariamente cuál de las descripciones prevalece.

La tercera obligó a precisar la regla de normalización que el proyecto había
adoptado en la revisión del modelo de datos, según la cual el identificador de
organización no se replica en ninguna entidad alcanzable a través de un padre
que ya lo posee. El vínculo entre paciente y profesional no tiene un padre sino
dos, ambos acotados por organización, y ese caso no estaba contemplado. El
criterio adoptado es que la columna se conserva cuando restringe algo que
ningún padre determina por sí solo. Con un único padre la columna es una
réplica pura y se omite, como en el consentimiento. Con dos padres, en cambio,
esa única columna es lo que obliga a que ambos pertenezcan a la misma
organización, restricción que ningún par de claves foráneas independientes
puede expresar; y, al ser leída por las dos claves foráneas compuestas en cada
escritura, tampoco puede divergir de ninguno de los dos padres, que era
precisamente el riesgo invocado para prohibir la réplica. La base de datos
rechaza así por sí misma el estado en que un paciente de una organización
aparece tratado por un profesional de otra, sin depender de que cada camino de
servicio recuerde verificarlo.

El mismo razonamiento se aplicó a la solicitud de receta, con una consecuencia
que se aparta de la letra del diagrama. Éste dibuja dos claves foráneas
independientes, hacia el paciente y hacia el profesional; se optó en cambio por
una única clave foránea compuesta contra el vínculo, que resuelve a la vez tres
cuestiones que las dos flechas dejaban abiertas: que ambas filas pertenezcan a
una organización, que pertenezcan a la misma, y que el paciente esté
efectivamente en tratamiento con el profesional a quien pide la receta, que es
el único caso que los requisitos describen. Como efecto derivado, la solicitud
no necesita identificador de organización propio, porque el del vínculo al que
apunta lo determina.

El consentimiento de tratamiento de datos, exigido por la Ley 25.326, se modeló
como una tabla de solo agregado: carece de columna de última modificación, y una
revocación se registra como una fila nueva en lugar de editar la anterior, de
modo que el historial de qué se aceptó y cuándo permanece íntegro. La marca
temporal del consentimiento se mantuvo en esta instancia separada de la de
creación de la fila, con el propósito de admitir consentimientos firmados en
papel sin falsear la fecha del acto; la tarea que implementó el comportamiento
del consentimiento revisó esa previsión y la revirtió, según se detalla más
adelante en esta misma subsección.

La tarea salda además una deuda que el propio esquema venía declarando. La traza
de auditoría reservaba desde su creación una columna para referenciar al
paciente, que hasta ahora era un identificador sin clave foránea porque la tabla
destino no existía; al existir, se la convirtió en clave foránea compuesta real.
La elección del comportamiento ante el borrado del paciente resultó
significativa: una clave foránea compuesta no admite anular la referencia,
porque anularía también el identificador de organización, que es no nulable, y
propagar el borrado destruiría la traza que documenta lo actuado sobre ese
paciente. Se optó por restringirlo, con la consecuencia deliberada de que un
paciente con historial de auditoría no puede eliminarse físicamente. Lejos de
constituir una limitación, ésa es la forma adecuada para datos de salud bajo la
norma citada: un pedido de supresión se atiende anonimizando el registro del
paciente, mientras la traza que acredita que la supresión ocurrió debe
sobrevivirle.

Por último, el carácter obligatorio de cada campo se decidió por la realidad del
dato y no por su deseabilidad. Sólo el documento, el nombre y el apellido son
obligatorios; la clave tributaria, la fecha de nacimiento y el teléfono celular
se declararon opcionales porque la migración de pacientes preexistentes
importará registros que hoy viven en planillas de cálculo y en papel, donde esos
datos suelen faltar. La exigencia de documento, fecha de nacimiento y contacto
de emergencia que fijan los requisitos es una regla del flujo de reserva de
turnos, no una razón para rechazar un registro histórico incompleto. La edad,
que los requisitos enumeran entre los datos filiatorios, no se almacena sino que
se deriva de la fecha de nacimiento, de modo que no puede quedar desactualizada.
Las fechas de nacimiento y de última consulta se modelaron como fechas de
calendario, sin hora ni huso horario, siguiendo la convención adoptada para las
ausencias, mientras que el consentimiento y la solicitud de receta se modelaron
como instantes, porque la norma se interesa por el momento del consentimiento y
porque los requisitos restringen el pedido de recetas al horario laboral, donde
la hora del día forma parte del hecho.

La verificación se realizó mediante pruebas de integración contra la instancia
local de PostgreSQL, planteadas de modo que cada garantía declarada en el
esquema quede demostrada como restricción efectiva de la base de datos y no como
intención documentada. Se comprueba el rechazo de un documento repetido dentro
de una organización junto con su admisión en otra, la independencia de las
observaciones y la prioridad entre dos vínculos del mismo paciente, el rechazo
del vínculo que cruza organizaciones reclamando la organización de cualquiera de
los dos lados, el rechazo de una solicitud de receta dirigida a un profesional
que no trata al paciente, el carácter acumulativo del historial de
consentimientos y la propagación en cascada del borrado del paciente sobre sus
entidades dependientes. El diagrama entidad-relación acotado a este subdominio
queda pendiente de incorporación como figura.

Estabilizado el modelo, la tarea siguiente construyó sobre él la gestión
propiamente dicha: el alta, la modificación, la baja y la consulta de pacientes,
la búsqueda por número de documento dentro de la organización, la administración
del vínculo con cada profesional tratante y un punto de consulta que responde
las dos reglas que los requisitos imponen antes de reservar un turno. El módulo
expone los datos que el flujo de reserva necesitará; la conversación que los
usará pertenece a fases posteriores.

La baja exigió una columna que el diagrama no contempla. Los requisitos piden
que sea lógica, y el diagrama no asigna al paciente ningún atributo de ciclo de
vida, de modo que se agregó una marca de estado activo con el mismo criterio ya
aplicado al profesional. La alternativa de eliminar la fila resultaba inviable
por dos razones convergentes, y ambas provienen del propio esquema: los vínculos
de tratamiento y los consentimientos se eliminarían en cascada, y la traza de
auditoría restringe por sí misma el borrado de un paciente que menciona. El
esquema, en otras palabras, ya había decidido que un paciente no se elimina; la
columna se limita a darle al módulo la forma de expresarlo. La reactivación se
resolvió como una modificación ordinaria de esa marca y no como un alta nueva,
porque un paciente que regresa tras años es la misma persona con el mismo
historial, y un alta chocaría además con la unicidad del documento.

La construcción del módulo obligó, además, a revisar una ubicación heredada del
diagrama entidad-relación. El contacto de emergencia y el correo electrónico
estaban modelados en el vínculo paciente-profesional, tal como el diagrama los
dibuja, y la tarea anterior los había respetado por fidelidad a esa fuente. Al
implementar la gestión se hizo evidente que la ubicación era incorrecta: se trata
de datos de la persona, que el propio documento de requisitos enumera entre los
datos de contacto del paciente y que una reserva exige con independencia del
profesional con quien sea. Sostener una copia por profesional tratante dejaba la
pregunta por el contacto de emergencia de un paciente con tantas respuestas
posibles como profesionales lo atendieran, y sin ninguna regla para elegir entre
ellas —exactamente el defecto que las restantes decisiones de modelado se habían
ocupado de evitar—. Ambos campos se trasladaron a la entidad Paciente mediante
una migración que copia los valores existentes antes de eliminar las columnas,
conservando, cuando hay varios vínculos con dato cargado, el del vínculo
modificado más recientemente. En el vínculo permanece únicamente aquello que sí
varía según quién observe al paciente: la prioridad que ese profesional asigna,
sus observaciones de uso interno, el tipo de paciente que es para él, el
indicador de primera sesión y la fecha de la última consulta. La divergencia
respecto del diagrama es deliberada y queda documentada como tal, tanto en el
esquema como en la bitácora de la tarea.

Como consecuencia de ese traslado, los tres datos que los requisitos exigen para
reservar —documento, fecha de nacimiento y contacto de emergencia— son atributos
del paciente y se validan en un único lugar, el alta. Sus columnas siguen
admitiendo nulos para que la migración de registros preexistentes pueda importar
legajos incompletos: la obligatoriedad reside en el contrato del endpoint
interactivo y no en la columna, asimetría que se documentó de forma explícita
para que no se interprete como un descuido. La vinculación con un profesional
queda entonces identificada por completo por la dirección del recurso, sin
cuerpo obligatorio.

La vinculación de un paciente con un profesional se definió como idempotente. El
vínculo queda identificado por completo por la dirección del recurso, y quien
más necesita esa operación, el asistente conversacional que toma una reserva, no
puede saber de antemano si el paciente ya estaba vinculado. Bajo la lectura
estricta del verbo de creación, ese desconocimiento obligaría a consultar antes
de vincular y a tratar el conflicto como caso corriente, siendo que para un
paciente recurrente es precisamente la situación normal. Una segunda invocación,
por tanto, no falla: aplica lo que el cuerpo haya traído y devuelve el vínculo
existente, respondiendo con el código de éxito ordinario en lugar del de
creación, porque la respuesta describe el estado del vínculo y no afirma haberlo
creado en esa invocación. El indicador de primera sesión conserva el valor por
defecto de la columna al crearse y no se altera al repetir la operación: ese
valor por defecto es, precisamente, la regla según la cual el sistema debe
validar si es la primera vez que el paciente se atiende con ese profesional, y
reafirmarlo desde el servicio le daría un segundo lugar desde el cual divergir.
La traza de auditoría sí distingue creación de actualización, de modo que la
diferencia que la respuesta deliberadamente no expone no se pierde.

El plazo de un año tras el cual el sistema solicita al paciente la actualización
de sus datos de contacto se trató como configuración de la organización y no
como una constante del código, en continuidad con el criterio de reglas de
negocio como datos adoptado en las fases anteriores. Se optó por una única clave
de configuración, aun cuando gobierna dos reglas distintas —la actualización de
datos y la reclasificación del paciente como nuevo, esta última correspondiente
a la tarea siguiente—, porque dos claves independientes admitirían valores
divergentes y producirían un paciente al que se le piden datos actualizados
mientras se lo sigue tratando como recurrente. Un valor configurado que no sea un
número entero positivo se descarta en favor del valor por defecto, dado que la
configuración se almacena como documento JSON y ninguna otra verificación se
interpone entre un error de tipeo y una fecha de corte carente de sentido.

La fila con el valor por defecto se crea mediante una migración de datos y no
únicamente en el seed, distinción que resultó relevante y que quedó incorporada
como convención del proyecto. El seed es dato de desarrollo y del piloto, y no se
ejecuta en un entorno real: una regla sembrada sólo allí existiría en producción
exclusivamente como constante del código, y la fila que un operador buscaría en
la tabla de configuración no estaría. La migración la inserta para toda
organización existente, el seed cubre las que se creen con posterioridad y el
servicio conserva el mismo valor como respaldo cuando la fila falta; los tres
caminos leen una única constante, de modo que no pueden discrepar entre sí.
Ninguno de ellos sobrescribe un valor ya modificado por la clínica: ambos
mecanismos de alta son idempotentes y respetan la decisión registrada.

La visibilidad de los datos incorporó una restricción más estrecha que la de
organización. Sobre el aislamiento por organización, que el sistema ya garantiza
en toda consulta, se añadió que el rol profesional alcance únicamente a los
pacientes con los que tiene vínculo de tratamiento. La restricción se
implementó como filtro del camino de lectura y no como verificación posterior,
de modo que un paciente que el profesional no atiende resulta indistinguible de
uno inexistente: responder que el acceso está prohibido confirmaría que el
registro existe, lo que constituye ya una revelación. Del mismo modo, cuando dos
profesionales comparten un paciente, cada uno recibe solamente su propio
vínculo, puesto que la prioridad, el contacto y —cuando la tarea correspondiente
las incorpore— las observaciones de uso interno son juicio de ese profesional y
no del colega; el personal administrativo, en cambio, sigue viendo el legajo
completo.

Toda mutación se audita dentro de la misma transacción que la produce y con la
referencia real al paciente que la clave foránea agregada en la tarea anterior
hizo posible, de modo que una consulta de cumplimiento pueda establecer qué se
hizo sobre un paciente sin interpretar identificadores genéricos. El detalle de
una modificación enumera qué campos cambiaron y nunca su contenido, porque el
contenido es dato personal y la traza no es el lugar donde deba replicarse. La
implementación reveló un matiz digno de mención: enumerar las claves del objeto
recibido no produce esa lista, porque la biblioteca de transformación instancia
todas las propiedades declaradas en el tipo, de modo que una modificación de un
único campo generaba una entrada que declaraba haber cambiado todos. Una traza
que sobredeclara cambios es peor que una sin detalle, ya que es contra ella que
se audita; la corrección se factorizó en una función propia con su prueba
unitaria y la regla se incorporó a las convenciones del repositorio.

La tarea aprovechó además para consolidar tres piezas que se repetían. La
conversión entre fechas de calendario y los valores que la biblioteca de acceso
a datos expone, que vivía dentro del servicio de ausencias, se extrajo a un
único lugar y se amplió con el desplazamiento por meses —ajustado al último día
del mes destino, para que un año antes del 31 de marzo sea el 28 o el 29 de
febrero y no el 3 de marzo— y con el cálculo de edad a partir de la fecha de
nacimiento. La verificación de que un profesional actúa sobre su propio registro
se generalizó para leer el identificador del parámetro de ruta que cada ruta
declare, ya que en este módulo el parámetro principal designa al paciente. Y la
validación de identificadores contra el catálogo de obras sociales se centralizó
en el servicio del catálogo, del que ahora dependen tanto el módulo de
profesionales, que registra cuáles acepta cada uno, como el de pacientes, que
registra cuál cubre a cada paciente.

La verificación se realizó nuevamente por la capa HTTP, con credenciales reales
de cada rol, cubriendo el ciclo completo de gestión, el rechazo con error de
validación de cada dato obligatorio ausente o mal formado, el conflicto ante un
documento repetido dentro de la organización junto con su admisión en otra, la
independencia de los vínculos de un mismo paciente con dos profesionales, el
alcance de cada rol sobre los datos, la obediencia al umbral de inactividad
configurado y la baja lógica con su recuperación posterior. Se incluyó
deliberadamente una prueba de que las observaciones de uso interno no aparecen
en ninguna respuesta del módulo, para que la restricción de acceso que la tarea
correspondiente diseñará no quede anticipadamente vulnerada. El mapa de
endpoints del módulo con sus permisos por rol queda pendiente de incorporación
como figura.

La tarea siguiente incorporó las reglas que gobiernan el vínculo entre paciente
y profesional: la clasificación del paciente como nuevo o recurrente, el
registro de su última consulta y la consulta de la prioridad que el profesional
mantiene. Las columnas correspondientes ya existían desde la primera tarea de la
fase, de modo que lo aportado aquí es la lógica y no el esquema. Los requisitos
establecen que un paciente es nuevo en su primera sesión con un profesional
determinado y pasa a recurrente una vez completada, que si deja de concurrir por
más de un año vuelve a marcarse como nuevo, y que la prioridad la define y carga
el profesional para que el motor de turnos la consuma al reasignar.

La reclasificación por inactividad se resolvió como evaluación en la lectura y
no mediante un proceso programado periódico, según indican las fuentes. Sobre
esa base quedaba abierta una segunda decisión: si la reclasificación debía
además escribirse en la base o recalcularse en cada lectura sin tocar la fila. Se
optó por escribirla. Calcularla sin persistirla dejaría la columna almacenada en
desacuerdo permanente con lo que los puntos de consulta informan, de modo que
cualquier lector ajeno a esos puntos —una consulta directa a la tabla, un
informe, el propio motor de turnos— observaría un valor que el sistema ya
considera caduco. La escritura se realiza mediante una única sentencia que nombra
exactamente los vínculos leídos, y no con una condición general sobre la tabla,
para que la lectura de un paciente no altere en silencio filas que quien
consulta nunca solicitó.

La regla misma se ubicó en una función pura, aislada del acceso a datos, y el
servicio que la rodea se limita a resolver las fechas que necesita y a persistir
lo que decide. La separación responde a la naturaleza de lo que debe validarse:
un límite entre dos días del calendario se comprueba mejor sin base de datos,
mientras que las consecuencias que ninguna función pura puede exhibir —que la
regla efectivamente se ejecute al leer, que la reclasificación quede escrita y
que todos los puntos de consulta coincidan— se verifican por separado mediante
pruebas de integración.

El límite temporal exigió una decisión explícita, por discrepancia entre las
fuentes. El documento de requisitos habla de "más de un año", expresión que en
sentido estricto excluye el aniversario exacto, mientras que el criterio de
aceptación de la tarea sitúa la reclasificación a los trescientos sesenta y
cinco días cumplidos; la tarea anterior, además, había implementado el mismo
umbral para la solicitud de actualización de datos con el límite excluyente. Se
adoptó el límite inclusivo para ambas reglas, unificándolas en una única función
compartida. El criterio que ordenó la elección fue la consistencia: los
requisitos enuncian las dos reglas con la misma frase, de manera que dos
comparaciones separadas por un día producirían un paciente considerado nuevo al
que sin embargo no se le solicitan datos actualizados, precisamente la clase de
divergencia silenciosa que el diseño del esquema se había ocupado de evitar. El
costo asumido es un desplazamiento de un día en el comportamiento previamente
entregado, que se juzgó preferible a sostener dos versiones de una misma regla.

El registro de la última consulta se definió como monótono: una sesión antigua
cargada con posterioridad no retrasa la fecha almacenada, puesto que el dato
existe para responder cuánto hace que el paciente no concurre y una carga tardía
no vuelve más lejana la última asistencia. La condición se expresó dentro de la
propia sentencia de actualización y no como comparación previa en el servicio,
dado que leer el valor y decidir después permitiría que dos turnos completados de
forma concurrente leyeran ambos la fecha anterior y que el último en escribir la
retrasara, patrón de lectura seguida de escritura que el nivel de aislamiento
predeterminado no protege.

Completar la primera sesión determina dos hechos simultáneos: que el paciente
deja de ser nuevo para ese profesional y que su primera sesión deja de estar
pendiente. Se resolvió escribir ambos campos en el mismo momento, en lugar de
diferir el segundo a la fase de turnos como preveía una anotación previa, porque
un indicador de primera sesión activo junto a un paciente ya recurrente
constituye un estado internamente contradictorio, y porque de ese indicador
dependerá el cálculo de la duración del turno: la primera sesión ocupa dos turnos
consecutivos, de modo que un valor caduco no sería un detalle cosmético sino una
agenda mal calculada. Ambos campos se escriben sin condicionarlos a la fecha,
porque registrar una sesión antigua sigue constituyendo constancia de
asistencia; si esa sesión resulta anterior al umbral, la regla de inactividad la
reclasifica en la lectura siguiente, de manera que ambas reglas se componen en
lugar de contradecirse.

El registro de la consulta se expuso como servicio interno y no como operación
del contrato HTTP. La única evidencia de asistencia que el sistema posee es un
turno que pasó a completado, y admitir la carga directa de la fecha equivaldría
a aceptar una afirmación sobre asistencia que ningún turno respalda. El servicio
admite que quien lo invoca le entregue la transacción en curso, de modo que un
turno confirmado como completado y una consulta nunca registrada no puedan
ocurrir por separado. Se optó por una dependencia directa entre módulos y no por
un puerto con su adaptador, en aplicación del criterio ya fijado: los puertos
existen para las integraciones externas y para los casos en que el módulo
productor no debe conocer a su consumidor, mientras que aquí la dirección es la
inversa —el módulo de turnos depende del de pacientes, que ya existe—, de modo
que no hay nada que invertir y un puerto sólo agregaría indirección.

El diagrama de estados del tipo de paciente, con las transiciones que lo
gobiernan, queda pendiente de incorporación como figura.

La última tarea de la fase incorporó el consentimiento del paciente para el
tratamiento de sus datos personales, exigido por la Ley 25.326. La entidad ya
existía desde la primera tarea, de modo que lo aportado aquí es su
comportamiento: el registro de la aceptación con fecha y hora, la consulta de si
está otorgada y desde cuándo, y un servicio interno de verificación que la capa
conversacional consultará antes de tomar una reserva. Los requisitos enuncian la
regla en una sola frase —el sistema guarda el consentimiento con fecha y hora y
solo lo solicita si no se registró previamente su aceptación—, de la que se
desprenden las dos propiedades que la implementación debía garantizar: que quede
constancia temporal del acto y que no se vuelva a pedir una vez otorgado.

El registro se definió idempotente, con el mismo razonamiento aplicado a la
vinculación con un profesional: quien más necesita la operación es el asistente
conversacional en medio de una conversación, que no puede saber si el paciente
aceptó meses atrás, y volver a preguntárselo es precisamente lo que la regla
prohíbe. Una segunda invocación devuelve entonces el consentimiento existente sin
agregar una fila y responde con el código de éxito ordinario, ya que la respuesta
describe el estado del consentimiento y no afirma haberlo creado. Consultar y
registrar comparten así una única forma de respuesta.

Impedir la duplicación exigió una decisión sobre concurrencia. Comprobar que no
existe aceptación previa e insertar a continuación es una lectura seguida de
escritura, que el nivel de aislamiento predeterminado no protege. La solución
habitual —una restricción de unicidad que la base rechazara— resultaba
inaplicable: la tabla es de solo agregado por diseño y debe admitir una segunda
fila cuando la fase de cumplimiento incorpore la revocación, de modo que la
restricción impediría aquello para lo cual la tabla fue concebida. Se recurrió
por tanto al mecanismo ya adoptado en el proyecto para esta clase de invariante,
ejecutando comprobación e inserción en una única transacción serializable, de
manera que la invocación perdedora reintente sobre la rama idempotente.

La marca temporal se tomó del reloj del sistema y el punto de acceso no admite
cuerpo alguno. Aceptar una fecha informada por el cliente equivaldría a admitir
una afirmación sobre cuándo se prestó el consentimiento sin respaldo, siendo este
el registro que la clínica exhibiría ante un organismo de control.

Este punto obligó además a revisar una decisión de la primera tarea de la fase.
La entidad se había modelado con dos marcas temporales, la del acto y la de
creación de la fila, previendo un consentimiento firmado en papel e incorporado
con posterioridad conservando su fecha real. Consultada la clínica, se estableció
que ese caso no existe: todo paciente, tanto los ya registrados como los nuevos,
presta el consentimiento a través del asistente conversacional. Desaparecido el
único escenario que las distinguía, ambas columnas sólo podían contener el mismo
valor, y una segunda copia de un instante es una copia que puede discrepar sin
que consulta alguna lo advierta, que es el mismo argumento con el que el esquema
rechaza replicar el identificador de organización. Se eliminó en consecuencia una
de las dos mediante una migración —sin pérdida de información, puesto que por
construcción ambas contenían el mismo valor—, junto con el índice que la usaba.
Se conservó la marca de creación de la fila, que es la que todas las demás
entidades ya llevan y cuyo valor por defecto proviene del reloj de la base de
datos; la alternativa de conservar la columna con nombre de dominio propio, por
su trazabilidad al diagrama entidad-relación, se descartó porque el alta de la
fila es el momento del consentimiento y un nombre distinto para el mismo instante
sólo agrega vocabulario, mientras que el término del diagrama se preserva como
comentario del glosario sobre la columna. El episodio ilustra un
criterio que el proyecto viene sosteniendo: una columna se conserva mientras
exista algo que sólo ella pueda expresar, y se elimina apenas ese algo deja de
existir, aun cuando conservarla parezca inofensivo.

Como la tabla es de solo agregado, el estado del consentimiento no es una
bandera sino la entrada más reciente, obtenida ordenando por la fecha del acto,
ordenamiento que el índice existente resuelve. Esa derivación se ubicó en un
único lugar, del que también lee el servicio de verificación en lugar de
recalcularla, de modo que la respuesta que ve el cliente y la compuerta que
consultará el asistente conversacional no puedan discrepar sobre qué significa
que el consentimiento esté aceptado. Aunque la revocación corresponde a la fase
de cumplimiento y quedó fuera de alcance, la lógica no supone que la última
entrada sea siempre una aceptación: si lo último registrado fuese una
revocación, el estado informado es la ausencia de consentimiento y una nueva
aceptación se registra como acto nuevo.

El recurso se modeló en singular, con una única dirección y sin entradas
direccionables individualmente, puesto que lo que un cliente pregunta es siempre
si el paciente consiente y desde cuándo; exponer el identificador de cada fila
habría sugerido operaciones de modificación o borrado incompatibles con una
tabla de solo agregado. Los permisos se repartieron según de quién es el acto:
registran el personal administrativo y el proceso automatizado, que es donde el
paciente efectivamente lo presta, y se excluyó al rol profesional, porque el
consentimiento es acto del paciente y el clínico no tiene nada que declarar en su
nombre; la consulta, en cambio, queda abierta a todos los roles bajo la
restricción ya vigente en el módulo, de manera que un profesional alcanza
únicamente el consentimiento de los pacientes que atiende y para los demás
obtiene la misma respuesta que para un paciente inexistente.

Al no llevar el consentimiento identificador de organización —por tratarse de
una entidad con un único padre que ya lo determina—, ninguna fila se alcanza por
su identificador y toda operación se ancla en el paciente. El camino de lectura
requería además la restricción por profesional tratante, que la verificación
existente no aplicaba, de modo que se incorporó una segunda verificación junto a
la anterior y se extrajo a un método común la consulta y el mensaje de error que
ambas comparten, dejando como única diferencia entre ellas el filtro de alcance.

El diagrama de secuencia del consentimiento, con la verificación previa, la
solicitud únicamente ante la ausencia de registro y el comportamiento
idempotente ante una aceptación ya existente, queda pendiente de incorporación
como figura.

La última tarea del módulo abordó el campo de observaciones de la relación de
tratamiento, que los requisitos describen como de uso interno y de manejo
exclusivo del profesional: sólo él lo carga y lo visualiza, no se utiliza en la
conversación con el asistente y no se revela nunca al paciente. El campo ya
existía en el modelo, incorporado con las entidades del módulo; lo que esta
tarea debía construir era la restricción de acceso, y la dificultad no estaba en
impedir una lectura sino en impedir todas las que el sistema ya ofrecía, presentes
y futuras, sobre una relación que se devuelve anidada en cada consulta de
paciente.

La restricción se resolvió, en consecuencia, en la propia forma en que se lee la
relación y no en el momento de responder. Hasta entonces las consultas
recuperaban la fila completa y la función de presentación decidía qué campos
exponer, de modo que la única barrera entre la observación y una respuesta era
el cuidado de quien escribiera cada presentador; un campo agregado por descuido a
esa función, o un presentador nuevo, habrían bastado para publicarla. Se pasó
entonces a enumerar explícitamente las columnas que cada lectura recupera, y la
enumeración compartida por todos los caminos del módulo no incluye las
observaciones. El efecto es doble: la columna no se trae siquiera desde la base
de datos en ninguna consulta general, y el tipo que el compilador deriva de esa
enumeración carece de la propiedad, de manera que ningún presentador puede
reenviarla aunque se lo proponga. Una segunda enumeración, definida como
extensión de la primera para que ambas no puedan divergir en los campos que
comparten, agrega la observación y la utilizan únicamente los dos caminos
autorizados a verla. La alternativa —seguir recuperando la fila entera y omitir
el campo al presentar— se descartó por ser exactamente la disciplina manual que
el requisito no puede permitirse.

Sobre esa base, la autorización quedó expresada en el enrutamiento. El proyecto
ya contaba con una comprobación que permite al personal administrativo actuar
sobre cualquier profesional y al profesional únicamente sobre sí mismo, que es la
regla adecuada para los datos que la administración legítimamente gestiona en su
nombre. Las observaciones exigen una regla más estrecha, en la que ni siquiera el
rol administrativo queda exento, por lo que se incorporó una segunda comprobación
que sólo admite al profesional nombrado en la dirección del recurso. Ambas
comparten todo salvo esa exención, de modo que se extrajo la parte común a una
clase base y cada una declara únicamente si el rol administrativo la evade; así,
la comprobación estricta no puede perder ninguna de las cautelas que la otra ya
tenía —entre ellas la de denegar cuando falta el identificador, que es el modo de
fallo obligatorio en una comprobación de acceso—. La comparación de identificadores
que ambas realizan se extrajo también a una función propia, que es hoy el único
lugar donde el sistema decide si dos identificadores designan al mismo
profesional.

La escritura se expuso como un subrecurso de la relación, con su propio cuerpo de
solicitud, y no como un campo más del punto de acceso que edita la prioridad.
Separarlos es precisamente lo que permite protegerlos de manera distinta: el
personal administrativo mantiene la prioridad de cualquier relación y queda
excluido de las observaciones, sin que ningún servicio deba recordar ignorar un
campo que le fue entregado. La validación por lista blanca ya vigente descarta
además el campo si se lo intenta introducir por el punto de acceso permitido, lo
que quedó verificado por una prueba. La lectura, en cambio, no recibió una
dirección propia: la relación ya se consulta por su dirección natural, y allí se
resolvió que la observación acompañe la respuesta únicamente cuando quien
consulta es el profesional de esa relación, mientras que el personal
administrativo obtiene la misma relación sin el campo. Se evaluó publicar
también una lectura específica del subrecurso y se descartó, por constituir un
segundo lugar donde la misma restricción podría implementarse mal.

La traza de auditoría se sujetó a la regla ya vigente en el proyecto, según la
cual una entrada nombra los campos modificados y nunca sus valores. Aquí esa
regla deja de ser una convención de higiene para volverse parte de la
restricción: registrar el contenido escrito trasladaría a la tabla de auditoría
—consultada con fines de rendición de cuentas— exactamente aquello que la tarea
mantiene fuera de toda respuesta. La escritura del campo y su entrada de
auditoría comparten transacción con el resto de las ediciones de la relación,
para lo cual se factorizó el envoltorio común que ambas operaciones repetían.

Las pruebas se orientaron a demostrar la ausencia del campo tanto como su
presencia: que el profesional de la relación lo escribe y lo recupera, que otro
profesional, el personal administrativo y el proceso automatizado reciben un
rechazo por falta de permisos, y que ninguna de las respuestas generales del
módulo —el detalle del paciente, su listado, la vinculación con un profesional y
la edición de la prioridad— contiene el valor almacenado, verificado buscándolo
en la respuesta serializada completa y no sólo en los campos esperados.

La última tarea de la fase atendió el requisito de carga de pacientes
preexistentes, cuyos registros la clínica conserva hoy en planillas de cálculo y,
en algunos casos, únicamente en papel. La particularidad de esta funcionalidad no
está en la escritura sobre la base, que reutiliza las entidades ya construidas,
sino en la naturaleza de su entrada: es el único punto de acceso del sistema cuyo
cuerpo de solicitud es un archivo y no un documento estructurado, y ese archivo lo
escribió una persona a lo largo de años, de manera que un archivo sin ninguna fila
defectuosa no existe. De allí la decisión que gobierna todo el diseño: la
importación es parcial por naturaleza, intenta todas las filas, persiste las
válidas y responde con un informe que enumera las rechazadas indicando el número
de fila, el campo y el motivo. Una carga de todo o nada habría significado que la
clínica no puede migrar hasta que la planilla sea perfecta, es decir, que no puede
migrar.

La identidad de una fila se hizo coincidir con la que el modelo de datos ya
declaraba para un paciente: su número de documento dentro de la organización. La
importación resulta así idempotente, y ejecutar dos veces el mismo archivo
actualiza los mismos registros en lugar de duplicarlos. Esa propiedad no es un
refinamiento sino la condición que vuelve seguro el ciclo natural de trabajo
—cargar, leer el informe, corregir en la planilla las filas señaladas y volver a
cargarla entera—, y se complementó con una segunda regla: sólo se escriben las
celdas que el archivo trae. Una celda vacía significa ausencia de información y
nunca orden de borrar, porque la planilla es una fotografía incompleta del pasado
y no la verdad sobre el presente; por la misma razón la marca de baja lógica queda
enteramente fuera del alcance de la importación, ya que un archivo que enumera a
todos los pacientes que la clínica alguna vez atendió no constituye un pedido de
dar de alta nuevamente a quienes fueron dados de baja.

La lectura del archivo se separó del dominio y quedó como componente común: recibe
la carga, resuelve el formato por la extensión —el tipo declarado en la solicitud
varía según el cliente y el software instalado, y no identifica nada de manera
confiable— y devuelve un encabezado normalizado junto con filas de texto numeradas
tal como las numera la aplicación de planillas, de modo que un error pueda
señalarse por la fila que la persona verá al abrir su archivo. Esa capa no sabe
nada de pacientes; qué significa cada columna y qué valores son admisibles
pertenece a quien pidió el archivo. Dos decisiones de detalle merecen mención por
tratarse de errores frecuentes en este tipo de funcionalidad. La primera es que
toda celda es texto y sigue siéndolo hasta que un cuerpo de solicitud la juzgue:
una planilla no tiene tipos, y una conversión automática transforma un documento
escrito con cero inicial en un número que pierde ese cero, y una fecha en un
instante situado en la zona horaria del servidor; la única excepción es la celda
con formato de fecha, que se convierte a día calendario por el mismo módulo con
que el proyecto realiza todos sus cruces entre instante y día. La segunda es que
el separador se detecta en lugar de suponerse, contando candidatos fuera de las
comillas en la primera fila: un archivo exportado desde una hoja de cálculo con
configuración regional en español está separado por punto y coma, y suponer la
coma habría significado rechazar justamente el archivo que la funcionalidad existe
para leer.

Las columnas se reconocen por un catálogo de nombres alternativos, sobre
encabezados normalizados a minúsculas y sin acentos ni puntuación, de manera que
una misma columna se identifique cualquiera sea la forma en que la clínica la
escribió. La coincidencia es del nombre completo y no de una parte, para que
"nombre completo" no sea leído como el nombre de pila, y una columna que el
sistema no modela —un importe, una anotación administrativa— se ignora en lugar de
provocar un rechazo, porque toda planilla real trae algunas. La obra social y el
profesional llegan por nombre, nunca por identificador interno, y se resuelven
contra la base; cuando un nombre no coincide con ninguno, o coincide con más de
uno, se informa como error de la fila en lugar de adivinar, ya que dos
profesionales homónimos son una posibilidad real y elegir entre ellos adosaría la
historia de un paciente al profesional equivocado. La resolución incluye
deliberadamente a los profesionales dados de baja, porque una planilla es historia
y un paciente atendido durante años por alguien que ya no trabaja en la clínica no
dejó de haber sido atendido por esa persona.

La validación de cada fila se apoyó en los mismos validadores del alta
interactiva, que hasta entonces cada cuerpo de solicitud del módulo declaraba por
separado; se extrajeron a un módulo común que hoy comparten el alta, la edición,
la búsqueda y la importación, de modo que un valor que la importación acepta es
uno que el alta habría aceptado también. La única asimetría es deliberada y va en
sentido contrario: la importación exige menos campos obligatorios, porque la fecha
de nacimiento y el contacto de emergencia son requisitos de la reserva de un turno
y no del registro de una persona, y rechazar por ellos dejaría fuera del sistema a
los pacientes más antiguos de la clínica. El punto de acceso que informa qué datos
le faltan a un paciente, construido en una tarea anterior de esta misma fase, es
lo que después señala esa carencia cuando el paciente vuelve a atenderse. La
validación se invoca de forma explícita sobre cada fila en lugar de delegarse en
la tubería global, porque su resultado no debe ser un rechazo de la solicitud sino
una entrada del informe, y se emite una entrada por campo y no una por restricción
incumplida, ya que tres líneas sobre una misma celda producen un informe que hay
que filtrar antes de poder leerlo. Se admitió además la fecha escrita en orden
día-mes-año, que es la forma usual en el país, con la precaución de no reparar en
silencio ningún otro valor: lo que no responde a esa forma pasa intacto al
validador, que lo rechaza e informa la fila. Esa tolerancia quedó confinada a las
planillas, mientras la interfaz de programación conserva un formato único, porque
un cliente se escribe una vez contra un contrato y una planilla la escribe una
persona.

Cada fila se aplica dentro de su propia transacción, de manera que el fallo de una
no deshaga las anteriores y que, dentro de ella, la escritura del paciente, el
vínculo con su profesional y las entradas de auditoría se confirmen como una
unidad. El requisito de marcar como recurrentes a los pacientes importados que
tengan historial de consultas se resolvió reutilizando el método que ya traducía
una asistencia en un tipo de paciente, una marca de primera sesión y una fecha,
escrito para el momento en que un turno se completa; la fecha de última consulta
que trae la planilla es ese historial, y reutilizar el método evita que la regla
tenga un segundo lugar donde divergir. Esa reutilización obligó a una
reorganización menor de los servicios de la relación paciente–profesional: tanto
ese método como el que crea el vínculo comprobaban la pertenencia del paciente
sobre el cliente exterior a la transacción, comprobación que fallaría sobre un
paciente recién creado dentro de la transacción de la importación y todavía sin
confirmar, de modo que en cada uno se separó el trabajo transaccional de las
comprobaciones previas, quedando el método público como punto de entrada de las
rutas y el trabajo disponible para un llamador que ya estableció ambos extremos.
La fecha de última consulta queda así escrita únicamente por dos caminos: el turno
completado y esta migración.

El punto de acceso se reservó al rol administrativo. Cargar una planilla es
trabajo administrativo que se realiza una vez durante la migración, con una
persona leyendo el informe que devuelve; el proceso automatizado que atiende la
conversación registra pacientes de a uno por el punto de acceso ya existente, y un
profesional queda excluido por la misma razón por la que no da de alta pacientes y
porque una importación escribe sobre toda la organización y no sólo sobre los
suyos. Los registros se cargan bajo la organización del usuario autenticado, de
modo que la misma planilla puede importarse de forma independiente en
organizaciones distintas. Las entradas de auditoría, además de nombrar los campos
escritos y nunca sus valores, indican que la escritura provino de una
importación: un registro cargado desde una planilla y uno escrito por una persona
responden de manera distinta ante una consulta de rendición de cuentas, y la traza
es donde esa distinción debe sobrevivir.

El cierre del módulo agregó los datos de ejemplo del catálogo de obras sociales y
verificó la cobertura de pruebas de todo el subdominio, sin cambios de esquema. El
seed del inquilino del piloto crea las obras sociales de ejemplo, entre ellas la
provincial, la única que la clínica acepta para las consultas. Se la sembró como un
nombre genérico de fantasía, como todo otro nombre del seed, ya que la entidad real
se carga con los datos de la clínica, y se la ubicó primero en la lista por ser la
que la especificación singulariza; la regla que distingue la consulta de la receta
—sólo la provincial para las consultas, cualquiera para las recetas y la
medicación— pertenece al motor de turnos y a la capa conversacional, y queda fuera
de esta fase. El catálogo se siembra de forma global y no bajo la organización del
piloto, en coherencia con la conversión de la obra social en catálogo global
descripta en las fundaciones: el seed la crea una sola vez y el piloto la alcanza a
través de ese catálogo compartido, apoyándose el alta en el nombre único de la obra
social de modo que dos corridas sucesivas no la dupliquen. La entidad no se
modificó: lo que en una primera lectura del diagrama entidad-relación podía tomarse
por atributos de cobertura de la obra social resultó corresponder a verbos de
relación del diagrama y no a columnas, de manera que la obra social conserva el
nombre como único dato propio y el seed la puebla tal como está.

La cobertura de pruebas del módulo se verificó y se completó sin reproducir lo ya
cubierto. Los siete escenarios que el requisito enumera —el alta, la baja y la
modificación del paciente y su vínculo con el profesional; los campos que una
reserva exige; la regla del año que alterna el tipo del paciente entre nuevo y
recurrente; el consentimiento que no se reitera una vez aceptado; las observaciones
accesibles sólo al profesional del vínculo; el importador con archivos válidos e
inválidos; y el aislamiento entre inquilinos— ya quedaban cubiertos de extremo a
extremo por las tareas previas de esta misma fase, cada una con su prueba.
Reescribirlos habría duplicado esa cobertura, contra la disciplina de no
redundancia del proyecto; se verificó su vigencia y se agregó sólo lo que esta
tarea introduce de nuevo: la prueba del propio seed, que comprueba que la obra
social provincial se crea y que una segunda ejecución no la duplica.

La revisión sistemática de código que precedió al inicio de los módulos posteriores
—descrita en la subsección de Profesionales— no halló defectos de comportamiento en
el módulo de Pacientes, cuyo filtrado por vínculo de tratamiento, restricción de las
observaciones al profesional del vínculo, auditoría dentro de la transacción e
idempotencia del consentimiento estaban correctamente implementados. Identificó, en
cambio, un vacío de verificación: dos invariantes que el código garantiza no contaban
con una prueba que los ejercitara. El primero es la unicidad del consentimiento bajo
concurrencia; la regla de que el consentimiento se solicita sólo si no fue registrado
antes se protege registrándolo en una transacción de aislamiento serializable con una
rama idempotente, y se agregó una prueba que somete ese invariante a una carrera real
—dos registros simultáneos del mismo paciente— y afirma que deja exactamente una fila y
una única entrada de auditoría, cualquiera sea el entrelazado que el planificador
imponga. El segundo es la rama de denegación del filtro de acceso del profesional: una
cuenta con rol de profesional que carece de profesional asociado es una cuenta rota que
debe rechazarse antes de consultar la base de datos, en lugar de degradar a un acceso
sin restricción, y como esa cuenta no puede construirse a través de la interfaz pública
—el inicio de sesión no la emite—, se la verificó con una prueba unitaria que ejercita
las dos ramas del filtro y, en particular, esa denegación previa a toda consulta. Ambas
adiciones son exclusivamente de verificación: no modifican el comportamiento del módulo,
sino que fijan por prueba una garantía que hasta entonces sólo el código sostenía.

Una revisión más amplia, que confrontó el conjunto de lo implementado con las dos fuentes
de verdad del proyecto, introdujo dos correcciones de comportamiento en este módulo. La
primera atañe al estado de actividad del paciente: era un campo editable del punto de
modificación general, alcanzable por el rol administrativo y por el proceso automático, de
modo que este último podía dar de baja a un paciente deslizando el atributo dentro de una
modificación ordinaria, y la baja quedaba registrada como una modificación genérica de
campos en lugar de la baja que era. Se retiró el atributo de ese punto de acceso y se
separaron el alta y la baja en puntos propios, ambos restringidos al rol administrativo,
convirtiendo la restricción en un hecho de enrutamiento —qué rol alcanza cada punto— en vez
de una comprobación dispersa que hay que recordar mantener, y logrando que cada operación se
audite con la semántica que le corresponde. La segunda corrige la precisión de la traza de
auditoría ante operaciones sin efecto: el registro de una consulta atendida nombraba siempre
los mismos tres campos como modificados aun cuando la fecha de última consulta no avanzaba, y
una modificación de cuerpo vacío dejaba una entrada de una operación que no cambiaba nada. Se
optó por derivar los campos efectivamente modificados y por omitir la escritura y su entrada
cuando la operación no cambia ningún campo, porque una traza que afirma cambios inexistentes
degrada el valor del registro contra el cual se rinde cuentas en materia de protección de
datos personales.

### 3.2.3 Motor de Turnos

El módulo de Turnos comenzó, igual que Profesionales y Pacientes antes que él, por
incorporar al esquema las entidades base del dominio antes de construir servicio o
lógica alguna sobre ellas: el turno, el feriado y la lista de espera, tomados del
diagrama entidad-relación y del documento de requisitos, módulo Turnos. El turno
agrega, además de sus datos propios, un enumerado de cinco estados —reservado,
confirmado, cancelado, reasignado y completado— y dos claves foráneas compuestas,
una hacia el paciente y otra hacia el profesional, siguiendo la misma forma que la
tarea de Pacientes había fijado para el vínculo paciente-profesional: con dos
padres acotados por organización, el identificador de organización propio del turno
es lo que obliga a que ambos padres pertenezcan a la misma organización, restricción
que ningún par de claves foráneas independientes puede expresar por sí solo. La
lista de espera, que enlaza igualmente un paciente y un profesional, recibió la
misma forma y el mismo razonamiento.

El feriado ilustra una tensión entre el diagrama y el diseño multi-tenant del
sistema: el diagrama lo declara con la fecha como clave primaria, pero esa clave
sólo es única dentro de una organización, ya que dos organizaciones observan
legítimamente el mismo feriado nacional cada una con su propia fila. Se le dio
entonces un identificador propio, como al resto de las entidades del esquema, y la
clave natural del diagrama se expresó como una restricción de unicidad acotada por
organización en lugar de como clave primaria literal — el mismo tratamiento que ya
había recibido el documento del paciente.

Dos campos del turno se poblaron deliberadamente por copia al momento de la
reserva y no por lectura en vivo de su origen: la duración, que puede diferir de la
configurada en el profesional para una primera sesión y que no debe cambiar si la
configuración del profesional cambia después de reservado el turno; y el indicador
de primera sesión, copiado del vínculo paciente-profesional por la misma razón. El
estado "reasignado" se modeló únicamente como un valor del enumerado, sin relación
ni lógica que vincule un turno reasignado con el que lo reemplaza, porque el
documento de requisitos atribuye esa creación al algoritmo de reasignación de una
fase posterior y no a esta entidad.

La tarea saldó además una deuda declarada desde las Fundaciones: la traza de
auditoría reservaba, desde su creación, una columna sin clave foránea para
referenciar al turno, porque la tabla destino no existía todavía. Al quedar creada
la entidad, la columna se convirtió en clave foránea compuesta real mediante un
renombrado —no un borrado y recreación— para no perder las referencias ya
registradas, con la misma restricción sobre el borrado que ya regía la referencia
al paciente: una entrada de auditoría no puede quedar huérfana, de modo que un
turno con historial de auditoría no puede eliminarse físicamente. La referencia
equivalente del registro operativo de la cerradura hacia el turno se dejó sin
conectar: esa columna está documentada como parte de un par junto con el futuro
código de acceso, y conectar sólo una de las dos no habilita ningún camino de
lectura por organización mientras la otra siga sin existir.

Sobre esa base de entidades se construyó el servicio de disponibilidad, que
calcula las franjas libres de la agenda de un profesional entre dos fechas
siguiendo el procedimiento del documento de requisitos: toma el horario de
atención del profesional para los días de la semana del rango, genera
franjas desde el inicio hasta el fin de cada bloque con paso igual a la
duración de consulta configurada, y descarta las que caen en un feriado del
inquilino, en una ausencia del profesional o en un turno ya reservado o
confirmado. El paso de generación es deliberadamente la duración de la
consulta y no la cadencia de apertura de franjas —una configuración distinta
del profesional, pensada para un espaciado posterior de franjas— porque así
lo especifica el documento de requisitos para este cálculo puntual. La
ausencia de una duración de consulta configurada se trata como un pedido mal
formado y no como una lista vacía, para distinguir "no hay franjas hoy" de
"falta configurar la agenda del profesional", dato que el cliente necesita
para actuar. El cálculo hereda además la misma simplificación de huso
horario que ya rige los horarios de atención y las ausencias del sistema:
combina el día calendario y la hora de reloj de pared del bloque como si
ambos fueran directamente UTC, en lugar de introducir aquí, de forma
aislada, una conversión de huso horario real que ningún otro componente
todavía tiene.

A diferencia del horario de atención y las ausencias, que viven anidados en
el módulo de Profesionales aunque su ruta cuelga del mismo prefijo, el
servicio de disponibilidad se organizó en un módulo propio: lee además el
feriado y el turno, entidades de otro subdominio, y está pensado para ser
inyectado más adelante tanto por el algoritmo de reasignación (M3/M4) como
por la capa conversacional del chatbot (M5), que ofrecerá al paciente las
mismas franjas que calcula este servicio. Deliberadamente no incorpora
todavía la regla de doble franja para paciente nuevo: el documento de
requisitos la asigna a una tarea posterior de la misma fase, que consumirá
este servicio y aplicará esa restricción por encima de las franjas
calculadas aquí.

Sobre el servicio de disponibilidad se construyó la reserva de turno, que
aplica en orden las cuatro validaciones previas que fija el documento de
requisitos antes de crear el turno: que el paciente tenga consentimiento
registrado, que tenga cargados los datos obligatorios para una reserva
(fecha de nacimiento y contacto de emergencia — el documento nacional de
identidad no se verifica aparte porque la columna que lo guarda ya es no
anulable), que el slot esté libre, y que el paciente no tenga ya otro turno
reservado o confirmado en el mismo instante con otro profesional. La
verificación del slot libre no reimplementa el criterio de "ocupado": se
agregó como un método nuevo del propio servicio de disponibilidad, para que
el cálculo de franjas y la reserva no puedan llegar a discrepar sobre qué
cuenta como un turno que ya ocupa un horario. Las dos últimas validaciones
—slot libre y no simultaneidad del paciente— son invariantes de
lectura-y-escritura, de modo que se ejecutan dentro de una transacción
serializable junto con la creación del turno, siguiendo la misma técnica
que ya usa el repositorio para el tope de matrículas y el reemplazo del
horario semanal; el consentimiento y los datos obligatorios, al no competir
con ninguna escritura concurrente, se verifican antes de abrir esa
transacción.

El turno creado hereda el mismo patrón de copia al momento de la reserva
que ya había fijado el modelado de la entidad: la duración se toma de la
configuración del profesional en ese instante, y el indicador de primera
sesión se copia del vínculo entre el paciente y el profesional. Cuando ese
vínculo todavía no existe, la reserva lo crea en el momento en lugar de
exigir un paso de vinculación previo: la primera reserva de un paciente con
un profesional es, por definición, su primera sesión con él, de modo que la
ausencia del vínculo es en sí misma la respuesta a la pregunta que el
sistema debe validar, y no un caso de error. Cierra la operación una entrada
en la traza de auditoría, con un motivo de texto libre propio de la reserva
en lugar de los motivos genéricos de alta, modificación y baja que usa el
resto del sistema — el mismo mecanismo que ya usan el consentimiento y el
vínculo paciente-profesional para sus propios eventos de cumplimiento
normativo.

La fase se cerró con la regla de primera sesión que el servicio de
disponibilidad había dejado pendiente: cuando la reserva es la primera
sesión de un paciente con un profesional, se aplican cuatro controles
adicionales antes de crear el turno. Dos son interruptores de
configuración del profesional que ya existían en el esquema desde
Profesionales sin uso hasta este punto — si el profesional dejó de aceptar
pacientes nuevos, o si solo atiende adultos, en cuyo caso la edad se
deriva de la fecha de nacimiento con la misma función que ya usa el resto
del sistema para ese cálculo — y se verifican como precondiciones
ordinarias, de la misma manera que el consentimiento y los datos
obligatorios de la reserva. Un tercero limita a un paciente nuevo por
jornada por profesional, y por ser una invariante sobre el estado de la
agenda se revalida dentro de la misma transacción serializable que ya
protegía el slot libre y la no simultaneidad, con el mismo razonamiento:
dos reservas concurrentes de dos pacientes nuevos distintos podrían, sin
esa revalidación, leer ambas "todavía no hay ninguno" y escribir ambas.

El cuarto control es la franja extra que el profesional puede configurar
en tres modalidades — que el doble turno de la primera sesión solo pueda
empezar en el primer turno libre de la jornada, que solo pueda terminar en
el último, o que pueda ubicarse en cualquier posición con dos turnos
consecutivos libres — y que, cuando está configurada, hace que la primera
sesión reserve dos turnos consecutivos en lugar de uno, cada uno con la
duración de consulta del profesional. El cálculo de qué instantes son un
inicio válido de doble turno se agregó como una capa sobre el cálculo de
franjas existente, no como una modificación de su algoritmo: la agenda que
ve un paciente que ya está en tratamiento debía seguir siendo exactamente
la misma. Los dos primeros modos exigieron distinguir el primer o el
último turno *real* de la jornada, según la grilla de horario configurada,
del primero o el último turno que *todavía* está libre — un profesional
cuyo primer turno del día ya fue tomado por otro paciente no ofrece esa
modalidad corriéndose al segundo turno disponible, sino que directamente
no la ofrece ese día, porque la regla existe justamente para reservarle a
la primera sesión el comienzo de la jornada. Cuando el profesional
todavía no configuró la franja extra, la primera sesión se reserva como
un turno único, sin restricción de colocación: no se convirtió en un
requisito bloqueante como sí lo es la duración de consulta, porque impedir
la atención de pacientes nuevos hasta que se configure una modalidad de
franja habría sido una exigencia que el documento de requisitos no
plantea, y porque así ya se venía comportando el sistema antes de esta
regla. El mismo cálculo que decide qué instantes son un inicio válido de
doble turno se expone también por consulta directa, para que el turno que
se le ofrece al paciente y el que finalmente reserva no puedan discrepar;
y como una primera sesión bajo una franja configurada crea dos turnos en
lugar de uno, la respuesta de la reserva pasó a tener siempre la forma de
una lista, con un único elemento en el caso ordinario, en lugar de cambiar
de forma según el caso.

La fase continuó con la máquina de estados del turno: reservado a
confirmado, reservado a cancelado, confirmado a completado, confirmado a
cancelado, y cancelado a reasignado, tal como los enumera el documento de
requisitos. La tabla de transiciones válidas se extrajo a una función
pura, separada del servicio y sin dependencias de infraestructura, con el
mismo criterio que ya había fijado la regla de inactividad de pacientes —
una tabla de datos fácil de agotar por completo con una prueba unitaria
por cada par de estados posible, en lugar de una cadena de condicionales
dispersa en el servicio. La transición de cancelado a reasignado quedó
declarada válida en esa tabla, pero ningún método del servicio la ejecuta
todavía: el documento de requisitos atribuye esa escritura al algoritmo de
reasignación de una fase posterior, que decide a qué paciente de la lista
de espera se reasigna el turno liberado, una decisión que esta tarea no
tiene información para tomar.

Sobre la tabla de transiciones se construyeron tres operaciones. La
confirmación registra la fecha de confirmación y quedó reservada a los
mismos roles que la propia reserva —administrador o el proceso
automatizado del chatbot—, porque el documento de requisitos describe la
confirmación como una acción que el paciente realiza por WhatsApp, no algo
que un profesional haga en su nombre. La cancelación admite tanto un turno
reservado como uno confirmado, sujeta a una anticipación mínima respecto
de la fecha y hora del turno: si el pedido llega con menos anticipación
que la configurada, se rechaza con un mensaje que indica que el paciente
debe contactar directamente a la clínica. Esa anticipación mínima se
modeló como configuración por organización y no como una constante del
código, con el mismo valor por defecto de cuatro horas que fija el
documento de requisitos, siguiendo exactamente el patrón que ya había
usado el umbral de inactividad de pacientes: una fila sembrada tanto en
una migración, para que una organización ya existente la reciba sin
depender del seed de desarrollo, como en el seed mismo, con el servicio
cayendo al mismo valor por defecto si la fila todavía no existe. El
completado de un turno confirmado, por último, actualiza dentro de la
misma transacción el vínculo entre el paciente y el profesional a través
del método que la tarea de tipo de paciente ya había dejado preparado
para este momento: el tipo pasa a recurrente, el indicador de primera
sesión se apaga, y la fecha de última consulta avanza si corresponde. Las
tres operaciones protegen la transición con una escritura condicionada al
estado de origen, en lugar de una transacción serializable completa: dos
transiciones concurrentes sobre el mismo turno hacen que la segunda no
encuentre ninguna fila que igualar y se traduzca en un conflicto, el mismo
mecanismo que ya usa el vínculo paciente-profesional para no retroceder la
fecha de última consulta ante dos completados simultáneos.

La autorización de "administrador o el profesional dueño del turno", que
exige tanto la confirmación como la cancelación, el completado y los dos
registros administrativos descritos a continuación, se resolvió en el
servicio y no con el guard de propiedad ya existente en Profesionales: ese
guard lee el identificador del profesional directamente de un parámetro de
la URL, y en estas rutas el parámetro nombra al turno, no al profesional,
de modo que la propiedad solo puede conocerse después de leer la fila. Se
aplicó la misma comparación que ya usa el vínculo paciente-profesional
para el mismo problema, dejando la verificación de organización —que sigue
resolviendo el filtrado automático por inquilino— como lo que en verdad
contiene el pedido.

La fase continuó, tras la máquina de estados, con dos registros
administrativos del turno sin restricción de estado —si se cobró la
consulta y si el paciente trajo la orden médica que exige la obra
social—, y con el método que expone, sin combinarlos, el tipo del vínculo
(nuevo o recurrente) y la prioridad que el profesional le asignó al
paciente: los dos datos que el algoritmo de reasignación de una fase
posterior necesitará leer para decidir, ante un turno liberado, a qué
paciente de la lista de espera ofrecérselo primero.

La fase cerró, por ahora, con los tres flujos de reprogramación que
completan el ciclo de vida administrativo del turno: la reprogramación
individual por su propio identificador, la reorganización manual de
varios turnos de una misma agenda en un solo pedido, y la cancelación
automática de los turnos afectados cuando se registra una ausencia del
profesional. Las dos primeras comparten la misma operación interna —slot
destino libre según el mismo servicio de disponibilidad que ya usa la
reserva, fecha futura, y, cuando el turno es una primera sesión,
revalidación de la aceptación de pacientes nuevos y de la restricción de
solo adultos— dentro de una transacción serializable con su propia entrada
de auditoría, siguiendo la misma técnica que ya fija la reserva para las
invariantes de lectura y escritura. No se revalida en cambio la
colocación de la franja extra de primera sesión: el documento de
requisitos describe la reprogramación como el movimiento de un turno por
su propio identificador, no como una nueva ubicación del par de turnos que
lo acompañó en la reserva original. La reorganización manual, además, no
se ejecuta como una única transacción de base de datos sobre todo el
lote: el documento de requisitos exige que un movimiento fallido se
informe sin abortar los que sí pudieron aplicarse, lo que exige que cada
uno persista con independencia del resultado de los demás.

La cancelación masiva por ausencia le dio, a su vez, su primer consumidor
real al punto de extensión que la gestión de ausencias había dejado
preparado desde la fase de Profesionales: el servicio de turnos implementa
`AbsenceEventsPort` en un adaptador propio que, ante el evento de ausencia
registrada, cancela todo turno reservado o confirmado del profesional
dentro del período informado, lo marca con un nuevo motivo de cancelación
específico para distinguirlo de una cancelación ordinaria, y publica un
segundo punto de extensión, `ReassignmentPort`, una vez por cada turno que
efectivamente cancela — un puerto enfocado y separado del de ausencias,
para no mezclar bajo un mismo contrato el registro de una ausencia con la
liberación puntual de un turno, y con el mismo adaptador *stub* que ya usa
el resto de los puertos de integración, a la espera del algoritmo de
reasignación de una fase posterior. Esa cancelación masiva se ejecuta de
forma sincrónica dentro del mismo pedido HTTP que registra la ausencia, y
por eso mismo no queda sujeta a la anticipación mínima que sí exige la
cancelación ordinaria — no es una decisión discrecional sujeta a un plazo
de aviso, sino la consecuencia obligada de que el profesional dejó de
estar disponible —, y cada turno del período se cancela en su propia
transacción, con su propio registro de error si algo falla, para que un
conflicto de concurrencia sobre un turno cualquiera no convierta en una
respuesta de error el registro de una ausencia que ya se persistió y
auditó correctamente, ni deje sin cancelar el resto del período.

Darle al puerto de eventos de ausencia su primer consumidor real expuso
además una restricción de composición que las fases anteriores no habían
enfrentado: el servicio de ausencias, alojado hasta entonces dentro del
módulo de Profesionales, pasó a depender del módulo de Turnos para
resolver ese puerto, mientras que el módulo de Turnos ya dependía del de
Profesionales para verificar la pertenencia de cada turno — proveer el
puerto directamente dentro de Profesionales habría cerrado un ciclo de
importación entre ambos. La solución fue extraer el servicio y el
controlador de ausencias a un módulo propio, ubicado por encima de los
otros dos, que importa a cada uno para lo que necesita sin que ninguno
necesite importarlo de vuelta — la misma clase de reorganización que ya
había exigido, en Fundaciones, la revisión de integridad y normalización
del esquema, aplicada esta vez al grafo de módulos de Nest en lugar de al
modelo de datos.

