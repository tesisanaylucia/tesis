# Fase 4 — Notificaciones y Scheduler (backend) — modelo Notificacion y endpoint GET /notificaciones para la app del profesional (TASK-76, P7.b)

## Qué se implementó

Un canal de notificaciones dentro de la propia aplicación, distinto del
canal de mensajería por WhatsApp que el resto del módulo construye para el
paciente: un modelo nuevo de Prisma, `Notification`, que registra un evento
puntual dirigido a un profesional —hoy, la cancelación de uno de sus turnos
o la reasignación automática de un turno liberado a un candidato de su
lista de espera— junto con el texto descriptivo del evento, generado por el
propio backend, y si el profesional ya lo leyó o no.

Sobre ese modelo se construyeron cuatro operaciones: el profesional
autenticado obtiene su propio listado, opcionalmente filtrado por leído/no
leído y paginado; marca una notificación puntual como leída; marca todas
las suyas pendientes como leídas de una sola vez; y un administrador puede
consultar el listado de cualquier profesional del mismo inquilino. La
creación de una notificación no es una operación expuesta por HTTP en sí
misma, sino un efecto colateral que dos servicios ya existentes disparan
cuando ocurre el evento que describen: `AppointmentsService`, al cancelar
un turno —tanto por la vía ordinaria como por la cancelación masiva que
dispara la ausencia de un profesional (sección 3.2.3)—, y
`WaitlistReassignmentService`, al reservar el turno liberado para el
candidato de la lista de espera que aceptó la oferta (también sección
3.2.3).

El requisito nombra además dos orígenes de notificación que todavía no
tienen ningún servicio del que colgarse: una solicitud de receta nueva por
parte de un paciente, y una alerta ante un error de la cerradura
inteligente. Ambos quedaron representados en el enumerado de tipos de
notificación, pero sin ningún punto de disparo real: la entidad de
solicitud de receta existe en el esquema desde una fase muy anterior del
proyecto (sección 3.2.2) pero nunca tuvo un servicio propio que la
gestionara, y la entidad de código de acceso de la cerradura todavía no
existe en absoluto, a la espera de la fase dedicada a la integración con
TTLock.

## Decisiones y por qué

**El módulo nuevo se llamó `InAppNotificationsModule`, no
`NotificationsModule`.** Ese segundo nombre ya pertenece al motor de
plantillas de mensajes construido al comienzo de este mismo módulo (más
arriba en esta misma sección): un componente sin relación conceptual con
este, que arma el texto de un mensaje saliente por WhatsApp hacia el
paciente a partir de una clave y parámetros, y que nunca escribe en la
tabla `Notification`. Reutilizar el nombre habría hecho que dos conceptos
distintos —una plantilla de mensaje saliente y un evento interno para la
propia aplicación— compartieran una única palabra en el código, exactamente
la clase de ambigüedad que el resto del proyecto evita nombrando cada
concepto por lo que hace.

**El modelo `Notification` no lleva columna de organización propia.** El
propio esquema documenta una regla general desde su encabezado: una entidad
alcanzada por un único padre acotado por inquilino no debe repetir esa
columna, porque el valor repetido puede llegar a discrepar del que ya tiene
el padre, una corrupción silenciosa que ninguna consulta puede detectar por
sí sola. `Notification` tiene un único padre de ese tipo, el profesional al
que pertenece, así que sigue el mismo patrón ya usado para la matrícula o
el horario de atención de un profesional: sin columna de organización
propia, alcanzable solo a través de ese profesional, y con cada método del
servicio anclado primero en la comprobación de que ese profesional
pertenece al inquilino de quien pregunta antes de tocar ninguna fila. El
propio documento de requisitos pedía la columna de organización de forma
literal, pero se interpretó, igual que otras veces a lo largo del proyecto,
como parte de la descripción funcional del requisito —"que las
notificaciones respeten el límite del inquilino"— y no como un contrato
textual sobre qué columnas debía tener la tabla, dado que la regla general
del esquema ya construida resuelve esa misma garantía sin necesidad de la
columna.

**Marcar una notificación ajena como leída devuelve 403, apartándose de la
convención general del proyecto de responder 404 ante cualquier intento
fuera del propio alcance.** El resto del sistema trata sistemáticamente un
recurso fuera del alcance de quien pregunta como si no existiera, para no
confirmar por el código de estado que el recurso sí existe en la base de
datos. Aquí, sin embargo, el propio documento de requisitos exige
explícitamente el código 403 como criterio de aceptación verificable para
este caso puntual —un profesional que intenta marcar como leída la
notificación de otro—, así que se siguió el texto literal del requisito en
lugar de la convención general, del mismo modo en que otras decisiones de
este proyecto se apartan puntualmente de un criterio por defecto cuando el
propio ticket lo pide de forma expresa y verificable.

**Los nombres de campo de las consultas se tradujeron al inglés** —`read`
en lugar de `leidas`, `professionalId` en lugar de `profesional`—, mismo
criterio ya aplicado en el módulo de Motor de Turnos al definir los
parámetros de consulta del listado de turnos: el texto en español del
ticket describe la funcionalidad, no un contrato literal sobre el nombre
del parámetro, y el contrato HTTP de este proyecto es en inglés salvo la
ruta misma.

**La notificación de cancelación se disparó también desde la cancelación
masiva por ausencia del profesional, no solo desde la cancelación
individual que el ticket nombra explícitamente por su nombre de método.**
El documento de requisitos, al describir el disparador, nombra únicamente
el método de cancelación individual, pero la fuente de verdad que cita ese
mismo documento —la necesidad funcional de "notificaciones in-app de
cancelaciones"— no distingue entre ambos orígenes: para el profesional que
consulta su listado, un turno cancelado es un turno cancelado, sin importar
si la cancelación la pidió una persona puntual o la disparó automáticamente
la ausencia registrada. Ambos caminos ya convergían, antes de esta tarea,
en la misma función auxiliar que dispara el mecanismo de reasignación, así
que la notificación se agregó a esa misma función compartida en lugar de
duplicarla en cada uno de los dos métodos que la invocan.

**La creación de una notificación no interrumpe la operación que la
origina si falla.** Igual que ya ocurre con el disparo del mecanismo de
reasignación tras una cancelación, escribir una notificación se envolvió en
su propio bloque de manejo de error, con el fallo apenas registrado en el
log: una cancelación de turno ya válida y ya escrita en la base de datos no
debe convertirse en un pedido fallido solo porque el aviso posterior al
profesional no pudo escribirse.

**El identificador de la tabla es un UUID generado por la base de datos, no
un entero autoincremental como pide literalmente el documento de
requisitos.** Todas las demás entidades del esquema, sin excepción, usan el
mismo mecanismo de identificador —un UUID generado con la extensión
`pgcrypto` de PostgreSQL— por una razón ajena a esta tarea puntual: expone
menos información sobre el volumen de filas de la tabla y evita que un
identificador secuencial permita adivinar identificadores vecinos. Se
mantuvo la consistencia con el resto del esquema en lugar de introducir el
único caso de un identificador con una forma distinta.

## Entidades / puertos / adaptadores tocados

- `Notification` (modelo nuevo de Prisma): evento dirigido a un profesional
  puntual, con tipo (`NotificationType`: `APPOINTMENT_CANCELLED` /
  `APPOINTMENT_REASSIGNED` / `PRESCRIPTION_REQUESTED` / `LOCK_ERROR`),
  referencia opcional al turno o a la solicitud de receta que lo origina,
  texto descriptivo y marca de leído.
- `InAppNotificationsModule` (nuevo, `src/in-app-notifications/`):
  `InAppNotificationsService` (creación interna; listado propio; marcar una
  o todas las notificaciones como leídas; listado por profesional para uso
  administrativo), `InAppNotificationsController` (`/notificaciones`, rol
  profesional) y `AdminNotificationsController`
  (`/admin/notificaciones`, rol administrador).
- `AppointmentsService`: la cancelación individual y la cancelación masiva
  por ausencia comparten ahora, además del disparo ya existente del
  mecanismo de reasignación, el disparo de la notificación de cancelación
  al profesional del turno.
- `WaitlistReassignmentService`: la reserva del turno de reemplazo para el
  candidato que acepta la oferta dispara la notificación de reasignación al
  profesional, tras confirmarse la transacción.

## Tests agregados

- `in-app-notifications.service.spec.ts` (nuevo, 14 pruebas): creación de
  una notificación; listado propio filtrado por leído/no leído y paginado;
  rechazo cuando quien pregunta no tiene profesional asociado; listado por
  profesional con anclaje de inquilino vía `assertOwned`, incluyendo el 404
  cuando el profesional pertenece a otro inquilino; marcar como leída,
  incluyendo el 404 por identificador inexistente o inquilino ajeno y el
  403 cuando la notificación pertenece a otro profesional; marcar todas
  como leídas.
- `test/in-app-notifications.e2e-spec.ts` (nuevo, 8 pruebas, base de datos
  real): el flujo completo que exige el propio criterio de aceptación del
  ticket —cancelar un turno por HTTP, obtener la notificación por HTTP,
  marcarla como leída, comprobar que desaparece del listado filtrado por no
  leídas—; marcar todas las notificaciones pendientes como leídas de una
  sola vez; el 403 cuando otro profesional intenta marcar una notificación
  ajena; el listado administrativo de cualquier profesional del inquilino y
  su 404 cuando lo pide un administrador de otro inquilino; el 401 sin
  autenticación y el 403 de un administrador sobre la ruta pensada para el
  profesional; y el flujo de reasignación —turno cancelado con lista de
  espera automática, oferta aceptada mediante el mismo mecanismo que ya usa
  la prueba de reasignación del Motor de Turnos— verificando que la
  notificación resultante llega al listado del profesional.
- Ejecución: 38 suites unitarias / 411 pruebas en verde; 36 suites e2e / 425
  pruebas en verde (`--runInBand`). Lint y verificación de tipos sin
  errores. Migración aplicada contra Postgres local
  (`prisma migrate deploy`).

## Figuras pendientes

Una figura nueva: diagrama entidad-relación y de flujo del modelo
`Notification` (el profesional como único padre acotado por inquilino, las
dos referencias opcionales hacia turno y solicitud de receta, y los dos
puntos de disparo ya conectados —cancelación de turno y reserva por
reasignación— frente a los dos todavía sin conectar —solicitud de receta y
error de cerradura, señalados como pendientes de una fase posterior).
Agregada a `figuras_pendientes.md` como figura 28, sección 3.2.4.

## Marco Teórico ofrecido

El tema "2.4 Arquitectura de software (extensión: scheduling)" que habilita
la Fase 4 según `mapa_fases_capitulos.md` había quedado sin ofrecerse ni
redactarse en las cinco tareas anteriores de esta fase (TASK-42 a TASK-45 y
TASK-82). Se ofreció explícitamente en esta sesión, en lugar de volver a
diferirlo, y la usuaria pidió redactarlo ahora: se agregaron a
`cap2_marco_teorico_a.md`, sección 2.4, tres párrafos conceptuales sobre
ejecución de tareas programadas (disparo por intervalo sin petición
externa, idempotencia frente a corridas repetidas, escritura condicionada
al estado esperado como defensa ante concurrencia con una acción manual, e
iteración explícita por organización en un sistema multi-tenant), sin
atarlos a ninguna decisión concreta de PSIQUE, con tres marcadores nuevos
`[CITA: ...]` agregados a `referencias_pendientes.md` (filas 22 a 24). El
título de la sección 2.4 se amplió para nombrar el tema nuevo.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-76-notification-model-and-endpoint`
  (creada desde `origin/main` fresco). Pusheada a `origin`, no fusionada
  aún.
- Ticket: TASK-76 (P7.b — Backend: modelo Notificacion y endpoint
  GET /notificaciones para la app del profesional). Numerado bajo la fase
  de Jira 7 (App móvil del profesional) pero de componente backend puro; se
  documentó en esta subsección (3.2.4 Notificaciones y Scheduler) en lugar
  de 3.2.7, por decisión de la usuaria, dado que `mapa_fases_capitulos.md`
  asigna la fase 7 íntegramente al componente móvil y esta tarea es la
  extensión del dominio de notificaciones ya cubierto aquí, con un
  disparador distinto (la app del profesional en vez de WhatsApp) en lugar
  de una pantalla o consumo de API móvil. Depende de TASK-38/TASK-40
  (cancelación y reasignación de turnos, [[FASE-3_PROMPT-2]],
  [[FASE-3_PROMPT-7]]), ya implementadas; declara además como dependencias
  TASK-52, TASK-55 y TASK-59 (solicitud de receta y error de cerradura), sin
  implementar todavía, por lo que esos dos orígenes de notificación quedan
  modelados sin conectar, señalado explícitamente en el propio código con
  el ticket que debe resolverlo.
