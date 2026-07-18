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
