# Fase 3 — Motor de Turnos (backend) — reserva de turno (TASK-36)

## Qué se implementó

Se implementó la reserva de turno (`AppointmentsService.book`,
`src/appointments/`) junto con el endpoint `POST /turnos` que la expone,
siguiendo el procedimiento del documento de especificación de requisitos,
módulo Turnos, sección "Reserva de turno". El servicio aplica, en el orden
que fija el documento, las cuatro validaciones previas a la creación del
turno:

1. Consentimiento: se verifica mediante `ConsentsService.verify` (P2.4) que
   el paciente tiene una aceptación registrada; si no la tiene, se rechaza
   el pedido con una instrucción explícita de registrar el consentimiento
   primero.
2. Datos obligatorios del paciente: se comprueban `birthDate` y
   `emergencyContact`. El DNI no se verifica porque la columna es no
   anulable en el esquema, de modo que todo paciente alcanzable ya lo tiene.
3. Slot libre: se verifica a través de un nuevo método del servicio de
   disponibilidad de TASK-35, `AvailabilityService.isSlotFree`, que
   consulta si existe un turno en estado `RESERVED` o `CONFIRMED` para ese
   profesional en ese instante exacto — la misma definición de "ocupado"
   que ya usa el cálculo de franjas de P3.2, para que ambos no puedan
   discrepar sobre qué significa "libre".
4. No simultaneidad: se verifica que el paciente no tenga ya un turno
   `RESERVED` o `CONFIRMED` en el mismo instante, con otro profesional.

Superadas las cuatro, el turno se crea en estado `RESERVED` (el valor por
defecto de la columna), con la duración copiada de
`Professional.consultationDuration` y el indicador de primera sesión
copiado de `PatientProfessional.firstSession` del vínculo entre ese
paciente y ese profesional — el mismo patrón de copia al momento de la
reserva que TASK-34 ya había fijado para ambos campos al modelar la
entidad. Si el vínculo paciente-profesional todavía no existe (la reserva
es, por definición, la primera vez que se registra la relación), se crea
en el momento con `PatientProfessionalsService.ensureLinked`, que deja
`firstSession` en su valor por defecto (verdadero). Por último se registra
una entrada en la traza de auditoría con `accion=RESERVE`, el identificador
del turno y el del paciente.

El endpoint es accesible por rol `ADMIN` o `SYSTEM` (el chatbot), conforme
al ticket; el rol `PROFESSIONAL` queda fuera de este flujo de reserva.

## Decisiones y por qué

**El slot libre y la no simultaneidad se verifican dentro de una
transacción `SERIALIZABLE`, junto con la escritura que reserva el turno;
el consentimiento y los datos obligatorios se verifican antes, fuera de
ella.** Las dos primeras son invariantes de lectura-y-escritura: dos
pedidos concurrentes sobre el mismo instante podrían leer ambos "libre" y
escribir ambos, dejando un estado que ninguno de los dos habría permitido
si hubiera visto al otro — la misma clase de problema que ya documenta
`CLAUDE.md` para el tope de matrículas y el reemplazo del horario semanal,
y que el repositorio resuelve con `runSerializable`. El consentimiento y
los datos obligatorios, en cambio, son lecturas de precondición sin nada
con qué competir en la escala de tiempo de un pedido HTTP, de modo que se
resuelven antes de abrir la transacción, sin pagar su costo para algo que
no lo necesita.

**La verificación de slot libre se agregó como método nuevo del servicio de
disponibilidad de P3.2 (`isSlotFree`) en lugar de una consulta propia del
servicio de reserva.** El documento de requisitos pide explícitamente que
la reserva verifique el slot "por" el servicio de disponibilidad, no que
reimplemente el criterio de qué cuenta como "ocupado". El método nuevo
acepta opcionalmente el cliente de la transacción del llamador —el mismo
patrón que ya usa `ConsentsService.latest`— para poder ejecutarse dentro de
la transacción serializable de la reserva sin que el servicio de
disponibilidad conozca nada de transacciones más allá de aceptar el cliente
que se le pasa.

**El vínculo paciente-profesional se crea, no se exige que exista, cuando
falta al momento de la reserva.** El documento de requisitos pide
"consultar" `PACIENTE_PROFESIONAL.primera_sesion` para determinar si es la
primera sesión, pero no dice qué hacer si el vínculo todavía no existe. Se
interpretó que la primera reserva de un paciente con un profesional es, por
definición, su primera sesión con él, de modo que la ausencia del vínculo
es en sí misma la respuesta a la pregunta que el sistema debe validar
("¿es la primera vez que el paciente se atiende con ese profesional?"), y
no un caso de error. Se usó `PatientProfessionalsService.ensureLinked` en
lugar de `.link()` porque ya se había verificado la pertenencia del
paciente y del profesional al inquilino antes de entrar a la transacción,
y `ensureLinked` es precisamente el método pensado para un llamador que ya
estableció esa garantía y necesita hacer la operación dentro de su propia
transacción.

**El motivo de la traza de auditoría se registró como `RESERVE` y no como
el `RESERVA` en español que usa el propio ticket.** El campo es texto
libre, pensado para nombrar tipos de acción de cumplimiento normativo que
no encajan en `CREATE`/`UPDATE`/`DELETE` — el mismo mecanismo que ya usan
`Consent` y `PatientProfessional` con sus propios valores. Se aplicó el
mismo criterio que ya se había fijado en TASK-35 para los nombres de
parámetros de consulta: el vocabulario en español del ticket describe la
acción en los términos del documento de requisitos, no exige literalmente
esa cadena en un contrato que en el resto del sistema es íntegramente en
inglés salvo la terminología clínica propia del dominio.

**La duración de la sesión no es un parámetro del pedido.** Se calcula a
partir de `Professional.consultationDuration` en el momento de la reserva,
con el mismo rechazo explícito (pedido mal formado) que ya usa el cálculo
de disponibilidad cuando el profesional no la tiene configurada — el mismo
razonamiento documentado en la entrada de TASK-35: sin ese valor no hay con
qué poblar la columna, y devolver un turno con una duración arbitraria
dictada por el cliente permitiría a un llamador reservar una sesión de
cualquier longitud.

## Alternativas descartadas

- **Exigir que el vínculo paciente-profesional ya exista antes de reservar**,
  devolviendo un error si falta: descartada porque el propio flujo de
  reserva es, en la práctica, el primer contacto formal entre un paciente y
  un profesional nuevo para él, y exigir un paso previo de vinculación
  explícita no está en el alcance funcional que describe el ticket.
- **Reimplementar la consulta de "turno reservado o confirmado en ese
  instante" directamente en el servicio de reserva**, en lugar de agregar
  un método al servicio de disponibilidad: descartada por el riesgo de que
  ambas consultas definan "ocupado" de manera distinta con el tiempo, y
  porque el propio ticket pide la verificación "por" el servicio de
  disponibilidad.
- **Verificar consentimiento y datos obligatorios también dentro de la
  transacción serializable**, por uniformidad con las otras dos
  validaciones: descartada porque ninguna de las dos es una invariante de
  lectura-y-escritura — no hay una escritura concurrente que pueda dejarlas
  en un estado inconsistente — y envolverlas habría alargado la transacción
  sin necesidad.

## Entidades / puertos / adaptadores tocados

- `src/appointments/appointments.service.ts` (nuevo): `AppointmentsService`
  con el método `book`.
- `src/appointments/appointments.controller.ts` (nuevo): controlador de
  `POST /turnos`.
- `src/appointments/appointments.module.ts` (nuevo): módulo que importa
  `PatientsModule`, `ProfessionalsModule` y `AvailabilityModule`.
- `src/appointments/appointment.presenter.ts` (nuevo): forma de la
  respuesta JSON del turno reservado.
- `src/appointments/dto/create-appointment.dto.ts` (nuevo): validación del
  cuerpo del pedido (`patientId`, `professionalId`, `scheduledAt`).
- `src/availability/availability.service.ts` (modificado): se agregó
  `isSlotFree`, el método que la reserva usa para verificar el slot.
- `src/app.module.ts` (modificado): registro de `AppointmentsModule`.

No se tocó el esquema de Prisma ni se agregaron puertos o adaptadores
nuevos: la tarea es exclusivamente de lógica de servicio sobre entidades ya
existentes.

## Tests y qué validan

- `src/appointments/appointments.service.spec.ts` (nuevo): el cliente de
  Prisma y los colaboradores (`PatientsService`, `ProfessionalsService`,
  `ConsentsService`, `PatientProfessionalsService`, `AvailabilityService`,
  `AuditService`) se simulan en memoria.
  - Creación de un turno `RESERVED` con la duración y el indicador de
    primera sesión calculados correctamente, en ambos valores del
    indicador.
  - Registro de la auditoría con `accion=RESERVE`, dentro de la
    transacción.
  - Rechazo cuando falta el consentimiento, cuando falta un dato
    obligatorio (`birthDate` o `emergencyContact`), cuando el profesional
    no tiene `consultationDuration` configurada, cuando el slot ya está
    ocupado y cuando el paciente ya tiene un turno simultáneo — los cuatro
    casos de error del ticket más el caso de configuración faltante ya
    cubierto por P3.2.
- `src/availability/availability.service.spec.ts` (ampliado): cobertura
  nueva para `isSlotFree` — verdadero sin ocupación, falso con un turno
  `RESERVED`/`CONFIRMED` en el instante, y verificación de que consulta a
  través del cliente que se le pasa (para poder ejecutarse dentro de una
  transacción ajena).
- `test/appointments-booking.e2e-spec.ts` (nuevo, 15 pruebas): contra la
  instancia local de PostgreSQL.
  - Rechazo del acceso sin autenticar (401) y del rol `PROFESSIONAL` (403).
  - Reserva exitosa como `ADMIN` y como `SYSTEM` (el chatbot), con el turno
    marcado como primera sesión y la entrada de auditoría correspondiente.
  - Los cuatro casos de error del ticket: sin consentimiento, con un dato
    obligatorio faltante, con el slot ya ocupado por otro paciente, y con
    el mismo paciente ya reservado a la misma hora con otro profesional.
  - Un turno cancelado en el mismo slot no impide una nueva reserva sobre
    él.
  - Rechazo cuando el profesional no tiene duración de consulta
    configurada.
  - Aislamiento por inquilino: 404 al reservar con un paciente o un
    profesional de otra organización.
  - Validación del cuerpo del pedido: identificador faltante o mal
    formado, y fecha sin componente de hora.
- Ejecución: suite unitaria en verde (22 suites / 171 pruebas) y suite
  end-to-end en verde para el nuevo archivo (15/15); la corrida completa de
  la suite end-to-end mostró fallas en `test/appointments-entities.e2e-spec.ts`
  únicamente al ejecutarse junto al resto de la suite (no de forma aislada,
  donde pasa igual que antes), atribuibles a interferencia entre archivos
  de prueba que comparten la base de datos de desarrollo y preexistentes a
  esta tarea — no se modificó ese archivo ni las entidades que usa. Los
  datos usados en las pruebas son ficticios (nombres de fantasía y
  documentos de ejemplo).

## Figuras pendientes

Se agregó una figura pendiente con el diagrama de secuencia de la reserva
de turno (ver `figuras_pendientes.md`, entrada 18).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-36-appointment-booking` (creada a
  partir de `main`, con TASK-34 y TASK-35 ya fusionados). Pusheada a
  `origin`. Commit `9b033c2` al momento de redactar esta entrada.
- Ticket: TASK-36 ("P3.3 – Reserva de turno"). Depende de TASK-34 (P3.1),
  TASK-35 (P3.2), TASK-29 (tipo de paciente), TASK-30 (consentimiento) y
  TASK-17 (auditoría), todas ya fusionadas a `main`.
