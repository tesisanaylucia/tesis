# Fase 4 — Notificaciones y Scheduler (backend) — turnos de corto plazo nunca reciben confirmación, y fallas de envío igual disparan la cancelación automática (TASK-157, corrección a TASK-43/TASK-44)

## Qué se implementó

TASK-157 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, validada por la usuaria
antes de implementarse. Reunía dos hallazgos relacionados sobre el job de
confirmación de turno (`AppointmentConfirmationCron`, sección 4.5) y su
efecto sobre el job de cancelación automática por falta de confirmación que
depende de él:

- **TUR-5**: un turno reservado con menos de 23 horas de anticipación nunca
  entra en la ventana de detección de 23-25 horas del job, porque el tiempo
  que falta hasta la cita sólo se acorta con el paso del tiempo — nunca
  vuelve a entrar en un rango que ya quedó atrás. `confirmedAt` quedaba
  entonces null para siempre, sin que el paciente hubiera tenido nunca la
  oportunidad de confirmar.
- **TUR-10**: `confirmationRequestedAt` (la columna que marca la solicitud
  como enviada) se escribía junto con el asiento de auditoría *antes* de
  renderizar y enviar el mensaje. Si el envío fallaba después de esa
  escritura, la columna quedaba de todos modos marcada, y
  `AppointmentAutoCancellationCron` — que sólo mira esa columna para saber
  si corresponde iniciar el plazo de 4h — cancelaba el turno por "falta de
  confirmación" sobre un mensaje que en los hechos nunca había salido.

La corrección extrajo la secuencia de guarda/renderizado/envío/auditoría de
`AppointmentConfirmationCron` a un servicio propio,
`AppointmentConfirmationService`, para que tanto la corrida horaria del job
como `AppointmentsService.book()` pudieran compartirla sin duplicarla.
`book()` ahora, al terminar de reservar el turno, evalúa si su propia
anticipación ya es menor o igual a la cota superior de la ventana de
detección y, si lo es, dispara la solicitud de inmediato en la misma
operación de reserva, en lugar de esperar una corrida del job que para ese
turno podría llegar demasiado tarde o no llegar nunca (TUR-5). Dentro del
servicio compartido, la marca de `confirmationRequestedAt` sigue
escribiéndose antes del envío — para reservar el turno frente a una corrida
concurrente que pudiera estar procesándolo al mismo tiempo — pero un
renderizado o envío que falla después de esa marca ahora la deshace,
dejando la columna otra vez en null para que una corrida posterior, mientras
el turno siga dentro de la ventana, pueda reintentarlo; el asiento de
auditoría `CONFIRMATION_SENT` se retrasó hasta después de que el envío
efectivamente se concreta (TUR-10).

## Decisiones y por qué

**Un servicio compartido, no un helper de fecha o una condición duplicada
en `book()`.** La alternativa más simple para TUR-5 —copiar dentro de
`book()` la lógica de armar el candidato, reclamarlo, renderizar y
enviar— habría dejado dos implementaciones de la misma secuencia
guardar/enviar/registrar, con el riesgo real de que una futura corrección
(como TUR-10, en el mismo ticket) sólo se aplicara a una de las dos copias.
Extraer `AppointmentConfirmationService` con un único método
`requestConfirmation` deja la secuencia completa en un solo lugar; el job y
`book()` sólo difieren en *quién* decide que un turno corresponde procesar
ahora, no en cómo se procesa una vez decidido.

**El servicio resuelve sus propios datos de paciente y profesional, en
lugar de exigirlos precargados por el llamador.** El job original
construía sus candidatos con una única consulta que ya traía embebidos el
nombre y el celular del paciente y el nombre del profesional, evitando una
consulta adicional por turno. `book()`, en cambio, sólo tiene a mano los
identificadores del turno recién creado. Exigirle al servicio compartido
que recibiera esos datos ya resueltos habría obligado a `book()` a hacer
las mismas dos consultas por su cuenta antes de llamarlo, sin ahorrar nada;
que el propio servicio las resuelva internamente evita esa duplicación, al
costo de dos consultas adicionales por turno en la corrida del job en lugar
del único `join` que tenía antes. Se aceptó ese costo porque, a la escala
de una sola clínica, es exactamente el mismo criterio que ya rige para no
denormalizar `organizationId` en ningún lado del esquema: la
denormalización (aquí, el `join` precargado) sólo se reconsidera frente a
un problema medido, nunca de forma preventiva.

**La deshecha de la marca ante un envío fallido se protege con el
`timestamp` exacto que la propia llamada escribió, no con una condición
genérica de "no nulo".** Deshacer la marca con una condición amplia habría
corrido el riesgo de borrar, por accidente, una marca distinta escrita por
una llamada concurrente entre el momento en que esta llamada la puso y el
momento en que decide deshacerla. Guardar el `timestamp` exacto que esta
misma llamada escribió y usarlo como condición de la deshecha hace que sólo
pueda deshacer su propia escritura, nunca la de otra.

**El asiento de auditoría se retrasó hasta después del envío, y ya no
comparte transacción con la marca de la columna.** El diseño anterior
escribía la marca y el asiento de auditoría dentro de la misma transacción,
antes de enviar — lo que garantizaba que ambos se confirmaran juntos, pero
también que el asiento `CONFIRMATION_SENT` pudiera mentir sobre un envío
que después fallaba. Como el asiento ahora sólo tiene sentido escribirlo
una vez conocido el resultado real del envío —una llamada de red externa
que no puede, de todos modos, ejecutarse dentro de una transacción de base
de datos—, deja de tener sentido envolverlo junto con la marca en una
transacción: la marca se escribe sola (una sola instrucción ya es atómica
por sí misma) y el asiento de auditoría se escribe, también solo, únicamente
si el envío tuvo éxito.

## Entidades / puertos / adaptadores tocados

- `src/appointments/appointment-confirmation.service.ts` (nuevo):
  `AppointmentConfirmationService`, con el método `requestConfirmation` que
  antes vivía dentro de `AppointmentConfirmationCron`, y la función pura
  `isDueForImmediateConfirmationRequest` que decide si un turno, al
  momento de evaluarlo, ya está dentro o más allá de la cota superior de
  la ventana de detección.
- `src/appointments/appointment-confirmation.cron.ts`: se simplificó a
  construir el lote de candidatos (ahora sin `join` a paciente/profesional)
  y delegar cada uno al servicio nuevo.
- `src/appointments/appointments.service.ts`: `book()` evalúa, sobre los
  turnos recién creados, cuáles ya están dentro de la ventana de
  detección y dispara `AppointmentConfirmationService.requestConfirmation`
  para esos — a mejor esfuerzo, igual que el resto de los efectos
  posteriores a la reserva en este mismo servicio. El usuario SYSTEM que
  atribuye ese envío se resuelve con `resolveSystemActor`, el mismo
  utilitario que ya usa la capa conversacional para atribuir escrituras
  sin una request HTTP detrás.
- `src/appointments/appointments.module.ts`: registra
  `AppointmentConfirmationService` como proveedor del módulo, consumido
  tanto por el job como por `AppointmentsService`.
- `prisma/schema.prisma`: se amplió el comentario de
  `Appointment.confirmationRequestedAt` para documentar que ahora puede
  dispararse desde `book()` además del job, y que sólo queda marcada una
  vez que el envío efectivamente se concreta. Cambio de comentario
  únicamente, sin migración.

## Tests y qué validan

- `appointment-confirmation.service.spec.ts` (nuevo): la secuencia
  guardar/renderizar/enviar/auditar en aislamiento — omite un turno sin
  celular sin marcarlo; no reclama (ni envía) un turno cuya fila ya
  cambió de estado o ya fue reclamada por otra corrida; el caso central de
  TUR-10, que un envío fallido deshace la marca y no deja asiento de
  auditoría; el mismo caso cuando falla el renderizado en lugar del envío;
  y que un envío fallido nunca se propaga como excepción al llamador.
- `appointment-confirmation.cron.spec.ts`: reescrito para verificar la
  construcción del lote y la delegación a un `AppointmentConfirmationService`
  simulado, ya no la secuencia de envío en sí misma (cubierta en el
  archivo anterior).
- `appointments.service.spec.ts`: se agregó un bloque dedicado que
  verifica que `book()` dispara la solicitud inmediata para un turno
  reservado ya dentro de la ventana, que no la dispara para uno reservado
  bien fuera de ella, y que ni una falla al resolver el usuario SYSTEM ni
  una falla del propio envío impiden que la reserva se complete y se
  devuelva con normalidad.
- `test/appointment-confirmation.e2e-spec.ts`: se corrigió para simular el
  puerto de mensajería en lugar de usar el adaptador real de WhatsApp —el
  comentario original decía que el adaptador ligado en pruebas "nunca
  lanza", una descripción que quedó desactualizada desde que ese adaptador
  dejó de ser un simulador (sección 4.6); con el adaptador real, cualquier
  corrida en un entorno sin credenciales válidas fallaba el envío, y bajo
  el comportamiento nuevo (TUR-10) eso alcanzaba para que la prueba del
  caso feliz fallara también.
- `test/appointment-reassignment.e2e-spec.ts`: se actualizó una prueba que
  reserva un turno de reemplazo con pocas horas de anticipación y
  esperaba que no se enviara ningún mensaje; con la corrección de TUR-5 ese
  turno cae dentro de la ventana de detección y `book()` le dispara
  correctamente la solicitud de confirmación, un efecto correcto y no
  relacionado con el mecanismo de reasignación que la prueba en sí
  verifica.

Suite completa sin regresiones nuevas: 88 suites unitarias / 908 pruebas en
verde, 51 suites e2e / 606 pruebas, de las cuales 2 fallan por una
inestabilidad preexistente y ya documentada de la suite completa de
reprogramación (ver [[FASE-3_PROMPT-14]] y la nota de TASK-78 sobre
`--runInBand`): se reproduce de forma idéntica corriendo el repositorio sin
los cambios de este ticket, y ambas pruebas pasan al ejecutarse en
aislamiento. `lint` y `tsc --noEmit` sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-157-appointment-confirmation-reliability`
  (creada desde `origin/main` fresco, tras el merge de TASK-153). Pusheada
  a `origin`; PR abierto, no fusionado aún.
- Ticket: TASK-157 ("Turnos de corto plazo nunca reciben confirmación, y
  fallas de envío igual disparan la cancelación automática"), tarea de
  auditoría automática (hallazgos TUR-5 y TUR-10) validada por la usuaria
  antes de implementarse. Misma convención de bitácora dedicada para una
  corrección puntual que TASK-129 ([[FASE-4_PROMPT-14]]) y TASK-84
  ([[FASE-2_PROMPT-11]]).
