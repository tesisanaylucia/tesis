# Fase 4 — Notificaciones y Scheduler (backend) — timeout real de 4h para la oferta de turno en lista de espera, modalidad AUTOMÁTICA (TASK-82, P4.5)

## Qué se implementó

El mecanismo real de espera con el que un candidato de la lista de espera
responde a la oferta de un turno liberado, bajo reasignación AUTOMÁTICA
(P3.7, TASK-40). Hasta esta tarea, `WaitlistResponsePort` —el puerto que
representa "¿el paciente aceptó el turno ofrecido?"— solo tenía un
adaptador stub que resolvía sincrónicamente a "no" en el mismo instante en
que se lo llamaba, porque ni el canal real de WhatsApp (M5) ni la ventana
de timeout (este ticket, M4) existían todavía. El propio puerto documentaba
esa limitación desde que se creó.

Se agregó un modelo nuevo de Prisma, `WaitlistOffer`, que registra cada
intento de ofrecer el turno liberado a un candidato puntual: a qué turno y
a qué entrada de la lista de espera corresponde, a qué paciente se ofreció,
el estado (`PENDING`, `ACCEPTED`, `REJECTED`, `EXPIRED`) y las marcas de
tiempo de oferta y de respuesta. El adaptador stub se reemplazó por uno
real, respaldado por esta tabla: en vez de resolver sincrónicamente,
registra la oferta y consulta su estado.

Ese cambio de fondo obligó a rediseñar cómo `WaitlistReassignmentService`
recorre la lista de espera bajo modalidad AUTOMÁTICA. Antes, el recorrido
completo —ofrecer a cada candidato, uno tras otro, hasta que alguno aceptara
o se agotara la lista— ocurría de manera sincrónica dentro de una sola
llamada, porque el stub respondía al instante. Una espera real de hasta
cuatro horas no puede bloquear ese recorrido de esa forma, así que el
servicio ahora ofrece a un único candidato por vez y se detiene: el turno
queda retenido (`holdUntil`) mientras esa oferta sigue pendiente, y avanzar
al siguiente candidato —o reservar el turno para quien aceptó— pasó a ser
responsabilidad de dos métodos nuevos y compartidos, `acceptOffer` y
`expireOffer`, que cualquier disparador externo puede invocar: hoy el cron
de esta misma tarea, y en el futuro un webhook real de WhatsApp (M5) cuando
el paciente responda. Un candidato sin celular registrado en su ficha se
trata como un rechazo instantáneo en vez de dejarlo esperando cuatro horas
que nadie podría responder, preservando el comportamiento previo a esta
tarea para ese caso puntual.

Se agregó `WaitlistOfferTimeoutCron`, un trabajo programado nuevo que corre
cada quince minutos y recorre cada organización bajo su propio contexto de
inquilino, con la misma infraestructura que los demás trabajos de este
módulo (`AppointmentConfirmationCron`, `AppointmentAutoCancellationCron`,
`AppointmentReminderCron`). Por cada oferta `PENDING` cuya marca de tiempo
de oferta supera las cuatro horas, la marca como `EXPIRED` y dispara el
avance al siguiente candidato —el mismo camino que seguiría un rechazo
explícito—, dejando un registro en la auditoría del evento.

## Decisiones y por qué

**La retención del turno liberado (`holdUntil`) pasó de ser un techo fijo
de sesenta minutos a seguir la ventana de respuesta real de la oferta
vigente.** Antes de esta tarea, el propio comentario en el código
reconocía que ese valor de sesenta minutos no era la regla de negocio real
("cuánto tiempo tiene un candidato para responder"), sino apenas una
salvaguarda para que un proceso que se cayera a mitad del recorrido no
dejara el turno retenido para siempre —porque el recorrido completo, al
ser sincrónico, terminaba en segundos de cualquier manera. Con el
recorrido ahora distribuido en el tiempo real, esa salvaguarda dejó de
tener sentido tal como estaba: se reemplazó por la ventana de cuatro horas
de la oferta vigente, más un margen de quince minutos que absorbe la
cadencia del propio cron de expiración, y se renueva cada vez que el
recorrido avanza a un candidato nuevo.

**Se agregó una tabla nueva (`WaitlistOffer`) en lugar de apoyarse en
columnas sueltas sobre `WaitlistEntry` o `Appointment`.** Una entrada de
lista de espera puede recibir, a lo largo de su vida, más de una oferta
—si la primera vence sin respuesta, se le puede volver a ofrecer un turno
distinto liberado más adelante—, y un turno liberado puede haber sido
ofrecido a más de un candidato en el mismo recorrido antes de que alguno
acepte. Ninguna de las dos relaciones es uno a uno, así que ninguna de las
dos tablas existentes podía representarla sin ambigüedad; una tabla nueva,
con una clave foránea compuesta hacia cada uno de sus dos padres acotados
por inquilino (turno y paciente, además de una referencia opcional hacia
la entrada de la lista de espera), sigue la misma regla 4 del esquema ya
aplicada a `Appointment` y `WaitlistEntry`: dos padres acotados por
inquilino, así que la columna de organización es la que fuerza a ambos a
coincidir en el mismo inquilino.

**La referencia hacia la entrada de la lista de espera es opcional y no se
libera con `SET NULL` de la base de datos, sino explícitamente desde el
servicio.** Cuando un candidato acepta una oferta, la entrada de lista de
espera correspondiente se elimina —comportamiento que ya existía desde
TASK-40—, pero la oferta debe sobrevivir a esa eliminación como registro
de qué se ofreció y qué se aceptó. Postgres no permite que una clave
foránea compuesta se resuelva con `SET NULL` cuando una de sus columnas
—en este caso, el identificador de organización— es obligatoria: intentarlo
generó una advertencia explícita de Prisma al validar el esquema. La
solución fue anular esa columna desde el propio servicio, dentro de la
misma transacción, inmediatamente antes de eliminar la entrada, en vez de
delegarle esa responsabilidad a la base de datos.

**Un candidato sin celular registrado se trata como rechazo instantáneo, no
como una oferta que expira a las cuatro horas.** El comportamiento previo a
esta tarea ya tomaba esa misma decisión —el propio comentario en el código
lo explicaba como "no puede ser contactado, lo que equivale a que no
responda"—; esta tarea lo preservó explícitamente al migrar hacia el
mecanismo real, en vez de dejar que un candidato incontactable ocupara
cuatro horas de la ventana de reasignación sin ninguna posibilidad real de
responder.

**`acceptOffer` y `expireOffer` son dos métodos públicos separados que
comparten toda su lógica interna**, en vez de un único método genérico de
"responder a la oferta" expuesto públicamente. La justificación es de cara
al futuro: cuando el webhook real de WhatsApp (M5) exista, va a necesitar
invocar la aceptación de una oferta con la misma forma exacta en que hoy la
invocan las pruebas; exponer dos métodos con nombres que describen la
intención de quien los llama deja esa integración futura más clara que un
único método con un parámetro de estado genérico.

## Entidades / puertos / adaptadores tocados

- `WaitlistOffer` (modelo nuevo de Prisma): oferta de un turno liberado a
  un candidato puntual de la lista de espera, con estado
  (`WaitlistOfferStatus`: `PENDING`/`ACCEPTED`/`REJECTED`/`EXPIRED`) y
  marcas de tiempo de oferta y respuesta.
- `WaitlistResponsePort` (`src/domain/ports/waitlist-response.port.ts`):
  cambió de forma — antes exponía `awaitResponse(oferta): Promise<boolean>`
  que resolvía sincrónicamente; ahora expone `registerOffer(oferta):
  Promise<string>` (persiste y devuelve el identificador de la oferta) y
  `getStatus(id): Promise<estado>`.
- `WaitlistResponseAdapter` (nuevo, reemplaza a `StubWaitlistResponseAdapter`,
  eliminado): implementación real del puerto, respaldada por la tabla
  `WaitlistOffer` a través del cliente de Prisma acotado por inquilino.
- `WaitlistReassignmentService`: el recorrido de la lista de espera bajo
  modalidad AUTOMÁTICA se reescribió para ofrecer a un único candidato por
  vez; se agregaron los métodos públicos `acceptOffer` y `expireOffer`
  como puntos de entrada compartidos para avanzar el recorrido o reservar
  el turno.
- `WaitlistOfferTimeoutCron` (nuevo, `src/waitlist/waitlist-offer-timeout.cron.ts`):
  trabajo programado que corre cada quince minutos y expira las ofertas
  `PENDING` que superaron las cuatro horas.
- `waitlist.constants.ts`: la constante `AUTOMATIC_REASSIGNMENT_HOLD_SAFETY_MINUTES`
  (el techo fijo de sesenta minutos) se reemplazó por
  `WAITLIST_OFFER_RESPONSE_TIMEOUT_HOURS` (la regla de negocio real, cuatro
  horas) y `WAITLIST_OFFER_HOLD_SAFETY_MINUTES` (el margen de cadencia del
  cron).

## Tests agregados

- `waitlist-reassignment.service.spec.ts` (reescrito): cubre el nuevo
  recorrido de a un candidato por vez —ofrece al primero y se detiene sin
  reservar todavía—, el rechazo instantáneo por falta de celular con avance
  automático al siguiente candidato, `acceptOffer` reservando el turno y
  sin avanzar más, `expireOffer` marcando la oferta vencida y avanzando al
  siguiente candidato o liberando la retención si no queda ninguno, y una
  prueba de condición de carrera (el cron de timeout llega después de que
  la oferta ya fue aceptada por otro camino, y no produce ningún efecto).
  El ranking por prioridad y el manejo de un candidato sin vínculo previo
  con el profesional, ya cubiertos desde TASK-40, se conservaron.
- `waitlist-offer-timeout.cron.spec.ts` (nuevo, 6 pruebas): expira una
  oferta con más de cuatro horas sin respuesta; no toca una de tres horas
  cincuenta y nueve minutos; no toca una oferta ya `ACCEPTED` sin importar
  su antigüedad; una organización sin usuario `SYSTEM` no interrumpe la
  corrida; una falla en una oferta no detiene el resto del lote; la
  corrida recorre cada organización bajo su propio contexto de inquilino.
- `integrations.module.spec.ts` (ajustado): la prueba de resolución del
  puerto por inyección de dependencias pasó de simular una respuesta
  sincrónica a registrar una oferta real y leer su estado de vuelta.
- Extremo a extremo: `waitlist-offer-timeout.e2e-spec.ts` (nuevo, 3
  pruebas) reproduce con reloj simulado —fijando directamente la marca de
  tiempo de la oferta al crear el dato de prueba, sin simular
  `Date.now()`, siguiendo el mismo criterio ya usado en
  `appointment-confirmation.e2e-spec.ts`— los tres criterios de aceptación
  del propio ticket: una oferta sin respuesta a las 4h01m expira y avanza
  al siguiente candidato; una oferta sin respuesta a las 3h59m sigue
  pendiente; una oferta aceptada antes del vencimiento queda intacta y su
  reserva se mantiene, aun cuando el cron corra sobre ella. Se
  reescribieron además `appointment-reassignment.e2e-spec.ts` y las
  pruebas de reasignación dentro de `appointment-engine-integration.e2e-spec.ts`,
  que dependían de la forma sincrónica anterior del puerto: donde antes se
  aceptaba o rechazaba una oferta con un doble de prueba, ahora se invoca
  `acceptOffer`/`expireOffer` directamente sobre el servicio real, dentro
  de su propio contexto de inquilino.
- Ejecución: 37 suites unitarias / 397 pruebas en verde; 35 suites e2e /
  417 pruebas en verde (`--runInBand`). Lint limpio y verificación de tipos
  sin errores. Migración aplicada contra Postgres local
  (`prisma migrate deploy`); la base de desarrollo de esta máquina no tenía
  su historial de migraciones registrado en la tabla de Prisma pese a tener
  ya todas las tablas anteriores —una situación previa a esta tarea, no
  causada por ella—, así que se estableció como línea base antes de aplicar
  la migración nueva.

## Figuras pendientes

Una figura nueva: diagrama de estados de una oferta de lista de espera
(`PENDING` → `ACCEPTED`/`REJECTED`/`EXPIRED`; qué evento dispara cada
transición — respuesta real por WhatsApp aún no construida, celular
ausente, vencimiento del cron de timeout— y qué ocurre del lado del
recorrido en cada caso). Agregada a `figuras_pendientes.md` como figura 27,
sección 4.5.

Además, la Figura 23 (diagrama de secuencia del algoritmo de reasignación
automática, registrada en Fase 3 y todavía sin dibujar) describía el
recorrido sincrónico anterior a esta tarea —"oferta vía MessagingPort y
consulta a WaitlistResponsePort por candidato" dentro de un único paso—,
que esta tarea reemplazó. Se actualizó su descripción en
`figuras_pendientes.md` para reflejar el recorrido de a un candidato por
vez y remitir al mecanismo de esta tarea.

## Marco Teórico ofrecido

El tema "2.4 Arquitectura de software (extensión: scheduling)" que habilita
la Fase 4 según `mapa_fases_capitulos.md` sigue sin ofrecerse ni redactarse
en ninguna de las cinco tareas de esta fase hasta el momento (TASK-42 a
TASK-45, y esta). Queda pendiente de ofrecer a la usuaria.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-82-waitlist-offer-timeout-cron`
  (creada desde `origin/main` fresco). Pusheada a `origin`, no fusionada
  aún.
- Ticket: TASK-82 (P4.5 — Timeout real de 4h para la oferta de turno en
  lista de espera, modalidad AUTOMÁTICA), quinta tarea de Módulo 4.
  Depende de TASK-40 (algoritmo de reasignación, [[FASE-3_PROMPT-7]]) y de
  la infraestructura de cron ya sentada por TASK-43/TASK-44
  ([[FASE-4_PROMPT-2]], [[FASE-4_PROMPT-3]]). Deja pendiente, declarado
  fuera de alcance por el propio ticket: el algoritmo de recorrido de la
  lista de espera en sí (ya implementado en TASK-40, sin cambios en esta
  tarea) y el envío real de la oferta por WhatsApp (M5).
