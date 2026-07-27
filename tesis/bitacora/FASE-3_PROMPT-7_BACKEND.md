# Fase 3 — Motor de Turnos (backend) — algoritmo de reasignación por prioridad (TASK-40)

## Qué se implementó

Se implementó el algoritmo de reasignación que el documento de
especificación de requisitos, módulo Turnos, describe para el turno que
queda liberado por una cancelación: según la modalidad de reasignación que
el profesional tiene configurada (automática o manual), el turno liberado
se ofrece por orden de prioridad a la lista de espera del profesional, o
simplemente queda disponible sin que se contacte a nadie. Se implementó
también la gestión de la lista de espera que el propio documento de
requisitos pide como soporte de ese algoritmo: alta de un paciente con su
obra social opcional, listado, reordenamiento manual por el profesional y
baja.

- El algoritmo se conectó al punto de extensión `ReassignmentPort`, que la
  tarea de reprogramación y reorganización (P3.6) había dejado publicado
  sin consumidor real, con un adaptador *stub* que solo registraba el
  intento. Este es su primer consumidor real: un adaptador propio del
  nuevo módulo de lista de espera que, ante cada turno cancelado, decide
  qué hacer según la modalidad del profesional.
- En modalidad manual, el turno cancelado queda simplemente disponible: no
  se contacta automáticamente a ningún paciente, y el profesional decide
  desde la aplicación a quién ofrecérselo.
- En modalidad automática, se recorre la lista de espera del profesional
  ordenada por prioridad — el campo que el vínculo paciente-profesional ya
  exponía desde la tarea de estados y prioridad (P3.5) sin consumidor
  todavía —, luego por orden de llegada a la lista y luego por fecha de
  solicitud. Para cada candidato se envía una oferta a través del puerto de
  mensajería existente y se espera una respuesta de aceptación; si el
  candidato no acepta, se continúa con el siguiente. Si alguno acepta, el
  turno original pasa al estado "reasignado" — el mismo valor que la
  máquina de estados de P3.5 ya había declarado alcanzable desde
  "cancelado" sin que ningún método lo ejecutara todavía —, se crea un
  turno nuevo para ese paciente en el mismo horario y con la misma
  duración, y el paciente se retira de la lista de espera. Si la lista se
  agota sin que nadie acepte, o si está vacía, el turno simplemente queda
  liberado, sin error.
- Como todavía no existe un canal real de contacto por WhatsApp ni el
  temporizador de espera que el documento de requisitos atribuye a fases
  posteriores (M4 y M5), la aceptación del paciente se resolvió detrás de
  un puerto de dominio nuevo, `WaitlistResponsePort`, cuyo adaptador *stub*
  registra la oferta pendiente y responde siempre que nadie aceptó — una
  respuesta negativa por defecto, para que ninguna reserva ocurra sin una
  aceptación real hasta que ese canal exista.
- La gestión de la lista de espera se expuso bajo `/lista-espera`: alta con
  posición asignada automáticamente (la siguiente disponible para ese
  profesional), listado ordenado, reordenamiento —reemplazo completo del
  orden a partir de la secuencia de identificadores que envía el
  profesional— y baja, con la misma restricción de "administrador o
  profesional dueño" que ya rige el resto de los recursos acotados a un
  profesional.

## Decisiones y por qué

**El gancho de reasignación se conectó también a la cancelación ordinaria
de un turno (`PATCH /turnos/:id/cancelar`), no solo a la cancelación
masiva por ausencia que P3.6 ya disparaba.** El documento de requisitos
describe el servicio de reasignación como algo que se invoca "cuando un
turno pasa a cancelado", en términos generales, sin acotarlo a la vía de
cancelación por la que llegó a ese estado; dejarlo conectado solo a la
cancelación por ausencia habría dejado sin cubrir el caso más común, la
cancelación voluntaria de un turno por el paciente o el profesional. Ambos
puntos de disparo se unificaron en un único método privado del servicio de
turnos, que arma el evento y absorbe cualquier error del algoritmo de
reasignación sin interrumpir la respuesta de la cancelación que lo
originó — el mismo criterio de mejor esfuerzo que ya regía la notificación
al paciente.

**El módulo nuevo de lista de espera no depende del módulo de Turnos, pese
a que provee la implementación real de un puerto que el módulo de Turnos
consume.** De haber dependido de `AppointmentsService` para crear el turno
de reemplazo y transicionar el original, se habría cerrado un ciclo de
importación: el módulo de Turnos ya necesita importar el de lista de
espera para obtener la implementación real del puerto de reasignación. La
solución fue que el algoritmo opere directamente sobre la tabla de turnos
a través del cliente de Prisma acotado por inquilino, sin pasar por el
servicio de turnos — la misma clase de restricción de composición que la
extracción de `AbsencesModule` ya había resuelto en P3.6, aplicada esta vez
en la dirección opuesta.

**La aceptación de una oferta se modeló con un puerto de dominio propio,
`WaitlistResponsePort`, separado del puerto de mensajería existente.** El
documento de requisitos indica que la oferta en sí se emite a través del
puerto de mensajería, pero la pregunta de si el paciente aceptó es una
cuestión distinta, que en el sistema real dependerá de una respuesta por
WhatsApp dentro de una ventana de tiempo controlada por un temporizador —
ambas piezas de una fase posterior. Mezclar esa pregunta dentro del puerto
de mensajería, cuyos métodos son de una sola vía y no devuelven una
respuesta, habría forzado a cambiar un contrato ya en uso por el resto del
sistema para una necesidad que le es ajena.

**El adaptador *stub* del nuevo puerto de aceptación responde siempre que
nadie aceptó, en lugar de simular una aceptación para poder ejercitar el
resto del algoritmo en un entorno sin WhatsApp real.** Una respuesta
afirmada por defecto habría reservado turnos a partir de una aceptación
que nadie emitió realmente, un comportamiento inseguro de dejar activo en
cualquier entorno donde el adaptador real todavía no esté conectado. Las
pruebas ejercitan el camino de aceptación sustituyendo el puerto por un
doble configurable, no cambiando el comportamiento por defecto del
adaptador.

**El criterio de prioridad — un paciente con prioridad mayor a cero
antepuesto a uno con prioridad cero, y un paciente recurrente antepuesto a
uno nuevo cuando el profesional así lo decidió — se resolvió enteramente a
través del campo de prioridad numérica que el vínculo paciente-profesional
ya exponía desde P3.5, sin agregar un interruptor adicional a nivel del
profesional.** El documento de requisitos no describe un campo de
configuración separado para preferir pacientes recurrentes; la lectura
adoptada es que un profesional que quiere anteponer a sus pacientes
recurrentes lo expresa subiéndoles la prioridad individualmente, el mismo
mecanismo que ya usa para cualquier otro paciente al que quiere anteponer,
en lugar de introducir una segunda vía de expresar la misma preferencia.
Un candidato de la lista de espera que todavía no tiene vínculo registrado
con el profesional —alguien que se anota en la lista sin haber tenido
nunca una sesión— se trata como de prioridad nula en lugar de interrumpir
el recorrido completo, ya que la ausencia de vínculo no es un error sino
un caso legítimo que el documento de requisitos no excluye.

**La atribución de las entradas de auditoría que genera una reasignación
automática recae en el mismo actor que canceló el turno que la disparó, no
en un usuario de sistema.** El sistema reserva un rol de sistema para
procesos automatizados, pero ese rol no tiene ninguna cuenta sembrada
detrás — no existe un usuario de sistema que pueda ser el autor de una
escritura autónoma. Como tanto la cancelación ordinaria como la
cancelación masiva por ausencia ya cuentan con un actor humano autenticado
en el momento en que se dispara la reasignación, ese mismo identificador se
agregó al evento que ambas cancelaciones publican y es el que queda
registrado en la traza de auditoría de la reasignación resultante.

## Alternativas descartadas

- **Agregar un interruptor de configuración nuevo en el profesional para
  preferir pacientes recurrentes sobre nuevos**: descartada por la razón
  ya expuesta — el documento de requisitos no lo pide como un campo
  separado, y el campo de prioridad ya existente permite expresar esa
  preferencia sin ampliar el esquema.
- **Simular una aceptación afirmativa por defecto en el adaptador *stub*
  del puerto de respuesta**, para poder demostrar el camino de reserva sin
  necesidad de sustituir el puerto en cada prueba: descartada porque
  convertiría el comportamiento por defecto del sistema, en cualquier
  entorno donde todavía no exista el canal real, en reservas automáticas
  sin aceptación genuina de nadie.
- **Hacer que el algoritmo de reasignación dependa de `AppointmentsService`
  para crear el turno de reemplazo**, reutilizando la operación de reserva
  ya existente: descartada por el ciclo de importación que habría cerrado
  entre los módulos de Turnos y de lista de espera, según se explica en la
  sección de decisiones.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: sin cambios — la lista de espera, el enumerado de
  modalidad de reasignación, el campo de prioridad y el estado
  "reasignado" ya existían desde P3.1, P1.5 y P3.5 respectivamente, sin
  consumidor hasta esta tarea.
- `src/domain/ports/reassignment.port.ts` (modificado): el evento
  `AppointmentCancelledEvent` incorporó la duración del turno liberado y el
  identificador del actor que disparó la cancelación.
- `src/domain/ports/waitlist-response.port.ts` (nuevo): puerto
  `WaitlistResponsePort` y el contrato de una oferta de reasignación.
- `src/infrastructure/adapters/stub-waitlist-response.adapter.ts` (nuevo):
  adaptador *stub* del puerto anterior.
- `src/infrastructure/adapters/logging-reassignment.adapter.ts`
  (eliminado): reemplazado como implementación real por
  `WaitlistReassignmentAdapter`, en el nuevo módulo de lista de espera.
- `src/infrastructure/integrations.module.ts` (modificado): dejó de proveer
  `ReassignmentPort` (trasladado al nuevo módulo) y pasó a proveer
  `WaitlistResponsePort`.
- `src/waitlist/` (nuevo módulo): `waitlist.module.ts`,
  `waitlist.service.ts` (alta, listado, reordenamiento y baja de la lista
  de espera), `waitlist.controller.ts` (`/lista-espera`),
  `waitlist.presenter.ts`, `waitlist.constants.ts`,
  `dto/create-waitlist-entry.dto.ts`, `dto/reorder-waitlist.dto.ts`,
  `waitlist-reassignment.service.ts` (el algoritmo en sí) y
  `reassignment.adapter.ts` (implementación real de `ReassignmentPort`,
  análoga a `AppointmentAbsenceEventsAdapter` de P3.6).
- `src/appointments/appointments.service.ts` (modificado): la cancelación
  ordinaria ahora también dispara el gancho de reasignación, a través de un
  método privado compartido con la cancelación masiva por ausencia.
- `src/appointments/appointments.module.ts` y `src/app.module.ts`
  (modificados): incorporación del nuevo módulo de lista de espera a la
  composición de módulos.

## Tests y qué validan

- `src/waitlist/waitlist.service.spec.ts` (nuevo): asignación automática de
  la primera posición y de la posición siguiente al máximo existente,
  validación de la obra social contra el catálogo, autorización del
  profesional dueño y rechazo del que no lo es sobre alta, reordenamiento y
  baja, rechazo de un reordenamiento que no coincide exactamente con la
  lista actual del profesional.
- `src/waitlist/waitlist-reassignment.service.spec.ts` (nuevo): modalidad
  manual no contacta a nadie; lista vacía no produce error; el primer
  candidato es ofrecido primero y, si rechaza, se continúa con el
  siguiente; cuando alguno acepta, el turno original pasa a reasignado, se
  crea el turno de reemplazo con los mismos horario y duración, el
  candidato se retira de la lista de espera y quedan las dos entradas de
  auditoría correspondientes; un candidato con prioridad mayor es ofrecido
  antes que uno sin prioridad; un candidato sin vínculo paciente-profesional
  todavía se trata como de prioridad nula sin interrumpir el recorrido.
- `src/appointments/appointments.service.spec.ts` y
  `appointments-rescheduling.service.spec.ts` (modificados): la cancelación
  ordinaria dispara el gancho de reasignación con los campos nuevos del
  evento, sin que una falla del algoritmo impida que la cancelación se
  complete; la aserción existente sobre la cancelación masiva por ausencia
  se amplió para cubrir esos mismos campos nuevos.
- `src/infrastructure/integrations.module.spec.ts` (modificado): la prueba
  de resolución de `ReassignmentPort` se reemplazó por la de
  `WaitlistResponsePort`, coherente con el traslado del primero fuera de
  este módulo.
- `test/waitlist.e2e-spec.ts` (nuevo): contra la instancia local de
  PostgreSQL — alta, listado, reordenamiento y baja de la lista de espera
  a través de la API HTTP, con las mismas reglas de aislamiento por
  inquilino y de autorización que la suite unitaria, verificadas de
  extremo a extremo con JWT reales.
- `test/appointment-reassignment.e2e-spec.ts` (nuevo): contra la instancia
  local de PostgreSQL, con el puerto de respuesta y el de mensajería
  sustituidos por dobles configurables — modalidad manual no contacta a
  nadie y deja el turno cancelado; modalidad automática con lista vacía no
  produce error; ningún candidato acepta y el turno queda liberado; el
  candidato que acepta obtiene el turno de reemplazo y se retira de la
  lista, con las entradas de auditoría correspondientes; un candidato con
  prioridad configurada es ofrecido antes que uno sin prioridad aunque
  haya llegado después a la lista.
- Ejecución: suite unitaria en verde (27 suites / 273 pruebas). Suite
  end-to-end completa en verde (27 suites / 335 pruebas), ejecutada en
  serie (`--runInBand`). Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Se agregaron dos figuras pendientes: el diagrama de secuencia del
algoritmo de reasignación automática (recorrido de la lista de espera por
prioridad, oferta y respuesta, reserva del turno de reemplazo) y el
diagrama entidad-relación / de flujo de la gestión de la lista de espera
(ver `figuras_pendientes.md`, entradas 23 y 24).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-40-waitlist-reassignment` (creada a
  partir de `main`, que ya tenía fusionadas las tres dependencias de esta
  tarea: TASK-34, TASK-38 y TASK-26).
- Ticket: TASK-40 ("P3.7 – Algoritmo de reasignación por prioridad").
  Depende de TASK-34 (P3.1), TASK-38 (P3.5) y TASK-26 (P1.6). El módulo de
  Turnos continúa con TASK-41 (P3.8 – tests integrales) y TASK-78 (P3.b –
  CRUD de feriados), ambas todavía pendientes.
