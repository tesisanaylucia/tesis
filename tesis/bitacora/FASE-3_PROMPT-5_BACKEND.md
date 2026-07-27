# Fase 3 — Motor de Turnos (backend) — máquina de estados, prioridad de recurrentes y registro de pago/orden (TASK-38)

## Qué se implementó

Se implementó la máquina de estados del turno, la regla de cancelación
temprana, el disparo hacia `PACIENTE_PROFESIONAL` al completar un turno, el
método que expone la prioridad del paciente para la reasignación de una
fase posterior, y los dos endpoints de registro administrativo del turno
(pago y orden médica), siguiendo el documento de especificación de
requisitos, módulo Turnos, secciones "Estados del turno", "Confirmación",
"Cancelación", "Registro de pago" y "Prioridad de pacientes recurrentes".

La tabla de transiciones válidas —reservado a confirmado, reservado a
cancelado, confirmado a completado, confirmado a cancelado, y cancelado a
reasignado— se extrajo a una función pura,
`isValidAppointmentTransition`/`assertValidAppointmentTransition`
(`src/appointments/appointment-transition.rule.ts`), separada del servicio,
siguiendo el mismo patrón que ya había fijado Pacientes para la regla de
inactividad (`patient-inactivity.rule.ts`): una tabla de datos sin
dependencias de infraestructura, fácil de agotar con una prueba unitaria
por cada par de estados posible. La transición de cancelado a reasignado se
declaró válida en la tabla, pero ningún método del servicio la ejecuta
todavía: el documento de requisitos atribuye esa escritura al algoritmo de
reasignación de una fase posterior (P3.7), y esta tarea solo deja la
transición reconocida para cuando ese algoritmo exista.

Sobre esa tabla se agregaron tres métodos nuevos al servicio de turnos:

- `confirm`, que aplica reservado a confirmado y registra la fecha de
  confirmación. Reservado a los mismos roles que la propia reserva
  (administrador o el proceso automatizado del chatbot), porque el
  documento de requisitos describe la confirmación como una acción que el
  paciente realiza por WhatsApp, no algo que un profesional haga en nombre
  del paciente.
- `cancel`, que aplica reservado a cancelado o confirmado a cancelado, con
  la regla de anticipación mínima: si faltan menos de las horas
  configuradas para el turno, la cancelación se rechaza con un mensaje que
  indica que el paciente debe contactar directamente a la clínica.
- `complete`, que aplica confirmado a completado y, dentro de la misma
  transacción, actualiza el vínculo paciente-profesional a través del
  método que ya existía desde la tarea de tipo de paciente (P2.3):
  `PatientProfessionalsService.recordConsultation`, que fija el tipo a
  recurrente, apaga el indicador de primera sesión, y adelanta la fecha de
  última consulta si corresponde.

El registro de pago y de orden médica se expuso como dos endpoints nuevos,
`PATCH /turnos/:id/pago` y `PATCH /turnos/:id/orden`, sin restricción de
estado: cualquiera de los dos puede registrarse en cualquier momento del
ciclo de vida del turno, porque el documento de requisitos no ata ninguno
de los dos campos a un estado particular.

Por último se expuso `AppointmentsService.getPatientPriority`, que
consulta el vínculo paciente-profesional y devuelve su tipo (nuevo o
recurrente) y su prioridad asignada por el profesional, sin combinar
ambos valores en un único orden: el documento de requisitos encomienda esa
combinación al algoritmo de reasignación de una fase posterior (P3.7), y
esta tarea solo deja disponibles los dos datos que ese algoritmo
necesitará leer.

## Decisiones y por qué

**La anticipación mínima de cancelación se modeló como configuración por
organización (`OrganizationConfig`, clave `appointment_cancellation_min_hours`,
valor por defecto 4), no como una constante en el código.** Es el mismo
criterio que `CLAUDE.md` ya fija para las reglas de negocio de alcance
organizacional, y el mismo patrón que ya había usado la tarea de tipo de
paciente para el umbral de inactividad: el valor se lee a través de
`ConfigTenantService`, con el mismo criterio de validación (un entero
positivo, o el valor por defecto si el dato guardado no lo es), y se
sembró tanto en una migración —para que una organización ya existente
reciba la fila sin depender de que el seed de desarrollo se ejecute— como
en el seed, siguiendo exactamente la justificación que ya deja registrada
la migración homóloga del umbral de inactividad.

**La transición de estado se protegió con una escritura condicionada al
estado de origen (`updateMany` con `where: { id, status: from }`), no con
una transacción serializable completa.** A diferencia de la reserva, donde
el turno todavía no existe y la invariante es sobre la ausencia de
conflicto, aquí la fila ya existe y la invariante es "nadie más cambió el
estado entre la lectura y la escritura" — el mismo problema, y la misma
solución, que ya usa el vínculo paciente-profesional para no retroceder la
fecha de última consulta ante dos completados concurrentes. Dos
transiciones concurrentes sobre el mismo turno hacen que la segunda
actualización no encuentre ninguna fila que igualar y se traduzca en un
conflicto (409), sin necesidad de pagar el costo de una transacción
serializable para una invariante que una condición en la propia escritura
ya resuelve.

**Completar un turno actualiza el vínculo paciente-profesional a través de
`recordConsultation` (que recibe la transacción en curso), no de
`registerConsultation` (que abre la suya propia).** La distinción ya
existía en el servicio de Pacientes desde P2.3, pensada exactamente para
este caso: un llamador que ya estableció que el turno pertenece al
inquilino actuante no necesita repetir esa verificación, y la escritura
del estado del turno y la actualización del vínculo deben confirmarse
como una sola unidad — un turno completado sin la consulta registrada, o
viceversa, es un estado que ninguna de las dos escrituras por separado
debería poder dejar.

**La autorización de "administrador o el profesional dueño del turno" se
resolvió en el servicio, después de cargar la fila, y no con el guard de
propiedad ya existente (`ProfessionalOwnershipGuard`).** Ese guard lee el
identificador del profesional directamente de un parámetro de la URL, y en
estas rutas el parámetro `:id` nombra al turno, no al profesional — la
propiedad solo puede conocerse después de leer la fila. Se aplicó la misma
comparación (`isSameProfessional`) que ya usa el controlador de
paciente-profesional para el mismo problema, dejando expresamente
documentado en el código que la comprobación de organización sigue siendo
la que en verdad contiene el pedido, siguiendo el mismo principio que
`CLAUDE.md` ya deja explícito sobre ese guard.

**El registro de pago y de orden médica no exige ningún estado
particular del turno.** El documento de requisitos describe ambos campos
como datos administrativos del turno (si se cobró, si el paciente trajo
la orden médica de la obra social), sin condicionarlos a que el turno
esté confirmado o completado; exigir un estado habría sido una regla
adicional que el documento no plantea.

## Alternativas descartadas

- **Devolver 403 en lugar de 404 cuando el turno pertenece a otra
  organización**: descartada por el mismo motivo que ya fija `CLAUDE.md`
  para toda restricción más estrecha que el inquilino — confirmar que el
  turno existe en otra organización es en sí mismo una fuga de
  información, así que el filtrado automático por inquilino que ya aplica
  la extensión de Prisma sobre `Appointment` (un turno de otra
  organización simplemente no aparece en la consulta) se dejó producir su
  404 natural.
- **Ejecutar la transición de cancelado a reasignado desde esta tarea**,
  ya que la tabla de estados la declara válida: descartada porque el
  documento de requisitos atribuye esa escritura al algoritmo de
  reasignación de P3.7, que decide *a qué* paciente de la lista de espera
  se reasigna el turno liberado — una decisión que esta tarea no tiene
  información para tomar.
- **Combinar tipo y prioridad en un único valor de orden dentro de
  `getPatientPriority`**: descartada por la misma razón que la anterior;
  el documento de requisitos deja esa combinación al algoritmo de P3.7, y
  anticiparla aquí habría fijado un criterio de orden que esa tarea
  todavía no definió.

## Entidades / puertos / adaptadores tocados

- `src/appointments/appointment-transition.rule.ts` (nuevo): tabla de
  transiciones válidas y las funciones de validación.
- `src/appointments/appointments.constants.ts` (nuevo): clave y valor por
  defecto de la anticipación mínima de cancelación.
- `src/appointments/appointments.service.ts` (modificado): se agregaron
  `confirm`, `cancel`, `complete`, `setPayment`, `setReferralOrder`,
  `getPatientPriority` y los métodos privados de apoyo (`loadOrThrow`,
  `assertOwnerOrAdmin`, `cancellationMinHours`, `applyStatusTransition`,
  `guardedStatusUpdate`, `applyFieldUpdate`).
- `src/appointments/appointments.controller.ts` (modificado): se
  agregaron `PATCH /turnos/:id/confirmar`, `/cancelar`, `/completar`,
  `/pago` y `/orden`.
- `src/appointments/dto/update-payment.dto.ts` y
  `update-referral-order.dto.ts` (nuevos): validación de los cuerpos de
  pago y orden médica.
- `prisma/migrations/20260727180000_seed_appointment_cancellation_config/`
  (nueva): siembra la configuración de anticipación mínima de cancelación
  para toda organización existente.
- `prisma/seed.ts` (modificado): se agregó la misma clave a la lista de
  configuración sembrada para organizaciones nuevas.

No se tocó el esquema de Prisma: los campos que esta tarea consume
(`status`, `paymentCompleted`, `broughtReferralOrder`, `confirmedAt`) ya
existían desde la tarea que modeló la entidad turno (P3.1), sin uso hasta
este punto.

## Tests y qué validan

- `src/appointments/appointment-transition.rule.spec.ts` (nuevo): agota
  el producto cartesiano de los cinco estados contra sí mismos — cada
  transición declarada válida se acepta, y cada una de las restantes se
  rechaza con el mensaje descriptivo esperado.
- `src/appointments/appointments.service.spec.ts` (ampliado): confirmación
  con marca de fecha y auditoría, rechazo de una transición inválida,
  404 sobre un turno inexistente en el inquilino, conflicto ante una
  transición concurrente, cancelación con y sin la anticipación mínima
  (incluido un umbral configurado por la organización distinto del
  valor por defecto), cancelación de un turno confirmado además de uno
  reservado, rechazo de la cancelación de un turno completado,
  autorización del profesional dueño y rechazo del que no lo es,
  completado con la llamada a `recordConsultation` con la fecha
  correcta, rechazo de completar un turno todavía no confirmado, registro
  de pago y de orden con su entrada de auditoría, comportamiento sin
  operación cuando el valor ya coincide, y la exposición de tipo y
  prioridad sin combinarlos.
- `test/appointments-states.e2e-spec.ts` (nuevo, 23 pruebas): contra la
  instancia local de PostgreSQL. Confirmación por administrador y por el
  proceso automatizado, rechazo al rol profesional, rechazo de una
  transición inválida con su mensaje, aislamiento por inquilino (404).
  Cancelación con y sin la anticipación mínima (turno reservado y
  confirmado), rechazo de cancelar un turno completado, autorización del
  profesional dueño y rechazo del que no lo es. Completado con
  verificación directa en la base de datos de que el vínculo
  paciente-profesional quedó con el indicador de primera sesión apagado,
  el tipo en recurrente y la fecha de última consulta escrita, más su
  propia entrada de auditoría; rechazo de completar sin confirmar antes;
  autorización equivalente a la de cancelación. Pago y orden médica:
  registro por administrador y por el profesional dueño, rechazo al
  profesional que no lo es, rechazo al proceso automatizado (sin criterio
  clínico para registrar estos datos), rechazo sin autenticar, y
  aislamiento por inquilino.
- Ejecución: suite unitaria en verde (23 suites / 217 pruebas). Suite
  end-to-end completa en verde (24 suites / 297 pruebas) ejecutada en
  serie (`--runInBand`); en paralelo reaparece la misma interferencia
  entre archivos de prueba que comparten la base de datos de desarrollo
  ya señalada en las entradas de TASK-36 y TASK-37 — se confirmó que es
  preexistente y no atribuible a esta tarea ejecutando el archivo
  afectado (`patient-consent.e2e-spec.ts`) de forma aislada, donde pasa
  sin fallas. Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Se agregó una figura pendiente con el diagrama de la máquina de estados
del turno (ver `figuras_pendientes.md`, entrada 20).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-38-appointment-states-priority-payment`
  (creada a partir de `feature/TASK-36-appointment-booking`, ya que TASK-36
  todavía no estaba fusionado a `main` al momento de esta tarea, mientras
  que TASK-34 y TASK-29, las otras dos dependencias, sí lo estaban).
  Commit `da0c3e9` al momento de redactar esta entrada.
- Ticket: TASK-38 ("P3.5 – Prioridad de recurrentes, estados y registro de
  pago/orden"). Depende de TASK-34 (P3.1), TASK-29 (P2.3) y TASK-36 (P3.3).
