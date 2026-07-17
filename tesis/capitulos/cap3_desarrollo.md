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
