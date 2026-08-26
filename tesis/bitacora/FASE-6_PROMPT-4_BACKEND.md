# Fase 6 — Cerradura TTLock (backend) — Envío cifrado del código y apertura ad-hoc (TASK-58)

## Qué se implementó

Se cerraron los dos procesos que el Módulo Cerradura Electrónica todavía
tenía pendientes según el SRS: la entrega del código al paciente una vez
que queda activo, y la apertura ad-hoc que un profesional o administrador
puede solicitar ante una falla del flujo normal.

**Envío del código al confirmar el turno.** `AppointmentsService.confirm`
ya llamaba a `AccessCodeService.generateForAppointment` desde TASK-56; el
método privado que envolvía esa llamada (antes `generateAccessCode`) se
renombró a `sendAccessCode` y se amplió para, sólo cuando
`generateForAppointment` devuelve un `CODIGO_ACCESO` realmente activo,
renderizar una nueva plantilla (`ACCESS_CODE_DELIVERY`, siguiendo el motor
de plantillas de P4.1/TASK-42) con el PIN y la fecha del turno, y enviarla
al paciente reutilizando el helper `notifyPatient` que ya usaban la
cancelación y la reprogramación. La verificación "no enviar si el código no
quedó activo" no necesitó código propio: `generateForAppointment` ya
devuelve `null` para cualquier caso que no termine en una fila activa (una
falla de `LockPort`, o una verificación de instalación que resultó
negativa), así que un resultado nulo simplemente nunca llega al envío.

**Apertura ad-hoc.** Se agregó `AccessCodeService.generateAdhoc`, el nuevo
endpoint `POST /cerradura/abrir-adhoc` (rol profesional o administrador del
propio inquilino) y un módulo controlador nuevo,
`AccessCodesController`. El flujo sigue el que describe el propio ticket:
genera un `CODIGO_ACCESO` temporal de quince minutos, fuera de cualquier
turno, vía `LockPort.createTemporaryCode`, lo verifica instalado igual que
el flujo normal, lo persiste con `id_turno` nulo, deja un asiento en
`REGISTRO_AUDITORIA` con la acción `ACCESS_CODE_ADHOC_OPENING` (traducción
al inglés de "APERTURA_ADHOC", misma convención que `ACCESS_CODE_EXPIRED`)
y devuelve el PIN en la respuesta HTTP — no por `MessagingPort`, ver más
abajo.

**Cambio de esquema.** Que el código ad-hoc no esté asociado a ningún turno
("id_turno=null en CODIGO_ACCESO", como lo pide el propio ticket) chocaba
con el diseño que TASK-55 le había dado a `AccessCode`: hijo de
`Appointment`, sin columna de organización propia, acotado por inquilino
únicamente a través de ese padre obligatorio. Un código sin turno no tiene
ningún padre del que heredar esa garantía. Se resolvió agregando
`organizationId` directamente a `AccessCode` —pasa a ser un caso híbrido,
acotado por inquilino de forma directa (como `Holiday` u
`OrganizationConfig`) y con una clave foránea compuesta *opcional* hacia
`Appointment`, sólo poblada para el flujo normal por turno— en lugar de
quitarle a `AccessCode` el acotamiento por inquilino por completo, que
habría dejado sin filtro automático a una tabla que gobierna el acceso
físico al edificio. La clave foránea compuesta usa el mismo mecanismo que
ya prueba `WaitlistOffer.waitlistEntryId` desde una fase anterior: Postgres
no exige la restricción mientras la columna opcional sea nula, así que un
código ad-hoc queda fuera de su alcance sin necesidad de una excepción
explícita.

## Decisiones y por qué

**El envío se ató a la confirmación, no a la generación en general.** El
propio código de `generateForAppointment` tiene un único llamador
(`AppointmentsService.confirm`) bajo el flujo normal; la reprogramación
(`rescheduleCore`, TASK-57) también puede generar un código nuevo para la
fecha nueva, pero deliberadamente no dispara un nuevo envío: los criterios
de aceptación del ticket sólo cubren el turno confirmado, y el paciente ya
tiene un PIN válido para la fecha anterior hasta el momento mismo en que
ese método lo anula. Reenviar en cada reprogramación queda como una mejora
razonable, no como parte de esta tarea.

**La apertura ad-hoc lanza en vez de tragarse un error de `LockPort`,
al revés que la generación y la revocación por turno.** Esas dos existen
para proteger una mutación ajena (confirmar, cancelar, reprogramar) de una
falla de la integración física; en la apertura ad-hoc no hay ninguna otra
mutación que proteger, generar el código *es* todo el pedido. El
solicitante —una persona esperando frente a la puerta— necesita enterarse
en el momento de que no funcionó, no leerlo después en un registro de
`LockLog`. El error de `LockPort` se traduce a `ServiceUnavailableException`
(503).

**El PIN se devuelve en la respuesta HTTP, no por `MessagingPort`.** El
propio ticket ofrece esa alternativa explícitamente ("enviar... o retornar
en respuesta"), y no hay ningún número de celular registrado para un
`Professional` o un `User` en este sistema — el solicitante ya está
autenticado en la aplicación que hace el pedido, así que la respuesta
misma es el canal.

**El código ad-hoc no gana ninguna columna propia para distinguirse del
flujo normal.** `appointmentId` nulo ya es esa marca; agregar un campo de
"tipo" o de "origen" habría sido una redundancia que el propio esquema ya
resuelve.

**El cron de expiración dejó de unirse contra `Appointment` para su filtro
por inquilino.** Al pasar `AccessCode` a estar acotado por inquilino de
forma directa, la extensión de Prisma que filtra y sella automáticamente
por `organizationId` empezó a alcanzarlo sin necesidad del `join` manual
que el cron usaba desde TASK-57 — y, como efecto secundario correcto, el
barrido pasó a recoger también los códigos ad-hoc vencidos, sin ningún caso
especial: la misma consulta por `estado=activo` y `valido_hasta` vencido
los alcanza a todos por igual.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: `AccessCode` gana `organizationId` (acotamiento
  directo) y `appointmentId` pasa a ser opcional, con la clave foránea
  compuesta descripta arriba; migración escrita a mano (con
  retrocompletado de `organizationId` para las filas existentes a partir de
  su turno) porque Prisma no puede inferir el valor por defecto de una
  columna `NOT NULL` nueva sobre una tabla con datos.
- `AccessCodeService`: nuevo método `generateAdhoc`; nuevas dependencias
  `AuditService` (ya usada por el cron de expiración) para el asiento de
  auditoría.
- `AccessCodesController` (nuevo) y `AccessCodesModule` actualizado.
- `AccessCodeExpirationCron`: dejó de unirse contra `Appointment` para
  filtrar por inquilino.
- `NotificationTemplateKey.ACCESS_CODE_DELIVERY` (nueva plantilla, motor de
  P4.1/TASK-42).
- `AppointmentsService.confirm`: el método privado que generaba el código
  se renombró y amplió para enviar el mensaje.
- `CLAUDE.md`: sección de multi-tenencia actualizada para documentar el
  nuevo caso híbrido y sacar a `AccessCode` de la lista de ejemplos del
  patrón "hijo sin columna propia".

## Tests agregados o modificados

- `access-code.service.spec.ts`: nueva batería para `generateAdhoc` (código
  activo de quince minutos sin turno, auditoría, y las tres formas en que
  `LockPort`/la configuración pueden fallar, todas terminando en
  `ServiceUnavailableException` sin persistir nada).
- `appointments.service.spec.ts`: dos casos nuevos sobre `confirm` (envía
  el PIN cuando el código queda activo; no envía nada cuando no queda
  ninguno activo).
- `notification-template.service.spec.ts`: caso nuevo de la plantilla
  `ACCESS_CODE_DELIVERY` en la batería exhaustiva existente.
- `test/access-code-delivery-and-adhoc-opening.e2e-spec.ts` (nuevo, contra
  Postgres real): envío del PIN al confirmar con código activo; silencio
  cuando no queda activo; rechazo sin autenticación; alta ad-hoc por un
  administrador con su fila, su `LockLog` y su asiento de auditoría
  verificados directamente contra la base; alta ad-hoc también permitida a
  un profesional del propio inquilino; 503 sin persistir nada cuando la
  verificación de instalación falla.
- `test/lock-log.e2e-spec.ts`: ajustado para pasar el nuevo `organizationId`
  obligatorio al crear el `AccessCode` de su fixture.

## Figuras pendientes

- Diagrama de secuencia del envío del código al confirmar (transición ya
  comprometida → `generateForAppointment` → ¿código activo? → rama de
  envío, con la plantilla `ACCESS_CODE_DELIVERY` renderizada y despachada
  por `MessagingPort`, frente a la rama sin código, que no envía nada).
- Diagrama de secuencia de la apertura ad-hoc (pedido autenticado →
  generación y verificación en la cerradura → persistencia con turno nulo
  y auditoría, todo dentro de una transacción → respuesta con el PIN),
  contrastado con el de la Figura 48 para remarcar la diferencia: aquí una
  falla se propaga como 503 en lugar de callarse.
- Corrección a la Figura 47 (diagrama entidad-relación del subdominio de
  cerradura): `AccessCode` ya no es hijo puro de `Appointment` — pasa a
  llevar `organizationId` propio y una clave foránea compuesta opcional
  hacia el turno.

## Componente y referencia

Backend. Rama `feature/TASK-58-access-code-delivery-and-adhoc-opening`
(todavía no fusionada al momento de esta entrada). Suite completa
verde: 71 suites unitarias / 717 tests, 46 suites e2e / 529 tests
(`--runInBand`), lint y `tsc --noEmit` limpios.
