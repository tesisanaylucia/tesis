# Fase 3 — Motor de Turnos (backend) — administración del calendario de feriados (TASK-78)

## Qué se implementó

Se implementaron los endpoints de administración que permiten a un
administrador del inquilino gestionar el calendario de feriados de su
organización: `GET /admin/feriados?year=`, `POST /admin/feriados`,
`PATCH /admin/feriados/:date` y `DELETE /admin/feriados/:date`, en un
módulo nuevo (`src/holidays/`). El modelo `Holiday` ya existía desde
TASK-34 y ya era consumido por `AvailabilityService.getSlots` (TASK-35);
esta tarea no agrega nada al esquema ni al algoritmo de disponibilidad,
sólo la superficie de administración que faltaba para que un
administrador pudiera modificar ese calendario sin intervención directa
sobre la base de datos.

Las cuatro operaciones quedan restringidas al rol administrador mediante
el decorador de roles aplicado a nivel de controlador, y cada una se
resuelve enteramente a través del cliente de Prisma acotado por
inquilino: un feriado de otra organización es, para cualquiera de las
cuatro operaciones, tan inexistente como un feriado que nunca se creó.
La búsqueda por fecha (clave natural acotada por inquilino, no el
identificador de la fila) se ancla mediante una consulta de
`findFirst` sobre el campo `date` con la extensión de Prisma inyectando
el identificador de organización, en lugar de una consulta por clave
compuesta única (`organizationId`, `date`) — la misma limitación de la
extensión de acotamiento por inquilino, que sólo sabe inyectar un
`organizationId` plano en la cláusula `where` y no puede poblar el
objeto anidado que exige una clave compuesta, ya documentada en la fase
de Fundaciones al resolver el mismo problema para la configuración por
inquilino.

La eliminación de un feriado no verifica si existen turnos activos en esa
fecha antes de proceder: el documento de requisitos pide explícitamente
que la eliminación se advierta, no que se bloquee, ya que ningún turno ya
reservado o confirmado deja de ser válido por dejar de estar la fecha
marcada como feriado — el efecto de la eliminación es únicamente que esa
fecha vuelve a ofrecer franjas libres en consultas futuras de
disponibilidad. El endpoint responde entonces con el conteo de turnos en
estado reservado o confirmado que ya ocupaban esa fecha, en el mismo
cuerpo de la respuesta de la eliminación exitosa, en lugar de la
respuesta sin cuerpo que usa el resto de las eliminaciones del sistema.

## Decisiones y por qué

**El acceso de un administrador de un inquilino a un feriado de otro
responde con "no encontrado" (404), no con "prohibido" (403) como
describe literalmente el documento de requisitos.** El resto del sistema
resuelve el acotamiento por inquilino haciendo que una fila de otra
organización sea invisible para la consulta, en lugar de visible pero
vedada — el mismo tratamiento que ya recibe, por ejemplo, un profesional
de otra organización al consultarlo por identificador. Tratar el
feriado de otro modo habría exigido primero comprobar su existencia sin
el acotamiento por inquilino para poder distinguir "no existe" de
"existe pero no es tuyo", una consulta que ningún otro recurso del
sistema necesita y que reintroduciría, sólo para este endpoint, la fuga
de información entre inquilinos que el acotamiento automático existe
para evitar. El texto en español del ticket se interpretó, en la misma
línea que decisiones anteriores de esta fase, como la descripción
funcional del aislamiento esperado y no como una exigencia literal del
código de estado HTTP.

**La ruta de administración no valida por separado si existen turnos
antes de eliminar un feriado — sólo informa.** Bloquear la eliminación
habría requerido definir una regla de negocio nueva, no pedida por el
ticket, sobre qué hacer con esos turnos (¿reprogramarlos?, ¿cancelarlos?);
el documento de requisitos es explícito en que la eliminación debe
proceder de todos modos, así que la única responsabilidad del endpoint es
dejar visible al administrador cuántos turnos coexisten con esa fecha
recién liberada, información que puede motivar una acción manual
posterior por su parte.

**El parámetro de fecha en la ruta (`:date`) se valida con un `PipeTransform`
nuevo, en vez de dejar que una fecha mal formada llegue a la consulta como
una fecha inválida.** El validador de fecha calendario existente en el
sistema (`IsCalendarDate`) está pensado para campos de un cuerpo de
solicitud validado por `class-validator`, no para un parámetro de ruta; sin
una validación explícita, una fecha mal formada en la URL se habría
convertido silenciosamente en una fecha inválida de JavaScript y, al no
coincidir con ningún feriado almacenado, habría respondido con 404 en
lugar de señalar el error de formato con 400 — el mismo criterio que ya
aplica el validador de cuerpo, llevado al parámetro de ruta equivalente.

## Alternativas descartadas

- **Responder 403 ante un feriado de otro inquilino**, siguiendo el texto
  literal del ticket: descartada por ser inconsistente con el tratamiento
  "no encontrado" que ya reciben todos los demás recursos acotados por
  inquilino del sistema, y por requerir una consulta adicional sin
  acotamiento para poder distinguir ambos casos.
- **Bloquear la eliminación de un feriado si existen turnos activos en esa
  fecha**: descartada porque el propio documento de requisitos pide que la
  eliminación proceda de todos modos, y porque definir qué hacer con esos
  turnos es una decisión de negocio fuera del alcance de este ticket.
- **Usar `findUnique` sobre la clave compuesta (`organizationId`, `date`)
  para las operaciones de modificación**, en lugar de `findFirst` seguido
  de una operación por identificador: descartada por la misma limitación
  de la extensión de acotamiento por inquilino ya documentada al resolver
  el mismo problema para la configuración por inquilino en la fase de
  Fundaciones.

## Entidades / puertos / adaptadores tocados

- `src/holidays/holidays.module.ts`, `holidays.controller.ts`,
  `holidays.service.ts`, `holiday.presenter.ts` (nuevos): el módulo de
  administración de feriados completo.
- `src/holidays/dto/create-holiday.dto.ts`, `update-holiday.dto.ts`,
  `find-holidays-query.dto.ts` (nuevos): validación de la creación, la
  edición de la descripción y el filtro por año del listado.
- `src/common/validation/parse-calendar-date.pipe.ts` (nuevo): validación
  del parámetro de ruta `:date` como fecha calendario existente.
- `src/app.module.ts` (modificado): registro de `HolidaysModule`.

No se modificó el esquema de Prisma ni ningún puerto o adaptador: la
entidad `Holiday` y su acotamiento por inquilino ya existían desde
TASK-34.

## Tests y qué validan

- `src/holidays/holidays.service.spec.ts` (nuevo, 9 pruebas): el cliente
  de Prisma se simula en memoria.
  - El listado filtra por el año recibido y, en su ausencia, por el año
    actual.
  - La creación escribe el feriado y la entrada de auditoría dentro de la
    misma transacción.
  - Una violación de la restricción de unicidad al crear se traduce en un
    rechazo 409, y cualquier otro error se propaga sin modificar.
  - La edición y la eliminación rechazan con 404 cuando no existe un
    feriado para la fecha pedida.
  - La eliminación cuenta los turnos en estado reservado o confirmado de
    esa fecha, borra el feriado y registra ambos datos en la auditoría,
    dentro de una misma transacción.
- `test/holidays.e2e-spec.ts` (nuevo, 12 pruebas): contra la instancia
  local de PostgreSQL, con dos organizaciones y un profesional con horario
  de atención configurado.
  - Rechazo del acceso sin autenticar (401) y del rol profesional en las
    cuatro rutas (403).
  - El criterio de aceptación central del ticket: agregar un feriado hace
    desaparecer las franjas de `GET /profesionales/:id/disponibilidad`
    para esa fecha, y eliminarlo las restituye.
  - Creación y listado filtrado por año.
  - Rechazo de una fecha duplicada para el mismo inquilino (409).
  - Rechazo de una fecha mal formada, tanto en el cuerpo de la creación
    como en el parámetro de ruta de la edición (400).
  - Edición de la descripción de un feriado existente, y 404 al editar una
    fecha sin feriado.
  - 404 al editar o eliminar un feriado desde el administrador de otra
    organización, y un listado que no incluye feriados ajenos.
  - Eliminación de un feriado con un turno confirmado en esa fecha: el
    feriado se elimina igual y la respuesta informa el turno afectado.
- Ejecución: suite unitaria completa en verde (28 suites / 282 pruebas).
  La suite end-to-end completa corre en verde en modo serie (29 suites /
  361 pruebas); en el modo paralelo por defecto persiste la misma
  intermitencia por conflictos de transacción serializable, ya señalada
  como condición preexistente del entorno de pruebas en la entrada de
  TASK-41 — se comprobó además que esa intermitencia reproduce igual
  sobre `main` sin ningún cambio de esta tarea, antes de descartarla como
  causa. Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-78-holiday-admin-crud` (creada a
  partir de `main` ya actualizado con TASK-41). Commit `c2f2714` al
  momento de redactar esta entrada. Pusheada a `origin`, pendiente de
  Pull Request en Bitbucket.
- Ticket: TASK-78 ("P3.b – FERIADO admin: endpoints CRUD para gestión del
  calendario de feriados por tenant"). Depende de TASK-34 (modelo
  `Holiday`) y TASK-35 (`AvailabilityService` lo consume), ambas ya
  fusionadas a `main`.
