# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Retención de datos y derechos del titular, Ley 25.326/26.529 (TASK-67, P8.2)

## Qué se implementó

Se implementó P8.2 ("Cumplimiento Ley 25.326 / 26.529") del SRS: (1) una
política de retención de datos, documentada directamente sobre las tablas y
columnas que gobierna mediante comentarios `COMMENT ON` de PostgreSQL,
aplicados por una migración de Prisma escrita a mano; (2) verificación de
que el registro de auditoría del sistema nunca guarda el contenido de los
datos del paciente, solo que un recurso fue tocado; (3) tres endpoints
administrativos nuevos bajo `/admin/pacientes/:id` que implementan los
derechos de acceso, supresión y rectificación de los Arts. 14 y 16 de la
Ley 25.326; (4) verificación de que el consentimiento informado sigue
siendo condición necesaria para reservar un turno, ya implementado en
tareas anteriores.

## Decisiones y por qué

**La política de retención se documentó en el propio esquema de la base de
datos, no en un documento aparte.** Ni la especificación de requisitos ni
el anteproyecto de la tesis fijan un plazo concreto de retención para
ningún tipo de dato —se verificó explícitamente contra el propio
anteproyecto antes de decidir un número—, de modo que los plazos elegidos
(cinco años desde la fecha de cada entrada de auditoría antes de ser
archivada, y cinco años desde la baja o supresión de un paciente antes de
que su fila sea elegible para archivado) son una decisión de diseño de
este sistema, no un mínimo legal transcripto de una fuente. Prisma no tiene
sintaxis para declarar un comentario de tabla o columna en su propio
archivo de esquema, así que la migración escribe sentencias `COMMENT ON`
directamente en SQL —el mismo patrón ya usado en tareas anteriores para
expresar una restricción `CHECK` que Prisma tampoco puede declarar—, y el
esquema referencia esa migración desde un comentario junto a cada columna
afectada, de modo que la política queda consultable con las herramientas
propias de PostgreSQL y no solo legible por quien lea el código fuente.
Ningún trabajo automático de archivado existe todavía: la política
documentada es la que ese trabajo, cuando se construya, deberá seguir.

**La limitación del registro de auditoría no requirió cambios de código,
solo verificación.** Se recorrieron los veintiún puntos del código que
escriben una entrada de auditoría y se confirmó que ninguno pasa el
contenido real de un dato del paciente —el texto de una observación, una
fecha de consulta, el motivo de una cancelación— sino únicamente
identificadores, nombres de campos que cambiaron o motivos codificados como
enumeración. Esa disciplina ya estaba estable desde tareas anteriores (la
función que calcula qué campos cambiaron, usada por cada punto de escritura
del módulo de pacientes, existe justamente para eso), así que el aporte de
esta tarea fue dejar esa verificación registrada explícitamente como regla
documentada, en lugar de una propiedad implícita que cada tarea nueva
debía descubrir por sí misma leyendo el código existente.

**Los tres endpoints de derechos del titular se separaron en un módulo
propio, distinto del módulo de pacientes existente.** La exportación de
datos necesita leer, además del paciente, sus turnos y sus solicitudes de
receta —entidades de otros dominios—, y el módulo de turnos ya depende del
módulo de pacientes, de modo que hacer depender a este último del de turnos
hubiera cerrado un ciclo de importación entre módulos. Se optó por un
módulo nuevo que se ubica por encima de ambos y lee las tablas de turnos y
solicitudes de receta directamente a través del cliente de Prisma con
alcance de tenant, sin pasar por el servicio de turnos —el mismo patrón que
ya usa el servicio de disponibilidad para leer feriados y turnos sin
depender de ese módulo—. Las tres operaciones quedan auditadas con nombres
de acción propios, distintos de los que ya usa la edición administrativa
ordinaria de un paciente, de modo que una consulta de cumplimiento pueda
distinguir una solicitud formal de derechos de una edición rutinaria.

**La supresión no es un borrado físico: es la misma baja lógica que ya
existía, ampliada con anonimización real de los campos identificatorios.**
El sistema ya tenía, de una tarea anterior, una baja lógica administrativa
que solo marca al paciente como inactivo sin tocar ningún otro campo,
pensada para un paciente que puede volver a ser atendido. Esa operación no
alcanza lo que la Ley 25.326 pide para una solicitud de supresión: se
decidió que la operación de este ticket, además de marcar la baja, reemplace
el documento, el nombre, el apellido y todo dato de contacto por un valor
que no identifica a ninguna persona real, conservando la fila —nunca se
consideró un borrado físico, porque una clave foránea ya existente impide
explícitamente eliminar un paciente mientras exista alguna entrada de
auditoría que lo nombre, precisamente para que el registro que prueba que
la supresión ocurrió sobreviva a la fila que describe—. Antes de anonimizar
se verifica que no existan turnos futuros pendientes de confirmación o ya
confirmados; si existen, la operación se rechaza con un mensaje que nombra
esos turnos, para que se cancelen primero. Esa verificación y la escritura
posterior se ejecutan bajo el nivel de aislamiento más estricto que ofrece
la base de datos, no bajo el nivel por defecto, porque de lo contrario un
turno reservado exactamente en el instante entre la verificación y la
anonimización quedaría huérfano bajo una identidad que ya no existe.

**La rectificación reutiliza el mismo método de edición que ya usaba el
endpoint administrativo ordinario, en lugar de reimplementar su
validación.** La única diferencia real entre "editar un paciente" y
"atender una solicitud formal de rectificación" es el nombre que la
entrada de auditoría debe llevar; el método existente se modificó para
aceptar ese nombre como parámetro opcional, con el valor que ya tenía por
defecto, de modo que el punto de entrada administrativo nuevo delega en él
sin duplicar el manejo de conflictos de documento ni el cálculo de qué
campos cambiaron.

**La exportación no incluye las observaciones internas del profesional
sobre el paciente, ni siquiera para una administradora que actúa sobre una
solicitud de la Ley 25.326.** Una decisión de una tarea anterior ya había
establecido que ese campo es de uso exclusivo del profesional que lo
escribe y que ninguna otra lectura del paciente lo expone, ni siquiera a
una administradora. Se decidió no relajar esa restricción para este caso:
una nota clínica interna de un profesional sobre su paciente no es un dato
que el derecho de acceso del Art. 14 alcance a exponer a través de un canal
administrativo, y la propia estructura de datos que ya arma la respuesta
del paciente excluye ese campo por construcción, no por una omisión
puntual de este endpoint.

## Entidades / servicios tocados

- `prisma/migrations/…_document_data_retention_policy/` (nueva): sentencias
  `COMMENT ON` de PostgreSQL sobre las tablas de auditoría y consentimiento
  y sobre la columna de baja del paciente.
- `src/patient-data-rights/` (módulo nuevo): controlador y servicio de los
  tres endpoints, presentador de la exportación con una selección propia
  por cada entidad exportada.
- `src/common/csv/to-csv.ts` (nuevo): serialización mínima a CSV con el
  escape que exige el formato, reutilizada por la exportación.
- `src/patients/patients.service.ts`: el método de edición acepta ahora un
  nombre de acción de auditoría opcional.
- `CLAUDE.md`: nueva sección de retención de datos y derechos del titular,
  y una regla nueva en la sección de auditoría verificando que ningún
  punto del código registra contenido del paciente.

## Tests

- `src/common/csv/to-csv.spec.ts` (nuevo): escape de comas, comillas y
  saltos de línea.
- `src/patient-data-rights/patient-data-export.presenter.spec.ts` y
  `patient-data-rights.service.spec.ts` (nuevos): armado de la respuesta de
  exportación, verificación del filtro de turnos pendientes antes de
  suprimir, y de la anonimización efectiva de los campos identificatorios.
- `test/patient-data-rights.e2e-spec.ts` (nuevo, contra PostgreSQL real):
  exportación completa en JSON y en CSV, ausencia de la observación
  clínica en la exportación, rechazo con 409 de una supresión con turno
  pendiente nombrándolo en el mensaje, supresión exitosa con verificación
  de que la entrada de auditoría histórica del alta del paciente
  sobrevive, y aislamiento entre organizaciones (404) en los tres
  endpoints.

Suite completa en verde al cierre: 75 suites unitarias (753 pruebas) y 48
suites de extremo a extremo (546 pruebas) contra PostgreSQL real, lint y
verificación de tipos sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-67-data-rights-compliance`, creada
  desde `main` para esta tarea. Sin commitear al cierre — pendiente de
  autorización explícita de la autora antes de commitear/pushear, según lo
  indicado para esta sesión.
