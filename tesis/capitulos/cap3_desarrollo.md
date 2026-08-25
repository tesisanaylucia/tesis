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

Una auditoría posterior de la base de datos encontró una invariante del
modelo `User` que ninguna restricción expresaba: una cuenta con rol
profesional debe estar siempre vinculada a un profesional, y una cuenta con
rol administrador o de sistema nunca debe estarlo, pero la columna que
sostiene ese vínculo era simplemente anulable. La omisión no tenía todavía
consecuencia observable, dado que la gestión de usuarios no cuenta aún con
ningún endpoint de creación o edición más allá del sembrado inicial, pero
dejarla pendiente para el momento en que ese endpoint se construyera
arriesgaba perderla de vista. Se agregó una restricción `CHECK` que expresa
la invariante con una sola condición de igualdad, válida para los tres
roles a la vez: el rol profesional exige el vínculo, y los otros dos lo
prohíben. Al aplicar la restricción se descubrió que la propia rutina de
sembrado la infringía al retirar del roster a un profesional dado de baja,
pues desvinculaba la cuenta asociada sin degradar su rol; se corrigió
eliminando la cuenta en ese caso, ya que ningún otro rol corresponde a un
inicio de sesión cuyo profesional fue eliminado.

Una auditoría de código posterior, centrada esta vez en contrastar el
código de notificaciones, capa de IA, seguridad y aspectos legales contra
el documento de requisitos, señaló que el servicio de usuarios era la
única pieza del sistema que seguía consultando la base a través del
cliente de Prisma sin extender, en lugar de hacerlo a través de la
extensión de acotamiento por tenant descripta al comienzo de esta sección.
No existía una fuga real en ese momento —el único método que leía varios
usuarios a la vez recibía el identificador de la organización de su único
invocador, pasado a mano y correcto—, pero esa corrección dependía
enteramente de que ese invocador siguiera pasándolo bien, exactamente la
disciplina manual que la extensión se había construido para no requerir.
Un método nuevo que se agregara a ese servicio sin recordar filtrar por
organización habría expuesto usuarios y contraseñas cifradas de otras
organizaciones sin que ningún mecanismo lo impidiera, ni en tiempo de
desarrollo ni en tiempo de ejecución. Se migró el servicio al cliente
acotado por tenant, y de paso se retiró el parámetro de identificador de
organización que el único método de lectura múltiple recibía: una vez que
el filtro lo aplica la extensión a partir del contexto de la solicitud,
sostener además un parámetro explícito con el mismo propósito no aporta
nada y reabre la posibilidad de que ambos valores diverjan, que es
precisamente el riesgo que la migración buscaba cerrar.

Dos de los tres métodos del servicio, sin embargo, no pudieron migrarse:
ambos se invocan desde el inicio de sesión, antes de que exista un token
válido y, por lo tanto, antes de que haya una organización en el contexto
de la solicitud que la extensión pueda leer —intentarlo simplemente habría
provocado que la propia extensión rechazara la consulta—. Se los dejó
deliberadamente sobre el cliente sin extender, documentando en el código
por qué cada uno es seguro sin un filtro manual de todos modos: la
búsqueda por correo electrónico, porque esa columna es única en toda la
base y no solo dentro de una organización, de modo que no hay ninguna fila
de otra organización con la que pueda confundirse; y la verificación de si
el profesional vinculado a una cuenta sigue activo, porque el identificador
de profesional es una clave primaria única en toda la base y una clave
foránea compuesta ya vigente garantiza, a nivel de esquema, que el
profesional vinculado a una cuenta solo puede pertenecer a la misma
organización que esa cuenta. Esta excepción quedó incorporada al documento
de convenciones del repositorio como una categoría propia, distinta de las
que ya reconocía para el diseño de un modelo (pertenencia directa, hijo de
un padre acotado por tenant, vínculo entre dos padres acotados por tenant):
un método de servicio, no un modelo, puede justificar por sí mismo evitar
el cliente acotado, pero solo con el mismo tipo de argumento —qué otra
propiedad de la consulta ya limita el resultado a una única organización—
dejado por escrito.

La prueba de que un método nuevo hipotético queda automáticamente acotado
sin depender de que quien lo escriba recuerde filtrar ya existía, de forma
genérica para todo modelo con identificador de organización, en la
suite de extremo a extremo de la extensión de acotamiento: una consulta sin
ningún filtro explícito de organización, emitida a través del cliente
acotado, solo devuelve filas de la organización activa en el contexto. Se
sumó cobertura unitaria propia del servicio de usuarios, que verifica cuál
de los dos clientes usa cada método —y en particular que el método migrado
no recibe ni necesita ningún argumento de organización—, dejando la prueba
de aislamiento real, contra una base de datos de verdad, a la suite ya
existente.

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

Una revisión posterior, ya con el motor de turnos en desarrollo, encontró una
inconsistencia entre P1.1/P1.2 y el criterio de aceptación del chatbot que da
origen a la entidad matrícula: la fuente de verdad exige que el asistente pueda
mostrar "sus dos matrículas" de un profesional, pero el ABM nunca había exigido
un mínimo, de modo que un profesional podía quedar sin matrícula alguna o con
varias del mismo tipo. La corrección se limitó deliberadamente a un único punto
de escritura en lugar de exigir el mínimo en toda alta o edición del
profesional: la aceptación de pacientes nuevos —el indicador que P1.5 agregó a
la configuración y que el asistente conversacional consulta antes de ofrecer un
profesional— no puede activarse, ni permanecer activa tras una edición de esa
configuración, si el profesional no cuenta con al menos una matrícula
provincial y una nacional. Exigir el mínimo en el alta misma se descartó
porque contradice el propio diseño de P1.2, que permite cargar las matrículas
de forma incremental después de crear el profesional, y porque habría exigido
adaptar la numerosa batería de pruebas de otros módulos que crean un
profesional sin matrículas por no ser su objeto de prueba y dependen del valor
por defecto del indicador. Por la misma razón se dejó deliberadamente sin
cerrar un resquicio simétrico: nada impide retirarle a un profesional que ya
acepta pacientes nuevos la matrícula que lo habilitaba, lo que lo dejaría por
debajo del mínimo sin que ninguna verificación lo advierta; cerrarlo habría
exigido extender la misma condición a la edición y a la baja de matrículas, y
esa extensión resultó incompatible con una prueba preexistente que edita la
única matrícula de un profesional para cambiarle el tipo, ajena por completo a
este invariante. La inconsistencia queda documentada como una limitación
conocida en lugar de resuelta a costa de una prueba que no la motivó. Por
tratarse, igual que el tope de tres matrículas, de un invariante de lectura
seguida de escritura —se cuentan las matrículas vigentes y solo entonces se
permite activar el indicador—, la verificación se incorporó a una transacción
con aislamiento serializable, de modo que una eliminación de matrícula
concurrente con la activación no pueda dejar pasar a ambas.

Una última revisión, ya cerrada la fase, encontró que la matrícula carecía de
toda restricción de unicidad: nada impedía cargar dos veces la misma
combinación de tipo y número para un mismo profesional, un dato redundante
sin significado adicional que el mínimo introducido por la corrección
anterior no llegaba a cubrir, por regir sobre la cantidad y no sobre el
contenido de las matrículas. La corrección agregó una restricción de
unicidad compuesta por profesional, tipo y número al esquema, en reemplazo
del índice simple preexistente sobre el profesional —al que la nueva
restricción ya sirve de cobertura, por compartir su columna inicial—, y
tradujo la violación de esa restricción a un error de validación legible en
los dos puntos de escritura del servicio de matrículas, en lugar de dejar
propagarse el error interno de la base de datos. Se prefirió tratar la
violación como un error de validación antes que como un conflicto de
concurrencia, a diferencia del criterio ya adoptado para otras restricciones
de unicidad del sistema: no se trata de dos escrituras que compiten por un
recurso todavía inexistente, sino de una entrada rechazada por ser
redundante con datos que ya existen y son perfectamente visibles para quien
la envía. La restricción, al ser una propiedad de la base de datos y no una
verificación de la capa de servicio, protege por igual el alta y la edición
de una matrícula sin necesitar el aislamiento serializable que otros
invariantes de lectura seguida de escritura de este módulo sí requirieron.

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

Una revisión posterior del esquema, ya con el motor de turnos construido y
consumiendo este campo, encontró que la prioridad —hasta entonces un entero
libre entre 1 y 999 que el profesional cargaba a mano, sin una escala
definida— excedía lo que el algoritmo de reasignación por prioridad
realmente necesitaba: distinguir quién tiene prioridad sobre quién, no un
ranking numérico fino. Se la convirtió en un enumerado de cuatro niveles
(bajo, medio, alto y urgente), nulo con el mismo significado de "sin
prioridad asignada" que antes tenía el entero ausente. El desempate entre
dos pacientes del mismo nivel pasó a depender enteramente del orden de la
lista de espera y de la fecha de creación del registro correspondiente, ya
que un enumerado de cuatro valores no admite, dentro de un mismo nivel, el
desempate más fino que sí permitía el rango numérico anterior.

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

La misma revisión automatizada contra las fuentes de verdad que introdujo
el estado ausente del turno en el módulo de Turnos (más adelante en este
capítulo) encontró aquí que el profesional no tenía forma de registrar
datos faltantes de sus propios pacientes, pese a que el documento de
requisitos lista esa acción entre las que la aplicación móvil del
profesional debe permitir: el punto de edición del paciente estaba
restringido a administración y al proceso automatizado. Se resolvió
habilitando el mismo rol profesional sobre el mismo punto de acceso y el
mismo cuerpo de solicitud que ya usa administración, en vez de construir
uno más angosto limitado a los datos faltantes que ninguna fuente
describe así, acotado a los pacientes con los que el profesional tiene un
vínculo de tratamiento vigente —la misma restricción que ya rige el resto
de sus lecturas— y con la respuesta recortada a ese vínculo, en lugar de
traer los de todos los profesionales que atienden al mismo paciente.

Una revisión posterior, de prioridad baja, agregó a la respuesta del
paciente una proyección de conveniencia: la fecha de su última consulta,
calculada como el máximo entre las fechas de última consulta de todos sus
vínculos con profesionales. El diseño por vínculo no se cuestionó ni se
modificó —una sesión sigue perteneciendo a la relación con un profesional
determinado, no a la persona en abstracto, como ya fundamenta la
introducción de esa columna más arriba en esta misma subsección— pero el
documento de requisitos habla de la última consulta del paciente en
singular, un dato que hasta entonces sólo se obtenía con una llamada aparte
al punto de estado de datos o calculando el máximo del lado cliente. Se
extrajo el cálculo, ya existente para ese punto de estado de datos, a una
función compartida, de modo que la nueva proyección y la respuesta que ya la
ofrecía no puedan llegar a calcular el máximo de dos maneras distintas. El
campo se resuelve sobre los mismos vínculos que la respuesta del paciente ya
trae, de modo que hereda sin código adicional el recorte por vínculo que un
llamante con rol profesional ya tenía: no se introdujo ninguna comprobación
de autorización propia. No hubo cambios de esquema.

Una auditoría posterior del código encontró que la configurabilidad del plazo
de inactividad, decidida en la tarea que introdujo la regla del año, nunca
había sido efectivamente alcanzable: el servicio la leía correctamente desde
la tabla de configuración por inquilino, pero ningún punto de acceso permitía
escribirla, de modo que toda organización quedaba fija en el valor por
defecto. Se encontró además que la lectura no aplicaba ningún límite
superior al valor configurado, pese a que el documento de requisitos enuncia
expresamente "un año máximo". Se agregó en consecuencia un punto de acceso
administrativo que valida el nuevo valor como un entero entre uno y doce
meses antes de persistirlo, y se incorporó el mismo tope como una segunda
comprobación en la lectura ya existente, de modo que una fila que llegara a
superarlo por una vía distinta al punto de acceso —una fila anterior a esta
corrección, o una escritura directa sobre la base— quede igualmente acotada.
Ambas comprobaciones leen una única constante compartida, para que el tope no
pueda actualizarse en un lugar y quedar desactualizado en el otro. Un valor
que exceda el tope se recorta a doce en la lectura en lugar de descartarse
como hace un valor sin sentido —no numérico o no positivo—, distinción
deliberada entre un dato inválido y un dato válido pero excesivo. La
escritura, al tratarse de la primera invocación real del método de
configuración por inquilino desde que existe, deja además una entrada de
auditoría con la clave y el nuevo valor, siguiendo el mismo criterio que el
proyecto ya aplica a otros cambios de configuración administrativa. El
inquilino sobre el que se escribe nunca es un parámetro de la solicitud sino
que se deriva del token de quien administra, de modo que el aislamiento entre
organizaciones no requirió una comprobación adicional a la del rol.

Una auditoría posterior, esta vez sobre la propia escritura que la tarea
anterior había agregado, encontró que el cambio de umbral y la entrada de
auditoría que lo describe no compartían ninguna garantía de todo-o-nada: el
método de configuración por inquilino escribía el nuevo valor sobre el
cliente de base de datos directo, y la entrada de auditoría se generaba
recién después, sin ningún `$transaction` que envolviera a ambas. Una falla
del proceso, o de la propia escritura de auditoría, justo después de una
configuración ya confirmada dejaba el umbral de inactividad de la
organización modificado sin ningún rastro de quién lo había cambiado ni
cuándo, un vacío de cumplimiento significativo porque esa regla reclasifica
pacientes y dispara los recordatorios de actualización de contacto que una
fase posterior construye sobre ella. La corrección adaptó el método de
configuración por inquilino para aceptar, además del cliente por defecto, el
handle de una transacción en curso, el mismo mecanismo que el registro de
auditoría ya ofrecía desde su creación, y envolvió la escritura del umbral y
su entrada de auditoría en una misma transacción, siguiendo el patrón ya
aplicado en esta misma subsección y, más adelante, en la de notificaciones y
trabajos programados, para el resto de las escrituras acompañadas de
auditoría.

Una tercera auditoría, esta vez orientada a las convenciones de nomenclatura
del propio proyecto, encontró que el punto de acceso administrativo del
umbral de inactividad rompía una regla explícita: el idioma de una ruta HTTP
en este sistema se decide por la superficie a la que pertenece, y toda ruta
que empieza con el segmento de configuración administrativa se mantiene en
español de punta a punta, salvo esta, cuyo segmento final había quedado en
inglés desde que se escribió. La inconsistencia no afectaba ningún
comportamiento observable —la validación, el cálculo del corte de fecha y el
resto de la lógica no se vieron tocados— pero sí el patrón del que una
persona que se incorpore al proyecto infiere la convención, ya que una única
excepción sin justificar debilita la regla para todas las rutas que la
respetan. La corrección reemplazó el segmento final por su forma en español,
sin abrir un período de convivencia entre ambas rutas: el propio relevamiento
que originó la tarea constató que el único consumidor de la ruta anterior era
la suite de pruebas del propio proyecto, de modo que mantener un alias en
inglés habría reintroducido la misma inconsistencia que se buscaba eliminar.

Una auditoría más sobre el importador de pacientes preexistentes encontró
que la lectura de un archivo separado por comas no contemplaba más que una
codificación. El lector decodifica todo `.csv` como UTF-8 sin ningún
resguardo, y una planilla histórica exportada por un Excel en español como
"CSV (delimitado por comas)" —el caso de uso exacto que esta funcionalidad
existe para migrar— se escribe en Windows-1252/ANSI, donde una letra
acentuada o una ñ ocupa un único byte que no es UTF-8 válido. Ante ese byte,
la decodificación no falla: lo reescribe en silencio como el carácter de
reemplazo Unicode, de modo que "José María" se convertía en "Jos� Mar�a" y la
fila se persistía igual, porque ese texto corrompido sigue siendo una cadena
no vacía y ningún validador de forma puede distinguirlo de un nombre real. La
corrección agregó, en la capa de lectura, un chequeo posterior a la
decodificación de cada fila —el mínimo que la propia auditoría proponía como
aceptable frente a una detección de codificación en sentido estricto, que
tendría que adivinar el origen de una información que la decodificación ya
perdió— y evalúa cualquier celda de la fila, no sólo las que el importador de
pacientes reconoce: si una columna que el importador ignora aparece
corrompida, el resto de la fila se decodificó con el mismo problema, aunque
"nombre" y "apellido" no tuvieran acentos en ese registro puntual y por eso no
lo delataran por sí solos. La validación de la fila corta antes de construir
el registro cuando esa marca está presente, de modo que la fila se rechaza
—nombrada en el informe, con la fila y el archivo señalados como el resto de
los rechazos del importador— en lugar de reintentarse con otra codificación
supuesta, opción descartada porque reintentar siempre produce algún
resultado, correcto o no, y arriesgaría introducir un nombre plausible pero
equivocado si el archivo estuviera en una tercera codificación distinta.

La misma auditoría señaló, sobre el mismo módulo, un punto que no calificó
como defecto sino como pregunta abierta: desde la tarea que incorporó la
prioridad del vínculo paciente-profesional, su edición admite tanto al
profesional del vínculo como a cualquier ADMIN, comportamiento documentado y
deliberado desde su origen —la misma comprobación que ya regía la lectura—
pero sobre el que no constaba si coincidía con la intención del documento de
requisitos, que atribuye la carga de la prioridad específicamente al
profesional. Antes de decidir, se releyó la fuente de verdad: el documento de
Especificación de Requisitos vigente enumera, sin mencionar ninguna vía
administrativa, que la prioridad "la define y carga el profesional desde la
app". La verificación resolvió la pregunta a favor de la exclusividad, y la
corrección restringió únicamente la escritura —dejando la lectura
administrativa intacta, ya que el texto verificado no dice nada sobre quién
puede leer la prioridad, a diferencia de la frase que rige las observaciones
de uso interno y que sí nombra explícitamente la lectura— reemplazando, sólo
en ese punto de acceso, la comprobación permisiva por la estricta que la
tarea de las observaciones ya había introducido para un caso hermano dentro
del mismo controlador. La corrección no agregó código de autorización nuevo:
reutilizó el par de comprobaciones existente, cambiando únicamente cuál de
las dos protege la escritura de la prioridad.

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
confirmado. El paso de generación se fijó en esta etapa en la duración de la
consulta y no en la cadencia de apertura de franjas —una configuración
distinta del profesional, agregada al modelo poco antes y todavía sin
consumidor—, en el entendido de que así lo especificaba el documento de
requisitos para este cálculo puntual; una revisión posterior, descrita al
cierre de esta subsección, encontró que esa lectura no era correcta y que la
cadencia debía ser el paso. La
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

La fase continuó con el algoritmo de reasignación que el punto de
extensión `ReassignmentPort` había dejado preparado sin consumidor real:
qué hacer con un turno que acaba de quedar liberado por una cancelación.
Según la modalidad de reasignación que el profesional tiene configurada
—un interruptor que ya existía en el esquema desde la fase de
Profesionales, sin uso hasta este punto—, el turno liberado se ofrece por
orden de prioridad a la lista de espera del profesional, o simplemente
queda disponible sin que se contacte a nadie. El recorrido de la lista de
espera en modalidad automática ordena a los candidatos por la prioridad
que el profesional les asignó individualmente —el mismo campo que la fase
de estados y prioridad había dejado expuesto sin consumidor—, luego por el
orden de llegada a la lista y luego por la fecha de solicitud; un
candidato que todavía no tiene vínculo registrado con el profesional se
trata como de prioridad nula en lugar de interrumpir el recorrido completo.
Para cada candidato se envía una oferta a través del puerto de mensajería
ya existente y se espera una respuesta de aceptación: si acepta, el turno
original pasa al estado "reasignado" —la transición que la máquina de
estados de la fase anterior ya había declarado válida sin que ningún
método la ejecutara—, se crea un turno nuevo para ese paciente en el mismo
horario y con la misma duración, y el paciente se retira de la lista de
espera; si rechaza, se continúa con el siguiente candidato; si la lista se
agota sin que nadie acepte, o si está vacía, el turno simplemente queda
liberado, sin error.

Como todavía no existe un canal real de contacto por WhatsApp ni el
temporizador de espera que el documento de requisitos atribuye a fases
posteriores, la aceptación del paciente se resolvió detrás de un puerto de
dominio nuevo y separado del de mensajería, cuyo adaptador *stub* responde
siempre que nadie aceptó —una respuesta negativa por defecto, para que
ninguna reserva ocurra a partir de una aceptación que nadie emitió
realmente mientras ese canal no exista—, en lugar de mezclar esa pregunta
dentro de un puerto de mensajería cuyos métodos son de una sola vía. El
gancho de reasignación, que la fase anterior solo disparaba desde la
cancelación masiva por ausencia, se conectó también a la cancelación
ordinaria de un turno: el documento de requisitos describe el servicio de
reasignación como algo que se invoca en general cuando un turno pasa a
cancelado, sin acotarlo a una única vía de llegar a ese estado, y dejarlo
conectado solo a la cancelación por ausencia habría dejado sin cubrir el
caso más frecuente, la cancelación voluntaria del paciente o del
profesional.

El nuevo módulo que implementa el algoritmo se organizó de manera que no
dependiera del módulo de Turnos, pese a proveer la implementación real de
un puerto que ese módulo consume: de haber necesitado el servicio de
turnos para crear el turno de reemplazo y transicionar el original, se
habría cerrado un ciclo de importación, ya que el módulo de Turnos
necesita a su vez importar el nuevo módulo para obtener esa implementación
real. El algoritmo opera en cambio directamente sobre la tabla de turnos a
través del cliente de Prisma acotado por inquilino — la misma clase de
restricción de composición que ya había resuelto, en la fase anterior, la
extracción del módulo de ausencias, aplicada esta vez en la dirección
opuesta. Por la misma razón por la que no existe un usuario de sistema
sembrado en el sistema, las entradas de auditoría que una reasignación
automática genera se atribuyen al mismo actor humano que canceló el turno
que la disparó, dato que ambas vías de cancelación ya tenían disponible en
el momento en que publican el evento.

La fase continuó con la gestión de la lista de espera que el
algoritmo anterior consume: alta de un paciente con su obra social
opcional y con la posición siguiente asignada automáticamente, listado
ordenado, reordenamiento manual por el profesional —reemplazo completo del
orden a partir de la secuencia de identificadores que este envía, con la
misma técnica de reemplazo idempotente que ya usa la grilla de horario
semanal— y baja, con la misma restricción de "administrador o profesional
dueño" que ya rige el resto de los recursos acotados a un profesional. El
criterio de anteponer un paciente recurrente a uno nuevo, que el documento
de requisitos deja a criterio del profesional, se resolvió enteramente a
través del campo de prioridad numérica ya existente, sin agregar un
interruptor de configuración adicional: un profesional que quiere
anteponer a sus pacientes recurrentes lo expresa subiéndoles la prioridad
individualmente, el mismo mecanismo que ya usa para cualquier otro
paciente al que quiere anteponer.

La fase se cerró, sin agregar entidades ni endpoints nuevos, con una suite
de tests de extremo a extremo que ejercita el motor de turnos completo tal
como lo describen en conjunto las tareas anteriores, en lugar de cada regla
por separado: disponibilidad, reserva, confirmación, cancelación,
reasignación y completado encadenados sobre un mismo turno, contra una base
de datos real y con los puertos sin canal externo todavía —mensajería y
respuesta de lista de espera— sustituidos por dobles configurables. A
diferencia de las suites de módulo, que sustituyen también el puerto de
reasignación por un espía para aislar la prueba de cancelación de la de
reasignación, esta suite deja conectado el adaptador real de ese puerto,
porque su objetivo es precisamente comprobar que una cancelación real
dispara una reasignación real, no una versión simulada de ella. Además del
recorrido completo, la suite aloja dos escenarios que ninguna suite de
módulo cubre por sí sola: las tres modalidades de franja extra para
paciente nuevo ejercitadas juntas, y un escenario con dos organizaciones
simultáneas cuyos profesionales y pacientes comparten nombre y documento
—un caso válido, porque la unicidad del documento está acotada por
organización y no es global—, para comprobar que el acotamiento por
inquilino sostiene a la vez la reserva, la cancelación, la reasignación y
la lista de espera. Los candidatos de lista de espera que usa la suite se
crean sin los datos obligatorios de una reserva ni consentimiento
registrado, porque el algoritmo de reasignación crea su turno directamente
y no pasa por esas validaciones. Ejecutada en serie, la suite end-to-end
completa del backend corre en verde; en el modo paralelo por defecto,
algunas pruebas de esta suite y de otras ya existentes fallan de forma
intermitente por conflictos de una transacción serializable al compartir
todos los archivos una misma base de datos local — una condición
preexistente del entorno de pruebas que esta tarea deja señalada como
pendiente de una fase de endurecimiento posterior, en lugar de resolverla
aquí.

El feriado, cuya entidad y consumo por el algoritmo de disponibilidad ya
había resuelto esta fase, todavía carecía de una vía para que un
administrador gestionara su propio calendario sin intervenir la base de
datos directamente: una tarea posterior cerró ese faltante con cuatro
endpoints de administración —alta, baja, edición de la descripción y
listado por año— restringidos al rol administrador y resueltos, como el
resto del sistema, mediante el acotamiento automático por inquilino del
cliente de Prisma; un feriado de otra organización resulta así
tan inexistente para estas operaciones como uno que nunca se creó, con
la misma respuesta de "no encontrado" que ya recibe cualquier otro
recurso ajeno al inquilino, en lugar del código de "prohibido" que
describe literalmente el documento de requisitos. La eliminación de un
feriado no bloquea su ejecución ante la presencia de turnos ya
reservados o confirmados en esa fecha —ningún turno existente deja de
ser válido porque la fecha deje de estar marcada como feriado—, pero
informa en la propia respuesta cuántos turnos coexisten con la fecha
recién liberada, a modo de advertencia para una eventual acción manual
del administrador.

Una revisión posterior del esquema completo del módulo, sin agregar
funcionalidad nueva propia, corrigió cuatro puntos concretos. Confirmó,
en primer lugar, que la lista de espera debe conservar su propio
identificador de organización: aunque tanto el paciente como el
profesional que enlaza ya lo llevan de forma directa, la lista de espera
tiene dos padres acotados por organización, y ese único campo es lo que
obliga a ambos a pertenecer a la misma, mediante las dos claves foráneas
compuestas que ya la vinculan a cada uno —el mismo caso, ya razonado en
esta fase, del turno y del vínculo paciente-profesional—. Retiró, en
segundo lugar, dos campos de la lista de espera que ninguna ruta de
lectura ni el propio algoritmo de reasignación consumían: la obra social
opcional que se había incorporado junto con el alta del recurso, y la
fecha de solicitud, duplicado exacto de la fecha de creación de la fila
—el registro de la lista de espera se crea en el mismo instante en que
el paciente pide el turno, de modo que ambas columnas sólo podían
contener siempre el mismo valor—. Retiró, con el mismo razonamiento, la
fecha de solicitud de la solicitud de receta del módulo de Pacientes,
duplicada de la misma manera respecto de su propia fecha de creación.

Agregó, en tercer lugar, un nuevo estado al turno —ausente— para el caso
en que el paciente no se presenta, distinto de completado, alcanzable
únicamente desde confirmado y con la misma restricción de origen que ya
regía la transición a completado desde la fase anterior: recién en ese
estado el profesional puede efectivamente distinguir si la sesión
ocurrió o no. Junto con el nuevo estado se incorporó una tarea
programada semanal que resuelve, como completado y nunca como ausente,
todo turno reservado o confirmado cuyo horario ya pasó sin que el
profesional lo haya marcado de una forma o de la otra —marcar una
ausencia es una decisión que el sistema no puede inferir por sí solo,
mientras que asumir que la sesión ocurrió es la opción conservadora que
el propio pedido de la revisión estableció como comportamiento por
omisión—.

Esa tarea programada expuso, en cuarto lugar, un vacío que ninguna
funcionalidad anterior había necesitado llenar: el rol reservado para
procesos automáticos, declarado desde la corrección de Fundaciones que
lo agregó, nunca había sido sembrado como una fila real, porque hasta
entonces toda entrada de auditoría se atribuía al actor humano o
conversacional de la solicitud que la originaba, y una tarea programada
no tiene una solicitud detrás. Se sembró entonces, por organización, el
usuario de ese rol que el propio rol ya anticipaba —con un correo
determinístico derivado del identificador de la organización, para que
la tarea programada pueda ubicarlo por organización y rol sin necesitar
almacenar ni propagar un identificador propio—, en una migración de
datos para las organizaciones ya existentes y en el mismo script de
siembra de desarrollo para las que se crean después, siguiendo el mismo
patrón de doble siembra que ya usa el umbral de inactividad de
pacientes. La alternativa de debilitar la traza de auditoría para
admitir una mutación sin actor se descartó por introducir, sólo para
este caso, una excepción a una regla que hasta entonces no tenía
ninguna.

Una revisión automatizada del código completo contra el anteproyecto de
tesis y el documento de requisitos, ejecutada con varios agentes en
paralelo y triada punto por punto por la usuaria, encontró seis brechas
concretas entre lo construido hasta ese momento y lo que esas fuentes
describen, y las cerró sin agregar entidades nuevas al esquema salvo una.
Encontró, primero, que el camino de escritura de turnos —la reserva y la
reprogramación— nunca consultaba el calendario de feriados, a diferencia
del cálculo de franjas libres que sí lo hacía desde la fase de
disponibilidad: un turno podía reservarse o reprogramarse directamente a
una fecha feriada sin pasar por las sugerencias que sí la excluían. Se
resolvió incorporando esa consulta al método que ambas rutas de escritura
ya comparten para decidir si un instante está libre, en vez de repetirla
en cada una por separado, para que las dos no puedan llegar a definir
"libre" de manera distinta entre sí.

Encontró, segundo, que un turno recién liberado por una cancelación era
inmediatamente reservable por cualquiera, tanto en la modalidad manual de
reasignación —donde el documento de requisitos pide una retención de
veinticuatro horas antes de que el bot pueda volver a ofrecerlo— como en
la automática, mientras el algoritmo todavía recorre la lista de espera
ofreciéndolo a sus candidatos. La retención se modeló con una única marca
de tiempo sobre el propio turno cancelado en lugar de una tabla aparte,
que alcanza para expresar los dos casos: en la modalidad manual es un
plazo real de veinticuatro horas que no necesita liberarse explícitamente,
porque el turno cancelado simplemente deja de contarse como ocupado en
cuanto el reloj supera esa marca; en la automática es sólo un techo de
seguridad ante una caída del proceso a mitad del recorrido, y la retención
real se libera explícitamente en cuanto ese recorrido concluye, sea porque
alguien aceptó la oferta o porque la lista se agotó sin que nadie
aceptara. La duración de ese techo se dejó como una constante aparte,
explícitamente distinta de la ventana real de "cuánto tiempo tiene un
candidato para responder", que pertenece a una fase posterior y todavía
no existe.

Encontró, tercero, que la reprogramación individual y la reorganización
manual de la agenda aplicaban el nuevo horario y recién después avisaban
al paciente, sin esperar ninguna respuesta suya, pese a que el documento
de requisitos pide que el sistema le pregunte si acepta el cambio antes
de aplicarlo. Se resolvió con un puerto de dominio nuevo que reproduce
deliberadamente la forma del que ya resuelve la aceptación de una oferta
de lista de espera: se pregunta antes de escribir, y el adaptador de
reemplazo —mientras no exista un canal real de WhatsApp— siempre responde
que no, para que ninguna reprogramación se dé por aceptada sin que nadie
realmente la haya aceptado. Esa confirmación sólo se exige cuando
administración o el propio profesional reprograman por su cuenta, nunca
cuando lo hace el proceso automatizado en nombre del paciente, porque en
ese caso el pedido del propio paciente ya es su aceptación; la
reorganización manual de la agenda, en cambio, la exige siempre, porque a
esa ruta nunca llega el proceso automatizado. Con el único adaptador
disponible respondiendo siempre que no, toda reprogramación unilateral
queda hoy efectivamente bloqueada hasta que una fase posterior conecte el
canal real — la misma situación, ya aceptada, en la que se encuentra la
reasignación automática de turnos desde que se implementó su propio
puerto de respuesta.

Encontró, cuarto, que un profesional no tenía forma de registrar datos
faltantes de sus propios pacientes, pese a que el documento de requisitos
lista esa acción entre las que la aplicación móvil del profesional debe
permitir: el punto de acceso de edición del paciente estaba restringido a
administración y al proceso automatizado. Se resolvió reutilizando
exactamente el mismo cuerpo de solicitud y el mismo servicio que ya usa
administración, en lugar de construir un formulario más angosto limitado
a los datos faltantes que el documento de requisitos no describe, con el
alcance acotado a los pacientes con los que el profesional tiene un
vínculo de tratamiento vigente —la misma restricción que ya rige el resto
de sus lecturas— y la respuesta recortada a su propio vínculo.

Encontró, quinto, que no existía ningún punto de acceso de backend para
la agenda propia del profesional, la fuente que las vistas diaria,
semanal y mensual de la aplicación móvil necesitan: sólo existía la
consulta de franjas libres, que responde una pregunta distinta. Se
agregó como un punto de acceso nuevo, separado de la disponibilidad y
restringido al propio profesional o a administración —a diferencia de la
disponibilidad y las ausencias, abiertas a cualquier usuario autenticado
del inquilino—, porque un turno nombra a un paciente concreto y "qué
pacientes ve este profesional y cuándo" no es información de agenda
pura.

De los dieciséis hallazgos que arrojó esa revisión, los seis anteriores
son los que se resolvieron de inmediato; el resto quedó explícitamente
marcado para otra etapa —ya sea por tratarse de una decisión de diseño ya
tomada y confirmada, ya sea por corresponder a un cimiento (precios y
copago, distinción de la obra social provincial, código de acceso y su
ventana de validez, orquestación de la cerradura electrónica,
notificación al profesional) que el propio anteproyecto ubica en una fase
posterior del proyecto.

Una auditoría posterior del código sobre `main`, hecha al preparar el
trabajo de la aplicación móvil del profesional (fase 7 del anteproyecto,
que da por existente un punto de acceso `GET /turnos` con filtros por
profesional y por rango de fechas) y el alta y baja de turnos desde
administración, encontró que ese punto de acceso general nunca se había
implementado: el quinto hallazgo de la revisión anterior había agregado
la agenda propia del profesional, acotada a un identificador de
profesional nombrado en la propia ruta y sin más filtro que el rango de
fechas, pero ningún ticket del módulo había expuesto la lectura de
turnos por paciente ni por estado, ni una que admitiera filtrar por
cualquier profesional del inquilino en una sola ruta. Se agregó entonces
`GET /turnos`, con cuatro filtros —profesional, paciente, rango de
fechas y estado— combinables entre sí, cada resultado acompañado de los
datos mínimos del paciente (nombre y apellido) que el documento de
requisitos pide para esta lectura en particular, y con la respuesta
paginada, a diferencia de la agenda propia del profesional, ya que aquí
el conjunto de resultados no está acotado de antemano por un único
identificador de profesional y un rango de fechas con tope.

El punto de acceso general reutiliza, para el acotamiento por rol, el
mismo criterio que ya aplican las transiciones de estado de un turno
individual: un profesional que nombra explícitamente el identificador de
otro recibe prohibido (403), no "no encontrado", porque ya conoce su
propio identificador y negarlo no revela nada nuevo; en cambio, cuando el
filtro es por paciente, se aplicó el criterio contrario —"no
encontrado" ante un paciente con el que el profesional no tiene vínculo
de tratamiento—, siguiendo la distinción ya fijada para el resto de las
lecturas de pacientes, porque en ese caso el profesional no tiene
conocimiento previo de a quién trata otro. Cuando ni un profesional ni un
paciente se nombran en la consulta, se exige al menos uno de los dos por
parte de administración —el propio documento de requisitos lo recomienda
para evitar un recorrido sin acotar de los turnos del inquilino—,
exigencia que nunca llega a manifestarse para un profesional porque su
propia identidad ya cumple ese rol de forma incondicional, se la nombre o
no en la consulta.

El segundo hallazgo de esa misma revisión —que un turno liberado por una
cancelación quedaba inmediatamente reservable por cualquiera, tanto en
modalidad manual de reasignación, donde el documento de requisitos exige
una retención de veinticuatro horas antes de que el sistema pueda volver a
ofrecerlo, como en la automática, mientras el algoritmo todavía recorre la
lista de espera— dio lugar más adelante a un ticket formal de corrección
sobre la tarea original del algoritmo de reasignación por prioridad. Al
retomar ese ticket se confirmó que la implementación ya estaba hecha,
fijada por la propia revisión el mismo día: una columna de fecha y hora en
el turno que expresa el plazo de retención, leída por el servicio de
disponibilidad como parte de su misma definición de "ocupado". Lo que
faltaba era la prueba directa de los tres criterios de aceptación que el
ticket fijaba en términos de ese servicio de lectura —un turno retenido
hace una hora no debe ofrecerse, uno retenido hace veinticinco sí, y la
modalidad automática no debe verse afectada—, cobertura que hasta entonces
sólo existía indirectamente, sobre el estado del turno cancelado o sobre
el chequeo interno de un instante puntual, nunca contra el propio cálculo
de franjas libres con una base de datos real. La corrección quedó así
acotada a tests: dos casos que fijan la marca de retención directamente en
los dos extremos del plazo de veinticuatro horas y verifican el resultado
de la consulta de disponibilidad, y una verificación del flujo de
cancelación real que confirma que la modalidad automática libera la
retención en cuanto concluye su propio recorrido de la lista de espera.

Una tercera auditoría automatizada, corrida el mismo día sobre `main`,
señaló un hallazgo de naturaleza distinta a los anteriores: no un
requisito sin implementar, sino la ausencia total de una funcionalidad ya
dada por cerrada. Reportó que el ABM administrativo del calendario de
feriados —controlador, servicio y las cuatro rutas de alta, consulta,
edición y baja— no existía en el código de `main`, pese a que el ticket
correspondiente figuraba como terminado; sólo encontró el modelo de datos
del feriado y su lectura de sólo lectura desde el cálculo de
disponibilidad. Se abrió un ticket formal para determinar con certeza el
estado real de esa funcionalidad antes de decidir cualquier acción sobre
el ticket original —reabrirlo, fusionar una rama pendiente o corregir un
cierre prematuro—, en lugar de confiar directamente en el hallazgo de la
auditoría. La verificación, hecha sobre una copia recién actualizada del
repositorio, encontró el controlador, el servicio y las cuatro rutas
presentes en `main`, incorporados por una fusión registrada el mismo día
en que corrió la auditoría; el ticket original no se cerró de forma
prematura ni quedó código huérfano en una rama sin fusionar; simplemente
la auditoría se ejecutó, con toda probabilidad, contra una copia del
repositorio anterior a esa fusión. No hizo falta entonces ningún cambio de
código ni de estado en Jira, sólo dejar registrada la verificación. El
episodio es afín a la lección ya dejada al implementar la máquina de
estados del turno —que una rama en curso debe compararse contra
`origin/main` recién traído y no contra una referencia local que no
avanza sola—, aplicada aquí no a un chequeo de conflictos antes de fusionar
sino a la interpretación de un hallazgo de auditoría: un reporte de
ausencia de código es tan dependiente de la frescura de la copia auditada
como un chequeo de conflictos, y merece la misma verificación explícita
antes de actuar sobre él, en particular antes de revertir el estado de un
ticket ya dado por terminado.

Una cuarta auditoría, dirigida esta vez puntualmente a la regla de
franja extra para paciente nuevo, encontró un hallazgo del primer tipo: un
campo de configuración que se persistía y se exponía sin que ninguna
lógica lo leyera. El diagrama de base de datos y la tarea que declaró la
configuración del profesional habían modelado la franja extra con dos
campos separados —una cantidad de horas y una modalidad de tres valores—,
pero al implementarse la regla de paciente nuevo la cantidad de horas
nunca se incorporó al cálculo: sólo la modalidad terminó gobernando dónde
se ubica el doble turno, exactamente como se describe más arriba en esta
misma sección. La auditoría no dio por sentado cuál de las dos lecturas
posibles correspondía —aplicar la cantidad de horas al cálculo, o
reconocerla como diseño descartado— y dejó la decisión, junto con su
justificación, a cargo de quien retomara el ticket. Releer el propio
documento de requisitos que dio origen a la regla de paciente nuevo
resolvió la pregunta: describe la franja extra únicamente en términos de
las tres modalidades de ubicación, y define el turno adicional con la
misma duración de consulta del profesional, sin ninguna referencia a una
cantidad de horas que ensanche o desplace ese turno. La cantidad en horas
no había quedado sin conectar por un olvido de implementación; había
quedado obsoleta en el momento en que la regla efectivamente
implementada resolvió el posicionamiento con las tres modalidades. Se
retiró entonces la columna con una migración que deja registrado el
motivo en su propio comentario, siguiendo el mismo criterio ya aplicado al
retirar el campo de diagnóstico agregado fuera de alcance en Plataforma
base, y se actualizaron en consecuencia el punto de configuración, la
respuesta que expone los datos del profesional y los datos de ejemplo
sembrados para el ambiente piloto.

Una quinta auditoría, de tipo multi-agente y con foco declarado en
multi-tenancy y seguridad, encontró un hallazgo distinto a los anteriores:
no un requisito sin implementar ni un dato huérfano, sino un chequeo de
autorización incompleto en un endpoint ya en producción. La consulta de
lista de espera por profesional aplicaba el chequeo de inquilino —que la
lista pedida pertenezca a la misma organización de quien pregunta— pero no
el chequeo de pertenencia dentro de ese inquilino, que sí aplicaban las
otras tres operaciones del mismo servicio (alta, reordenamiento y baja de
una entrada). La consecuencia era que un profesional autenticado podía
pedir la lista de espera de un colega de su propia clínica y recibir el
listado completo de pacientes en espera de ese colega, con quienes no
tenía ningún vínculo de tratamiento —una fuga de datos de pacientes entre
profesionales de la misma organización, distinta en naturaleza de las
fugas entre organizaciones que el chequeo de inquilino ya impedía. La
corrección reusó el mismo mecanismo de pertenencia que ya centralizaban
las otras tres operaciones del servicio, en vez de introducir un segundo
mecanismo de control de acceso —un guardián de ruta— para la misma regla
dentro del mismo módulo: la consulta ahora recibe también la identidad de
quien pregunta y aplica esa verificación antes de tocar la base de datos,
de modo que un profesional ajeno recibe una respuesta de acceso denegado
sin que la lista llegue siquiera a leerse. Se revisaron además el resto de
los accesos de lectura a la lista de espera dentro del módulo para
descartar el mismo hueco en otro lugar; los únicos otros puntos de lectura
resultaron ser internos al propio motor de reasignación, disparados por
eventos del sistema y no por un identificador de profesional que un
llamador externo pueda elegir, por lo que no compartían el problema.

Una sexta auditoría, dirigida esta vez al ángulo motor de turnos/crons,
encontró un hallazgo de una tercera naturaleza distinta a las cinco
anteriores: no un requisito sin implementar, un dato huérfano ni un
chequeo de autorización incompleto, sino dos operaciones del sistema —una
de este módulo, la reprogramación de turnos, y dos del módulo de
Notificaciones y Scheduler— que dejaron de coordinarse entre sí a medida
que se implementaron por separado. La reprogramación actualiza la fecha
de un turno sin tocar dos marcas de tiempo que sólo tienen sentido para
los trabajos programados: cuándo se le pidió confirmación al paciente y
cuándo se le envió el recordatorio. Mientras esas dos marcas existieron
en aislamiento no había ningún problema visible; el defecto sólo se
manifiesta en la intersección de ambos módulos, cuando un turno al que ya
se le pidió confirmación —o ya se le envió recordatorio— se reprograma
antes de que ese ciclo termine: el trabajo de auto-cancelación por falta
de respuesta cancelaba el turno reprogramado sin haber preguntado nunca
por la fecha nueva, y el de recordatorio se saltaba en silencio el aviso
de la fecha nueva por creer, con la marca de la fecha vieja todavía
puesta, que ya lo había enviado. La corrección resetea ambas marcas en el
mismo punto de escritura que cambia la fecha del turno, dentro de la
misma transacción, en vez de repartir la responsabilidad de mantener esa
invariante entre el punto que reprograma y cada uno de los trabajos que la
consumen — la propia guarda de idempotencia que cada trabajo ya usaba para
no repetirse dos veces es lo que, puesta de nuevo en su valor inicial,
hace que cada uno vuelva a tratar la fecha nueva como si nunca la hubiera
procesado.

La misma sexta auditoría encontró, dentro del mismo módulo de lista de
espera, un segundo hallazgo también nacido en la intersección de dos
módulos que se implementaron por separado: la baja de una entrada de la
lista de espera podía fallar con un error de integridad referencial sin
manejar en vez de completarse. Cuando un candidato recibe una oferta
automática de un turno liberado y la rechaza, o la oferta vence sin
respuesta, el registro de esa oferta queda en la base con su estado
final pero sigue apuntando a la entrada de lista de espera que la
originó —solo el camino de aceptación de una oferta limpiaba esa
referencia antes de borrar la entrada correspondiente, porque hasta
entonces era el único lugar del sistema que borraba una entrada de lista
de espera—. Una baja explícita posterior sobre esa misma entrada, pedida
por quien administra la lista, chocaba entonces con la restricción de
integridad que impide borrar un registro todavía referenciado, y el
candidato quedaba sin ninguna vía para salir de la lista de espera. La
corrección generaliza al camino de baja explícita el mismo mecanismo que
ya usaba el de aceptación: limpiar esa referencia dentro de la misma
transacción, inmediatamente antes de borrar la entrada, de modo que los
dos únicos puntos del sistema que borran una entrada de lista de espera
mantienen la invariante de la misma manera.

Una séptima auditoría, dirigida esta vez al ángulo de las convenciones
documentadas en el propio archivo de reglas del repositorio, encontró un
hallazgo distinto a los anteriores: no una omisión funcional ni una
integridad referencial rota, sino una traza de auditoría que registraba
más de lo que la propia regla del proyecto permite. La entrada que deja
la reprogramación de un turno anotaba, en su campo de detalle, los dos
timestamps involucrados —el horario anterior y el nuevo—, mientras que la
regla del repositorio es explícita en que ese campo debe nombrar qué
cambió, nunca el valor que tomó. El camino hermano de escritura de
campos sueltos del turno, usado por ejemplo al registrar el pago o la
orden de derivación, ya seguía la regla correctamente, de modo que la
reprogramación resultó ser el único punto de mutación de turnos que se
había apartado de ella. El motivo por el que importa: una exportación de
cumplimiento pensada para responder "qué cambió en este turno" terminaba
revelando, de paso, el propio dato personal —el horario de agenda del
paciente— que la regla existe para mantener fuera de esa columna. La
corrección reemplaza los dos timestamps por el nombre del único campo
que este flujo modifica, siguiendo el mismo patrón ya usado en el camino
hermano, sin alterar la lógica de reprogramación en sí.

La misma séptima auditoría, en su ángulo de reuso y simplificación,
encontró un hallazgo relacionado en cuatro puntos distintos del motor de
turnos y de la lista de espera: el aviso de que un turno fue
reprogramado, el pedido de confirmación previo a un reagendado
unilateral, el aviso de cancelación masiva por ausencia de un profesional
y la oferta de un turno liberado a un candidato de la lista de espera
armaban su texto a mano, en inglés, en vez de pasar por el motor de
plantillas de mensajes construido en la fase de notificaciones y
scheduler (sección 3.2.4). Dos de las cinco claves de ese motor —el aviso
de cancelación y la oferta de lista de espera— ya existían con su texto
en español y ya eran configurables por organización, pero sin ningún
punto del código que realmente las usara: sólo los dos trabajos
programados de confirmación y recordatorio pasaban efectivamente por el
motor, un desvío que sólo se nota leyendo en paralelo ambos módulos. El
efecto concreto es que una clínica que personalizara el texto de
cualquiera de estos cuatro mensajes en su configuración no veía ningún
cambio, y el texto en inglés incumplía además la convención del proyecto
de que el contenido cara al paciente debe estar en español. La corrección
agrega al catálogo las dos claves que todavía no existían —el aviso de
reprogramación y su pedido de confirmación previo— y hace que los cuatro
puntos rendericen su texto a través del motor antes de enviarlo, en vez
de construirlo por su cuenta.

Una octava auditoría, dirigida al ángulo de auditoría, fechas y reglas de
negocio tratadas como dato, encontró un hallazgo del mismo tipo que ya
había motivado la ventana mínima de cancelación configurable por
organización: la edad mínima de la modalidad "solo mayores", con la que
un profesional puede restringir la aceptación de pacientes nuevos a
adultos, estaba fijada como una constante de dieciocho años dentro del
código, comparada directamente contra la edad calculada del paciente en
dos puntos —la reserva de un turno y la revalidación de esas mismas
reglas al reprogramar el turno de un paciente nuevo—. A diferencia de un
tope anti-abuso, se trata de una regla de negocio real, la mayoría de
edad vigente, que un tenant white-label o uno radicado en una
jurisdicción con otra mayoría de edad no podía ajustar sin modificar el
código y volver a desplegar. La ventana mínima de cancelación, que vive
en el mismo archivo, ya resolvía exactamente este mismo problema para
otra regla: leer el valor desde la configuración de la organización, con
una constante como resguardo si la fila todavía no existe. La corrección
replica ese mecanismo sin variaciones para la edad mínima —misma clave
de configuración por organización, misma validación de que el valor
configurado sea un entero positivo antes de confiar en él, mismo
resguardo de dieciocho años— y reemplaza ambas comparaciones
hardcodeadas por una consulta a esa configuración. Una migración de
datos siembra el valor vigente para toda organización ya existente, con
el mismo razonamiento que la migración equivalente de la ventana de
cancelación: los datos de siembra del entorno de desarrollo no corren en
un entorno productivo, de modo que sin esa migración la regla sólo
seguiría existiendo como constante de código para cualquier organización
creada antes del cambio.

Una novena auditoría, dirigida esta vez al ángulo de completitud de
módulos respecto del anteproyecto, encontró un hallazgo de una naturaleza
distinta a las ocho anteriores: no un requisito sin implementar, un dato
huérfano, un chequeo de autorización incompleto ni una regla de negocio
hardcodeada, sino un cambio de esquema ya fusionado a la rama principal
sin ningún ticket ni entrada de bitácora que lo respaldara —un hueco de
trazabilidad documental, no funcional—. Una revisión previa del propio
esquema, la que dio origen a las restricciones de integridad reflejadas
más arriba en esta sección, había agregado tres restricciones `CHECK`
—sobre la duración de un turno, el rango de fechas de una ausencia y la
posición en la lista de espera— y había eliminado tres índices de una
sola columna sobre el identificador de organización que resultaban
redundantes frente a un índice compuesto ya existente en cada uno de los
tres modelos afectados. El cambio en sí es correcto y ya estaba verificado
contra la base de datos viva desde su fusión; lo que faltaba era
exclusivamente su documentación. La corrección consistió en escribir esa
documentación en forma retroactiva —tanto la entrada de bitácora
correspondiente como esta misma ampliación del capítulo—, sin modificar,
rehacer ni revertir el cambio de esquema en sí.

La misma novena auditoría examinó además por qué el mecanismo de captura
automática de la tesis, pensado precisamente para que un caso así no
ocurra, no lo había detectado. El hook de cierre de sesión que actúa como
red de seguridad de ese mecanismo compara dos marcas de tiempo: la del
último cambio de código y la de la entrada de bitácora más reciente del
componente, bloqueando el cierre cuando la primera es posterior a la
segunda. Esa comparación detecta que la bitácora quedó desactualizada en
términos de reloj, pero no si la entrada más reciente efectivamente
describe el cambio de código puntual que se acaba de hacer —dos
propiedades que el hook trataba, hasta este hallazgo, como si fueran la
misma—. En el caso del cambio de esquema, esto significó que el hueco dejó
de ser detectable en cuanto se escribió cualquier otra entrada de
bitácora del backend posterior a esa fusión, sin que el hook llegara nunca
a verificar que el cambio de esquema específicamente hubiera quedado
documentado. Corregir esa comparación para que evalúe cobertura en lugar
de sólo recencia es un cambio de diseño no trivial, ajeno al alcance
puramente documental de esta corrección, por lo que quedó registrado como
un ticket de seguimiento propio en lugar de resolverse dentro de esta
misma tarea.

Una décima auditoría, dirigida esta vez al ángulo de reuso,
simplificación y eficiencia, encontró un hallazgo de rendimiento en el
algoritmo de ranking de la lista de espera: por cada candidato de una
lista, el código consultaba por separado su vínculo con el profesional y
aplicaba sobre él, de a uno, la regla que actualiza el tipo de paciente
según su inactividad —pese a que esa regla ya estaba escrita para recibir
muchos vínculos de una sola vez y aplicarse sobre el conjunto completo
con una única escritura. El efecto práctico es que rankear una lista de
N candidatos costaba un número de consultas secuenciales proporcional a
N, y ese ranking se repite en cada paso del recorrido automático de la
lista —la cancelación original de un turno, y de nuevo cada vez que un
candidato rechaza una oferta o esta expira sin respuesta—, de modo que el
costo total de una reasignación con varios candidatos rechazando en
cadena crecía más rápido de lo necesario. La corrección reemplaza la
consulta y la aplicación de la regla por candidato con una carga
batcheada de todos los vínculos de la lista en una sola consulta y una
única aplicación de la regla sobre el conjunto, sin alterar el criterio
de ranking en sí ni el resultado que produce para una lista dada.

La misma décima auditoría encontró, en el mismo módulo, un segundo
hallazgo de eficiencia: la operación que reordena manualmente la lista de
espera de un profesional escribía la nueva posición de cada entrada una
por una, esperando la respuesta de la base de datos antes de emitir la
siguiente escritura, aun cuando esas N escrituras son independientes
entre sí —cada una toca una fila distinta identificada por su propio
identificador— y toda la operación corre dentro de una única transacción
con aislamiento serializable. Con el tope máximo de doscientas entradas
que ya regía este endpoint, un reordenamiento completo podía llegar a
serializar hasta doscientos viajes de red sucesivos en vez de solaparlos,
extendiendo sin necesidad cuánto tiempo permanece abierta esa transacción
—y con ella, el conjunto de bloqueos que retiene— en un endpoint
administrativo de uso frecuente. La corrección lanza las N escrituras en
paralelo dentro de la misma transacción en vez de esperarlas una por una,
sin alterar ni el resultado final del reordenamiento ni el manejo de un
conflicto de serialización, que sigue resolviéndose exactamente igual que
antes.

Una undécima auditoría, esta vez de contraste entre el comportamiento
implementado y el texto del SRS, encontró que la retención de veinticuatro
horas de la modalidad manual de reasignación —descrita más arriba— tenía un
alcance mayor que el que la regla admite. El SRS define esa ventana como una
restricción sobre el bot: ante una cancelación, la franja *queda libre en la
agenda del profesional para que la asigne al paciente que desee*, y lo único
que la ventana impide es que el bot la ofrezca durante ese lapso, dándole
tiempo al profesional de asignarla por su cuenta. La implementación, en
cambio, registraba de la retención únicamente su vencimiento, y el predicado
que decide si un instante está ocupado —compartido, deliberadamente, entre el
listado de disponibilidad y los caminos de escritura, para que ambos no
puedan discrepar sobre qué significa "libre"— trataba cualquier turno
cancelado bajo retención vigente como ocupado sin distinguir quién estaba
preguntando. El resultado invertía el propósito de la regla: la ventana
pensada para reservarle la franja al profesional era exactamente lo que le
impedía usarla, y el rechazo se le presentaba además afirmando que ya tenía
un turno reservado o confirmado en un horario que su agenda mostraba libre.

El hallazgo señala un límite del principio de predicado compartido tal como
estaba aplicado. Compartir la definición de "ocupado" entre lectura y
escritura es correcto y sigue siéndolo —es lo que impide ofrecer un horario
que la reserva luego rechazaría—, pero presupone que "ocupado" es una
propiedad del instante y no de la relación entre el instante y quien
pregunta. Una retención no es una propiedad del instante en ese sentido: es
una restricción dirigida a un actor concreto. El diseño anterior no podía
expresar esa dirección porque la retención se representaba con un único dato,
su vencimiento, que dice cuánto dura pero no a quién alcanza.

La corrección introduce ese segundo dato. Cada retención pasa a registrar
también su motivo —reasignación manual, u oferta automática en curso—, y el
predicado compartido pasa a recibir de quién proviene la consulta: la oferta
del bot, o una asignación directa en la que el profesional o un administrativo
nombran ellos mismos el instante. La oferta del bot sigue viendo ocultas
ambas clases de retención, que es lo que la regla del SRS exige. La
asignación directa queda bloqueada únicamente por una retención de oferta
automática, y la razón por la que esa sí bloquea es simétrica a la anterior:
un recorrido automático en curso tiene un candidato que puede estar aceptando
en ese mismo momento, y una reserva manual por encima dejaría a dos pacientes
creyendo tener el mismo horario. Es decir, la distinción no consiste en que
las retenciones dejen de bloquear escrituras, sino en que cada una bloquee a
quien fue creada para bloquear.

El parámetro que identifica al llamador se declaró obligatorio en lugar de
asumir un valor por omisión. Un valor por defecto habría reintroducido en
silencio el mismo defecto: un camino de código que no declara de qué lado de
la distinción está la obtendría mal sin que nada lo señalara. La consecuencia
práctica es que, cuando la capa conversacional reserve turnos en nombre del
paciente sobre la agenda que el propio bot ofrece, ese camino deberá
declararse como oferta del bot y no como asignación directa, o el chatbot
ocuparía justamente el horario que el SRS reserva para la mano del
profesional.

Los dos datos que describen una retención —vencimiento y motivo— quedan
además atados entre sí por una restricción de integridad en la base de datos,
que obliga a que se escriban y se limpien juntos. La elección no es
cosmética: una fila con vencimiento y sin motivo dejaría de emparejar con el
filtro por motivo, y el turno que el bot no debe ofrecer volvería a ser
ofrecible —es decir, la inconsistencia de datos se manifestaría exactamente
como el defecto que esta corrección elimina. Se prefirió declarar el motivo
como dato propio de la retención antes que derivarlo de la modalidad
configurada en el profesional, porque esa modalidad es su configuración
actual y puede cambiar después de colocada una retención: lo que hace falta
registrar no es cómo está configurado el profesional hoy, sino qué originó la
retención que está en vigor.

La corrección cierra el defecto en el backend, pero su consecuencia alcanza
a una capa que todavía no se construyó, y conviene dejarla asentada aquí
porque se decidió junto con ella. La ruta que publica los horarios libres de
un profesional responde hoy con el acceso del bot para todos los llamadores
por igual, de modo que un horario retenido queda oculto también para la
aplicación del profesional. Eso es correcto para el chatbot y no lo es para
la aplicación, por el mismo motivo que originó esta corrección: la ventana de
veinticuatro horas existe para darle tiempo al profesional de asignar ese
horario a mano, y un selector que no se lo muestra anula la ventana
exactamente como lo hacía el rechazo, una capa más arriba. Como la reserva
directa ya funciona, el horario queda reservable pero no listado —el peor de
los dos estados, porque sólo lo aprovecha quien ya sabe que debe escribir la
hora—. Queda registrado, entonces, como requisito de la fase de la aplicación
móvil y no como alternativa a evaluar: el selector debe ofrecer ese horario,
pidiéndolo con el acceso de asignación directa que el servicio ya sabe
responder. Ese requisito arrastra dos condiciones que no son preferencias.
La primera es que la variante debe negarse al rol bajo el cual corren los
procesos automáticos y correrá el chatbot, porque concedérsela le devolvería
justamente el horario que el SRS reserva para la mano del profesional —y la
ruta está hoy abierta a cualquier usuario autenticado de la organización, de
modo que la restricción hay que agregarla y no se hereda—. La segunda es que
la respuesta debe poder indicar que un horario está retenido, para que la
aplicación lo distinga de una hora libre cualquiera: un profesional que no
puede diferenciarlos podría suponer que el bot se lo quitará, que es
precisamente la incertidumbre que la ventana existe para evitar.

La misma auditoría encontró un defecto de naturaleza distinta a los
anteriores, no en una regla mal aplicada sino en una configuración que nunca
llegaba a aplicarse. La fuente de verdad distingue dos parámetros de agenda
del profesional y los ilustra con dos ejemplos que nombran ambos: atención
cada una hora con sesiones de cuarenta y cinco minutos, o cada treinta
minutos con sesiones de veinte. Cada ejemplo separa cada cuánto se abre un
horario ofrecible —la cadencia— de cuánto dura la sesión que lo ocupa. La
etapa de configuración del profesional ya había separado ambos datos en el
modelo, tras detectar que el diseño inicial los colapsaba en uno solo, y la
había dejado registrada como configuración destinada a la futura generación
de agenda. Esa generación se construyó después, y avanzó siempre con la
duración de la sesión: la cadencia quedó como un dato que el profesional
podía guardar y que no cambiaba nada de lo que veía.

La corrección hace de la cadencia el paso entre inicios de turno
consecutivos, dejando la duración de la consulta como longitud de la sesión y
como valor informado en cada franja —es el dato que el asistente comunica al
paciente al confirmar, de modo que la cadencia no debía filtrarse a la
respuesta—. La condición de corte de cada bloque de atención sigue exigiendo
que entre la sesión completa y no la cadencia, de manera que un bloque de
09:00 a 17:00 con cadencia de sesenta minutos y sesiones de cuarenta y cinco
ofrece hasta las 16:00 y no un inicio a las 17:00. Un profesional sin cadencia
configurada conserva exactamente la agenda anterior, porque el paso vuelve a
caer en la duración.

Lo que hizo que el defecto sobreviviera no fue una fórmula equivocada sino
una duplicación. Existían dos recorridos que producían instantes de turno
—el de la agenda ordinaria y el de la grilla sobre la que se apoya la regla
de doble franja para paciente nuevo—, cada uno con su propia copia del paso,
de modo que la cadencia tenía que ser incorporada dos veces y no lo fue en
ninguna. La corrección los unificó en un único recorrido que devuelve la
grilla completa de instantes por día: la agenda ordinaria es esa grilla menos
lo que ya está ocupado, y la regla de doble franja necesita la grilla sin esa
resta, porque debe distinguir "el primer turno del día está tomado" —caso en
que no ofrece nada— de "ese turno no existe en la grilla", distinción que la
lista de franjas libres no puede hacer, ya que ambos casos se le presentan
igual, como ausencia. La lectura e interpretación del par de configuraciones
se concentró además en un módulo compartido que consumen tanto el servicio de
disponibilidad como el de turnos, de modo que los instantes que la agenda
ofrece y los que una reserva ocupa provienen de una única definición.

Esa segunda propiedad no es un refinamiento sino una condición de corrección,
y lo ilustra el punto donde el defecto tenía su consecuencia más seria. La
reserva de una primera sesión validaba el instante de inicio contra la grilla
de paciente nuevo y luego derivaba el segundo turno del par sumando la
duración de la sesión. Con cadencia de sesenta y sesiones de cuarenta y
cinco, eso habría escrito el segundo turno en un instante que la agenda no
ofrece y que ninguna consulta de disponibilidad vuelve a mostrar, solapado
además con el horario siguiente, que sí continúa ofreciéndose porque la
verificación de ocupación compara instantes exactos y no intervalos. Es
decir, hacer efectiva la cadencia sin corregir también ese punto habría
abierto un camino de doble reserva que antes no existía —no porque el cálculo
fuera correcto, sino porque la cadencia no llegaba a producir la
discrepancia—. Por el mismo motivo el emparejamiento de la regla de doble
franja pasó a exigir que los dos turnos disten una cadencia y no una
duración: la fuente de verdad pide "dos turnos consecutivos de su agenda", y
en una agenda con cadencia el turno siguiente empieza una cadencia después.

Hacer efectiva la configuración obligó por último a acotarla. Mientras la
cadencia no se usaba, el par no podía contradecirse en ningún efecto
observable; al usarse, una cadencia menor que la duración de la sesión abre
el turno siguiente mientras la sesión anterior todavía transcurre, y la
agenda pasa a ofrecer dos horarios solapados que —de nuevo por comparar
instantes exactos— son ambos reservables. Se agregó entonces la invariante de
que la cadencia no sea menor que la duración, con valores iguales admitidos,
que son la agenda consecutiva de siempre. La comprobación se ubicó en el
servicio y no en la validación del cuerpo del pedido porque abarca el par tal
como quedará almacenado: la actualización de configuración es parcial y puede
traer una sola de las dos columnas, de modo que una validación que sólo
mirara el pedido se satisfaría enviando las dos mitades por separado. Eso la
convierte en una invariante de lectura y escritura, y se ejecuta dentro de la
transacción serializable que ese punto de acceso ya abría por otra
comprobación, para que dos actualizaciones concurrentes —una fijando la
duración, otra la cadencia— no puedan leer cada una un estado que permite su
propia escritura y dejar entre ambas un par que ninguna habría aceptado. Se
descartó expresarla como restricción de integridad en la base de datos:
ambas columnas son anulables hasta que la configuración las fija, la mitad no
configurada no puede contradecir a la otra, y una comprobación de tabla
debería tolerar los estados intermedios sin poder distinguir después cuál de
las dos mitades el operador acaba de mover.

Una última revisión del motor de turnos expuso que la regla de feriados, que
el servicio de disponibilidad aplica tanto al construir la agenda ofrecible
como al verificar un instante puntual, no alcanzaba al motor de reasignación
de la lista de espera. La causa es la misma decisión de arquitectura que se
describió al presentar ese motor: para evitar un ciclo de importación con el
módulo de turnos, el motor escribe directamente sobre la tabla de turnos, y
por esa vía quedaba fuera de la única verificación que aplica la regla. El
escenario que ello habilita se apoya en un segundo hecho del sistema, a
saber, que dar de alta un feriado no cancela los turnos ya reservados en esa
fecha: un turno agendado con anticipación puede quedar sobre una fecha
declarada feriado después y, si el paciente original lo cancela, la
reasignación automática podía asignárselo a un integrante de la lista de
espera sobre un día en que la clínica no atiende.

La corrección introduce la verificación en dos momentos distintos, con
respuestas deliberadamente distintas. En el paso de ofrecimiento la
condición se comprueba una vez por recorrido, antes de contactar a nadie:
que la fecha sea feriado es una propiedad de la franja liberada y no del
candidato, idéntica para toda la lista, de modo que comprobarla por candidato
sólo multiplicaría las consultas y llevaría a ofrecer sucesivamente a cada
integrante un turno que la clínica no podría honrar, reteniendo entretanto
una franja inutilizable durante toda la ventana de respuesta de cada uno.
Verificada la condición, si la fecha es feriado no se registra oferta alguna
y la retención sobre la franja se libera de inmediato. En el paso de reserva
la condición se comprueba nuevamente, esta vez dentro de la misma transacción
que crea el turno, porque el alta del feriado puede ocurrir mientras una
oferta ya emitida espera respuesta; allí la operación falla de forma
explícita y libera la retención, en lugar de reservar en silencio.

Dos aspectos de esa corrección merecen constancia por no ser evidentes. El
primero es que no se reutilizó la verificación de franja libre, que es la que
aplican las demás rutas de escritura y la que la formulación literal del
requerimiento sugería emplear. Esa verificación considera ocupada una franja
tanto cuando existe un turno vivo sobre ella como cuando existe un turno
cancelado bajo una retención de reasignación vigente; ahora bien, el turno
que el motor está reasignando es exactamente eso, una fila cancelada que el
propio motor retuvo al iniciar el recorrido, de modo que la comprobación lo
haría colisionar consigo mismo y devolvería "ocupada" de manera determinista.
Emplearla habría inutilizado por completo la reasignación automática en lugar
de corregir el defecto. Se extrajo entonces la mitad pertinente —la consulta
al calendario de feriados— a un método propio del servicio de
disponibilidad, que la verificación de franja libre pasa a consumir
internamente; se descartó replicar la consulta en el módulo de lista de
espera, pese a estar admitido, porque una tercera definición de la misma
regla es precisamente la divergencia que el criterio de definición única
busca evitar.

El segundo aspecto concierne al tratamiento de la falla en el paso de
reserva. Consignar la oferta como rechazada y avanzar al siguiente candidato
resultaría cómodo, pero introduciría en el registro de auditoría un rechazo
que el paciente nunca expresó, cuando en rigor aceptó y fue la franja la que
dejó de existir; dado que la trazabilidad exigida por la Ley 25.326 se apoya
en que ese registro refleje lo ocurrido, la oferta permanece aceptada y la
falla se propaga como excepción. Por análoga razón, la liberación de la
retención se acotó mediante un tipo de error específico a la causa de
feriado: un cambio de estado concurrente indica que la franja pertenece a
quien la tomó, y liberarla allí la expondría además a la agenda del
asistente, mientras que un fallo de escritura del registro de auditoría no
debe quedar enmascarado.

Cabe señalar que esta corrección impide que la reasignación agregue turnos
nuevos sobre un feriado, pero no altera la situación de los turnos ya
agendados sobre una fecha declarada feriado con posterioridad, que
permanecen en la agenda. Esa carencia del alta de feriados queda registrada
como observación pendiente.

La misma revisión expuso una segunda omisión en el motor de reasignación,
esta vez en el orden mismo en que recorre la lista de espera. La
especificación del algoritmo fijó dos criterios de prioridad: la que el
profesional carga a mano sobre su relación con un paciente determinado, y el
tipo de vínculo, que distingue al paciente recurrente del nuevo y que el
sistema deriva por sí mismo de las consultas registradas. La implementación
cargaba ambos datos —la estructura que representa a un candidato del ranking
declara los dos campos y la consulta los resuelve juntos—, pero el comparador
que ordena a los candidatos leía únicamente la prioridad explícita y, tras
ella, el orden de llegada a la lista. El tipo se escribía y no se volvía a
leer en ninguna parte. Como la prioridad explícita es opcional y se asigna a
mano, el caso ordinario es que ningún candidato la tenga, y en ese caso la
lista quedaba ordenada sólo por orden de llegada: un paciente recurrente de
años no obtenía ventaja alguna sobre uno nuevo que se hubiera anotado antes.
El criterio quedaba incumplido sin que nada fallara.

La corrección incorpora el tipo como escalón intermedio del comparador, entre
la prioridad explícita y el orden de llegada. Esa ubicación no es arbitraria:
la prioridad que el profesional asigna es una decisión deliberada sobre un
paciente concreto, mientras que el tipo es una condición general que el
sistema infiere, de modo que anteponer el tipo subordinaría lo específico a lo
automático y dejaría a un paciente nuevo marcado como urgente detrás de
cualquier recurrente sin prioridad. El orden resultante queda entonces en
cuatro escalones —prioridad, tipo, orden de llegada y fecha de creación del
registro—, cada uno desempatando al anterior, y se expresó como una función
pura con nombre, extraída del ordenamiento, junto a la que ya traducía la
prioridad a un peso comparable.

Un aspecto de esta corrección merece constancia por no ser evidente en el
punto donde se compara: el tipo que el comparador lee no es el almacenado en
la relación sino el vigente. Los vínculos de los candidatos se cargan por la
vía que aplica antes la regla de inactividad descripta en el módulo de
Pacientes, de manera que un vínculo registrado como recurrente cuya última
consulta es anterior al umbral del inquilino ya fue degradado a nuevo cuando
llega al ranking. Sin ese orden, la corrección habría otorgado ventaja en la
lista de espera a pacientes que el resto del sistema ya considera nuevos, es
decir, habría hecho que el motor de reasignación contradijera a las demás
lecturas del mismo dato.

Corresponde señalar, por último, cómo se resolvió una ambigüedad de la
formulación original del requerimiento, que enunciaba la regla como que los
pacientes recurrentes tienen prioridad sobre los nuevos *si el profesional así
lo configuró*. El sistema no contaba con ningún parámetro que activara o
desactivara esa preferencia, de modo que la condición no correspondía a nada
existente y admitía dos lecturas: agregar la configuración faltante, o
entender la regla como incondicional. Consultada la dueña del producto, se
confirmó lo segundo. Se descartó por lo tanto agregar la columna, en una
decisión que conviene distinguir del criterio general adoptado en este
trabajo, según el cual las reglas de negocio se expresan como datos
configurables antes que como condicionales fijos en el código: ese criterio
rige sobre aquello que la clínica efectivamente decide, y una opción que nadie
va a modificar no es configuración sino un parámetro muerto —una columna, un
campo del punto de acceso de configuración, un campo de la respuesta y las
pruebas de todos ellos— sosteniendo una decisión ya tomada.

Una revisión posterior de las cuatro rutas de administración del calendario
de feriados expuso que la restricción al rol administrador, razonable para
las tres escrituras, alcanzaba también a la lectura sin ninguna excepción, y
que ninguna otra ruta del sistema exponía el calendario a un profesional: ni
la disponibilidad ni el listado general de turnos devuelven una marca de "día
feriado". El documento de requisitos prevé que la futura vista de agenda de
la aplicación móvil del profesional —todavía no implementada— marque los
feriados con color, precondición que sin una ruta de lectura accesible no
podía cumplirse. La corrección amplía el rol de la lectura mediante un
decorador de roles declarado sobre el propio método, que se resuelve antes
que el del controlador; las tres escrituras conservan el rol administrador
exclusivo. Al tratarse de una anulación puramente de autorización, no hizo
falta tocar el servicio: el listado ya resolvía sus resultados a través del
cliente de Prisma acotado por inquilino, de modo que un profesional, igual
que un administrador, sólo alcanza a ver los feriados de su propia
organización. La alternativa de incorporar el feriado a la respuesta de
disponibilidad o de turnos, que el propio requerimiento contemplaba, se
descartó por acoplar una preocupación administrativa a dos contratos ya
probados con otro propósito; también se descartó exponer el listado bajo una
ruta nueva sin el prefijo administrativo, por no haber en el sistema
precedente de un mismo recurso bajo dos rutas según el rol que lo consulta,
cuando el patrón ya establecido para un recurso que ambos roles comparten es
una única ruta con el rol declarado por operación.

### 3.2.4 Notificaciones y Scheduler

El módulo de Notificaciones y recordatorios se abrió con el motor de
plantillas de mensajes, el componente sobre el que se apoyan tanto los
trabajos programados de este mismo módulo —confirmación de turno,
recordatorio, aviso de cancelación— como el chatbot de la capa
conversacional en una fase posterior, sin que ninguno de los dos exista
todavía: la tarea se acotó deliberadamente a producir el texto final de
un mensaje a partir de una clave y un conjunto de parámetros, sin enviar
nada, dejando el envío efectivo para cuando el puerto de mensajería se
integre con un proveedor real.

El documento de requisitos nombra las cinco plantillas base y su clave en
español —confirmación de turno, recordatorio de turno, aviso de
cancelación, aviso de reasignación y solicitud de consentimiento—, pero
esa nomenclatura se tradujo a identificadores en inglés siguiendo la
misma convención de renombrado ya aplicada en tareas anteriores del
Motor de Turnos: el texto en español del ticket describe la
funcionalidad que el requisito pide, no un contrato literal sobre el
nombre del símbolo. La clave de reasignación se nombró además reutilizando
el vocabulario ya existente para ese mismo evento en el puerto de
respuesta de lista de espera, en lugar de introducir un segundo nombre
para el mismo concepto. El texto de cada plantilla, en cambio, se dejó en
español sin traducir: a diferencia de un identificador de código, es
contenido que el paciente lee directamente por WhatsApp, y por lo tanto
sigue el idioma del dominio y no el de la implementación.

El almacenamiento de las plantillas siguió el mismo mecanismo de
configuración por inquilino ya construido para las reglas de negocio del
Motor de Turnos y de Pacientes: cada plantilla vive bajo una clave propia
en la configuración de la organización, y el servicio recurre al texto
base del sistema únicamente cuando el inquilino no definió el suyo. Esa
misma configuración es la que permite, sin cambios de código, que cada
organización redacte sus mensajes con un tono propio —el requisito de
marca blanca que el documento de requisitos plantea para el sistema en su
conjunto—. Siguiendo el criterio ya fijado para toda regla de alcance
tenant —que la fila por defecto debe existir desde el principio en la
configuración de cada organización y no aparecer recién la primera vez
que alguien la personaliza—, las cinco plantillas base se sembraron
también mediante una migración de datos, además de en el script de
siembra de desarrollo, con la misma condición de no sobrescribir nunca un
texto que la clínica ya hubiera modificado.

La validación del motor de renderizado se ajustó estrictamente a los
criterios de aceptación del ticket: una clave de plantilla inexistente y
un parámetro faltante son, cada uno, un error descriptivo y no un
resultado parcial. En particular, se descartó devolver el texto con el
marcador de posición sin reemplazar ante un parámetro faltante —un
mensaje que llegara al paciente con una llave literal en su texto sería
peor que no enviar ningún mensaje—, de modo que la ausencia de un
parámetro requerido interrumpe el renderizado antes de producir ningún
texto. La extracción de los nombres de parámetro que exige una plantilla
se hizo a partir del propio texto de la plantilla, y no de una lista
declarada aparte para cada clave, para que la plantilla personalizada de
un inquilino —con sus propios parámetros, potencialmente distintos de los
de la plantilla base— no pudiera quedar en desacuerdo con lo que el
servicio efectivamente exige.

Con el motor de plantillas disponible, la segunda tarea del módulo puso
en marcha el primer trabajo programado que efectivamente lo consume: un
job que corre cada hora y detecta los turnos reservados que entran en la
ventana de 24 horas antes de la cita, tal como lo pide el documento de
requisitos. La ventana de detección no se definió como un instante
puntual sino como un rango —entre 23 y 25 horas de anticipación—, ancho
suficiente para que un turno no pudiera atravesar sin ser detectado la
brecha entre dos corridas horarias consecutivas del job. Para cada turno
que entra en esa ventana, el job renderiza la plantilla de confirmación
con los datos del paciente, del profesional y de la cita, y entrega el
texto resultante al puerto de mensajería —hoy el adaptador de prueba, ya
construido en la fase de Fundaciones, hasta que la integración real con
WhatsApp se incorpore en una fase posterior— y dejó registrado el intento
en la auditoría del sistema, con el mismo mecanismo ya usado en el resto
del proyecto para dejar constancia de quién hizo qué y sobre qué recurso.

La condición de no reenvío que exige el documento de requisitos —que un
turno ya procesado no vuelva a recibir el mensaje en una corrida
posterior— se resolvió con una columna nueva en el turno, dedicada
exclusivamente a registrar el momento en que se envió la solicitud de
confirmación, en lugar de reutilizar la columna que ya existía para
registrar la respuesta del paciente. Esa columna previa, incorporada
durante el desarrollo del Motor de Turnos, tiene un significado distinto
y ya construido —se completa únicamente cuando el paciente responde que
confirma su asistencia—, de modo que superponerle también el significado
de "ya se envió la pregunta" habría hecho ambigua la columna y, además,
no habría resuelto el problema real: con una ventana de detección de dos
horas y un job que corre cada hora, un turno sin respuesta habría vuelto
a aparecer como candidato en la corrida siguiente y recibido el mensaje
por segunda vez. El texto del documento de requisitos, que describe la
condición de no reenvío en términos de esa columna anterior, se
interpretó como la descripción funcional del requisito de idempotencia y
no como un contrato literal sobre qué campo debía usarse —el mismo
criterio ya aplicado al traducir los nombres de las claves de plantilla a
identificadores en inglés, más arriba en esta misma sección.

Un turno cuyo paciente no tiene un número de celular registrado se deja
sin marcar en lugar de tratarse como procesado, siguiendo el mismo
criterio de tolerancia ya adoptado en el motor de reasignación de lista
de espera: el sistema simplemente no puede contactar a ese paciente, y
dejarlo sin marcar permite que una corrida posterior lo intente de nuevo
si el dato llegara a completarse mientras el turno sigue dentro de la
ventana. La respuesta del paciente al mensaje de confirmación —si acepta
o no— quedó deliberadamente fuera del alcance de esta tarea: su
procesamiento corresponde a la capa conversacional que se incorporará en
una fase posterior, que es quien efectivamente invoca la operación de
confirmar o cancelar el turno ya existente en el Motor de Turnos.

Con la solicitud de confirmación ya en marcha, la tercera tarea del módulo
cerró el otro extremo de ese mismo flujo: un segundo trabajo programado
que, cada 15 minutos, cancela automáticamente todo turno reservado cuya
solicitud de confirmación lleva 4 horas o más sin respuesta, tal como lo
exige el documento de requisitos, y dispara el mismo mecanismo de
reasignación que ya usa una cancelación pedida por una persona, para que
el turno liberado pueda ofrecerse a la lista de espera del profesional.
El documento de requisitos original pedía además "contemplar que el
mensaje puede haberse enviado en fin de semana"; esa condición se
interpretó, siguiendo una aclaración explícita del propio ticket, no como
una extensión del plazo de 4 horas sino como una exigencia sobre el propio
trabajo programado: que siga corriendo también los fines de semana para
detectar a tiempo los casos que cruzan ese límite, sin ninguna rama de
código adicional más allá de la comparación de marcas de tiempo que el
job ya hace. El motivo de la cancelación se registró extendiendo el mismo
enumerado que ya distinguía la cancelación masiva por ausencia del
profesional de una cancelación ordinaria, en lugar de introducir un
mecanismo paralelo, con un valor nuevo agregado mediante una migración
escrita a mano —Prisma no soporta, en la versión usada por este proyecto,
declarar el agregado de un valor a un enumerado de PostgreSQL de forma
automática—, el mismo procedimiento ya empleado para incorporar el rol
reservado a procesos automatizados en una fase anterior. La escritura de
la transición de estado no pasa por la ventana de aviso mínimo que sí
rige una cancelación pedida por una persona, porque esa ventana existe
para desalentar una cancelación de último momento sin avisar y no tiene
sentido aplicada a una cancelación que el propio sistema dispara al no
recibir respuesta; y queda protegida contra condiciones de carrera con la
misma técnica que el resto del proyecto —la actualización solo afecta al
turno si su estado sigue siendo el esperado en el momento de escribir—,
de modo que un turno que el paciente confirma o cancela mientras el job
está corriendo simplemente deja de ser candidato en lugar de competir con
esa decisión.

La cuarta tarea del módulo cerró el tercer trabajo programado del flujo de
notificaciones, el recordatorio de turno, y dejó además el andamiaje del
futuro trabajo de expiración de códigos de acceso de la cerradura
inteligente, que se completa recién en una fase posterior. A diferencia
del job de confirmación, cuya ventana de 24 horas es una regla fija del
documento de requisitos, el recordatorio se pidió explícitamente
configurable por inquilino —"por defecto X horas antes del turno,
configurable por tenant"—, de modo que la cantidad de horas de
anticipación se incorporó como una fila más de la configuración de
organización, con el mismo mecanismo de valor por defecto y siembra por
migración ya usado para las demás reglas de negocio de alcance
organizacional del proyecto. El valor por defecto elegido, 24 horas,
coincide con la ventana que ya usa el job de confirmación y con el propio
texto de la plantilla base del recordatorio, que anuncia el turno para
"mañana". El job solo selecciona turnos en estado confirmado, no
reservado —el recordatorio es una cortesía para un turno que el paciente
ya confirmó, un paso distinto de la solicitud de confirmación misma—, y
reutiliza la misma técnica de banda de detección de dos horas alrededor
del valor configurado que ya usa el job de confirmación, para que un
turno no pueda atravesar sin ser detectado la brecha entre dos corridas
horarias consecutivas. La condición de no reenvío se resolvió de la misma
manera que en el job de confirmación: una columna nueva y dedicada en el
turno, distinta de la que registra la respuesta del paciente, que solo
registra que el recordatorio ya se envió.

El segundo trabajo programado que exige el ticket, la expiración de
códigos de acceso temporales de la cerradura TTLock, no pudo
implementarse todavía en su forma real porque la entidad que representa
esos códigos no existe aún en el modelo de datos: su incorporación está
planificada para una fase posterior, cuando se integre el proveedor de
cerradura inteligente. El ticket pide explícitamente, para esta tarea,
solo el andamiaje: un trabajo programado real, con su firma definitiva,
que se ejecute sin errores y no produzca ningún efecto todavía. Se
construyó como un módulo nuevo y separado, pensado como el lugar donde
esa futura entidad y su lógica de expiración van a vivir, con un
comentario en el código que documenta la consulta que el trabajo va a
resolver una vez que la entidad exista. Se programó con una frecuencia de
ejecución distinta a la del recordatorio —cada quince minutos en lugar de
cada hora—, tanto para dejar documentada la independencia entre ambos
trabajos que pide el ticket como porque un código de acceso vigente más
allá de lo debido es una exposición de acceso físico, de una naturaleza
distinta a una demora en el envío de un mensaje.

Ambos trabajos se cubrieron con pruebas unitarias, sin agregar pruebas de
extremo a extremo nuevas: la lógica de selección, ventana e idempotencia
del recordatorio es enteramente reproducible con un cliente de Prisma
simulado y un reloj controlado, siguiendo el mismo criterio ya aplicado a
los otros trabajos programados del módulo, y el trabajo de expiración,
al no tener todavía ninguna lógica real que ejercitar, se probó
únicamente en que se ejecuta sin lanzar errores y no produce ningún
efecto observable más allá del mensaje de registro que declara su propio
estado de placeholder.

La última pieza de este módulo cerró una limitación que había quedado
declarada desde que se construyó el algoritmo de reasignación automática
en el módulo de Turnos (sección 3.2.3): la pregunta "¿el candidato de la
lista de espera aceptó el turno liberado que se le ofreció?" se resolvía
detrás de un puerto —`WaitlistResponsePort`— cuyo único adaptador
existente respondía "no" en el mismo instante en que se lo llamaba,
porque todavía no existían ni el canal real de conversación por WhatsApp
ni una ventana de tiempo real dentro de la cual esperar una respuesta. El
propio código dejaba constancia de esa limitación como una decisión
deliberada, a la espera de que ambas piezas llegaran.

La ventana de tiempo llegó con esta tarea, aunque el canal de WhatsApp
siga sin existir. El requisito de negocio es el mismo que ya gobierna la
autocancelación por falta de confirmación (sección 3.2.3): un candidato
tiene cuatro horas para responder a la oferta de un turno liberado antes
de que el sistema la dé por vencida y continúe con el siguiente candidato
de la lista, en el mismo orden de prioridad ya establecido. Sostener esa
espera obligó a reemplazar el mecanismo por el que el algoritmo de
reasignación recorría la lista: donde antes recorría a todos los
candidatos uno tras otro dentro de una única operación —viable solo
porque la respuesta llegaba al instante—, ahora ofrece el turno a un
único candidato, dentro de una tabla nueva que registra esa oferta y su
estado, y se detiene; avanzar al candidato siguiente, o reservar el turno
para quien acepta, quedó como una operación separada que cualquier
disparador externo puede invocar en el momento en que corresponda. Hoy el
único disparador es un trabajo programado nuevo, que corre cada quince
minutos con la misma infraestructura por inquilino que los demás trabajos
de este módulo y vence las ofertas que llevan más de cuatro horas sin
respuesta; el día que el canal real de WhatsApp exista, la respuesta
efectiva del paciente va a invocar la misma operación de aceptación por
el mismo camino, sin que el algoritmo de recorrido en sí —ya construido en
el módulo de Turnos— necesite cambiar de nuevo.

Esta tarea también resolvió, aunque de forma incidental, un caso que el
mecanismo anterior dejaba sin cubrir con precisión: un candidato sin
número de celular registrado no tiene ninguna vía real por la que
recibir ni responder una oferta, así que se sigue tratando, como ya
ocurría antes de esta tarea, como un rechazo inmediato en lugar de
ocupar cuatro horas de la ventana de reasignación sin ninguna
posibilidad real de respuesta.

El módulo cierra, por ahora, con un canal de notificaciones distinto de los
anteriores: no un mensaje saliente hacia el paciente por WhatsApp, sino un
aviso dentro de la propia aplicación, dirigido al profesional. Dos eventos
que el sistema ya sabía reconocer —la cancelación de uno de sus turnos, ya
sea individual o por la cancelación masiva que dispara su propia ausencia,
y la reasignación automática de un turno liberado a un candidato de su
lista de espera— pasaron a dejar, además de todo lo que ya registraban,
una fila en una tabla nueva que el profesional consulta desde su propio
listado, filtrable por leído o no leído y paginado, con operaciones para
marcar una notificación puntual o todas las pendientes como leídas. Un
administrador puede consultar el listado de cualquier profesional del
mismo inquilino, pero un profesional que intenta marcar como leída una
notificación ajena recibe un 403 explícito —una excepción puntual a la
convención general del proyecto de responder 404 ante cualquier intento
fuera del propio alcance, seguida aquí porque el propio documento de
requisitos exige ese código de estado como criterio de aceptación
verificable—.

El documento de requisitos nombra dos orígenes más de notificación —una
solicitud de receta nueva del paciente, y una alerta ante un error de la
cerradura inteligente— que quedaron representados en el tipo de evento sin
ningún disparador real todavía: ninguno de los dos servicios que deberían
dispararlos existe en el código a esta altura del proyecto, uno porque
nunca se construyó pese a que su entidad de datos ya existe desde una fase
muy anterior, y el otro porque su propia entidad está planificada recién
para la fase de integración con la cerradura TTLock. La tabla que sostiene
este canal de notificaciones no lleva columna de organización propia,
siguiendo la misma regla general del esquema ya aplicada a la matrícula o
al horario de atención de un profesional: al tener un único padre acotado
por inquilino —el profesional al que pertenece—, repetir esa columna
introduciría un valor que podría llegar a discrepar del que ya tiene ese
padre, así que cada operación se ancla primero en la comprobación de que
el profesional en cuestión pertenece al inquilino de quien pregunta.

Una auditoría de la base de datos viva, posterior a la incorporación de la
tabla de ofertas de lista de espera descripta más arriba en esta sección,
detectó que dos de sus tres claves foráneas compuestas acotadas por
inquilino tenían el índice de soporte correspondiente, pero la tercera —la
que apunta, de forma opcional, hacia la entrada de lista de espera de
origen— no. PostgreSQL no indexa automáticamente las columnas de una clave
foránea, así que cada operación de escritura sobre una entrada de lista de
espera —en particular, la que el propio trabajo programado de vencimiento
de ofertas dispara al aceptar o expirar una— forzaba un recorrido
secuencial completo de la tabla de ofertas para resolver esa restricción.
La corrección agregó el índice faltante, con la misma forma que sus dos
pares ya existentes, sin ningún cambio de lógica.

Una segunda auditoría, sobre la propia tabla de notificaciones descripta
más arriba en esta sección, encontró un problema análogo al de la
sección de Motor de Turnos: las dos columnas opcionales que enlazan una
notificación con el turno o la solicitud de receta que la originaron
eran claves foráneas simples, sin ningún componente que las atara al
profesional dueño de la notificación. Nada impedía, a nivel de base de
datos, que una notificación de un profesional terminara apuntando a un
turno o una solicitud de otro profesional —incluso de otra
organización—, y ninguna de las dos columnas tenía, además, el índice de
soporte que sí tienen el resto de las claves foráneas opcionales del
esquema. La corrección convirtió ambas en claves foráneas compuestas
contra el mismo profesional que ya identifica a la notificación, y
agregó los dos índices faltantes. A diferencia de los casos anteriores de
este mismo patrón, aquí no había una columna de organización disponible
para componer la clave —la tabla de notificaciones, como ya se explicó,
deliberadamente no lleva una propia—, así que la clave compuesta se
armó, en cambio, contra el profesional: exactamente el dato que el padre
de la notificación ya determina. El borrado de un turno o una solicitud
referenciados ahora arrastra en cascada a la notificación que los
mencionaba, en lugar de dejarla con una referencia rota, una decisión
tomada después de confirmar contra la batería de pruebas existente que
ningún flujo del sistema depende de que esa referencia sobreviva al dato
que describe.

Una tercera auditoría, esta vez centrada en el punto donde se resuelve el
destino de una oferta de turno de lista de espera —aceptada, rechazada o
vencida por el propio trabajo programado de vencimiento descripto más
arriba—, encontró que el cambio de estado de la oferta se escribía sobre
el cliente de base de datos directo, sin transacción, y que la entrada de
auditoría correspondiente se generaba varias líneas después, sin
compartir ninguna garantía de todo-o-nada con esa escritura. Una
auditoría que fallara justo después de una confirmación ya aplicada
dejaba una oferta resuelta sin ningún rastro en la traza de
cumplimiento, el mismo riesgo que ya se había resuelto, en una fase
anterior de esta misma sección, para el trabajo programado de
autocancelación por falta de confirmación. La corrección aplicó
exactamente ese patrón ya existente: la escritura del estado y su
entrada de auditoría pasaron a compartir una misma transacción, mientras
que la lectura posterior del turno asociado —que solo alimenta la
continuación del recorrido de la lista de espera, no la mutación que se
audita— quedó deliberadamente fuera de ella.

Una cuarta auditoría, dirigida específicamente a los dos trabajos
programados que envían mensajes por WhatsApp con base en una ventana de
detección temporal —el de solicitud de confirmación y el de recordatorio,
ambos descriptos más arriba en esta sección—, encontró que los dos
compartían la misma falla de orden de operaciones: el mensaje se enviaba
antes de aplicar la comprobación de idempotencia que registra el intento, y
ninguna de las dos comprobaciones volvía a verificar el estado del turno en
el momento mismo del envío, solo la consulta inicial que había armado el
lote de candidatos algunos segundos antes. Un turno que el paciente
confirmaba, o que se cancelaba, en el intervalo entre esa consulta inicial y
el procesamiento de su turno dentro del lote, igual recibía el mensaje —una
comunicación contradictoria para quien la lee, generada por una ventana de
carrera que la comprobación de idempotencia, tal como estaba escrita, no
alcanzaba a cerrar. La corrección invirtió el orden en ambos trabajos: la
escritura guardada que marca el intento —ahora también condicionada al
estado vigente del turno, no solo a que el campo de idempotencia siguiera
vacío— se ejecuta primero, junto con su entrada de auditoría dentro de la
misma transacción, y el mensaje solo se renderiza y se envía si esa
escritura efectivamente afectó una fila. La contrapartida de este orden,
dejada asentada junto al código, es que un envío que fallara después de ese
punto ya quedaría registrado como intentado y no se reintentaría en la
corrida siguiente —un costo menor y necesario para cerrar la ventana de
carrera que motivó la corrección, frente al riesgo de volver a mensajear un
turno que ya había cambiado de estado.

Una quinta auditoría, esta vez orientada a reuso y simplificación en lugar
de a un defecto de comportamiento, observó que los cinco trabajos
programados de esta sección —el de solicitud de confirmación, el de
autocancelación por falta de respuesta, el de recordatorio, el de
vencimiento de ofertas de lista de espera y el de autocompletado semanal de
turnos vencidos, descripto en la sección del motor de turnos— repetían,
cada uno de forma independiente, el mismo bloque inicial: consultar todas
las organizaciones, recorrerlas abriendo el contexto de inquilino de cada
una, resolver el usuario SYSTEM correspondiente y omitir con una advertencia
la organización que no tuviera ninguno. Cada archivo llevaba, además, una
copia del mismo comentario que explica por qué el callback pasado al
contexto de inquilino debe ser asincrónico y esperar su propio trabajo, bajo
riesgo de perder ese contexto apenas el bucle que lo invoca retorna de
forma síncrona. La duplicación no era solo repetición de código: cualquier
mejora futura a esa forma común —agrupar en una sola consulta la
resolución del usuario SYSTEM, sumar métricas por organización— habría
exigido editar los cinco archivos de manera idéntica, con el riesgo
adicional de que un sexto trabajo programado copiara uno de los cinco en
lugar de reutilizar un mecanismo compartido. La corrección extrajo ese
bloque a una única función, ubicada junto al resto de las utilidades
transversales del backend con la misma forma que ya tenía la que envuelve
una transacción serializable —una función asincrónica que recibe sus
dependencias como parámetros explícitos, no un servicio resuelto por
inyección—, y migró los cinco trabajos programados a usarla, sin modificar
la lógica de negocio particular de ninguno ni el comportamiento observable
del conjunto: la corrida sigue siendo independiente por organización, y una
organización sin usuario SYSTEM se sigue omitiendo con una advertencia en
lugar de abortar el resto del proceso.

Una sexta auditoría, también orientada a reuso, señaló que el trabajo de
solicitud de confirmación y el de recordatorio declaraban, cada uno por su
cuenta, la misma función auxiliar de formateo de hora en reloj de pared que
antepone un cero a un número de un solo dígito. Al revisar el estado real
del código sobre una copia recién actualizada del repositorio —la misma
precaución ya aplicada frente a un hallazgo de auditoría similar en el
Motor de Turnos— esa duplicación resultó ya no existir: una corrección
anterior, hecha al mover los mensajes de reagendado, cancelación por
ausencia y oferta de lista de espera a pasar por el motor de plantillas de
notificaciones, ya había extraído ambas funciones de formateo —y su único
auxiliar de relleno de cero— a ese archivo compartido de formateo de
notificaciones, como efecto colateral de un trabajo con un objetivo
distinto. La auditoría que originó esta observación se había ejecutado, con
toda probabilidad, sobre una copia del repositorio anterior a esa
corrección. No hubo entonces código que modificar: se documentó la
verificación y se cerró la observación dejando constancia de su causa más
probable, para que una auditoría posterior no vuelva a levantar la misma
alarma sin antes refrescar su copia del repositorio.

Una séptima auditoría, esta vez de cotejo entre el código y el documento de
requisitos, señaló una omisión en el contenido del propio mensaje: el
documento pide que la solicitud de confirmación informe al paciente cuánto
dura la sesión, y ni la plantilla base declaraba un marcador de posición
para la duración ni el trabajo programado que la renderiza seleccionaba esa
columna del turno, pese a estar disponible en la misma consulta que ya
recuperaba los datos del paciente y del profesional. La corrección agregó
el marcador al texto base y el dato a la consulta, con dos decisiones de
redacción del mensaje que conviene dejar asentadas. La primera es que el
marcador transporta únicamente el número y la palabra "minutos" quedó en el
texto de la plantilla, la misma separación que ya seguían el marcador de
fecha y hora y el de hora sola, cuyos formateadores compartidos devuelven un
valor y nunca la redacción que lo rodea: un parámetro es un valor, y las
palabras que lo acompañan son justamente lo que cada organización
personaliza bajo el requisito de marca blanca. La segunda es que el
marcador se nombró incluyendo su unidad, porque quien edita ese texto desde
la configuración de la organización es personal administrativo de la
clínica y el nombre del marcador es lo único que declara qué significa el
número. La duración se leyó, además, de la columna del propio turno y no de
la configuración vigente del profesional: un turno conserva la duración con
la que fue reservado —una decisión ya tomada en el Motor de Turnos para que
un cambio posterior de configuración no reescriba turnos ya agendados—, de
modo que la configuración actual no es necesariamente lo que dura ese turno
en particular.

El alcance real de la corrección resultó, sin embargo, mayor que el que
enunciaba la observación. Como el texto base de cada plantilla se siembra
en la configuración de cada organización mediante una migración de datos
—la decisión, descripta más arriba en esta misma sección, de que la fila
por defecto exista desde el principio en lugar de aparecer recién cuando
alguien la personaliza—, y como el motor de plantillas prefiere siempre la
fila del inquilino por sobre el texto base del sistema, modificar la
constante del código habría dejado el mensaje exactamente igual en
cualquier base de datos ya migrada, incluida la del piloto: el criterio de
aceptación se habría cumplido únicamente en las pruebas. Se agregó
entonces una segunda migración de datos que actualiza esa fila, con la
misma condición no destructiva que rige a todas las migraciones de siembra
del proyecto —sólo se reescribe la fila que conserva íntegro el texto base
anterior, es decir, aquella que la clínica nunca editó—, aceptando como
contrapartida que una organización que sí hubiera personalizado su
plantilla siga sin mencionar la duración, porque corregir su redacción es
una decisión de la clínica y no de una migración.

La verificación del criterio de aceptación se hizo sobre el texto final
entregado al puerto de mensajería y no sobre los parámetros pasados a un
doble de prueba del motor de plantillas. El motivo es que un trabajo
programado que enviara un parámetro con un nombre distinto del que la
plantilla declara produce el error de parámetro faltante descripto al
comienzo de esta sección, que el manejo tolerante por turno del propio
trabajo convierte en una advertencia en el registro y descarta para seguir
con el resto del lote: el paciente no recibiría nada y una prueba con el
motor simulado no lo advertiría. La prueba agregada arma por eso el motor
de plantillas real, con únicamente la configuración por inquilino simulada,
y verifica que el mensaje efectivamente enviado contiene la duración de la
sesión.

### 3.2.5 Capa conversacional y WhatsApp

El Módulo 5 se abrió con el adaptador de IA (P5.1, TASK-46), la pieza que
conecta el futuro orquestador del chatbot con un modelo de lenguaje real. El
diseño del puerto y del adaptador se apoya en los conceptos de *function
calling* ya introducidos en el Marco Teórico (2.2). El puerto `AIPort`, ya
declarado en la fase de Fundaciones como una interfaz de una sola operación
resuelta por un adaptador *stub*, recibió en esta tarea su primera
implementación real —`OpenAiAdapter`, sobre el SDK oficial `openai`, contra
el modelo GPT-4o mini— y, con ella, un contrato más completo:
`processMessage` pasó a aceptar el historial completo de la conversación,
el conjunto de herramientas disponibles para ese turno y, opcionalmente, la
instrucción de sistema, devolviendo una respuesta que puede traer texto,
una o más solicitudes de uso de herramienta (*tool_use*), o ambas a la vez.
La ampliación del contrato, más allá de la firma mínima sugerida por el
ticket, fue necesaria porque una interfaz de un solo mensaje sin
herramientas ni instrucción de sistema no habría sido utilizable por el
orquestador que la consumirá en una fase posterior (P5.3, TASK-48): sin
conversación multi-turno no hay chatbot, y sin instrucción de sistema no
hay forma de que el orquestador incorpore el contexto del inquilino, que el
propio ticket reserva explícitamente para esa fase posterior.

La elección de GPT-4o mini como proveedor no surgió del texto del ticket
—cuyo título en el tablero, "Adaptador de IA con Sonnet", nombra a Claude
Sonnet de Anthropic— sino del documento de evaluación de tecnologías de
inteligencia artificial conversacional elaborado como insumo del Objetivo
Específico 2 del anteproyecto. Ese documento ubica la tarea que debe cubrir
el modelo de lenguaje en este chatbot como de complejidad conversacional
media —comprensión de intención, conducción del diálogo, invocación de
funciones del backend y redacción de respuestas breves—, sin requerir las
capacidades de razonamiento de frontera de los modelos más costosos, y
prioriza en consecuencia el equilibrio costo–calidad entre los tiers
económicos de cada proveedor. GPT-4o mini resultó allí sensiblemente más
económico que el tier equivalente de Anthropic (Claude Haiku 4.5), y el
documento señala además la madurez de su function calling —clave porque el
bot debe invocar de forma confiable las funciones del backend: consultar
grilla, reservar, generar PIN— como una fortaleza distintiva. Una primera
versión de esta tarea, escrita antes de que este proveedor se verificara
contra ese documento, se había implementado por error sobre Claude Sonnet;
la implementación se rehizo por completo sobre GPT-4o mini antes de cerrar
la tarea, y esta sección describe únicamente la versión final.

Las formas de dato del puerto —mensaje, bloque de texto, bloque de uso de
herramienta, bloque de resultado de herramienta, respuesta— se definieron
como tipos propios del dominio en `ai.port.ts`, en vez de reexportar los
tipos de un SDK de proveedor concreto. La decisión sigue el mismo principio
que ya regía a `MessagingPort` y `LockPort` desde Fundaciones: el dominio
no debe conocer el proveedor concreto detrás de un puerto, y usar los tipos
del SDK en la firma del puerto habría filtrado un detalle de
infraestructura —qué forma exacta tiene un bloque de contenido para ese
proveedor— hasta la capa de dominio, acoplando a cualquier futuro
consumidor del puerto (el orquestador, y eventualmente sus pruebas) a esa
forma concreta en lugar de a la abstracción. La traducción entre ambos
mundos —dominio y SDK— se aisló en un módulo de funciones puras
(`openai.mappers.ts`), separado del adaptador, para poder probarla de forma
independiente del cliente HTTP y de la política de reintentos. El cambio de
proveedor a mitad de la propia tarea terminó siendo, de hecho, la prueba de
que esa separación cumple su propósito: `ai.port.ts` no necesitó ningún
cambio de tipos al pasar de Anthropic a OpenAI, sólo el adaptador y su
módulo de mapeo.

El modelo y la clave de API se resuelven como configuración global del
sistema —variables de entorno `OPENAI_MODEL` (con `gpt-4o-mini` como valor
por defecto) y `OPENAI_API_KEY`—, nunca por inquilino, siguiendo la
instrucción explícita del ticket de que la marca blanca de este módulo se
resuelve enteramente en la instrucción de sistema que el orquestador arme
por inquilino, no en qué modelo ni qué credencial se usa. La clave se lee
una única vez, al construirse el cliente del SDK, con el mismo criterio de
*fail-fast* que ya aplica `JWT_SECRET` en el módulo de autenticación: si
falta, el arranque de la aplicación falla de inmediato en lugar de fallar
recién en el primer mensaje que un paciente le escriba al chatbot.

El manejo de errores es la parte del ticket con más decisiones de diseño
propias. El SDK `openai` ya reintenta automáticamente errores 429 y 5xx con
backoff exponencial, pero esa política vive dentro del transporte HTTP del
cliente y no es observable ni fácil de ejercitar con un doble de prueba
determinista. Se optó entonces por apagar el reintento incorporado del SDK
(`maxRetries: 0` al construir el cliente) e implementar la política pedida
por el ticket —reintento sólo ante 429/5xx, backoff exponencial, tope de
tres intentos— explícitamente en `OpenAiAdapter`, distinguiendo las clases
de error tipadas que expone el SDK (`RateLimitError`, `InternalServerError`)
de cualquier otro error, que se propaga sin reintentar. Agotado el cupo de
intentos, o ante un error no reintentable, el adaptador lanza
`AIProcessingError` —un `Error` de dominio, no una `HttpException`, con el
mismo criterio ya aplicado a `MissingTenantContextError` y a los errores de
`NotificationTemplateService`: el puerto no está detrás de un controlador
propio, así que no hay una respuesta HTTP concreta a la cual atarse— con un
mensaje descriptivo que nombra el estado y el tipo del error subyacente,
nunca su cuerpo completo ni, por construcción, la clave de API:
`OpenAiAdapter` nunca la recibe directamente, sólo el cliente del SDK ya
construido, inyectado por un token de dependencias separado
(`OPENAI_CLIENT`) que además permitió, en las pruebas, sustituir el cliente
real por un doble sin tocar el resto del cableado de inyección de
dependencias del módulo.

La traducción hacia la API de OpenAI expuso dos diferencias reales frente
al diseño original pensado sobre Anthropic, no sólo cosméticas. La primera
es que Chat Completions no tiene un campo `system` separado del resto de
los mensajes como sí tiene la API de Anthropic: el adaptador antepone la
instrucción de sistema como un mensaje más, con rol `system`, al principio
del historial. La segunda es que un turno de usuario con varios resultados
de herramienta no tiene equivalente de un único mensaje: mientras Anthropic
admite varios bloques `tool_result` dentro de un mismo turno de usuario,
OpenAI exige un mensaje `tool` independiente por cada llamada que se
responde, de modo que el módulo de mapeo traduce con un `flatMap` —un
mensaje de dominio puede convertirse en cero, uno o varios mensajes de
OpenAI— en lugar de una correspondencia uno a uno. Se agregó además un
resguardo defensivo específico de este proveedor: a diferencia de
Anthropic, que entrega los argumentos de una llamada a herramienta ya
parseados, OpenAI los entrega como una cadena JSON cruda que el propio
proveedor advierte que el modelo no siempre genera válida, por lo que un
parseo fallido se envuelve en un `AIProcessingError` descriptivo en lugar
de propagar una excepción de sintaxis sin contexto.

Con el adaptador real reemplazando al stub, el binding de `AI_PORT` en
`IntegrationsModule` deja de ser el único puerto de integración externa sin
implementación real: junto con `WaitlistResponsePort` (P4.5), es el segundo
en dejar de depender de un adaptador de prueba. `MessagingPort` y
`LockPort` siguen detrás de sus stubs, a la espera de las integraciones de
WhatsApp y TTLock de fases posteriores. La definición concreta de las
herramientas que el chatbot podrá invocar (P5.2, TASK-47) y el orquestador
que arme el historial, la instrucción de sistema por inquilino y el ciclo
de ejecución de herramientas (P5.3, TASK-48) quedan fuera del alcance de
esta tarea; ambos consumen el puerto tal como quedó definido aquí, sin
necesitar cambios adicionales de contrato para lo que se conoce hasta el
momento.

La segunda tarea del módulo (P5.2, TASK-47) definió el catálogo completo de
herramientas que el modelo puede invocar —disponibilidad, reserva,
confirmación, reprogramación, cancelación y consulta de turnos; búsqueda o
alta de paciente; registro y verificación de consentimiento; solicitud de
receta; y preguntas frecuentes—, cada una como un adaptador delgado sobre
un servicio de dominio ya construido en los módulos de Turnos y Pacientes,
sin lógica de negocio propia. Dos piezas de dominio no existían todavía y
se agregaron porque las herramientas no tenían a qué delegar sin ellas: el
modelo `Faq` —el diagrama entidad-relación ya lo dibujaba, pero nada antes
del chatbot lo necesitaba— con su propio servicio de lectura, y el servicio
que efectivamente escribe una `PrescriptionRequest`, entidad que existía
sólo como esquema desde el Motor de Turnos sin ningún llamador.

Las once herramientas comparten una única función de ensamblado en lugar de
repetir, once veces, las mismas cuatro partes que el ticket pide de cada
una: validar la entrada, resolver quién ejecuta la escritura, llamar al
servicio de dominio y devolver un resultado estructurado sin nunca lanzar
una excepción al modelo. Esa función compone tres piezas menores y también
reutilizables: la validación por class-validator —el mismo pase que ya
corren los DTO HTTP, y a mano, el importador de pacientes—, la resolución
del actor que ejecuta la escritura, y la conversión de cualquier excepción
en un resultado descriptivo. Esta última reutiliza directamente la
jerarquía de excepciones HTTP de Nest en lugar de inventar un esquema de
error propio: cada servicio de dominio ya lanza `NotFoundException`,
`BadRequestException` o `ConflictException` con un mensaje pensado para
mostrarse a quien hizo el pedido, y ese mismo mensaje —en vez del cuerpo de
una respuesta HTTP que aquí no existe— es el que la herramienta devuelve.

La resolución del actor es la pieza que conecta esta tarea con una decisión
tomada en Fundaciones y hasta ahora sin más consumidor que los trabajos
programados: el rol SYSTEM, sembrado una vez por organización, para
atribuir escrituras que no tienen un usuario humano detrás. Varios métodos
del servicio de turnos —reservar, confirmar, cancelar, reprogramar— ya
distinguían ese rol explícitamente antes de esta tarea, en particular
reprogramar, cuyo comentario nombra al bot como el caso que no debe exigirle
confirmación al paciente porque la propia solicitud del paciente ya es esa
confirmación. Lo único que faltaba era cómo resolver, desde un llamador sin
una petición HTTP detrás, cuál es el usuario SYSTEM de la organización en
curso — y ahí la diferencia con los trabajos programados es la que importa:
un cron no tiene ningún contexto de inquilino todavía y debe recorrer todas
las organizaciones desde afuera, mientras que una herramienta del chatbot
corre dentro de una conversación que ya pertenece a un inquilino. La
función nueva asume entonces que ese contexto ya está abierto —el futuro
orquestador lo abre una vez por turno de conversación, tal como una petición
HTTP lo abre una vez por request— y lee el usuario SYSTEM a través del
cliente ya acotado, sin recibir ni aceptar un identificador de organización
por parámetro. Esa ausencia es, a la vez, la respuesta concreta al
requisito de marca blanca del ticket: ningún `input_schema` declara ese
campo, así que no hay ningún camino por el que una herramienta pueda
terminar actuando para un inquilino distinto del que ya está en curso.

La búsqueda de preguntas frecuentes se resolvió con una similitud de
palabras normalizadas —minúsculas, sin tildes ni puntuación, comparadas
como conjuntos— en lugar de incorporar un motor de búsqueda semántica o una
extensión de base de datos para eso. El conjunto de FAQ de una organización
es, en la escala de este caso de estudio, unas pocas decenas de filas
curadas por la propia clínica, no el volumen para el que existen esas
herramientas, y ninguna de las dos estaba integrada en el proyecto en
ningún otro punto; por debajo de un puntaje mínimo la herramienta responde
que no encontró nada en lugar de forzar una respuesta, dejando en manos del
modelo —no de la herramienta— decidir qué decirle al paciente, en línea con
el resto del diseño: ninguna herramienta decide qué se comunica, sólo
entrega datos.

La herramienta de búsqueda o alta de paciente terminó exigiendo más datos
que los que el ticket enumera en su firma ilustrativa. El DTO de alta de
paciente, construido en el módulo de Pacientes con el comentario explícito
de que lo usan tanto un administrador como el chatbot, exige fecha de
nacimiento y contacto de emergencia además de nombre y apellido, porque el
documento de requisitos los pide como datos obligatorios para poder
reservar un turno; aceptar la firma abreviada del ticket habría creado
pacientes no reservables, pateando el mismo error hacia el momento de la
reserva en lugar de evitarlo en el alta. La herramienta terminó
reutilizando ese DTO completo —no un DTO abreviado propio—, con los cuatro
campos adicionales declarados opcionales en su propio contrato: el chatbot
puede invocarla primero sólo con el documento para verificar si el
paciente ya existe, y una segunda vez con el resto si tiene que darlo de
alta.

La herramienta de solicitud de receta, por último, se acotó deliberadamente
a lo que el ticket pide —dejar la solicitud con estado pendiente en base,
sin generar ninguna receta— y no disparó la notificación al profesional
que el tipo de evento correspondiente ya tiene declarado en el esquema
desde el Motor de Turnos, con un comentario explícito de que todavía no
tiene ningún llamador: conectarla aquí hubiera sido una decisión —a quién y
cuándo notificar— que no le corresponde a esta tarea, sino a la que
construya el flujo conversacional completo alrededor de esta herramienta
en una fase posterior.

La tercera tarea del módulo (P5.3, TASK-48) cerró el círculo entre el
adaptador de IA y el catálogo de herramientas: el orquestador de
conversación, la pieza que efectivamente conduce un turno del diálogo con
el paciente. `OrquestadorService.procesar(sessionId, mensajeEntrante,
organizationId)` recupera el historial de la sesión en curso, arma la
instrucción de sistema del inquilino, llama a `AIPort` con el catálogo
completo de herramientas de la tarea anterior y, si la respuesta trae una o
más solicitudes de uso de herramienta, las ejecuta, agrega cada resultado
al historial como el bloque `tool_result` correspondiente y vuelve a llamar
al modelo — repitiendo el ciclo hasta que la respuesta trae únicamente
texto, o hasta un tope de diez vueltas pensado como resguardo contra un
modelo que entrara en un ciclo de invocación sin fin. Lo primero que hace
`procesar` es abrir el contexto de inquilino para todo el turno con el
identificador de organización recibido por parámetro, exactamente como un
interceptor HTTP lo abre una vez por petición: es la promesa que el propio
comentario de la resolución del actor SYSTEM, escrito durante la tarea
anterior, ya dejaba anotada como pendiente para "el futuro orquestador", y
con ese contexto abierto tanto la configuración del inquilino como cada
herramienta que el modelo invoque durante el resto del turno leen a través
del cliente de Prisma ya acotado, sin necesitar un identificador de
organización propio en ningún punto intermedio.

El historial de la conversación se resuelve con el mismo criterio que ya
gobierna otra regla dependiente del tiempo en el sistema: sin un trabajo
programado dedicado. `ConversationSessionStore` guarda el historial en un
mapa en memoria, de alcance de proceso —suficiente para el despliegue de
instancia única previsto para el piloto, ya que el stack del proyecto no
declara ninguna infraestructura de caché además de Postgres—, y cada
lectura compara la marca de tiempo de la última actividad de esa sesión
contra el umbral de inactividad vigente del inquilino, descartando la
entrada ahí mismo si ya venció en lugar de esperar a que un trabajo
programado la recorra: la misma lógica de "la lectura es lo que nota que el
plazo venció" que ya aplica la regla de inactividad de pacientes del Módulo
2. El umbral, que el ticket da como "configurable, valor por defecto 30
minutos" sin precisar si es una configuración global o por inquilino, se
resolvió como dato por inquilino en `OrganizationConfig`, siguiendo la
misma instrucción general que ya rige el resto de los plazos configurables
del sistema. El identificador de sesión en sí —celular del paciente más
identificador de organización, para que dos inquilinos nunca compartan una
entrada del mapa aunque un paciente les escriba a ambos desde el mismo
número— queda como una función pura exportada (`buildSessionId`),
lista para que el futuro webhook de WhatsApp (P5.8, TASK-53) la use al
recibir cada mensaje entrante, sin que el orquestador necesite conocer ni
imponer ese formato.

El system prompt sigue exactamente el mismo mecanismo de personalización
por inquilino que ya resolvieron las plantillas de mensajes de
notificación: un texto base en el propio módulo, sustituible por el que el
inquilino haya guardado en su configuración, con el nombre de la
organización interpolado sobre el resultado mediante el mismo marcador
`{placeholder}` que esas plantillas ya usan para `{patientName}` o
`{professionalName}`. El texto base declara el rol del bot ("gestiona
únicamente turnos médicos de la organización"), el tono esperado —claro,
cordial, breve, sin jerga médica— y los límites explícitos que el propio
ticket pide como refuerzo de los guardrails de una fase posterior: nunca
revelar observaciones del profesional, nunca informar montos de copago,
nunca derivar a un operador humano, y redirigir directamente al profesional
ante una emergencia. Las dos funciones puras que resuelven la sustitución
de `{placeholder}` se trasladaron a una ubicación compartida bajo
`src/common/`, en lugar de duplicarlas dentro del módulo del chatbot: eran,
literalmente, el mismo mecanismo que ya vivía junto a las plantillas de
notificación de P4.1, así que el módulo de notificaciones pasó a importarlas
desde ahí en vez de conservar su propia copia. Tanto el umbral de
inactividad como el texto base del system prompt se sembraron, además, en
una migración de datos para toda organización ya existente —no sólo en el
seed de desarrollo—, con la misma razón ya documentada para la regla de
inactividad de pacientes y para las propias plantillas de notificación: sin
esa fila sembrada, el requisito de "configurable por inquilino" existiría
únicamente como una constante del código, invisible para cualquier
organización productiva.

Cuatro lugares distintos del código —la regla de inactividad de pacientes,
dos umbrales del Motor de Turnos y la ventana del recordatorio de turno—
repetían ya, con comentarios casi idénticos entre sí, la misma
comprobación: aceptar un valor de configuración sólo si es un número entero
positivo, y devolver una constante de reserva en cualquier otro caso. El
umbral de inactividad de sesión de esta tarea necesitaba exactamente esa
misma lógica, así que en lugar de sumar una quinta copia se generalizó en
un método nuevo del servicio de configuración por inquilino
(`getPositiveInteger`), dejando los cuatro sitios preexistentes sin tocar
—una migración de esos cuatro al método nuevo excede el alcance de esta
tarea— pero disponible ya para cualquier regla futura con esa misma forma.

Un detalle de implementación no evidente hasta que se lo pisó por accidente
fue el manejo del arreglo de mensajes dentro del ciclo. Cada vuelta agrega
el turno del modelo y el resultado de cada herramienta al mismo arreglo en
construcción; pasar ese arreglo por referencia a `AIPort.processMessage` en
cada llamada funciona con el adaptador real —que serializa el pedido antes
de devolver el control—, pero deja una dependencia implícita sobre el
comportamiento de cualquier adaptador futuro que retuviera esa misma
referencia más allá de la llamada. Se optó por pasarle una copia superficial
del arreglo en cada vuelta en lugar del arreglo mutable en construcción, un
costo mínimo que evita esa clase de error para cualquier implementación de
`AIPort` que llegue a existir, no sólo la actual. Agotadas las diez vueltas
sin que el modelo devuelva una respuesta de sólo texto, el orquestador
lanza un error de dominio (`ToolCallLimitExceededError`) en lugar de armar
algún texto de disculpa por su cuenta —este servicio no está todavía detrás
de ningún controlador propio, así que qué llega a leer el paciente ante ese
error queda para cuando el webhook de WhatsApp exista— y deliberadamente no
guarda el historial de ese turno atascado, de modo que el próximo mensaje
del paciente arranca desde el último turno que sí produjo una respuesta, en
vez de repetir el ciclo trabado.

Con el orquestador ya implementado, el Módulo 5 completa el trayecto entre
el adaptador de IA (P5.1) y el catálogo de herramientas (P5.2): las tres
piezas encajan sin que ninguna de las dos anteriores haya necesitado
cambios de contrato para esta tarea, incluido el parámetro `system` que
P5.1 ya había agregado previendo exactamente este uso. Los guardrails que
refuerzan los límites del system prompt sobre la respuesta final del modelo
(P5.4, TASK-49), los flujos de negocio concretos por sobre este ciclo
genérico de herramientas (P5.5, TASK-50) y la integración real con
WhatsApp —incluido quién construye el identificador de sesión y quién
atrapa el error de tope de iteraciones— (P5.8, TASK-53) quedan para fases
posteriores del mismo módulo.

La cuarta tarea del módulo (P5.4, TASK-49) agregó la capa de guardrails que
el system prompt de la tarea anterior ya anticipaba como refuerzo, no como
reemplazo: un conjunto de reglas determinísticas —basadas en palabras clave
y expresiones regulares, no en una segunda llamada al modelo— que revisa la
conversación en dos puntos concretos del ciclo del orquestador. El primero
es el mensaje entrante del paciente, revisado antes de que el orquestador
llame al modelo o a cualquier herramienta: si contiene una palabra clave de
urgencia médica o de crisis de salud mental —el foco explícitamente
psiquiátrico/psicológico de la clínica hizo que este segundo grupo se
incorporara junto al de urgencias médicas generales—, el turno se corta ahí
mismo con el mensaje fijo de fuera de alcance, sin gastar ninguna llamada al
modelo sobre una respuesta que ya está decidida de antemano. El segundo
punto es la respuesta final en texto plano del modelo, revisada antes de
que se guarde en el historial de la sesión o se devuelva al paciente, contra
cuatro reglas independientes entre sí: ninguna mención del campo
observaciones del profesional, ningún monto de copago, ninguna derivación a
un operador humano, y ningún dato —nombre, turno, DNI— de un paciente ajeno
a la conversación en curso. Cualquiera de las cuatro reemplaza la respuesta
completa por uno de tres textos canónicos, nunca sólo el fragmento que la
violó, y ese texto reemplazado —nunca el original generado por el
modelo— es el que efectivamente queda guardado en el historial, para que un
turno bloqueado no reingrese como contexto en el que el propio modelo
"cree" haber dicho ya lo que el guardrail impidió decir.

A diferencia del umbral de inactividad de sesión y del texto del system
prompt (P5.3), las cinco reglas de esta tarea no son configurables por
inquilino: el propio ticket lo exige de forma explícita —los guardrails
son globales y ninguna configuración de organización puede desactivarlos—,
una excepción deliberada al criterio general del proyecto de tratar las
reglas de negocio como datos por inquilino en lugar de como constantes de
código, justificada de la misma forma que ya lo están los topes anti-abuso
del sistema: esto es un piso de seguridad, no una preferencia configurable
de la clínica. La regla sobre datos de otros pacientes resultó la más
limitada de las cinco por esa misma razón de diseño: el ticket la describe
en términos de "el profesional de la sesión activa", pero identificar a qué
paciente y a qué profesional pertenece la conversación en curso es
información que todavía no llega hasta esta capa —el guardrail, por
diseño del propio ticket, sólo analiza el texto ya generado, no los
parámetros con los que el modelo invocó cada herramienta durante el
turno—, y resolverla en esta tarea habría significado adelantar una
decisión de diseño reservada a los flujos de negocio de la fase siguiente
(P5.5, TASK-50). La regla se implementó en su lugar como dos resguardos de
texto deterministas y explícitamente heurísticos: un número suelto de siete
u ocho dígitos, la forma de un documento de identidad argentino, y una
referencia en tercera persona a un paciente nombrado ("el turno de
\<Nombre\>"), dado que una respuesta legítima siempre se dirige al paciente
en segunda persona.


Con el ciclo genérico de herramientas y la capa de guardrails ya en su
lugar, la última tarea de esta sección (P5.5, TASK-50) construyó los cinco
flujos conversacionales que el paciente recorre efectivamente —reservar,
confirmar, reprogramar, cancelar y consultar turnos— junto con los dos
pasos previos comunes a todos ellos: la identificación del paciente por
número de documento y la verificación del consentimiento antes de tomar
una reserva. En una arquitectura de *function calling* como la descripta,
implementar un flujo no consiste en escribir una máquina de estados por
flujo dentro del orquestador: quien conduce la conversación es el propio
modelo, y lo que el sistema aporta son el manual de procedimiento que ese
modelo lee en cada turno, las herramientas sin las cuales el
procedimiento no puede ejecutarse, y la aplicación determinística de
aquellos pasos que el documento de requisitos atribuye al sistema y no al
bot. La alternativa de codificar cada flujo como una máquina de estados
imperativa se descartó por duplicar, en código, la conducción que el
catálogo de herramientas ya delega en el modelo, anulando la razón misma
por la que se eligió esta arquitectura.

El manual de procedimiento se anexa al *system prompt* de cada turno, y
no se incorporó a la plantilla base que el inquilino puede sobrescribir.
La división separa dos cosas de naturaleza distinta: el texto base
describe rol, tono y límites del bot —exactamente la parte que una
implantación de marca blanca tiene razones legítimas para reescribir—,
mientras que el manual describe el procedimiento que imponen las
herramientas del propio sistema, de modo que una clínica que lo
reescribiera no estaría personalizando la reserva sino rompiéndola. Es el
mismo criterio aplicado a los guardrails en la tarea anterior, trasladado
del plano de la seguridad al del procedimiento. Al redactar el manual
apareció además una carencia no prevista en el enunciado de la tarea: la
consulta de disponibilidad recibe un rango de fechas y un modelo de
lenguaje no dispone de reloj, con lo cual ninguna expresión como "la
semana que viene" —que es como un paciente pide un turno por mensajería—
podía traducirse a un rango concreto. El *prompt* pasó entonces a
informar la fecha del día, resuelta en cada turno y no al construir la
constante, para que una conversación sostenida al día siguiente no reciba
la fecha del anterior.

Del lado de las herramientas, el catálogo de P5.2 resultó incompleto para
sostener el flujo de reserva: tanto la consulta de disponibilidad como la
reserva reciben el identificador del profesional, y el modelo no tenía
forma de obtener uno que no fuera inventándolo. Se agregó una herramienta
de listado de profesionales que delega en la misma lectura que ya servía
el listado HTTP del Módulo 1 y que, al resolverse por el cliente de
persistencia acotado por inquilino, queda confinada por construcción a la
organización de la sesión —la respuesta concreta al requisito de marca
blanca de que el chatbot sólo muestre profesionales del inquilino—. La
herramienta no recibe parámetros y devuelve únicamente identificador,
nombre y matrículas: el documento de requisitos establece que la
especialidad no modifica el flujo de reserva y que el paciente elige al
profesional con independencia de ella, y no exponer el dato resulta una
garantía más fuerte que instruir al modelo para que no filtre por él, ya
que un filtro que la herramienta no puede expresar es un filtro que el
modelo no puede aplicar.

El mismo criterio estructural resolvió el dato de contacto del paciente.
El enunciado de la tarea listaba el celular entre los datos que el bot
debía solicitar a quien no estuviera registrado, pero el número ya está en
la conversación: la clave de sesión definida en P5.3 es, precisamente, la
organización más el número desde el que el paciente escribe. Pedírselo
obliga a retipear un dato ya entregado de forma implícita, con la
posibilidad de equivocarse y de que la clínica quede con un número por el
que no puede contactarlo. El parámetro se retiró del esquema de la
herramienta de alta —no sólo de las instrucciones del manual— y el valor
se lee de un contexto de conversación por turno, equivalente del contexto
de inquilino ya existente y abierto por el orquestador dentro de él, de
modo que la herramienta lo obtiene sin que el modelo llegue a conocerlo.
Se descartó tanto sumar un parámetro adicional a la operación del
orquestador, que duplicaría un dato ya transportado por la clave de sesión
con posibilidad de discrepancia, como interpolar el número en el *prompt*
para que el modelo lo reenviara como argumento. El número se escribe
únicamente en el alta y nunca sobre una ficha existente, porque
reescribir el dato de contacto registrado por la clínica con cualquier
número que escriba sería una actualización destructiva silenciosa que el
documento de requisitos no contempla: lo que éste prevé es una
actualización de datos recién tras un año sin concurrir.

La verificación de consentimiento se reformuló para devolver, en el mismo
resultado, el texto que debe enviarse al paciente cuando no hay
aceptación registrada, renderizado con el mismo motor de plantillas —y
por lo tanto con la misma personalización por inquilino— que los avisos
de la sección anterior. La alternativa de exponer esa redacción como una
herramienta separada se descartó porque verificación y solicitud son un
único paso del flujo, y un modelo obligado a pedir la redacción por
separado tarde o temprano la improvisaría; tratándose del consentimiento
que exige la Ley 25.326, la redacción es precisamente lo que no admite
improvisación. La consulta de turnos del paciente, por su parte, pasó de
delegar en el listado general a devolver sólo los turnos activos y
futuros con el nombre del profesional de cada uno, porque los cuatro
flujos que parten de ella —consultar, confirmar, reprogramar y cancelar—
necesitan exactamente el conjunto de turnos sobre los que el paciente
todavía puede actuar, y ofrecerle al modelo un turno ya transcurrido sólo
habilita que el bot proponga cancelarlo. El conjunto de estados
considerados activos no se enumeró de nuevo sino que se deriva de la
máquina de estados del turno introducida en 3.2.3, como aquellos desde
los que la cancelación sigue siendo alcanzable, para que no pueda
desfasarse de ella en silencio.

El único paso de los cinco flujos que se implementó de forma
determinística, además de pedirse en el manual, es el ofrecimiento de
reprogramación posterior a una cancelación. El documento de requisitos lo
redacta como una acción del sistema —ante una cancelación, el sistema le
pregunta al paciente si desea reprogramar— y no como una sugerencia de
estilo para el bot; dejarlo enteramente en manos del modelo, además,
habría vuelto inverificable el criterio de aceptación correspondiente,
dado que una prueba con el puerto de inteligencia artificial simulado no
puede demostrar nada sobre un texto que la propia prueba redacta. El
ofrecimiento se aplica sobre el texto final del turno, en el mismo punto
y por la misma razón que los guardrails: una regla que debe cumplirse
para todos los pacientes no puede depender de que el modelo haya seguido
el *prompt* en esa ocasión. La condición que lo dispara son las
herramientas que efectivamente tuvieron éxito durante el turno y no el
texto de la respuesta, de modo que una cancelación rechazada por la
ventana mínima de anticipación —regla que ya vivía en el servicio de
turnos desde 3.2.3 y que deliberadamente no se replicó en esta capa— deja
el turno en pie y no genera ofrecimiento alguno; y una respuesta
bloqueada por un guardrail nunca recibe el agregado, porque anexarle una
pregunta reabriría la conversación que la regla acaba de cerrar. Si el
modelo ya ofreció la reprogramación con palabras propias, esa redacción
se conserva intacta.

La validación de los cinco flujos se hizo con pruebas de conversación
extremo a extremo contra la base de datos real, sustituyendo únicamente
el puerto de inteligencia artificial por un guion de llamadas a
herramientas y dejando reales el orquestador, el catálogo de
herramientas, los servicios de dominio de los Módulos 2 y 3 y el motor de
persistencia. Cada flujo se verifica por el estado que deja en la base y
no por el texto de la respuesta —que, con el modelo simulado, lo escribe
la propia prueba—: la reserva de un paciente nunca visto se comprueba por
el registro del paciente, la fila de consentimiento y el turno
resultantes; la confirmación, por la transición de estado y su marca
temporal; la reprogramación, por el instante actualizado del turno; y la
cancelación por debajo de la ventana mínima, por el turno que permanece
intacto. El doble del puerto verifica además que ninguna herramienta haya
fallado de forma inadvertida, ya que un guion cuyas herramientas fallaran
todas seguiría, de otro modo, dando por buena la conversación.
