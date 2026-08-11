# Fase 3 — Motor de Turnos (backend) — revisión ultrareview de TASK-89

## Qué se implementó

Esta tarea no partió de un ticket nuevo sino de una revisión automatizada
del código ya construido (`/code-review ultra`, ejecutada por varios agentes
en paralelo, cada uno cubriendo un módulo o un ángulo distinto) contra las
dos fuentes de verdad del proyecto: el anteproyecto de tesis y el documento
de requisitos (SRS) de "PSIQUE Neurociencias". De los dieciséis hallazgos
que arrojó esa revisión, la usuaria clasificó cada uno por separado —
corregir ahora, dejar como está, o no implementar todavía— y esta entrada
cubre los seis que se marcaron para corregir de inmediato:

1. **El camino de escritura de turnos no validaba feriados.** `getSlots`
   (las sugerencias que ofrece el sistema) ya excluía los feriados desde la
   fase de disponibilidad, pero `isSlotFree` —el chequeo del que dependen
   la reserva y la reprogramación— nunca lo hacía, de modo que un turno
   podía reservarse o reprogramarse directamente a una fecha feriada sin
   pasar por las sugerencias.
2. **Un turno liberado por una cancelación era inmediatamente reservable
   por cualquiera**, tanto en la modalidad manual de reasignación (donde el
   documento de requisitos pide que quede retenido 24 horas antes de que el
   bot pueda volver a ofrecerlo) como en la automática (mientras el
   algoritmo todavía está recorriendo la lista de espera ofreciéndolo a sus
   candidatos).
3. **La reprogramación individual y la reorganización manual de la agenda
   aplicaban el cambio de horario y recién después avisaban al paciente**,
   sin esperar ninguna respuesta suya, pese a que el documento de
   requisitos pide que el sistema le pregunte si acepta el nuevo horario
   antes de aplicarlo.
4. **Un profesional no tenía forma de registrar datos faltantes de sus
   propios pacientes**: el punto de acceso de edición del paciente
   (`PATCH /pacientes/:id`) estaba restringido a administración y al
   proceso automatizado, pese a que el documento de requisitos lista esa
   acción entre las que la aplicación móvil del profesional debe permitir.
5. **No existía ningún punto de acceso de backend para la agenda propia del
   profesional** (las vistas diaria, semanal y mensual que la aplicación
   móvil necesita mostrar) — sólo existía la consulta de franjas libres
   (`disponibilidad`), que responde una pregunta distinta.

## Decisiones y por qué

**El chequeo de feriados se incorporó dentro de `isSlotFree` en lugar de
repetirlo en cada punto de escritura.** `isSlotFree` ya era, desde la
reserva original, el único lugar del que dependen tanto la creación como la
reprogramación de un turno para decidir si un instante está ocupado; sumar
allí la consulta de feriado evita que las dos rutas de escritura puedan
llegar a definir "libre" de manera distinta entre sí, y mantiene esa
definición alineada con la que ya aplicaba `getSlots` del lado de las
sugerencias.

**La retención de un turno liberado se modeló con una única columna de
fecha y hora (`Appointment.holdUntil`) en lugar de una tabla aparte o un
indicador booleano.** Una marca de tiempo permite expresar dos semánticas
distintas con el mismo mecanismo: en modalidad manual es un plazo real —
veinticuatro horas desde la cancelación, tal como lo fija el documento de
requisitos, sin que ningún proceso tenga que liberarlo explícitamente,
porque `isSlotFree`/`getSlots` simplemente dejan de contar como ocupado un
turno cancelado en cuanto el reloj supera esa marca—; en modalidad
automática es, en cambio, sólo un techo de seguridad ante una caída del
proceso a mitad del recorrido de la lista de espera, y la retención real se
libera explícitamente apenas ese recorrido concluye, acepte alguien la
oferta o se agote la lista. La duración del techo de seguridad se dejó como
una constante propia, distinta y documentada aparte de la ventana real de
"cuánto tiempo tiene un candidato para responder" que todavía no existe
(depende del temporizador y el canal real de WhatsApp de una fase
posterior), para no confundir una salvaguarda técnica con una regla de
negocio que el sistema todavía no implementa.

**La confirmación del paciente en una reprogramación se resolvió con un
puerto nuevo, `RescheduleResponsePort`, que reproduce deliberadamente la
forma y el criterio de `WaitlistResponsePort`**: se pregunta antes de
escribir, y el adaptador de reemplazo mientras no exista un canal real de
WhatsApp siempre responde que no, para que una aceptación jamás se dé por
sentada. Sólo se exige esa confirmación cuando quien reprograma actúa por su
cuenta —administración o el propio profesional—, nunca cuando lo hace el
proceso automatizado en nombre del paciente, porque en ese caso el pedido
del propio paciente ya es su aceptación. La reorganización manual de la
agenda, en cambio, la exige siempre: a esa ruta nunca llega el proceso
automatizado, así que toda reorganización es, por definición, unilateral.
Con el único adaptador disponible respondiendo siempre que no, toda
reprogramación iniciada por administración o por el profesional queda hoy
efectivamente bloqueada hasta que una fase posterior conecte el canal real
— la misma situación, ya aceptada, en la que se encuentra la reasignación
automática de turnos desde que se implementó su propio puerto de
respuesta.

**La edición de datos del paciente por el profesional se resolvió
reutilizando exactamente el mismo cuerpo de solicitud y el mismo servicio
que ya usa administración, en lugar de crear un punto de acceso o un
formulario más angosto limitado a los "datos faltantes".** El documento de
requisitos no describe una superficie más estrecha que esa, así que
angostarla habría sido una regla inventada; lo que sí cambia es el alcance:
un profesional sólo puede tocar los pacientes con los que tiene un vínculo
de tratamiento vigente, la misma restricción que ya aplica a cualquier otra
lectura de un paciente de su parte, y su respuesta llega recortada a su
propio vínculo, en vez de traer los de todos los profesionales que atienden
a ese paciente.

**La agenda propia del profesional se expuso como un punto de acceso nuevo,
separado de la disponibilidad, y restringido al propio profesional o a
administración**, a diferencia de la disponibilidad y las ausencias, que
cualquier usuario autenticado del inquilino puede consultar: un turno
nombra a un paciente concreto, así que "qué pacientes ve este profesional y
cuándo" no es información de agenda pura, y se le aplicó el mismo criterio
de acceso que ya rige las mutaciones propias del profesional (ausencias,
matrículas, horarios), llevado también a esta lectura.

## Alternativas descartadas

- **Repetir la consulta de feriados en cada método de escritura de turnos**
  en lugar de incorporarla a `isSlotFree`: descartada por el riesgo de que
  las distintas rutas de escritura llegaran a definir "ocupado" de manera
  distinta entre sí, exactamente el problema que este mismo chequeo ya
  evita para el par lectura/escritura.
- **Una tabla aparte para la retención del turno liberado**, en lugar de la
  columna `holdUntil` sobre el propio turno cancelado: descartada porque
  cada retención nace siempre de una cancelación concreta, así que no había
  necesidad de desacoplar el estado de retención de la fila que ya la
  origina.
- **Que la retención en modalidad automática usara también un plazo fijo**
  (por ejemplo, el mismo techo de seguridad como ventana real): descartada
  porque mezclaría una salvaguarda técnica con una regla de negocio que
  todavía no está definida (el tiempo real de espera de un candidato
  pertenece a una fase posterior).
- **Aplicar la reprogramación igual que antes y sólo dejar registrada la
  confirmación pendiente para revisarla después**: descartada por no
  implementar lo que pide el documento de requisitos, que es preguntar
  *antes* de aplicar el cambio, no después.
- **Un formulario o punto de acceso separado, más angosto, para que el
  profesional registre sólo los datos faltantes de su paciente**:
  descartada por no estar pedida así en el documento de requisitos, y por
  duplicar la validación que el punto de acceso existente ya aplica.
- **Abrir la lectura de la agenda del profesional a cualquier usuario
  autenticado del inquilino**, siguiendo el criterio ya usado para
  disponibilidad y ausencias: descartada porque, a diferencia de esos dos
  recursos, un turno nombra a un paciente concreto.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: `Appointment.holdUntil` (nuevo campo, con su
  migración).
- `src/availability/availability.service.ts`: `isSlotFree` y
  `loadBookedTimes` ahora comparten una única definición de "ocupado"
  (turno vigente, retención activa, o feriado).
- `src/waitlist/waitlist-reassignment.service.ts`,
  `waitlist.constants.ts`: fijación y liberación de la retención en ambas
  modalidades de reasignación.
- `src/domain/ports/reschedule-response.port.ts` (nuevo),
  `src/infrastructure/adapters/stub-reschedule-response.adapter.ts`
  (nuevo), registrado en `integrations.module.ts`.
- `src/appointments/appointments.service.ts`: `reschedule`,
  `reorganizeAgenda` y `rescheduleCore` ahora piden confirmación antes de
  escribir cuando corresponde; nuevo método `listForProfessional`.
- `src/appointments/dto/find-appointments-query.dto.ts`,
  `professional-appointments.controller.ts` (nuevos): la agenda propia del
  profesional, `GET /profesionales/:id/turnos`.
- `src/patients/patients.controller.ts`, `patients.service.ts`,
  `dto/update-patient.dto.ts`: el rol profesional puede ahora editar sus
  propios pacientes.

## Tests y qué validan

- Se actualizaron las pruebas unitarias y end-to-end existentes de
  disponibilidad, reasignación y reprogramación para reflejar la nueva
  definición de "ocupado" y la exigencia de confirmación.
- Pruebas nuevas: la reserva/reprogramación rechaza un feriado; la
  reasignación fija y libera la retención en ambas modalidades, con la
  duración correcta en cada una; la reprogramación pide confirmación antes
  de escribir y la rechaza (409) si el paciente no confirma, salvo cuando
  el proceso automatizado reprograma en nombre del paciente; la
  reorganización manual reporta un movimiento no confirmado como fallido
  sin abortar el resto del lote; el profesional tratante puede completar
  datos de su paciente y recibe 404 ante uno que no atiende; la agenda
  propia del profesional lista turnos de cualquier estado dentro del rango,
  respeta el aislamiento por inquilino y la propiedad del profesional.
- Ejecución: suite unitaria completa en verde (29 suites / 313 pruebas).
  Suite end-to-end completa en verde en modo serie (31 suites / 382
  pruebas, `--runInBand`). Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-89-db-adjustments` (continuación de
  la misma rama que TASK-89). Commit `a3db129`. Pusheada a `origin`,
  pendiente de Pull Request en Bitbucket.
- Origen: hallazgos 2, 3, 4, 11, 14 y 15 de una revisión `/code-review
  ultra` contra el anteproyecto de tesis y el SRS de PSIQUE Neurociencias,
  triados por la usuaria (los hallazgos 1, 5, 6, 12 y 13 se dejaron como
  están; los hallazgos 7, 8, 9, 10 y 16 —precios/copago, distinción de obra
  social provincial, código de acceso, ventana de validez del PIN,
  orquestación de la cerradura, notificación al profesional, apertura
  remota de la cerradura— quedaron marcados explícitamente como no
  implementar todavía).
