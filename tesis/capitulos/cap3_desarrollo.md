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
