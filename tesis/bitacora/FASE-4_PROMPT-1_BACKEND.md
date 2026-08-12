# Fase 4 — Notificaciones y Scheduler (backend) — motor de plantillas de mensajes (TASK-42, P4.1)

## Qué se implementó

`NotificationTemplateService.render(key, params)` (`src/notifications/`),
que produce el texto final de una notificación a partir de una clave de
plantilla y un objeto de parámetros. No envía nada — es el primer
componente de Módulo 4 (Notificaciones y recordatorios) y está pensado
para ser consumido por los cron jobs de M4 y el chatbot de M5, ninguno
de los dos implementado todavía.

Se cargaron las cinco plantillas base que pide el ticket: confirmación
de turno, recordatorio de turno, aviso de cancelación, aviso de
reasignación (oferta de lista de espera) y solicitud de consentimiento.
Cada plantilla se resuelve primero contra la configuración del tenant
(`OrganizationConfig`, vía `ConfigTenantService`) y cae al texto base del
sistema si el tenant no la personalizó.

## Decisiones y por qué

**Claves de plantilla traducidas a identificadores en inglés.** El
ticket nombra las claves en español (`CONFIRMACION_TURNO`,
`RECORDATORIO_TURNO`, `AVISO_CANCELACION`, `AVISO_REASIGNACION`,
`SOLICITUD_CONSENTIMIENTO`) y los criterios de aceptación incluso llaman
a `render('CONFIRMACION_TURNO', ...)` literalmente. Se tradujo de todos
modos a `NotificationTemplateKey` en inglés
(`APPOINTMENT_CONFIRMATION`, `APPOINTMENT_REMINDER`,
`APPOINTMENT_CANCELLATION`, `WAITLIST_OFFER`, `CONSENT_REQUEST`),
siguiendo la convención de renombrado de CLAUDE.md ("Domain naming") ya
aplicada en TASK-35 (`from`/`to` en vez de `desde`/`hasta`) y TASK-78
(404 en vez del 403 que pide el texto del ticket): el español del ticket
se trató como la descripción funcional del requisito, no como un
contrato literal sobre el identificador. `AVISO_REASIGNACION` se nombró
`WAITLIST_OFFER` reutilizando el vocabulario que ya existe para el mismo
evento en `WaitlistOffer`
(`src/domain/ports/waitlist-response.port.ts`), en vez de acuñar un
segundo nombre para el mismo concepto.

**El texto de las plantillas se dejó en español.** A diferencia de los
identificadores de código, el texto es contenido que el paciente lee
directamente por WhatsApp — se trató como dato de dominio, igual que un
nombre de paciente, no como un símbolo del lenguaje de programación.

**Parámetros del texto también en inglés** (`patientName`,
`professionalName`, `scheduledAt`, `time`), no `nombrePaciente` /
`nombreProfesional` / `fechaHora` / `hora` del ticket — mismo criterio
que el contrato JSON en inglés en todo el resto de la API (ver "Domain
naming" en CLAUDE.md). `scheduledAt` reutiliza el nombre que ya usa
`WaitlistOffer` para el mismo dato.

**Parámetro faltante y clave desconocida son errores descriptivos, no
resultados parciales.** El criterio de aceptación es explícito: no debe
devolverse texto con el marcador de posición sin reemplazar. Se
implementó extrayendo los nombres de parámetro directamente del texto de
la plantilla (`placeholdersIn`, `notification-template.renderer.ts`) en
vez de declarar aparte, por clave, qué parámetros exige cada una —así una
plantilla personalizada por el tenant, con parámetros potencialmente
distintos a los de la base, no puede quedar en desacuerdo con lo que el
servicio efectivamente exige. Ambos errores son subclases planas de
`Error` (`NotificationTemplateNotFoundError`,
`MissingNotificationTemplateParamsError`), no `HttpException`: la tarea
no expone ningún endpoint propio, siguiendo el mismo criterio que
`MissingTenantContextError`.

**Las cinco plantillas base se sembraron también en una migración de
datos**, no sólo en `prisma/seed.ts` — mismo criterio ya fijado en
CLAUDE.md para toda regla de alcance tenant (`patient_inactivity_months`,
`appointment_cancellation_min_hours`): el seed es dato de desarrollo y
del piloto y no corre en un entorno real, así que sin la migración la
fila no existiría en la configuración de ninguna organización
productiva, justo el punto de la tarea (que el texto sea algo que el
admin pueda encontrar y editar). La migración es idempotente
(`ON CONFLICT ... DO NOTHING`) y no sobrescribe un texto que la clínica
ya haya modificado.

## Alternativas descartadas

- **Mantener las claves y los parámetros en español**, como los da el
  ticket literalmente: descartada por el mismo argumento que ya se aplicó
  en TASK-35/TASK-78 — el código es en inglés salvo las dos excepciones
  explícitas de CLAUDE.md (rutas HTTP y glosario de dominio en
  comentarios), y ni la clave de plantilla ni el nombre de un parámetro
  del contrato son ninguna de las dos cosas.
- **Declarar aparte, por clave, la lista de parámetros requeridos**: se
  descartó a favor de extraerlos del propio texto de la plantilla, para
  que una plantilla personalizada por el tenant no pudiera exigir un
  parámetro distinto al que el servicio validaba.
- **No sembrar las plantillas en una migración**, dejándolas existir sólo
  como constante de código hasta que alguien las personalice: descartada
  por ser exactamente el caso que la regla de CLAUDE.md sobre datos
  tenant-wide ya cubre, con el agravante de que el propio objetivo de la
  tarea es que el texto sea editable desde el día uno.

## Entidades / puertos / adaptadores tocados

- `src/notifications/notification-template.constants.ts`: enum
  `NotificationTemplateKey`, plantillas base
  (`DEFAULT_NOTIFICATION_TEMPLATES`) y la función que arma la clave de
  configuración por plantilla.
- `src/notifications/notification-template.renderer.ts`: función pura de
  extracción de parámetros (`placeholdersIn`) y sustitución
  (`interpolate`) sobre el texto de una plantilla — sin I/O, misma
  separación que `patient-inactivity.rule.ts`.
- `src/notifications/notification-template.errors.ts`: los dos errores
  descriptivos del servicio.
- `src/notifications/notification-template.service.ts`:
  `NotificationTemplateService`, inyecta `ConfigTenantService`.
- `src/notifications/notifications.module.ts`: módulo Nest, registrado en
  `AppModule`.
- `prisma/migrations/20260812140000_seed_notification_templates/`:
  siembra las cinco plantillas base en `OrganizationConfig` para toda
  organización existente.
- `prisma/seed.ts`: agrega las cinco plantillas a `TENANT_CONFIG` (cuyo
  tipo de valor se amplió de `number` a `Prisma.InputJsonValue` para
  admitir texto además de números).

No se tocó `MessagingPort` ni ningún cron: ambos quedan fuera de alcance
de esta tarea (P4.2/P4.3 y M5/TASK-46, según el propio ticket).

## Tests y qué validan

- `notification-template.renderer.spec.ts`: extracción y sustitución de
  parámetros como funciones puras (deduplicación, plantilla sin
  parámetros, parámetro repetido).
- `notification-template.service.spec.ts`, con `ConfigTenantService`
  mockeado: el ejemplo exacto del criterio de aceptación
  (`CONFIRMACION_TURNO` → `APPOINTMENT_CONFIRMATION`); las cinco
  plantillas base con parámetros completos, sin marcador de posición sin
  reemplazar en el resultado; las cinco con parámetros incompletos,
  rechazadas con `MissingNotificationTemplateParamsError`; un parámetro
  vacío tratado como faltante; una clave desconocida rechazada con
  `NotificationTemplateNotFoundError`; texto personalizado por el tenant
  usado en lugar de la plantilla base; un valor configurado en blanco
  tratado como no configurado.
- Migración y seed verificados contra Postgres local
  (`docker-compose up -d db`, `prisma migrate deploy`, `prisma db seed`):
  las cinco filas quedan en `OrganizationConfig` con el texto esperado,
  incluidos los acentos y signos de interrogación invertidos.
- Ejecución: 32 suites unitarias / 357 pruebas en verde (22 nuevas); 33
  suites e2e / 409 pruebas en verde (`--runInBand`, sin cambios respecto
  a la fase anterior — la tarea no agrega ningún endpoint). Lint limpio.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-42-notification-template-engine`
  (creada desde `origin/main` fresco, tras el merge de TASK-86). Pusheada
  a `origin`, no fusionada aún.
- Ticket: TASK-42 (P4.1 — Motor de plantillas de mensajes), primera tarea
  de Módulo 4 (TASK-7). Depende de TASK-15 (configuración por tenant,
  [[FASE-0_PROMPT-5]]) y es requisito declarado de P4.2/P4.3 (envío por
  WhatsApp) y de la integración real de M5/P5.1 (TASK-46), ninguna
  implementada todavía.
