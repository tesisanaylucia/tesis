# Fase 3 — Motor de Turnos (backend) — la retención de 24h en modalidad manual bloqueaba también la asignación manual del profesional (TASK-113, corrección a TASK-80)

## Contexto

TASK-80 ([[FASE-3_PROMPT-13]]) implementó la retención de 24h que el SRS
exige sobre un turno liberado cuando el profesional tiene configurada la
modalidad de reasignación MANUAL. El criterio de aceptación original de ese
ticket era explícito sobre el límite de la retención: *"El profesional puede
seguir asignando el turno manualmente desde la app durante el hold — esto no
lo bloquea, solo bloquea la oferta automática/chatbot"*, en línea con el
texto del SRS (Anexo, Módulo Turnos, reasignación ante cancelación, inciso
b): *"el profesional se encarga de asignar el turno y el bot no interviene;
la franja queda libre en su agenda para que el profesional la asigne al
paciente que desee. El bot no podrá ofrecer ese turno durante 24 horas,
dándole tiempo al profesional de asignarlo por su cuenta"*.

Lo implementado no respetó esa distinción. `Appointment.holdUntil` guardaba
únicamente *hasta cuándo* dura la retención, y no *a quién* alcanza; el
predicado compartido `occupyingStatusFilter` de `AvailabilityService` trataba
todo turno CANCELADO con `holdUntil` vigente como ocupado sin distinguir quién
intentaba reservar. Como `book()` y `rescheduleCore()` se anclan en el mismo
`isSlotFree` que alimenta el listado de disponibilidad del bot, un ADMIN o el
propio profesional que intentaba asignar ese horario a otro paciente recibía un
400. La ventana pensada para proteger la decisión del profesional era
exactamente lo que se la impedía, y el efecto era además desconcertante: el
mensaje del error afirmaba que el profesional *"already has a reserved or
confirmed appointment"* en un horario que, en su agenda, figuraba libre.

El comportamiento estaba incluso fijado como esperado por la suite: el test de
`availability.e2e-spec.ts` que cubría la retención la describía en su
comentario como *"unofferable, unbookable"*, invirtiendo el criterio de
aceptación que decía documentar. La detección proviene de una auditoría de
código sobre `psique-back/main` (2026-08-14, agente "Audit waitlist/
reassignment/holidays vs SRS").

## Qué se implementó

- Nuevo enum de Prisma `HoldReason` (`MANUAL_REASSIGNMENT`,
  `AUTOMATIC_OFFER`) y columna `Appointment.holdReason`, junto a
  `holdUntil`: la retención pasa a registrar no sólo su vencimiento sino qué
  la originó, que es lo que determina a quién alcanza.
- Nuevo enum de aplicación `SlotAccess` (`BOT_OFFER`,
  `DIRECT_ASSIGNMENT`) en `AvailabilityService`, con una tabla única
  `BLOCKING_HOLD_REASONS` que declara qué retenciones afectan a cada tipo de
  llamador. `occupyingStatusFilter` pasa a recibir ese acceso y a filtrar por
  `holdReason` además de por `holdUntil`.
- `isSlotFree` recibe el acceso como parámetro obligatorio. `loadBookedTimes`
  —el camino de lectura detrás de `getSlots`/`getNewPatientSlots`, es decir,
  la agenda que el bot ofrece— lo fija en `BOT_OFFER` y sigue ocultando ambos
  tipos de retención. `book()` y `rescheduleCore()` pasan
  `DIRECT_ASSIGNMENT` y quedan bloqueados únicamente por una retención
  `AUTOMATIC_OFFER` en curso.
- `WaitlistReassignmentService` escribe el motivo al retener
  (`MANUAL_REASSIGNMENT` en la rama manual, `AUTOMATIC_OFFER` en cada paso
  del recorrido de la lista de espera) y limpia ambas columnas al liberar.
- Restricción `CHECK` a nivel de base de datos
  (`Appointment_hold_until_reason_consistency`) que obliga a que las dos
  columnas se escriban y se limpien juntas, con migración que además rellena
  el motivo de cualquier retención preexistente.
- Reformulación del mensaje del 400 de conflicto de horario, que ahora nombra
  las causas posibles en vez de afirmar la primera.

## Decisiones y por qué

**El motivo se guarda como columna propia y no se deriva de
`Professional.reassignmentMode`.** Derivarlo evitaría una columna, pero el
modo es la configuración *actual* del profesional y puede cambiarse —o el
recorrido de la lista de espera puede terminar— después de colocada la
retención; la columna registra qué motivó la retención que está efectivamente
en vigor. No es, por lo tanto, una réplica de un dato que un padre ya
determina, que es lo que la regla 2 del esquema prohíbe: es un hecho propio de
esa cancelación. Además evita sumar un `join` contra `Professional` a la
consulta más caliente del motor de turnos, la que decide si un instante está
libre.

**La consistencia entre `holdUntil` y `holdReason` se garantiza en la base y
no sólo en el servicio.** Una fila con `holdUntil` y sin motivo no es una
inconsistencia cosmética: la cláusula `holdReason: { in: [...] }` dejaría
silenciosamente de emparejarla, y el turno que el bot no debe ofrecer se
volvería ofrecible. Prisma no expresa restricciones `CHECK` en el esquema
(6.19.3), de modo que se declara en SQL en la migración, siguiendo el
precedente de `User_role_professional_id_consistency` (TASK-93).

**`isSlotFree` exige el acceso en vez de asumir uno por defecto.** Un valor
por defecto reintroduciría exactamente el defecto que se corrige: un llamador
que no declara de qué lado de la distinción está la obtendría mal en silencio.
Hoy todos los llamadores son `DIRECT_ASSIGNMENT`, porque tanto `POST /turnos`
como la reprogramación son rutas de personal; queda documentado en el código
que cuando M5 reserve en nombre del paciente sobre la agenda del propio bot,
ese camino deberá pasar `BOT_OFFER`, o el chatbot ocuparía el turno que el SRS
reserva para la mano del profesional.

**La retención `AUTOMATIC_OFFER` sí sigue bloqueando la escritura directa.**
Es la razón por la que la distinción es una columna y no un "las retenciones
nunca bloquean escrituras": un recorrido en curso tiene un candidato que puede
estar aceptando en ese mismo momento, y una reserva manual por encima dejaría
a dos pacientes creyendo tener el mismo instante.

**La retención de la cancelación original no se limpia al reasignar el turno a
mano.** Registra por qué se retuvo *esa* cancelación, que es un hecho
histórico cierto, y el instante queda de todos modos ocupado por el turno
nuevo — que es lo que impide que el bot lo ofrezca, con retención o sin ella.
Limpiarla habría agregado una escritura al camino caliente de la reserva sin
cambiar ningún comportamiento observable.

## Alternativas descartadas

- **Distinguir el tipo de retención por la existencia de una `WaitlistOffer`
  PENDING sobre el turno, sin columna nueva.** Descartada por dos motivos: el
  recorrido automático retiene el turno *antes* de rankear candidatos (red de
  seguridad ante caídas), de modo que existe una ventana en la que la
  retención es automática y todavía no hay ninguna oferta que lo evidencie; y
  obligaría a sumar un `join` a la consulta de disponibilidad.
- **Resolver el problema en `AppointmentsService`, saltando la verificación de
  retención sólo en `book()`.** Descartada porque duplicaría en el servicio de
  turnos la definición de "ocupado" que `AvailabilityService` es el único lugar
  autorizado a tener; la distinción pertenece al predicado compartido, donde
  ambos caminos de lectura y escritura la ven expresada una sola vez.
- **Mantener el texto anterior del 400.** Descartada: el mensaje afirmaba una
  de las causas posibles como si fuera la única, y esa afirmación falsa es lo
  que hizo que el defecto se leyera, desde la app, como una reserva fantasma
  sobre un horario libre.

## Entidades / puertos / adaptadores tocados

- Esquema: enum `HoldReason` y columna `Appointment.holdReason`; migración
  `20260819120000_appointment_hold_reason` (tipo, columna, relleno de
  retenciones preexistentes a partir de `Professional.reassignmentMode`, y
  `CHECK` de consistencia).
- `AvailabilityService` (`src/availability/availability.service.ts`): enum
  exportado `SlotAccess`, tabla `BLOCKING_HOLD_REASONS`,
  `occupyingStatusFilter`, `isSlotFree`, `loadBookedTimes`.
- `AppointmentsService` (`src/appointments/appointments.service.ts`): los tres
  llamadores de `isSlotFree` (`book()` — turno simple y segunda mitad del
  turno doble de primera sesión — y `rescheduleCore()`), más el nuevo helper
  privado `slotTakenMessage`.
- `WaitlistReassignmentService` (`src/waitlist/waitlist-reassignment.service.ts`):
  `holdSlot` (recibe el motivo), `releaseHold` (limpia ambas columnas) y las
  dos ramas que retienen.
- `AvailabilityController` (`src/availability/availability.controller.ts`):
  sólo comentario — deja asentado, sobre la propia ruta, el requisito que esta
  tarea registra para la fase 7 (ver la sección homónima más abajo), con un
  puntero cruzado desde `SlotAccess.BOT_OFFER`.
- Sin cambios de contrato HTTP: ninguna ruta, DTO ni forma de respuesta se
  modificó, por lo que la colección de Postman no requiere sincronización.

## Tests agregados o modificados

**Agregados**

- `test/appointment-reassignment.e2e-spec.ts`: los dos criterios de
  aceptación del ticket, contra base de datos real y por las rutas HTTP
  reales. *"MANUAL: the professional can still assign the held slot by hand
  during the 24h window"* cancela un turno de un profesional MANUAL,
  comprueba que la retención está efectivamente vigente, y reserva el mismo
  instante para otro paciente vía `POST /turnos` esperando 201 — el caso que
  antes devolvía 400. *"AUTOMATIC: a slot held by a PENDING offer still
  refuses a direct assignment"* es la mitad complementaria: con una oferta
  PENDING viva, la misma reserva directa sigue siendo rechazada con 400 y no
  crea ningún turno.
- `test/appointment-hold-consistency-check.e2e-spec.ts` (nuevo archivo):
  cuatro pruebas sobre la restricción `CHECK`, siguiendo el patrón de
  `user-role-professional-id-check.e2e-spec.ts` — rechaza `holdUntil` sin
  motivo, rechaza motivo sin `holdUntil`, acepta ambas puestas y ambas nulas,
  y rechaza limpiar sólo una de las dos (el camino que más fácilmente rompe
  el par, porque libera dos columnas en vez de escribirlas).
- `availability.service.spec.ts`: dos pruebas unitarias que fijan qué motivos
  pide cada acceso a Postgres, verificando la consulta emitida y no sólo el
  booleano devuelto — un `findFirst` simulado responde lo que se le indique
  con independencia del filtro recibido, de modo que sólo inspeccionar la
  consulta prueba que la distinción llega hasta la base.

**Modificados**

- `availability.service.spec.ts`: las pruebas preexistentes de `isSlotFree`
  pasan el acceso explícito; la verificación de la cláusula de retención
  incorpora `holdReason`.
- `test/availability.e2e-spec.ts`: los dos tests de la retención de 24h siguen
  probando lo mismo a nivel `getSlots` (la agenda del bot sigue ocultando
  ambas retenciones, que es lo correcto), pero su comentario deja de describir
  el turno como "unbookable" — la exclusión es sólo de la oferta, y remite al
  e2e que prueba la otra mitad. Sus fixtures pasan a fijar también el motivo.
- `appointment-reassignment.e2e-spec.ts`: los tests existentes verifican ahora
  el motivo de cada retención y que la liberación limpia ambas columnas; el
  fixture de paciente pasa a crear pacientes reservables (fecha de nacimiento,
  contacto de emergencia y consentimiento), de modo que un mismo fixture sirva
  tanto a los caminos de lista de espera como a los de reserva.
- `waitlist-reassignment.service.spec.ts` y
  `test/waitlist-offer-timeout.e2e-spec.ts`: motivo en las aserciones y en los
  fixtures de retención.
- `test/appointments-rescheduling.e2e-spec.ts`: dos aserciones que buscaban la
  subcadena `'already has a'` en el 400 pasan a verificar la parte estable del
  mensaje (qué instante fue rechazado), tras la reformulación del texto.

**Resultado**: suite completa verde — 39 suites unitarias / 437 pruebas; 38
suites e2e / 445 pruebas (`--runInBand`). `tsc --noEmit` y ESLint sin errores.
`prisma migrate diff` no reporta diferencias entre el esquema y la base tras
aplicar la migración.

## Requisito que esta tarea deja registrado para la fase 7

La corrección cierra el defecto en el backend, pero deja una consecuencia
pendiente en la capa que todavía no existe. La ruta
`GET /profesionales/:id/disponibilidad` responde con acceso `BOT_OFFER` para
*todos* los llamadores, la app del profesional incluida. Es lo que TASK-113
delimitó y es lo correcto para el chatbot; no lo es para la app, y el motivo es
el mismo que motivó esta tarea: la ventana de veinticuatro horas existe para
darle tiempo al profesional de asignar ese horario a mano, de modo que un
selector que no se lo muestra anula la ventana exactamente como lo hacía el
400, una capa más arriba. Como `POST /turnos` ya acepta la reserva, el horario
queda reservable pero no listado —el peor de los dos estados, porque funciona
únicamente para quien ya sabe que tiene que escribir la hora.

Queda registrado, entonces, como requisito y no como alternativa: **el
selector de horarios de la app del profesional debe ofrecer el horario
retenido bajo `MANUAL_REASSIGNMENT`.** No es un defecto abierto contra
TASK-113 porque hoy nada está roto: la agenda del profesional es otra ruta
(`GET /profesionales/:id/turnos`, que lista turnos y no huecos), y allí la hora
liberada se ve simplemente como que no hay nada reservado a esa hora —
exactamente la formulación del SRS, *"la franja queda libre en su agenda"*.

El mecanismo del lado del servicio ya está construido y no requiere cambios:
esa pantalla pedirá los horarios como `DIRECT_ASSIGNMENT` en vez de como
oferta del bot. Lo que la fase 7 debe agregar son tres piezas:

1. Una forma de pedir esa variante por HTTP —un parámetro de consulta en esa
   ruta, o una ruta hermana—. `AvailabilityService` ya recibe el acceso, de
   modo que el cambio es de superficie y no de lógica.
2. Una restricción sobre quién puede pedirla. No es una preferencia:
   responder `DIRECT_ASSIGNMENT` al chatbot le devolvería precisamente el
   horario que el SRS reserva para la mano del profesional, así que la
   variante debe negarse al rol `SYSTEM` —bajo el cual corren los procesos
   automáticos y correrá el chatbot— y permitirse a `ADMIN`/`PROFESSIONAL`.
   La restricción hay que agregarla explícitamente: esta ruta está hoy
   abierta a cualquier usuario autenticado del inquilino.
3. Una forma de que la respuesta indique que un horario está retenido, para
   que la app pueda etiquetarlo en vez de mostrarlo como una hora libre
   cualquiera. `AvailabilitySlot` no tiene hoy ese campo. Un profesional que
   no puede distinguir ambos casos podría suponer razonablemente que el bot se
   lo va a quitar, que es justamente la incertidumbre que la ventana existe
   para evitar.

El requisito quedó además asentado sobre la propia ruta, como comentario en
`AvailabilityController`, con puntero cruzado desde el enum `SlotAccess`, para
que sea visible desde el código y no sólo desde esta bitácora.

## Figuras pendientes

Ninguna nueva. La corrección no agrega un flujo que la tesis no describa ya:
refina el alcance de la retención introducida en TASK-80, que ya está
documentada en 3.2.3.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-113-manual-hold-allows-direct-booking` (creada desde `main`,
  en sincronía con `origin/main`). Sin commit ni push al momento de redactar
  esta entrada, a pedido de la usuaria.
- Ticket: TASK-113 (Jira), "[CORRECCIÓN] TASK-80 – El hold de reasignación
  MANUAL bloquea también la asignación manual del profesional". Misma
  convención de bitácora dedicada para tareas puntuales dentro de la fase del
  ticket original que TASK-79/TASK-81/TASK-86/TASK-94/TASK-95/TASK-96/
  TASK-100/TASK-108/TASK-110 ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]],
  [[FASE-3_PROMPT-15]], [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]],
  [[FASE-3_PROMPT-18]], [[FASE-3_PROMPT-19]], [[FASE-3_PROMPT-23]],
  [[FASE-3_PROMPT-24]]).
- Fuente de verdad consultada: SRS "Secretaria Virtual — PSIQUE
  NEUROCIENCIAS" (Google Drive, *PROPUESTA PSIQUE - documento reunion
  ACTUALIZADO*), Anexo → Módulo turnos → "Reasignación de turnos ante
  cancelación", inciso b (modalidad manual).
