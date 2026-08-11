# Fase 3 — Motor de Turnos (backend) — ajustes al modelo de datos (TASK-89)

## Qué se implementó

A diferencia de las tareas anteriores, TASK-89 no agrega un módulo nuevo
sino que revisa y corrige cinco puntos concretos del modelo de datos ya
construido en las fases 0, 2 y 3, señalados por una revisión del esquema
previa a continuar con nuevas funcionalidades:

1. Se revisó si `WaitlistEntry` (LISTA_ESPERA) necesita conservar su propio
   `organizationId`, dado que sus dos entidades relacionadas (`Professional`
   y `Patient`) ya lo llevan de forma directa. La conclusión, documentada en
   el propio esquema, fue conservarlo: `WaitlistEntry` tiene *dos* padres
   acotados por inquilino, y ese único campo es lo que obliga a ambos a
   pertenecer a la misma organización mediante dos claves foráneas
   compuestas — exactamente el mismo caso que `PatientProfessional` y
   `Appointment` ya resuelven así. Retirarlo habría dejado "un profesional
   de una organización enlazado a un paciente de otra en la misma lista de
   espera" como un estado que la base de datos aceptaría y sólo un chequeo
   de servicio impediría.
2. Se eliminó de `WaitlistEntry` el campo `healthInsurerId` (id_obra_social):
   ninguna ruta de lectura ni el algoritmo de reasignación por prioridad lo
   consumían, y se determinó que no era necesario mantenerlo.
3. Se eliminó de `WaitlistEntry` el campo `requestedAt` (fecha_solicitud),
   duplicado exacto de `createdAt`: el registro de la lista de espera se
   crea en la base de datos en el mismo instante en que el paciente pide el
   turno, por lo que ambas columnas sólo podían contener siempre el mismo
   valor. Se aplicó el mismo criterio a `PrescriptionRequest`
   (SOLICITUD_RECETA), que tenía la misma duplicación entre su propio
   `requestedAt` y `createdAt`.
4. El campo `PatientProfessional.priority` (prioridad), hasta ahora un
   entero libre (1-999) cargado a mano por el profesional, se convirtió en
   un enum fijo (`PatientProfessionalPriority`: `LOW`, `MEDIUM`, `HIGH`,
   `URGENT`), nulo cuando el profesional todavía no asignó ninguna.
5. Se agregó al enum `AppointmentStatus` (estado del turno) un nuevo valor
   `ABSENT` (ausente), para el caso en que el paciente no se presenta al
   turno — un turno ausente no es, en el modelo, un turno completado. Junto
   con el nuevo estado se agregó un job programado semanal
   (`AppointmentAutoCompletionCron`, mediante `@nestjs/schedule`) que
   resuelve por omisión, como completado, cualquier turno reservado o
   confirmado cuyo horario ya pasó y que el profesional no marcó como
   ausente ni como completado.

## Decisiones y por qué

**El campo `priority` se convirtió en un enum de cuatro niveles
(`LOW`/`MEDIUM`/`HIGH`/`URGENT`) en lugar de mantenerse como un entero
acotado.** El entero anterior permitía al profesional cargar cualquier
posición entre 1 y 999 sin una escala definida, lo que el propio ticket
señaló como innecesario para lo que el algoritmo de reasignación (Fase 3,
prompt 7) realmente necesita: distinguir quién tiene prioridad sobre quién,
no un ranking numérico fino. El campo se mantiene nulo por omisión (sin
prioridad asignada) con el mismo significado que antes tenía el valor 0/nulo
en el algoritmo de reasignación; el desempate entre pacientes del mismo
nivel de prioridad pasa a depender enteramente del orden de la lista de
espera y de la fecha de creación del registro, en lugar de la posición
exacta dentro del rango numérico anterior.

**El turno ausente y el turno completado comparten la misma transición de
origen (`CONFIRMADO`), y no se habilitó `RESERVADO -> AUSENTE`.** Se siguió
la misma restricción que ya tenía `CONFIRMADO -> COMPLETADO` desde la fase
3, prompt 5 (P3.5): recién en el estado confirmado el profesional puede
efectivamente distinguir si la sesión ocurrió o no, y no se encontró en el
ticket ni en el código existente una razón para tratar la ausencia de forma
distinta a la finalización en ese sentido.

**El job semanal resuelve por omisión un turno no marcado como completado,
nunca como ausente.** El ticket es explícito en que, ante la falta de
acción del profesional, el sistema debe "completar" los turnos, no
marcarlos ausentes — marcar una ausencia es una decisión clínica/
administrativa que el sistema no puede inferir por sí solo, mientras que
asumir que la sesión ocurrió es la opción conservadora que el propio ticket
eligió como comportamiento por omisión. El job trata un turno reservado
vencido igual que uno confirmado vencido: ambos se completan, ya que ambos
pueden llegar a esa situación por la misma causa (el profesional no
actualizó el estado a tiempo).

**Se sembró un usuario de rol `SYSTEM` por organización (migración de datos
más `seedTenant`), algo que el sistema no tenía hasta esta tarea.** El rol
`SYSTEM` existía desde la fase 0 (corrección de TASK-72) explícitamente
reservado para "procesos automáticos/internos (cron de 24 h, expiración de
código de acceso)", pero ningún proceso automático lo había necesitado
todavía: cada entrada de auditoría escrita hasta ahora se atribuye a un
actor real de la solicitud HTTP que la origina (un administrador, un
profesional o el chatbot). El job semanal no tiene una solicitud detrás y,
sin embargo, la traza de auditoría es obligatoria para toda mutación del
sistema (ver la sección correspondiente del marco normativo); se decidió
entonces sembrar el usuario `SYSTEM` que el propio rol ya anticipaba, en
lugar de debilitar la traza de auditoría o inventar un mecanismo distinto
para este caso. El correo del usuario sembrado es determinístico
(`system@<id-de-organización>.internal`), de modo que el job puede
localizarlo por organización y rol sin necesidad de almacenar ni propagar
un identificador.

## Alternativas descartadas

- **Retirar `organizationId` de `WaitlistEntry`**, apoyándose en que ya es
  derivable de sus dos entidades relacionadas: descartada tras la revisión,
  por ser exactamente el caso de dos padres acotados por inquilino que
  justifica conservarlo como clave foránea compuesta, no como una copia
  redundante.
- **Mantener `priority` como entero, sólo acotando su rango de forma más
  estricta**: descartada porque no resolvía el pedido concreto del ticket
  de dejar de ser un entero, y porque un rango acotado sigue sin expresar
  una escala con significado para quien la usa.
- **Habilitar `RESERVADO -> AUSENTE` además de `CONFIRMADO -> AUSENTE`**:
  descartada por inconsistencia con la misma restricción ya aplicada a
  `COMPLETADO` desde la fase anterior, y por no estar pedida por el ticket.
- **Que el job semanal marcara como ausentes, en lugar de completados, los
  turnos vencidos sin resolver**: descartada porque el ticket pide
  explícitamente lo contrario, y porque asumir una ausencia sin que nadie la
  haya constatado sería una afirmación que el sistema no puede respaldar.
- **Debilitar la traza de auditoría para el job (por ejemplo, permitiendo
  un `userId` nulo) en lugar de sembrar un usuario `SYSTEM`**: descartada
  por ser una excepción a una regla que hasta ahora no tenía excepciones, y
  porque el rol `SYSTEM` ya existía con ese propósito declarado desde la
  fase 0.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: `WaitlistEntry` (retiro de `healthInsurerId` y
  `requestedAt`), `PatientProfessional.priority` (entero → enum
  `PatientProfessionalPriority`), `PrescriptionRequest` (retiro de
  `requestedAt`), `AppointmentStatus` (nuevo valor `ABSENT`). Dos
  migraciones nuevas: una de esquema (con conversión por rangos del valor
  entero anterior de `priority` a los cuatro niveles del enum, en lugar de
  descartarlo) y una de datos (siembra del usuario `SYSTEM` por
  organización existente).
- `prisma/seed.ts`: siembra del usuario `SYSTEM` para cada organización que
  el script crea.
- `src/appointments/appointment-auto-completion.cron.ts` (nuevo): el job
  semanal, registrado mediante `@nestjs/schedule` (`ScheduleModule.forRoot()`
  agregado a `AppModule`).
- `src/appointments/appointment-transition.rule.ts`,
  `appointments.service.ts` (nuevo método `absent`), `appointments.
  controller.ts` (nueva ruta `PATCH /turnos/:id/ausente`): el estado
  `ABSENT` y su transición.
- `src/waitlist/` (`waitlist.service.ts`, `waitlist.module.ts`,
  `waitlist.presenter.ts`, `dto/create-waitlist-entry.dto.ts`,
  `waitlist-reassignment.service.ts`): retiro de `healthInsurerId` y
  `requestedAt`, y adaptación del algoritmo de ranking al enum de
  prioridad.
- `src/patients/` (`patient-professionals.service.ts`, `patient.
  presenter.ts`, `patients.constants.ts`, `dto/link-patient-professional.
  dto.ts`, `dto/update-patient-professional.dto.ts`): el enum de
  prioridad.

## Tests y qué validan

- Se actualizaron las pruebas unitarias y end-to-end existentes de los
  módulos de lista de espera, pacientes/profesionales y turnos para
  reflejar los campos retirados y el nuevo enum de prioridad.
- `src/appointments/appointment-auto-completion.cron.spec.ts` (nuevo): el
  cliente de Prisma se simula en memoria.
  - Completa un turno confirmado o reservado vencido, registra la consulta
    en `PatientProfessional` y audita la acción contra el usuario `SYSTEM`.
  - No toca un turno cuya sesión, por su duración, todavía no terminó.
  - Omite, con una advertencia y sin interrumpir el resto de la corrida,
    una organización sin usuario `SYSTEM`.
  - Un conflicto de concurrencia en un turno (ya resuelto por el
    profesional entre la lectura y la escritura del job) no interrumpe el
    resto de la corrida.
  - Recorre cada organización bajo su propio contexto de inquilino.
- `test/appointment-auto-completion.e2e-spec.ts` (nuevo): contra la
  instancia local de PostgreSQL, con una organización con usuario `SYSTEM`
  sembrado y otra sin él.
  - Completa turnos confirmados y reservados vencidos, deja intactos los
    que aún no terminaron y los que ya estaban en un estado resuelto
    (`ABSENT`), y omite sin fallar la organización sin usuario `SYSTEM`.
- `test/appointments-states.e2e-spec.ts`: nuevo bloque
  `PATCH /turnos/:id/ausente` (marca un turno confirmado como ausente sin
  alterar `PatientProfessional`; rechaza un turno todavía reservado;
  respeta la propiedad del profesional y el aislamiento por inquilino).
- Ejecución: suite unitaria completa en verde (29 suites / 302 pruebas).
  Suite end-to-end completa en verde en modo serie (30 suites / 370
  pruebas, `--runInBand`). El nuevo job de auto-completado recorre *todas*
  las organizaciones de la base sin acotarse a las que crea cada archivo de
  prueba — a diferencia del resto de la suite end-to-end, que sólo toca sus
  propias organizaciones — por lo que su ejecución en paralelo junto a
  otras suites que dejan turnos reservados o confirmados vencidos (por
  ejemplo, las de estados de turno o de reasignación) puede alterar datos
  de esas otras suites; se documentó esta condición en el propio archivo de
  prueba y se verificó la corrida completa en modo serie. Los datos usados
  en las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-89-db-adjustments` (creada a partir
  de `main`, ya actualizado con TASK-78 y el endurecimiento de integridad
  posterior). Pusheada a `origin`, pendiente de Pull Request en Bitbucket.
- Ticket: TASK-89 ("Ajustar la base de datos"). Sin descripción propia en
  Jira; el alcance surge del pedido directo de la usuaria, transcripto al
  inicio de esta entrada de bitácora en los cinco puntos de la sección "Qué
  se implementó". Toca entidades de la fase 0 (`Organization`), la fase 2
  (`PatientProfessional.priority`) y la fase 3 (`WaitlistEntry`,
  `PrescriptionRequest`, `Appointment`), pero se registra bajo la fase 3
  por ser donde concentra la mayor parte del código nuevo (el estado
  `ABSENT` y el job de auto-completado).
