# Fase 3 — Motor de Turnos (backend) — entidades base del dominio (TASK-34)

## Qué se implementó

Se incorporaron al esquema de Prisma las tres entidades base del dominio
Turnos tomadas del diagrama entidad-relación (`modelo_base_de_datos.png`) y
del documento de especificación de requisitos, módulo Turnos: `Appointment`
(turno), `Holiday` (feriado) y `WaitlistEntry` (lista de espera). Se agregó
el enumerado `AppointmentStatus` con los cinco valores del estado del turno
(reservado, confirmado, cancelado, reasignado, completado), siguiendo la
convención del repositorio de nombrar modelos, columnas y valores de
enumerado en inglés.

La tarea abarca únicamente el esquema y su migración
(`20260723171019_add_appointment_holiday_waitlist`); no introduce lógica de
disponibilidad (P3.2), reserva (P3.3) ni las reglas de ubicación de la
franja extra de primera sesión (P3.4), que corresponden a tareas
posteriores de la misma fase.

Adicionalmente se saldó una deuda declarada en el propio esquema desde las
fases de Fundaciones y Pacientes: la columna que la traza de auditoría
reservaba para referenciar al turno era, desde su creación, un identificador
sin clave foránea, porque la tabla destino no existía todavía. La tarea la
convirtió en clave foránea compuesta real y, al dejar de ser un marcador de
posición de una tabla nombrada en español, la renombró a inglés como el
resto de las columnas — el mismo tratamiento que la referencia al paciente
recibió en TASK-27.

## Decisiones y por qué

**El turno y la lista de espera conservan el identificador de organización
por tener dos padres acotados por organización.** Ambas entidades enlazan un
paciente y un profesional, de modo que se les aplicó el mismo criterio que
la tarea de Pacientes fijó para el vínculo paciente-profesional: con un único
padre la columna es una réplica pura y se omite, pero con dos padres
acotados por organización esa única columna es lo que obliga a que ambos
pertenezcan a la misma organización, restricción que ningún par de claves
foráneas independientes puede expresar. Ambas entidades quedaron entonces con
`organizationId` propio y con claves foráneas compuestas hacia `Patient` y
hacia `Professional`, y en cascada con ambos: un turno o un lugar en la lista
de espera no tiene sentido sin el paciente o el profesional al que se
refiere, igual que el vínculo paciente-profesional.

**El feriado usa un identificador propio en lugar de la fecha como clave
primaria.** El diagrama declara la fecha como clave primaria de la entidad,
pero esa clave sólo es única *dentro* de una organización: dos organizaciones
observan legítimamente el mismo feriado nacional, cada una con su propia
fila. Se le dio entonces un identificador propio, como al resto de las
entidades del esquema, y se expresó la clave natural del diagrama como una
restricción de unicidad acotada por organización (organización, fecha) en
lugar de como clave primaria literal — el mismo razonamiento que ya se había
aplicado al documento del paciente en TASK-27.

**La duración y el indicador de primera sesión del turno se registran en el
turno, no se leen en vivo del profesional.** El profesional configura una
duración de consulta y, para pacientes nuevos, una franja extra (P1.4); pero
un turno reservado bajo una configuración debe conservar esa configuración
aunque el profesional la cambie después. Se registró la duración como un
campo propio del turno, poblado al momento de la reserva, y no como un
valor derivado de `Professional.consultationDuration` en cada lectura. El
mismo razonamiento se aplicó al indicador de primera sesión: se copia desde
`PatientProfessional.firstSession` al reservar, en lugar de leerse en vivo
del vínculo, para que el turno describa el hecho tal como era al momento de
reservarlo.

**El estado "reasignado" es transitorio y no crea el nuevo turno.** El
documento de requisitos aclara que el estado marca el turno original como
superado cuando su franja se entrega a otro paciente, pero que la creación
del turno que lo reemplaza es responsabilidad del algoritmo de reasignación
(M3/M4), no de esta entidad. El enumerado se limitó a declarar el estado; no
se agregó ninguna relación ni lógica que vincule un turno reasignado con su
reemplazo, porque esa relación no la sostiene una fila sino un algoritmo
todavía no implementado.

**El campo de referencia (`trajo_orden`) se registra para todo turno, no
sólo para los de obra social.** El documento de requisitos lo describe en el
contexto de la obra social, pero no condiciona su registro al tipo de
atención del profesional; se dejó como un campo booleano simple del turno,
sin una regla condicional que dependa de `Professional.careType`, en línea
con el principio del repositorio de tratar las reglas de negocio como datos
y no como condicionales dispersos — si el requisito llega a acotarse a un
tipo de atención, esa validación pertenece a la capa de servicio de reserva
(P3.3), no al esquema.

**La referencia del turno en la traza de auditoría restringe el borrado, no
lo propaga.** Se replicó el mismo criterio que ya rige la referencia al
paciente: una clave foránea compuesta no admite anular la referencia al
eliminar la fila referida, porque anularía también el identificador de
organización, que es no nulable; y propagar el borrado destruiría la traza
que documenta lo actuado sobre el turno. Se restringió el borrado, con la
consecuencia deliberada de que un turno con historial de auditoría no puede
eliminarse físicamente — coherente con que el turno en sí, hacia sus propios
padres (paciente y profesional), sí cascada, porque en ese caso lo que se
elimina es el padre, no el turno mismo intentando borrarse.

**El renombrado de la columna de la traza de auditoría se hizo con
`RENAME COLUMN`, no con borrado y recreación.** Prisma generó por defecto una
migración que eliminaba la columna en español y agregaba una nueva en
inglés, lo que habría perdido cualquier referencia ya registrada. Se
reescribió a mano siguiendo el patrón ya establecido en TASK-27 para el
paciente: renombrar la columna y su índice, limpiar los valores existentes
(la columna no tenía restricción hasta esta migración, de modo que cualquier
valor previo podía apuntar a un turno inexistente) y recién entonces agregar
la clave foránea compuesta.

**La referencia equivalente en el registro operativo de la cerradura
(`LockLog.turnoId`) se dejó sin tocar.** El esquema documenta esa columna
como una referencia futura conjunta con el código de acceso
(`accessCodeId`), a resolverse cuando ambas tablas existan — porque el
registro operativo sólo puede resolver la organización de un evento uniendo
por el turno, y esa unión no aporta nada mientras el código de acceso siga
sin existir. Se actualizó el comentario del esquema para nombrar la entidad
real (`Appointment`) en lugar del marcador anterior, pero se dejó la columna
sin renombrar ni conectar, a la espera de la tarea del código de acceso
(M6/P6.1, TASK-55).

## Alternativas descartadas

- **Omitir el identificador de organización en `Appointment` y
  `WaitlistEntry`**, por adhesión literal a la regla general de
  normalización: descartada por el mismo argumento que en TASK-27 para el
  vínculo paciente-profesional — dejaría el cruce entre organizaciones
  impedido únicamente por la capa de servicios.
- **Conectar de inmediato `LockLog.turnoId` hacia `Appointment` ahora que la
  entidad existe**, dejando pendiente sólo `accessCodeId`: descartada porque
  el registro operativo seguiría sin tener una vía de lectura por
  organización mientras falte el código de acceso, de modo que conectar una
  sola de las dos referencias no habilita ningún caso de uso nuevo y adelanta
  un cambio de esquema sin beneficio verificable en esta tarea.
- **Condicionar el registro de `trajo_orden` al tipo de atención del
  profesional dentro del esquema**: descartada por corresponder a una regla
  de negocio de la reserva (P3.3), no a una restricción de la base de datos.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: se agregaron los modelos `Appointment`, `Holiday` y
  `WaitlistEntry`, el enumerado `AppointmentStatus`, y la relación real de la
  traza de auditoría hacia el turno (`AuditLog.appointmentId`); se actualizó
  el comentario transicional de `LockLog` para reflejar que `Appointment` ya
  existe mientras `accessCodeId` sigue pendiente.
- Migración nueva
  `prisma/migrations/20260723171019_add_appointment_holiday_waitlist/`:
  creación del enumerado y las tres tablas con sus índices y claves foráneas,
  y conversión de la referencia al turno de la traza de auditoría en clave
  foránea compuesta mediante renombrado de columna — no borrado y
  recreación — para no perder las referencias ya registradas.
- `src/audit/audit.service.ts`: renombrado del parámetro de referencia al
  turno, en línea con el renombrado de la columna.
- `CLAUDE.md` del repositorio backend: se actualizaron los ejemplos de las
  reglas de integridad, claves foráneas compuestas y los tres patrones de
  multi-tenancy para incluir `Appointment`, `Holiday` y `WaitlistEntry`, y se
  precisó la nota transicional sobre `LockLog`.

No se tocaron puertos ni adaptadores: la tarea es exclusivamente de modelado
de datos.

## Tests y qué validan

- `test/appointments-entities.e2e-spec.ts` (nuevo): valida contra la
  instancia local de PostgreSQL que cada garantía declarada en el esquema es
  efectivamente una restricción de la base de datos.
  - Alta de un turno con sus valores por defecto (reservado, sin primera
    sesión, sin pago, sin orden, sin confirmación) y lectura de vuelta.
  - Alta de un turno de primera sesión con todos los campos explícitos.
  - Rechazo de un turno que vincula un paciente o un profesional de otra
    organización.
  - Admisión del mismo feriado (misma fecha) en dos organizaciones distintas,
    y rechazo de un segundo feriado con la misma fecha en la misma
    organización.
  - Alta de un lugar en la lista de espera con obra social opcional y sin
    ella, y rechazo de una lista de espera que vincula un paciente o un
    profesional de otra organización.
  - Rechazo de la consulta de turnos sin organización en contexto, y
    estampado y aislamiento por organización del turno, el feriado y la
    lista de espera — el criterio de aceptación explícito del ticket.
- `test/audit.e2e-spec.ts` (modificado): la prueba de la referencia al turno
  pasó a crear un turno real (profesional, paciente y turno), ya que la
  clave foránea rechaza ahora un identificador arbitrario.
- `src/audit/audit.service.spec.ts` (modificado): renombrado del campo y de
  los nombres de entidad de ejemplo (`Turno` → `Appointment`).
- Ejecución: suite unitaria en verde (20 suites / 148 pruebas) y suite
  end-to-end en verde (21 suites / 244 pruebas) tras aplicar la migración.
  Los datos usados en las pruebas son ficticios (nombres de fantasía y
  números de documento de ejemplo).

## Figuras pendientes

- Se registró una figura pendiente con el diagrama entidad-relación acotado
  al subdominio Turnos, incluyendo las claves foráneas compuestas del turno y
  de la lista de espera hacia paciente y profesional (ver
  `figuras_pendientes.md`).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-34-appointment-holiday-waitlist`
  (creada a partir de `main`), pusheada a `origin`. Commit
  `40f4d57` al momento de redactar esta entrada.
- Ticket: TASK-34 ("P3.1 – Entidades de Turno, Feriado y Lista de espera").
  Depende de TASK-14 (Prisma), TASK-15 (multi-tenancy), TASK-21 (entidades de
  Profesional) y TASK-27 (entidades de Paciente), todas ya fusionadas.
