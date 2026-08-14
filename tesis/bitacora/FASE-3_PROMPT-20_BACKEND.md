# Fase 3 — Motor de Turnos (backend) — mensajes al paciente de reagendado, cancelación por ausencia y oferta de lista de espera esquivaban el motor de plantillas (TASK-101, corrección a TASK-39/TASK-40)

## Contexto

Una auditoría multi-agente de `psique-back` en `main` (ángulos
reuse/simplificación y auditoría/fechas/reglas-como-datos, hallado de
forma independiente por ambos agentes, 2026-08-12) detectó que cuatro
puntos del código armaban a mano, en inglés, el texto de un mensaje al
paciente y lo enviaban directo por `MessagingPort.sendMessage`, en vez de
pasar por `NotificationTemplateService.render` (P4.1, TASK-42):

- `AppointmentsService.rescheduleCore` (`src/appointments/appointments.service.ts`):
  el aviso de que un turno fue reprogramado, enviado una vez confirmada la
  escritura.
- `AppointmentsService.requestRescheduleConfirmation` (mismo archivo): el
  pedido de confirmación al paciente antes de aplicar un reagendado
  unilateral (ADMIN o el profesional tratante).
- `AppointmentsService.cancelForAbsence` (mismo archivo): el aviso de
  cancelación masiva cuando un profesional registra una ausencia.
- `WaitlistReassignmentService.tryOffer` (`src/waitlist/waitlist-reassignment.service.ts`):
  la oferta de un turno liberado a un candidato de la lista de espera.

`NotificationTemplateKey.APPOINTMENT_CANCELLATION` y `.WAITLIST_OFFER` ya
existían en `notification-template.constants.ts`, con texto por defecto
en español y ya configurables por tenant vía `OrganizationConfig`, pero
sin ningún llamador real — sólo `AppointmentConfirmationCron` (P4.2) y
`AppointmentReminderCron` (P4.4) pasaban efectivamente por el motor de
plantillas. El efecto: una clínica que personalizara
`notification_template_appointment_cancellation` o
`notification_template_waitlist_offer` en su configuración no veía
ningún cambio en estos cuatro flujos, un desvío que sólo se nota leyendo
en paralelo el módulo de notificaciones y el de turnos/lista de espera.
El texto hardcodeado en inglés además incumplía la convención de idioma
de `CLAUDE.md` ("Domain naming"): el contenido cara al paciente debe ser
español.

## Qué se implementó

- Se agregaron dos claves nuevas a `NotificationTemplateKey`
  (`notification-template.constants.ts`), con su texto por defecto en
  español y su entrada en `DEFAULT_NOTIFICATION_TEMPLATES`, siguiendo el
  mismo patrón de configuración por tenant que las cinco claves
  existentes (namespaced vía `notificationTemplateConfigKey`):
  - `APPOINTMENT_RESCHEDULE`: el aviso posterior a la escritura
    (`rescheduleCore`). Texto: *"Tu turno con {professionalName} fue
    reprogramado para el {scheduledAt}."*
  - `APPOINTMENT_RESCHEDULE_CONFIRMATION_REQUEST`: el pedido de
    confirmación previo a la escritura
    (`requestRescheduleConfirmation`), mismo sufijo `_REQUEST` que ya usa
    `CONSENT_REQUEST` para un mensaje que espera una respuesta SI/NO.
    Texto: *"Tu profesional {professionalName} propone reprogramar tu
    turno para el {scheduledAt}. ¿Aceptás? Respondé SI o NO."*
- Los cuatro call sites listados ahora arman su texto con
  `NotificationTemplateService.render(key, params)` antes de llamar a
  `MessagingPort.sendMessage`, usando `APPOINTMENT_RESCHEDULE`,
  `APPOINTMENT_RESCHEDULE_CONFIRMATION_REQUEST`,
  `APPOINTMENT_CANCELLATION` y `WAITLIST_OFFER` respectivamente.
  `NotificationTemplateService` se agregó como dependencia del
  constructor de `AppointmentsService` y de `WaitlistReassignmentService`
  (ambas ya importaban `NotificationsModule` indirectamente o se les
  agregó el import — ver más abajo).
- Se agregó `AppointmentsService.professionalName(professionalId)`, una
  lectura chica y dedicada del nombre del profesional — necesaria porque
  `appointmentSelect` deliberadamente no lo incluye — reusada por los
  tres call sites de `AppointmentsService`, en vez de repetir la consulta
  tres veces.
- `WaitlistReassignmentService.tryOffer` obtiene el nombre del
  profesional con una consulta propia (`professional.findFirstOrThrow`,
  en paralelo con la consulta del teléfono del paciente vía
  `Promise.all`), en vez de encadenarlo desde `handleAppointmentCancelled`:
  ese método también es el punto de entrada de `respondToOffer` cuando un
  candidato rechaza o una oferta vence, un camino que reconstruye el
  evento desde la fila de `Appointment` y no tiene el nombre del
  profesional a mano, así que la única ubicación que lo necesita de forma
  consistente en los dos caminos de entrada es el propio `tryOffer`.
- Se extrajo `formatScheduledAt`/`formatTime` — antes duplicadas de forma
  idéntica en `AppointmentConfirmationCron` y
  `AppointmentReminderCron` respectivamente — a un módulo compartido
  nuevo, `notifications/notification-template.format.ts`, y los cuatro
  call sites nuevos (más los dos crons existentes, actualizados para
  importar en vez de redefinir) lo reusan. Evita agregar una tercera
  (y cuarta) copia de la misma función al resolver este ticket.
- `WaitlistModule` ahora importa `NotificationsModule` (ya lo hacía
  `AppointmentsModule`, para los dos crons existentes).

## Decisiones y por qué

**Nombres en inglés para las dos claves nuevas, siguiendo la convención
ya documentada en el propio archivo de constantes** ("Domain naming",
`CLAUDE.md"): se usó "reprogramación", el término que ya usa todo
`appointments.service.ts` para este flujo (P3.6), en vez de "reagendado"
(la palabra que usa el ticket) o "REPROGRAMACION_TURNO" tal cual —
mismo tratamiento que TASK-35 le dio a `from`/`to` y TASK-78 a
"404 en vez de 403": el texto del ticket en español es la descripción
funcional de la clave para el SRS, no un contrato literal sobre el
identificador.

**Consulta dedicada al nombre del profesional, no ampliar
`appointmentSelect`.** `appointmentSelect` es la fuente única de columnas
que expone la respuesta HTTP de un turno (comentario propio del archivo);
agregarle el nombre del profesional sólo para que estos mensajes lo
usaran hubiera filtrado un dato ajeno a ese contrato hacia todas las
respuestas de la API. Una lectura chica y separada, igual de barata a
esta escala (una fila por clave primaria), mantiene ambas
responsabilidades separadas.

**`tryOffer` resuelve el nombre por su cuenta, en vez de heredarlo del
llamador.** `advanceWaitlist` (y por lo tanto `tryOffer`) tiene dos
puntos de entrada: `handleAppointmentCancelled` (la cancelación
original) y `respondToOffer` (una respuesta o vencimiento, que puede
llegar en un tick de cron completamente separado y reconstruye el evento
desde la fila de `Appointment`, sin conexión con la llamada original).
Encadenar el nombre sólo desde el primero habría dejado al segundo sin
él; resolverlo en el propio `tryOffer` cubre ambos caminos de la misma
forma.

## Alternativas descartadas

- **Ampliar `appointmentSelect` con el nombre del profesional** — se
  descartó por la razón de arriba (contaminaría el contrato de la
  respuesta HTTP de turnos con un dato que ningún endpoint pidió).
- **Encadenar el nombre del profesional desde
  `handleAppointmentCancelled` hasta `tryOffer` como parámetro** — se
  descartó porque no cubre el camino de reentrada por
  `respondToOffer`/`expireOffer`/`acceptOffer` (rechazo o vencimiento de
  una oferta), que no pasa por `handleAppointmentCancelled`.

## Entidades / puertos / adaptadores tocados

- `notifications/notification-template.constants.ts`: dos claves nuevas
  en `NotificationTemplateKey` y `DEFAULT_NOTIFICATION_TEMPLATES`.
- `notifications/notification-template.format.ts` (nuevo):
  `formatScheduledAt`/`formatTime`, extraídas de los dos crons de P4.1.
- `appointments/appointment-confirmation.cron.ts`,
  `appointments/appointment-reminder.cron.ts`: importan el formato
  compartido en vez de redefinirlo localmente; sin cambio de
  comportamiento.
- `appointments/appointments.service.ts`: agrega `NotificationTemplateService`
  al constructor; nuevo método privado `professionalName`; los tres call
  sites listados arriba ahora renderizan antes de enviar.
- `appointments/appointments.module.ts`: comentario actualizado (sin
  cambio de imports — `NotificationsModule` ya estaba importado para los
  crons).
- `waitlist/waitlist-reassignment.service.ts`: agrega
  `NotificationTemplateService` al constructor; `tryOffer` resuelve el
  nombre del profesional y renderiza `WAITLIST_OFFER` antes de enviar.
- `waitlist/waitlist.module.ts`: agrega el import de `NotificationsModule`.

## Tests agregados o modificados

- `notifications/notification-template.service.spec.ts`: las dos claves
  nuevas se agregaron al mapa `completeParams` que ya cubría,
  paramétricamente (`it.each(Object.values(NotificationTemplateKey))`),
  los cinco casos existentes de "renderiza sin placeholders quedando
  colgados" y "rechaza con un parámetro faltante" — extenderlo alcanza
  para que ambas claves nuevas hereden esa cobertura genérica de
  configuración-vs-default por tenant, sin duplicar los dos tests
  dedicados que ya prueban esa dualidad de forma explícita
  (`'uses the tenant-configured text...'`, `'falls back to the base
  template...'`).
- `src/appointments/appointments-rescheduling.service.spec.ts`: agrega el
  mock de `NotificationTemplateService` (con un `render` que devuelve la
  propia clave, para poder verificar tanto qué clave se pidió como que el
  texto renderizado llegó a `MessagingPort` sin tener que repetir el
  texto en español acá); dos casos nuevos (`reschedule` renderiza y envía
  tanto el pedido de confirmación previo como el aviso posterior;
  `cancelForAbsence` renderiza y envía `APPOINTMENT_CANCELLATION`) más el
  mock de `professional.findFirst` ampliado con un `name`.
- `src/appointments/appointments.service.spec.ts`: agrega el mismo mock
  de `NotificationTemplateService` a los cuatro bloques de
  `TestingModule` de este archivo, sólo para satisfacer la inyección de
  dependencias — ningún test de este archivo ejercita `reschedule` ni
  `cancelForAbsence`, así que no necesitó aserciones nuevas.
- `src/waitlist/waitlist-reassignment.service.spec.ts`: mismo patrón de
  mock; el caso existente "offers the freed slot to the top-ranked
  candidate" se amplió para verificar la clave y los parámetros
  renderizados, además del texto ya enviado a `MessagingPort`; el mock de
  `professional.findFirstOrThrow` se amplió con un `name`.

Suite completa verde tras el cambio: 38 suites unitarias / 428 pruebas;
37 suites e2e / 439 pruebas (`--runInBand`). Lint y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva — es una corrección puntual sobre cuatro puntos de envío ya
documentados (P3.6 reprogramación/cancelación por ausencia, P3.7 oferta
de lista de espera), que ahora pasan por el motor de plantillas P4.1 ya
documentado en [[FASE-4_PROMPT-1]] (o la entrada correspondiente); no
introduce ningún flujo ni diagrama nuevo.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-101-notification-template-callsites`
  (creada desde `origin/main` fresco, tras el merge de TASK-100). Pusheada
  a `origin`, PR abierta, no fusionada aún.
- Ticket: TASK-101 ("[CORRECCIÓN] Mensajes al paciente (reagendado,
  cancelación por ausencia, oferta de lista de espera) esquivan
  NotificationTemplateService"), corrección a TASK-39 (P3.6, reprogramación
  y cancelación por ausencia) y TASK-40 (P3.7, oferta de lista de espera),
  que depende de TASK-42 (P4.1, el motor de plantillas que ambos deberían
  haber usado desde el principio). Misma convención de bitácora dedicada
  para correcciones pequeñas dentro de la fase del ticket original que
  TASK-79/TASK-81/TASK-86/TASK-94/TASK-95/TASK-96/TASK-100
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]], [[FASE-3_PROMPT-15]],
  [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]], [[FASE-3_PROMPT-18]],
  [[FASE-3_PROMPT-19]]) y TASK-91/TASK-92/TASK-97/TASK-98
  ([[FASE-4_PROMPT-7]], [[FASE-1_PROMPT-9]], [[FASE-4_PROMPT-9]],
  [[FASE-2_PROMPT-13]]).
