# Fase 3 — Motor de Turnos (backend) — suite de tests integrales del motor de turnos (TASK-41)

## Qué se implementó

Se agregó una suite de tests de extremo a extremo
(`test/appointment-engine-integration.e2e-spec.ts`) que ejercita el motor de
turnos completo tal como lo describe el documento de especificación de
requisitos para las tareas P3.1 a P3.7 combinadas, en lugar de cada regla
por separado: disponibilidad, reserva, confirmación, cancelación,
reasignación y completado encadenados sobre un mismo turno, contra una base
de datos PostgreSQL real y con los puertos de mensajería y de respuesta de
lista de espera sustituidos por dobles configurables, sin ninguna llamada a
WhatsApp, TTLock ni al proveedor de IA.

La suite no duplica la cobertura ya existente por módulo (reserva, estados,
disponibilidad, lista de espera, reasignación y reprogramación, cada una
con su propia suite de extremo a extremo desde las tareas P3.3 a P3.7): en
su lugar, encadena esas piezas en un único recorrido continuo y aloja
además los escenarios que el documento de requisitos pide para la tarea
P3.8 y que ninguna suite de módulo cubre por sí sola — las tres modalidades
de franja extra para paciente nuevo ejercitadas juntas dentro del mismo
archivo, y un escenario con dos organizaciones simultáneas cuyos
profesionales y pacientes comparten nombre y documento, para comprobar que
el acotamiento por inquilino sostiene a la vez la reserva, la cancelación,
la reasignación y la lista de espera.

- Un test único recorre el ciclo completo: la franja aparece en la
  disponibilidad del profesional, se reserva sobre ella, se confirma, se
  cancela con más de cuatro horas de anticipación —lo que dispara la
  reasignación automática—, el único candidato de la lista de espera
  acepta y ocupa el mismo horario, y ese turno de reemplazo se confirma y
  completa, verificando el efecto sobre el vínculo paciente-profesional
  (primera sesión, tipo de paciente y fecha de última consulta).
- Un grupo cubre las exclusiones de disponibilidad por feriado y por
  ausencia del profesional.
- Un grupo cubre las tres modalidades de franja extra para paciente
  nuevo —primer turno del día, último turno del día y dentro de la franja
  habitual—, reservando el doble turno bajo cada una, además de los topes
  de configuración (un paciente nuevo por jornada, `acepta_nuevos` en
  falso, `solo_mayores` con un paciente menor de edad).
- Un grupo cubre la ventana mínima de cuatro horas para cancelar.
- Un grupo cubre la reasignación automática cuando ningún candidato acepta
  y la modalidad manual, que no contacta a nadie.
- Un grupo cubre la reprogramación masiva por ausencia, verificando que
  también dispara la reasignación automática sobre el turno cancelado.
- Un grupo cubre el escenario multi-tenant descripto arriba.

## Decisiones y por qué

**La suite no sustituye `ReassignmentPort`.** A diferencia de
`appointment-reassignment.e2e-spec.ts` y
`appointments-rescheduling.e2e-spec.ts`, que sustituyen ese puerto por un
espía para aislar la prueba de cancelación de la de reasignación, esta
suite deja conectado el adaptador real —el mismo que usa el sistema en
producción— porque el objetivo es comprobar la cadena completa: cancelar
debe disparar la reasignación real, no una versión simulada de ella. Solo
se sustituyen los dos puertos sin implementación real todavía
—`WaitlistResponsePort` y `MessagingPort`—, los únicos que dependen de un
canal externo (WhatsApp) que no existe hasta una fase posterior.

**Los candidatos de lista de espera se crean sin los datos obligatorios de
una reserva ni consentimiento registrado.** El algoritmo de reasignación
(P3.7) crea el turno del candidato directamente, no a través del endpoint
de reserva, por lo que no pasa por las cuatro validaciones de la tarea
P3.3; exigirle a un candidato de prueba esos datos habría ejercitado una
restricción que el propio sistema no aplica en ese camino.

**El escenario multi-tenant reutiliza el mismo documento de identidad en
ambas organizaciones.** La unicidad del documento del paciente está
acotada por organización, no es global, de modo que dos organizaciones
distintas registrando pacientes con el mismo documento —y el mismo
profesional con el mismo nombre— es un caso legítimo y no un conflicto; es
también el caso más exigente para probar el acotamiento, porque descarta
que el aislamiento dependa accidentalmente de que los datos de prueba sean
distintos entre organizaciones.

## Alternativas descartadas

- **Repartir los escenarios de la tarea P3.8 entre las suites de cada
  módulo existente**, en lugar de un archivo nuevo: descartada porque el
  documento de requisitos pide explícitamente una suite integral que
  ejercite el motor completo, y desperdigar el recorrido de punta a punta
  entre archivos independientes no deja en ningún lugar la prueba de que
  las piezas efectivamente encajan entre sí.
- **Sustituir también `ReassignmentPort` por un espía**, como hacen las
  suites de módulo: descartada porque habría dejado sin probar,
  precisamente en la suite pensada para hacerlo, que una cancelación real
  dispara una reasignación real.

## Entidades / puertos / adaptadores tocados

Ninguno: la tarea es exclusivamente de tests, sin cambios de esquema ni de
código de producción.

- `test/appointment-engine-integration.e2e-spec.ts` (nuevo).

## Tests y qué validan

- `runs the full turno lifecycle end to end`: el recorrido completo descripto
  arriba, encadenado sobre un único turno.
- `Disponibilidad excludes blocked periods` (2 pruebas): un feriado y una
  ausencia del profesional excluyen el día correspondiente de la
  disponibilidad.
- `New-patient rules` (6 pruebas): doble turno bajo cada una de las tres
  modalidades de franja extra (con el caso adicional, dentro de
  `FIRST_SLOT_OF_DAY`, de que un horario que no es el primero del día no
  ofrece nada), tope de un paciente nuevo por jornada, `acepta_nuevos` en
  falso y `solo_mayores` con un paciente menor de edad.
- `Cancellation window` (1 prueba): una cancelación con menos de cuatro
  horas de anticipación se rechaza y el turno permanece reservado.
- `Waitlist reassignment on cancellation` (2 pruebas): modalidad automática
  sin ningún candidato que acepte deja el turno cancelado y libre;
  modalidad manual no contacta a nadie.
- `Mass rescheduling on absence` (1 prueba): la ausencia registrada cancela
  el turno dentro del período, lo etiqueta con el motivo correspondiente y
  dispara la reasignación automática sobre él, dejando intacto el turno
  fuera del período.
- `Multi-tenant scoping across the appointment engine` (1 prueba): dos
  organizaciones con profesional y paciente de igual nombre y documento
  reservan sobre el mismo horario sin conflicto entre sí, una organización
  no puede reservar contra el paciente o el profesional de la otra, y la
  lista de espera de cada una permanece separada.
- Ejecución: 14 pruebas nuevas, todas en verde, entre 4 y 8 segundos en
  aislamiento. Suite unitaria completa en verde (27 suites / 273 pruebas).
  Suite end-to-end completa en verde (28 suites / 349 pruebas) ejecutada en
  serie (`--runInBand`); en paralelo —el modo por defecto de
  `npm run test:e2e`— algunas pruebas de esta suite y de otras ya
  existentes fallan de forma intermitente por conflictos de una transacción
  serializable al compartir todos los archivos la misma base de datos
  local; es una condición preexistente del entorno de pruebas, no
  introducida por esta tarea, que también se reproduce en archivos sin
  modificar (`appointments-rescheduling.e2e-spec.ts`, entre otros), y queda
  señalada como pendiente de investigar en una tarea de endurecimiento
  posterior más que resuelta aquí. Los datos usados en las pruebas son
  ficticios.

## Figuras pendientes

Ninguna figura nueva: la tarea documenta comportamiento ya diagramado (o
pendiente de diagramar) por las entradas de bitácora de P3.1 a P3.7.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-41-appointment-engine-integration-tests`
  (creada a partir de `main`, que ya tenía fusionadas las siete
  dependencias de esta tarea: TASK-34 a TASK-40).
- Ticket: TASK-41 ("P3.8 – Tests integrales del motor de turnos"). Cierra,
  junto con TASK-78 (P3.b – CRUD de feriados, todavía pendiente), la fase
  del Motor de Turnos.
