# Fase 2 — Pacientes (backend) — entidades base del dominio (TASK-27)

## Qué se implementó

Se incorporaron al esquema de Prisma las cuatro entidades base del dominio
Pacientes tomadas del diagrama entidad-relación (`modelo_base_de_datos.png`)
y del documento de especificación de requisitos, módulo Pacientes:
`Patient` (paciente), `PatientProfessional` (vínculo paciente-profesional),
`Consent` (consentimiento de tratamiento de datos) y `PrescriptionRequest`
(solicitud de receta). Se agregaron los enumerados `PatientType` (nuevo /
recurrente) y `PrescriptionRequestStatus` (pendiente / generada), siguiendo
la convención del repositorio de nombrar modelos y columnas en inglés.

La tarea abarca únicamente el esquema y su migración
(`20260721034219_add_patient_entities`); no introduce endpoints ni
servicios, que corresponden a tareas posteriores de la misma fase (P2.2 a
P2.6).

Adicionalmente se saldó una deuda declarada en el propio esquema: la
columna que la traza de auditoría reservaba para referenciar al paciente
era, desde su creación, un identificador sin clave foránea, porque la tabla
destino no existía. La tarea la convirtió en clave foránea compuesta real y,
al dejar de ser un marcador de posición de una tabla nombrada en español, la
renombró a inglés como el resto de las columnas.

## Decisiones y por qué

**El identificador del paciente es único por organización, no globalmente.**
El documento de requisitos establece que el sistema identifica al paciente
por su documento. Se declaró la unicidad sobre el par (organización,
documento) en lugar de sobre el documento solo: en un sistema diseñado para
marca blanca, la misma persona puede ser paciente de dos organizaciones
distintas, y cada una debe poder mantener su propio registro sin que el
alta de una impida la de la otra.

**Los datos que varían por profesional viven en el vínculo, no en el
paciente.** El diagrama ubica en la tabla de vínculo la prioridad, las
observaciones de uso interno, el correo electrónico, el contacto de
emergencia, el indicador de primera sesión, el tipo de paciente y la fecha
de última consulta. Se respetó esa ubicación porque expresa un hecho del
dominio: una misma persona puede ser paciente nuevo para un profesional y
recurrente para otro, con prioridad y observaciones distintas en cada caso.
Modelar esos campos en el paciente habría obligado a elegir arbitrariamente
cuál de las descripciones prevalece.

**El vínculo conserva el identificador de organización; las demás hijas,
no.** Ésta fue la decisión de diseño central de la tarea y exigió precisar
la regla de normalización vigente en el repositorio, que prohíbe replicar el
identificador de organización en toda entidad alcanzable a través de un
padre que ya lo posee. El criterio adoptado es qué *restringe* la columna. Con
un único padre, la columna es una réplica pura —el padre ya la determina— y
por tanto se omite: es el caso del consentimiento. Con dos padres acotados
por organización, en cambio, esa única columna es lo que obliga a que ambos
pertenezcan a la *misma* organización, restricción que ningún par de claves
foráneas independientes puede expresar; y, al ser leída por las dos claves
foráneas compuestas en cada escritura, tampoco puede divergir de ninguno de
los dos padres, que es el riesgo que motivaba la prohibición. El vínculo
paciente-profesional conserva entonces la columna, y la base de datos rechaza
por sí misma el estado "paciente de una organización tratado por un
profesional de otra", sin depender de que todos los servicios recuerden
verificarlo.

**La solicitud de receta referencia el vínculo, no al paciente y al
profesional por separado.** El diagrama dibuja dos claves foráneas
independientes. Se optó por una única clave foránea compuesta contra la
tabla de vínculo, que resuelve simultáneamente tres cuestiones que las dos
flechas dejaban abiertas: que ambas filas pertenezcan a una organización,
que pertenezcan a la misma, y que el paciente esté efectivamente en
tratamiento con el profesional a quien solicita la receta, que es el único
caso descripto en los requisitos. Como consecuencia, la solicitud no necesita
identificador de organización propio: el de su vínculo lo determina.

**El consentimiento es una tabla de solo agregado.** Se modeló sin columna
de última modificación y con marca temporal propia distinta de la de
creación de la fila. Una revocación se registra como una fila nueva y no
como la edición de la anterior, de modo que el historial de qué se aceptó y
cuándo permanece íntegro; la distinción entre el momento del consentimiento
y el momento del registro permite cargar consentimientos firmados en papel
sin falsear la fecha del acto.

> Nota posterior (TASK-30): la clínica estableció que no existe consentimiento
> en papel —todo paciente, ya cargado o nuevo, lo presta por el asistente
> conversacional—, de modo que ambas marcas temporales sólo podían contener el
> mismo valor. En esa tarea se eliminó mediante migración la marca con nombre de
> dominio propio y se conservó la de creación de la fila, que es la convención
> del resto del esquema; el carácter de solo agregado de la tabla se mantiene sin
> cambios. Ver `FASE-2_PROMPT-4_BACKEND.md`.

**La referencia del paciente en la traza de auditoría restringe el
borrado.** Una clave foránea compuesta no admite anular la referencia al
eliminar la fila referida, porque anularía también el identificador de
organización, que es no nulable; y propagar el borrado destruiría la traza
que documenta lo actuado sobre el paciente. Se optó por restringir, con la
consecuencia deliberada de que un paciente con historial de auditoría no
puede eliminarse físicamente. Esa consecuencia es la forma correcta para
datos de salud bajo la Ley 25.326: un pedido de supresión se atiende
anonimizando el registro del paciente, mientras la traza que acredita que la
supresión ocurrió debe sobrevivirle.

**Campos opcionales por realidad del dato, no por comodidad.** Sólo el
documento, el nombre y el apellido son obligatorios. La clave tributaria, la
fecha de nacimiento y el celular se declararon opcionales porque la tarea de
migración de pacientes preexistentes (P2.6) importará registros que hoy
viven en planillas de cálculo y en papel, donde esos datos suelen faltar. La
exigencia de documento, fecha de nacimiento y contacto de emergencia que
fijan los requisitos es una regla del flujo de reserva, no una razón para
rechazar un registro histórico incompleto. La edad, que los requisitos
enumeran entre los datos filiatorios, no se almacena: se deriva de la fecha
de nacimiento, de modo que no puede quedar desactualizada.

**Fechas de calendario frente a instantes.** La fecha de nacimiento y la de
última consulta se modelaron como fechas de calendario, sin hora ni huso
horario, siguiendo la convención ya adoptada para las ausencias: son días tal
como los lee la clínica, y adjuntarles un instante desplaza el día
representado en una zona UTC-3. El consentimiento y la solicitud de receta,
en cambio, se modelaron como instantes, porque la norma se interesa por el
momento del consentimiento y porque los requisitos restringen el pedido de
recetas al horario laboral, donde la hora del día forma parte del hecho.

## Alternativas descartadas

- **Omitir el identificador de organización también en el vínculo
  paciente-profesional**, por adhesión literal a la regla de normalización:
  descartada porque dejaría el cruce entre organizaciones impedido
  únicamente por la capa de servicios, y porque el argumento que sustenta la
  regla —una copia que puede divergir del padre sin que ninguna consulta lo
  detecte— no aplica cuando dos claves foráneas compuestas la verifican en
  cada escritura.
- **Modelar la solicitud de receta con dos claves foráneas independientes**,
  como las dibuja el diagrama: descartada por dejar sin garantizar la
  coherencia de organización entre ambas y por admitir solicitudes a un
  profesional que no trata al paciente.
- **Anular la referencia al paciente en la traza de auditoría al
  eliminarlo**: descartada porque la clave foránea compuesta no lo permite y
  porque desvincular la traza del sujeto contradice su finalidad probatoria.
- **Reponer un catálogo de diagnósticos**, que los requisitos mencionan
  entre las tablas auxiliares del módulo: descartada por decisión previa y
  vigente del proyecto —el diagrama entidad-relación no lo incluye y los
  datos clínicos quedan fuera del alcance del sistema—; se dejó constancia
  del punto en un comentario del esquema para que la mención de los
  requisitos no se reinterprete en tareas futuras.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: se agregaron los modelos `Patient`,
  `PatientProfessional`, `Consent` y `PrescriptionRequest`, los enumerados
  `PatientType` y `PrescriptionRequestStatus`, y la relación real de la
  traza de auditoría hacia el paciente; se documentó además la regla de
  tenencia para entidades con dos padres acotados por organización.
- Migración nueva `prisma/migrations/20260721034219_add_patient_entities/`:
  creación de los dos tipos enumerados y las cuatro tablas con sus índices y
  claves foráneas, y conversión de la referencia al paciente de la traza de
  auditoría en clave foránea compuesta, mediante renombrado de la columna
  —no borrado y recreación— para no perder las referencias ya registradas.
- `src/audit/audit.service.ts`: renombrado del parámetro de referencia al
  paciente, en línea con el renombrado de la columna.
- `CLAUDE.md` del repositorio backend: se incorporó la cuarta regla de
  diseño de esquema (entidades con dos padres acotados por organización) y
  el tercer patrón de multi-tenancy correspondiente.

No se tocaron puertos ni adaptadores: la tarea es exclusivamente de modelado
de datos.

## Tests y qué validan

- `test/patients-entities.e2e-spec.ts` (nuevo): valida contra la instancia
  local de PostgreSQL que cada garantía declarada en el esquema es
  efectivamente una restricción de la base de datos.
  - Alta de un paciente con obra social opcional y lectura de vuelta, con
    verificación de que la fecha de calendario no se desplaza.
  - Alta de un paciente sólo con los datos de identidad, como hará el
    importador de pacientes preexistentes.
  - Rechazo de un segundo paciente con el mismo documento en la misma
    organización, y admisión del mismo documento en otra organización.
  - Vinculación de un paciente con dos profesionales distintos con
    observaciones, prioridad y tipo independientes, y verificación de que los
    valores por defecto de un vínculo no dependen del otro.
  - Rechazo de un vínculo duplicado y rechazo del vínculo entre un paciente
    y un profesional de otra organización, reclamando la organización de
    cualquiera de los dos lados.
  - Historial de consentimientos de solo agregado: la revocación es el estado
    vigente y la aceptación original permanece registrada.
  - Alta de una solicitud de receta contra un vínculo existente, y rechazo de
    una solicitud dirigida a un profesional que no trata al paciente.
  - Propagación en cascada del borrado del paciente sobre su vínculo, sus
    consentimientos y sus solicitudes.
  - Rechazo de la consulta de pacientes sin organización en contexto, y
    estampado y aislamiento por organización del paciente y del vínculo.
- `test/audit.e2e-spec.ts` (modificado): la prueba de la referencia al
  paciente pasó a usar un paciente real, ya que la clave foránea rechaza un
  identificador arbitrario, y se agregó una prueba de que una entrada no
  puede apuntar a un paciente de otra organización.
- `src/audit/audit.service.spec.ts` (modificado): renombrado del campo.
- Ejecución: suite unitaria en verde (10 suites / 50 pruebas) y suite
  end-to-end en verde (15 suites / 131 pruebas) tras aplicar la migración.
  La comparación entre el historial de migraciones y el esquema no reporta
  diferencias. Los datos usados en las pruebas son ficticios (nombres de
  fantasía y números de documento de ejemplo), y las observaciones de uso
  interno se representan con textos genéricos sin contenido clínico.

## Figuras pendientes

- Se registró una figura pendiente con el diagrama entidad-relación acotado
  al subdominio Pacientes, incluyendo la clave foránea compuesta de la
  solicitud de receta hacia el vínculo (ver `figuras_pendientes.md`).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-27-patient-entities` (creada a partir
  de `main`). Sin commit al momento de redactar esta entrada: los cambios
  quedaron en el árbol de trabajo a la espera de autorización.
- Ticket: TASK-27 ("P2.1 – Entidades de Paciente y catálogos"). Depende de
  TASK-14 (Prisma), TASK-15 (multi-tenancy) y TASK-21 (entidades de
  Profesional), todas ya fusionadas.
