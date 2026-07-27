# Fase 3 — Motor de Turnos (backend) — regla de paciente nuevo, doble turno y franja extra (TASK-37)

## Qué se implementó

Se implementó la regla de primera sesión sobre las bases sentadas por P3.2
(disponibilidad, TASK-35) y P3.3 (reserva, TASK-36): cuando la reserva es
la primera sesión de un paciente con un profesional (`primera_sesion`,
consultada en el vínculo `PACIENTE_PROFESIONAL`, con la ausencia del
vínculo como "sí, es la primera" por el mismo criterio fijado en TASK-36),
se aplican cuatro controles adicionales antes de reservar:

1. `acepta_nuevos`: si el profesional tiene desactivado el ingreso de
   pacientes nuevos, se rechaza la reserva con un mensaje explícito.
2. `solo_mayores`: si el profesional solo atiende adultos, se calcula la
   edad del paciente a partir de `Patient.birthDate` (con `ageInYears`, ya
   existente en `common/dates/calendar-date.ts`) y se rechaza si es menor
   de 18 años.
3. Un máximo de un paciente nuevo por jornada por profesional: si ya existe
   un turno `RESERVED`/`CONFIRMED` con `isFirstSession=true` para ese
   profesional en ese día, no se ofrece ni se acepta otro.
4. La franja extra (`modalidad_franja_nueva`, `NewPatientSlotMode` en el
   esquema): si el profesional la tiene configurada, la primera sesión
   ocupa dos turnos consecutivos —el segundo comienza exactamente donde
   termina el primero, cada uno con la duración de
   `Professional.consultationDuration`— colocados según el modo elegido:
   - `FIRST_SLOT_OF_DAY` (primer turno del día): el doble turno solo puede
     empezar en el primer slot libre de la jornada.
   - `LAST_SLOT_OF_DAY` (último turno del día): el doble turno solo puede
     terminar en el último slot libre de la jornada.
   - `WITHIN_SCHEDULE` (dentro de la franja): cualquier posición donde haya
     dos slots libres consecutivos es válida.

   Si el profesional todavía no configuró la franja extra (el campo es
   nulo hasta que P1.4 lo fija), la primera sesión se reserva como un único
   turno ordinario, exactamente el comportamiento que ya tenía TASK-36 —
   ver "Decisiones y por qué" para el motivo.

El cálculo de qué instantes son un inicio válido de doble turno se agregó
como un método nuevo del servicio de disponibilidad de P3.2,
`AvailabilityService.getNewPatientSlots`, que reutiliza `getSlots` para la
lista de slots libres y aplica el modo configurado sobre ella. Se expone
también por HTTP como `GET /profesionales/:id/disponibilidad?newPatient=true`,
de modo que el chatbot (M5) y la app puedan pedir directamente la vista de
"turnos para paciente nuevo" con el mismo endpoint que ya usan para la
disponibilidad ordinaria. `AppointmentsService.book` llama a este mismo
método antes de reservar, de manera que el slot que se le ofreció al
paciente y el que efectivamente reserva no puedan discrepar.

Como una primera sesión bajo una franja configurada crea dos turnos, la
respuesta de `POST /turnos` pasó a ser siempre un arreglo (un elemento para
una reserva ordinaria, dos para una primera sesión con franja configurada),
en lugar de cambiar de forma según el caso.

## Decisiones y por qué

**La franja extra y el tope de un paciente nuevo por día se implementaron
como una capa sobre `getSlots`, no como una modificación de su algoritmo.**
El propio comentario dejado en TASK-35 anticipaba esta división ("esa
tarea llama a este servicio y agrega la restricción sobre los slots que
este devuelve"): la agenda ordinaria que ve un paciente que vuelve debe
seguir siendo exactamente la misma, y mezclar ambas reglas en un único
método habría acoplado un cálculo genérico a una regla que solo aplica a
una fracción de las reservas.

**Cuando `newPatientSlotMode` es nulo, la primera sesión se reserva como un
turno único, no se rechaza ni se le exige a la clínica configurar la
franja extra antes de poder atender pacientes nuevos.** Esta decisión no
surge de una lectura literal del ticket sino de una restricción de
compatibilidad descubierta al revisar las pruebas end-to-end ya existentes
de TASK-36: todas sus reservas exitosas son, por construcción, primeras
sesiones (cada paciente de prueba se crea sin vínculo previo), y ninguna
configura `newPatientSlotMode` en el profesional de prueba. Tratar la
ausencia de configuración como un rechazo habría roto ese comportamiento
ya validado y aceptado; tratarlo como "sin regla de franja que aplicar,
reservar como siempre" preserva exactamente el comportamiento de TASK-36
para el caso no configurado, y es consistente con el resto del esquema,
donde un campo de P1.4 nulo significa "todavía no configurado", no "estado
inválido".

**`FIRST_SLOT_OF_DAY`/`LAST_SLOT_OF_DAY` se anclan al primer/último turno
real de la grilla de horario del profesional, no al primer/último turno
que todavía esté libre.** Una lectura más simple —tomar el primer elemento
de la lista de slots libres que ya devuelve `getSlots`— confunde dos casos
distintos: "el primer turno de la jornada está libre" y "el primer turno
de la jornada ya está ocupado, así que el primero *disponible* es el
segundo". El texto del ticket ("el doble turno solo puede empezar en el
primer slot disponible de la jornada") se interpretó como una regla
rígida —si el primer turno de la jornada está ocupado, la modalidad no
ofrece nada ese día, no se corre al siguiente libre—, porque de lo
contrario un paciente nuevo terminaría ocupando el segundo turno del día
bajo una modalidad que existe justamente para reservarle el primero. Esto
exigió calcular, además de los slots libres, la grilla completa de la
jornada (`AvailabilityService.rawSlotInstantsByDay`, un método nuevo y
deliberadamente separado de `getSlots` en lugar de una refactorización de
este —ver alternativas descartadas) para poder distinguir "el primer turno
de la jornada está ocupado" de "el primer turno de la jornada no existe en
absoluto".

**El tope de un paciente nuevo por día se revalida dentro de la
transacción `SERIALIZABLE` que ya envuelve la reserva, no solo antes de
entrar a ella.** Es la misma clase de invariante de lectura-y-escritura que
ya documentaba TASK-36 para el slot libre y la no simultaneidad: dos
reservas concurrentes de dos pacientes nuevos distintos para el mismo
profesional el mismo día podrían ambas leer "todavía no hay ninguno" y
ambas escribir. La verificación previa a la transacción (a través de
`getNewPatientSlots`) queda entonces como una comprobación de
presentación, útil para dar un mensaje de error específico y para no
abrir una transacción sobre un pedido que ya se sabe inválido, pero no es
la que garantiza la invariante bajo concurrencia.

**`acepta_nuevos` y `solo_mayores` se verifican fuera de la transacción,
como precondiciones ordinarias, mientras que el tope diario y la
verificación de ambos slots del doble turno se verifican dentro.** Mismo
criterio que TASK-36 ya había fijado para consentimiento/datos
obligatorios frente a slot libre/simultaneidad: los dos primeros son
lecturas sin nada concurrente con qué competir, mientras que los dos
segundos son invariantes sobre el estado de la agenda que, si se leen y se
escriben en pasos separados, permiten una condición de carrera.

**La respuesta de `POST /turnos` pasó a ser siempre un arreglo.** La
alternativa —devolver un objeto para una reserva ordinaria y un arreglo
que solo aparece cuando se crea un doble turno— haría que la forma de la
respuesta dependiera de una regla de negocio que el cliente no controla
(la configuración del profesional), obligando a quien consume el endpoint
a inspeccionar el cuerpo antes de saber cómo interpretarlo. Se prefirió
una forma uniforme, con un único elemento en el caso ordinario, coherente
con el principio ya declarado en `CLAUDE.md` de que el contrato JSON no
debe cambiar de forma según el caso.

## Alternativas descartadas

- **Refactorizar `getSlots` para que genere internamente la grilla cruda
  de la jornada y la filtre después por ocupación**, en lugar de agregar
  `rawSlotInstantsByDay` como método separado y algo redundante (vuelve a
  leer horarios/feriados/ausencias): descartada por el riesgo de alterar,
  aunque fuera sin intención, el algoritmo de `getSlots` ya cubierto por
  la suite de pruebas de TASK-35, a cambio de un ahorro de consultas que
  no es significativo a la escala de una sola clínica.
- **Aplicar el tope de un paciente nuevo por día también en el cálculo de
  disponibilidad ordinario (`getSlots`)**: descartada porque el tope es
  una restricción propia de la regla de paciente nuevo, no de la agenda en
  general; un paciente que ya está en tratamiento debe poder reservar sin
  que la presencia de un paciente nuevo ese mismo día se lo impida.
- **Exigir `newPatientSlotMode` configurado como condición para aceptar
  cualquier paciente nuevo**, análogo a como `consultationDuration`
  ausente rechaza la reserva: descartada, según se explica arriba, por
  romper el comportamiento ya validado de TASK-36 y porque el ticket
  distingue expresamente la franja extra (una modalidad de *colocación*)
  de `acepta_nuevos` (un interruptor de *si* se aceptan pacientes nuevos en
  absoluto).
- **Vincular los dos turnos del doble turno con una columna nueva en el
  esquema** (una referencia del segundo turno al primero): descartada
  porque ni el ticket ni el diagrama de base de datos (`TURNO
  (es_primera_sesion, duracion)`) piden un vínculo explícito, y ambos
  turnos ya son identificables como par por compartir
  paciente/profesional/`isFirstSession=true` y ser consecutivos en el
  tiempo — agregar una columna solo para ese fin habría sido una
  desnormalización sin un caso de uso que la exigiera todavía.

## Entidades / puertos / adaptadores tocados

- `src/availability/availability.service.ts` (modificado): se agregaron
  `getNewPatientSlots` (método público) y `doubleSlotStarts`/
  `withinScheduleStarts`, `rawSlotInstantsByDay`, `loadDaysWithNewPatient`
  (privados).
- `src/availability/availability.controller.ts` (modificado): el query
  param `newPatient` cambia `GET /profesionales/:id/disponibilidad` entre
  la vista ordinaria y la de doble turno.
- `src/availability/dto/availability-query.dto.ts` (modificado): se agregó
  `newPatient` (booleano opcional).
- `src/appointments/appointments.service.ts` (modificado): `book` ahora
  aplica los cuatro controles de paciente nuevo y crea uno o dos turnos
  según corresponda; se agregaron `loadProfessionalConfig` y
  `assertValidNewPatientStart` (privados).
- `src/appointments/appointments.controller.ts` (modificado): la respuesta
  de `POST /turnos` pasó de un objeto a un arreglo.

No se tocó el esquema de Prisma: los campos que la regla consume
(`acceptsNewPatients`, `adultsOnly`, `newPatientSlotMode`) ya existían
desde P1.4 (TASK-24), sin uso hasta esta tarea.

## Tests y qué validan

- `src/availability/availability.service.spec.ts` (ampliado, 12 pruebas
  nuevas para `getNewPatientSlots`): los tres modos de franja extra con
  todos los slots libres, el caso "el primer/único slot libre relevante ya
  está ocupado" para `FIRST_SLOT_OF_DAY` y `WITHIN_SCHEDULE`, el tope de un
  paciente nuevo por día, el reintegro a los slots ordinarios cuando el
  modo no está configurado, `acceptsNewPatients=false`, y la ausencia total
  de horario configurado.
- `src/appointments/appointments.service.spec.ts` (ampliado, 8 pruebas
  nuevas): rechazo por `acceptsNewPatients=false` (y que ese rechazo *no*
  aplica a un vínculo que ya superó su primera sesión), rechazo/aceptación
  por `solo_mayores` según la edad, rechazo por tope diario, reserva de
  turno único cuando la franja no está configurada, reserva del doble
  turno con los datos correctos cuando sí lo está, rechazo cuando el
  instante pedido no es un inicio válido según el modo configurado, y
  rechazo cuando el segundo slot del doble turno se ocupa entre la
  comprobación previa y la transacción.
- `test/appointments-booking.e2e-spec.ts` (ampliado, 8 pruebas nuevas):
  contra la instancia local de PostgreSQL. Rechazo por `acceptsNewPatients`
  y su no aplicación a un paciente que ya no es nuevo, rechazo/aceptación
  por `solo_mayores`, rechazo del segundo paciente nuevo el mismo día,
  reserva exitosa de un doble turno bajo `WITHIN_SCHEDULE` (dos turnos
  creados, ambos con `isFirstSession=true`, horarios consecutivos, dos
  entradas de auditoría), rechazo de un instante que la modalidad
  configurada no ofrece, y que `GET
  /profesionales/:id/disponibilidad?newPatient=true` presenta exactamente
  los mismos inicios que la reserva termina aceptando.
- Ejecución: suite unitaria en verde (22 suites / 189 pruebas). Suite
  end-to-end completa en verde (23 suites / 282 pruebas) ejecutada en
  serie (`--runInBand`); en paralelo (el modo por defecto de
  `npm run test:e2e`) reaparece la misma interferencia entre archivos de
  prueba que comparten la base de datos de desarrollo ya señalada en la
  entrada de TASK-36 — se confirmó que es preexistente y no atribuible a
  esta tarea revirtiendo los cambios y observando la misma clase de fallas
  (en archivos distintos según la corrida) en la rama base sin modificar.
  Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Se agregó una figura pendiente con el diagrama de los tres modos de
franja extra (`FIRST_SLOT_OF_DAY`/`LAST_SLOT_OF_DAY`/`WITHIN_SCHEDULE`)
sobre una jornada de ejemplo (ver `figuras_pendientes.md`, entrada 19).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-37-new-patient-double-slot` (creada
  a partir de `feature/TASK-36-appointment-booking`, todavía no fusionada
  a `main` al momento de esta tarea). Pusheada a `origin`. Commit
  `cfd1bc6` al momento de redactar esta entrada.
- Ticket: TASK-37 ("P3.4 – Regla de paciente nuevo / doble turno / franja
  extra"). Depende de TASK-35 (P3.2), TASK-36 (P3.3) y TASK-24 (P1.4,
  configuración del profesional), todas ya implementadas.
