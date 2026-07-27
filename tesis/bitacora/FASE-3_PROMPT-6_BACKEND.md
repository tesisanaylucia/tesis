# Fase 3 — Motor de Turnos (backend) — reprogramación, reorganización y reprogramación masiva por ausencia (TASK-39)

## Qué se implementó

Se implementaron los tres flujos de reprogramación que fija el documento de
especificación de requisitos, módulo Turnos, secciones "Reprogramación de
turno", "Reorganización de agenda" y "Reprogramación masiva por ausencia
del profesional": la reprogramación individual de un turno por su propio
identificador, la reorganización manual de varios turnos de una misma
agenda en una sola operación, y la cancelación automática de todos los
turnos afectados cuando se registra una ausencia del profesional, con
auditoría en los tres casos.

- `PATCH /turnos/:id/reprogramar` mueve un turno reservado o confirmado a
  un nuevo instante, conservando su identificador: valida que el nuevo
  slot esté libre (a través del mismo servicio de disponibilidad que ya
  usa la reserva), que la nueva fecha sea futura, y, si el turno es una
  primera sesión, que el profesional todavía acepte pacientes nuevos y
  que la restricción de solo adultos siga cumpliéndose.
- `POST /profesionales/:id/reorganizar-agenda` permite que el profesional
  mueva varios turnos propios en un mismo pedido; cada movimiento se
  valida y aplica de forma independiente, y un movimiento que falla se
  informa en la respuesta sin abortar los que sí se pudieron aplicar.
- El registro de una ausencia (`POST /profesionales/:id/ausencias`, ya
  existente desde P1.3) ahora dispara, a través del punto de extensión que
  esa tarea había dejado preparado, la cancelación de todo turno reservado
  o confirmado del profesional dentro del período de la ausencia, marcado
  con un nuevo motivo de cancelación específico, y el disparo de un evento
  de reasignación por cada turno cancelado.

Las tres operaciones delegan la notificación al paciente al puerto de
mensajería ya existente (`MessagingPort`), que en esta tarea sigue siendo
el adaptador *stub*: el contenido real de los mensajes y la integración
con WhatsApp quedan fuera de alcance, atribuidos por el propio documento
de requisitos a una tarea de un módulo posterior.

## Decisiones y por qué

**El punto de extensión `AbsenceEventsPort`, definido en P1.3 sin
consumidor real, obtuvo aquí su primer consumidor real.** El servicio de
turnos implementa el puerto en un adaptador propio
(`AppointmentAbsenceEventsAdapter`) que, al recibir el evento de ausencia
registrada, ejecuta la cancelación masiva. Esto obligó a reorganizar el
grafo de módulos: el servicio de ausencias vivía dentro del módulo de
Profesionales, que a su vez es una dependencia del módulo de Turnos (este
último necesita verificar la pertenencia del profesional); si el módulo de
Turnos hubiera pasado a proveer el puerto directamente dentro del módulo
de Profesionales, se habría formado un ciclo de importación entre ambos
módulos. La solución fue extraer el servicio y el controlador de
ausencias a un módulo propio (`AbsencesModule`), ubicado por encima de los
otros dos: importa el módulo de Profesionales para la verificación de
pertenencia y el módulo de Turnos para el token del puerto, sin que
ninguno de los dos necesite importarlo de vuelta.

**Se definió un segundo puerto de dominio, `ReassignmentPort`, con el
mismo criterio que ya había fijado `AbsenceEventsPort`.** El documento de
requisitos encomienda el algoritmo de reasignación en sí a una tarea
posterior (P3.7), que decide a qué paciente de la lista de espera ofrecer
el turno liberado; esta tarea solo necesita señalar que un turno quedó
disponible para reasignación. En lugar de reutilizar el puerto de eventos
de ausencia para ese aviso —que mezclaría dos preocupaciones distintas, el
registro de una ausencia y la liberación de un turno concreto—, se
introdujo un puerto separado y enfocado, publicado una vez por cada turno
que la cancelación masiva efectivamente cancela, con un adaptador *stub*
que solo registra la emisión, siguiendo el mismo patrón que el resto de
los puertos de integración del sistema.

**La cancelación masiva no se protegió con la anticipación mínima que sí
exige la cancelación ordinaria (`PATCH /turnos/:id/cancelar`).** Esa regla
existe para que un paciente o un profesional no cancele a último momento
por decisión propia; aquí la cancelación no es una decisión discrecional
sino la consecuencia obligada de que el profesional dejó de estar
disponible, motivo que el propio documento de requisitos distingue con su
propio valor de motivo de cancelación.

**Cada turno dentro del rango de la ausencia se cancela en su propia
transacción, con su propio bloque de captura de errores, en lugar de
cancelar todo el lote dentro de una única transacción.** La cancelación
masiva se ejecuta sincrónicamente dentro del mismo pedido HTTP que registra
la ausencia, antes de que ese pedido responda; si un conflicto de
concurrencia sobre un turno cualquiera (alguien lo confirmó o canceló en
el mismo instante) abortara todo el lote, un problema ajeno a la ausencia
en sí convertiría el registro de una ausencia —ya persistido y auditado—
en una respuesta de error, dejando además sin cancelar turnos que sí
podían cancelarse. Cada fallo se registra en el log de la aplicación en
lugar de interrumpir el resto del lote.

**Se agregó un nuevo motivo de cancelación (`CancellationReason`,
inicialmente con un único valor, `PROFESSIONAL_ABSENCE`) como columna
anulable del turno, en lugar de inferir la ausencia por otro medio.** El
documento de requisitos pide explícitamente que el turno cancelado por
esta vía quede marcado con ese motivo; una cancelación ordinaria no
escribe ningún valor en esa columna, que queda nula.

**La reprogramación individual y la reorganización manual comparten la
misma operación interna** (validaciones, transacción serializable y
auditoría), variando solo en cómo se resuelve la autorización de cada
turno: la primera confirma que el llamador es administrador, el proceso
automatizado o el profesional dueño del turno; la segunda ya resolvió esa
autorización a nivel de toda la agenda a través del guard de propiedad
existente, y solo necesita además confirmar que cada turno del lote
pertenece efectivamente al profesional de la URL, un turno de otro
profesional se informa como un movimiento fallido más, sin distinguirlo de
una falla de disponibilidad.

**La reorganización manual no se ejecuta como una única transacción de
base de datos que abarque todo el lote.** El propio documento de
requisitos pide que un error parcial no aborte los movimientos exitosos,
lo que exige que cada movimiento pueda persistir de forma independiente
del resultado de los demás.

## Alternativas descartadas

- **Revalidar en la reprogramación la colocación completa de la franja
  extra de primera sesión (par de turnos consecutivos según la modalidad
  configurada)**: descartada porque el documento de requisitos describe la
  reprogramación como el movimiento de un turno por su propio
  identificador, sin mencionar la re-vinculación con el turno que lo
  acompañó en la reserva original; se revalidan en cambio los dos
  interruptores de configuración que si podrían haber cambiado desde la
  reserva (aceptación de pacientes nuevos, restricción de solo adultos).
- **Restaurar automáticamente los turnos cancelados cuando se cancela la
  ausencia que los canceló** (`AbsenceCancelledEvent`, ya existente):
  descartada porque el documento de requisitos no describe ese flujo, y
  porque el slot original de cada turno puede ya no estar libre para el
  momento en que se cancela la ausencia; la cancelación de la ausencia
  queda registrada en el log de la aplicación, y la restauración, de
  requerirse, queda como una decisión manual del lado de la clínica.
- **Reutilizar `AbsenceEventsPort` para señalar también la liberación de
  un turno destinado a reasignación**: descartada por la razón ya
  expuesta en la sección de decisiones — mezclaría dos eventos de dominio
  distintos bajo un mismo contrato.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma` (modificado): enumerado `CancellationReason`
  (`PROFESSIONAL_ABSENCE`) y columna anulable `cancellationReason` en
  `Appointment`; migración
  `prisma/migrations/20260727193810_add_appointment_cancellation_reason/`.
- `src/domain/ports/reassignment.port.ts` (nuevo): puerto
  `ReassignmentPort` y el evento `AppointmentCancelledEvent`.
- `src/infrastructure/adapters/logging-reassignment.adapter.ts` (nuevo):
  adaptador *stub* del puerto anterior, análogo al ya existente para el
  puerto de eventos de ausencia.
- `src/infrastructure/adapters/logging-absence-events.adapter.ts`
  (eliminado): reemplazado como implementación real por
  `AppointmentAbsenceEventsAdapter`.
- `src/domain/ports/absence-events.port.ts` (modificado): se agregó
  `actorId` a `AbsenceRegisteredEvent`, para que la cancelación masiva
  pueda auditar con el mismo actor que registró la ausencia.
- `src/appointments/absence-events.adapter.ts` (nuevo):
  `AppointmentAbsenceEventsAdapter`, implementación real de
  `AbsenceEventsPort` que dispara `AppointmentsService.cancelForAbsence`.
- `src/appointments/appointments.service.ts` (modificado): se agregaron
  `reschedule`, `reorganizeAgenda`, `cancelForAbsence` y los métodos
  privados de apoyo (`rescheduleCore`, `assertNewPatientRulesStillApply`,
  `notifyPatient`).
- `src/appointments/appointments.controller.ts` (modificado): se agregó
  `PATCH /turnos/:id/reprogramar`.
- `src/appointments/agenda.controller.ts` (nuevo):
  `POST /profesionales/:id/reorganizar-agenda`.
- `src/appointments/dto/reschedule-appointment.dto.ts` y
  `dto/reorganize-agenda.dto.ts` (nuevos).
- `src/appointments/appointment.presenter.ts` (modificado): se agregó
  `cancellationReason` a la selección y a la respuesta del turno, y el
  tipo de respuesta de la reorganización de agenda.
- `src/appointments/appointments.module.ts` (modificado): pasa a proveer
  `ABSENCE_EVENTS_PORT` (con el adaptador real) y a importar
  `IntegrationsModule` para los puertos de mensajería y de reasignación.
- `src/professionals/absences.module.ts` (nuevo): módulo propio para el
  servicio y el controlador de ausencias, extraídos de
  `ProfessionalsModule` por la razón ya expuesta.
- `src/professionals/professionals.module.ts` y `src/app.module.ts`
  (modificados): ajuste de la composición de módulos descripta arriba.

## Tests y qué validan

- `src/appointments/appointments-rescheduling.service.spec.ts` (nuevo):
  reprogramación exitosa con auditoría del slot anterior y del nuevo,
  autorización del profesional dueño y rechazo del que no lo es, rechazo
  de un turno cancelado, de una fecha no futura, de un slot ya ocupado y
  de un conflicto de concurrencia, revalidación de las reglas de paciente
  nuevo, comportamiento sin operación cuando el nuevo instante coincide
  con el actual, notificación al paciente cuando tiene teléfono
  registrado; reorganización de agenda con éxito total, con un movimiento
  fallido que no aborta los demás, y con un turno que no pertenece al
  profesional de la URL; cancelación masiva por ausencia que marca el
  motivo y dispara la reasignación por cada turno, que no hace nada
  cuando ningún turno cae en el rango, y que continúa con el resto del
  lote cuando uno de los turnos entra en conflicto de concurrencia.
- `src/appointments/absence-events.adapter.spec.ts` (nuevo): el adaptador
  invoca `cancelForAbsence` con los campos del evento de ausencia
  registrada, y no falla ante una ausencia cancelada.
- `src/infrastructure/integrations.module.spec.ts` (modificado): la
  prueba de resolución de `AbsenceEventsPort` se reemplazó por la de
  `ReassignmentPort`, coherente con el traslado del primero fuera de este
  módulo.
- `test/professional-schedules.e2e-spec.ts` (modificado): la aserción
  sobre el contenido del evento de ausencia registrada ahora incluye
  `actorId`.
- `test/appointments-rescheduling.e2e-spec.ts` (nuevo, 12 pruebas): contra
  la instancia local de PostgreSQL, con los puertos de mensajería y de
  reasignación sustituidos por dobles para poder verificar su invocación.
  Reprogramación individual: movimiento exitoso con liberación
  verificable del slot anterior y auditoría del slot anterior y del
  nuevo, autorización del profesional dueño, del proceso automatizado y
  rechazo del que no es dueño, rechazo de un slot ya ocupado, de una
  fecha no futura y de un turno cancelado, aislamiento por inquilino
  (404). Reorganización de agenda: lote exitoso completo, lote con un
  movimiento fallido que no aborta el resto, rechazo al profesional que
  no es dueño de la agenda. Reprogramación masiva por ausencia: registro
  de una ausencia que cancela los turnos reservados y confirmados dentro
  del rango con el motivo correspondiente, deja sin tocar un turno fuera
  del rango, y dispara el puerto de reasignación una vez por cada turno
  cancelado.
- Ejecución: suite unitaria en verde (25 suites / 253 pruebas). Suite
  end-to-end completa en verde (25 suites / 317 pruebas) ejecutada en
  serie (`--runInBand`); en paralelo reaparece la misma interferencia
  entre archivos de prueba que comparten la base de datos de desarrollo ya
  señalada en entradas anteriores de esta fase, confirmada nuevamente como
  preexistente y no atribuible a esta tarea (los archivos afectados
  variaron entre corridas paralelas sucesivas y cada uno pasa sin fallas
  ejecutado de forma aislada). Los datos usados en las pruebas son
  ficticios.

## Figuras pendientes

Se agregaron dos figuras pendientes: el diagrama de secuencia de la
cancelación masiva por ausencia y la reorganización del grafo de módulos
que produjo (ver `figuras_pendientes.md`, entradas 21 y 22).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-39-appointment-reschedule-reorganization`
  (creada a partir de `main`, que ya tenía fusionadas las tres
  dependencias de esta tarea: TASK-35, TASK-38 y TASK-23).
- Ticket: TASK-39 ("P3.6 – Reprogramación, reorganización y reprogramación
  masiva"). Depende de TASK-35 (P3.2), TASK-38 (P3.5) y TASK-23 (P1.3).
