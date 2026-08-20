# Fase 4 — Notificaciones y Scheduler (backend) — la duración de la sesión en el mensaje de confirmación (TASK-118, corrección a TASK-43)

## Qué se implementó

El SRS pide que el mensaje de confirmación a 24 horas informe cuánto dura
la sesión ("sesiones de 45 minutos"). Una auditoría de código de
`psique-back/main` (2026-08-14, agente "Audit Módulo Turnos core lifecycle
vs SRS") detectó que no lo hacía: la plantilla base
`APPOINTMENT_CONFIRMATION` no tenía ningún marcador de posición para la
duración, y el trabajo programado que la renderiza
(`appointment-confirmation.cron.ts`) nunca seleccionaba la columna
`Appointment.duration`, pese a existir en el modelo y estar disponible en
la misma consulta que ya trae paciente y profesional.

El defecto es independiente del estado del adaptador de WhatsApp: el
renderizado de plantillas es real y está en producción, sólo el envío final
está detrás del adaptador de prueba, de modo que la corrección tiene efecto
completo hoy sobre el texto que el motor produce.

La corrección tiene tres partes:

1. **Plantilla base** (`notification-template.constants.ts`): el texto de
   `APPOINTMENT_CONFIRMATION` pasa a
   `"Hola {patientName}, tu turno con {professionalName} es el
   {scheduledAt} y dura {durationMinutes} minutos. ¿Confirmás asistencia?
   Respondé SI o NO."`.
2. **Cron** (`appointment-confirmation.cron.ts`): `duration` se agrega al
   `select` de `ConfirmationCandidate` y se pasa como parámetro
   `durationMinutes` al renderizar.
3. **Migración de datos**
   (`20260820120000_appointment_confirmation_template_duration`): actualiza
   la fila de `OrganizationConfig` que todavía contiene, palabra por
   palabra, el texto base anterior.

El punto (3) no está pedido por el ticket y es lo que hace que la
corrección efectivamente se vea. La migración
`20260812140000_seed_notification_templates` (TASK-42) ya había escrito el
texto viejo en `OrganizationConfig` de toda organización existente, y
`NotificationTemplateService.resolveTemplate` prefiere la fila del inquilino
por sobre `DEFAULT_NOTIFICATION_TEMPLATES`: sin esta tercera parte, cambiar
la constante habría dejado el mensaje exactamente igual en cualquier base
ya migrada —incluida la del piloto—, y el criterio de aceptación se habría
cumplido sólo en los tests.

## Decisiones y por qué

**El marcador de posición lleva el número y la palabra "minutos" queda en
el texto de la plantilla.** Es la misma separación que ya aplican
`{scheduledAt}` y `{time}`, cuyos formateadores compartidos
(`notification-template.format.ts`) devuelven un valor —"13/08/2026 12:00",
"12:00"— y nunca la redacción que lo rodea. Un parámetro es un valor; las
palabras alrededor son justamente lo que cada organización personaliza en
`OrganizationConfig` bajo el requisito de marca blanca. La alternativa
—un formateador `formatDuration` que devolviera "45 minutos" ya armado—
habría dejado la unidad fuera del alcance de esa personalización, sin
ninguna ganancia: la conversión es un `String(...)` sobre un entero, no un
formateo con reglas propias como sí lo es una fecha en reloj de pared.

**El parámetro se llama `durationMinutes` y no `duration`.** Quien edita
este texto en la configuración de la organización es personal de la
clínica, y el número aparece desnudo: el nombre del marcador es lo único
que declara la unidad. La columna del modelo sigue llamándose `duration`,
como en el resto del proyecto.

**La duración se lee de la columna del turno, no de la configuración
vigente del profesional.** Un turno conserva la duración con la que fue
reservado (`appointments.service.ts` la copia al crearlo, precisamente para
que un cambio posterior de la configuración no reescriba turnos ya
agendados), de modo que la configuración actual no es necesariamente lo que
dura *este* turno. Es además el dato que el ticket señala como
"trivialmente disponible": está en la misma fila que la consulta ya trae.

**La migración sólo toca la fila que conserva el texto base anterior
íntegro.** Un texto que la clínica editó es una decisión suya y no se
sobrescribe, el mismo criterio no destructivo que ya siguen las migraciones
de siembra de reglas de negocio de este proyecto. La contrapartida
aceptada: si una organización personalizó su plantilla y esa
personalización tampoco menciona la duración, seguirá sin mencionarla —
corregirla es una decisión de la clínica, no de una migración que
sobrescribiría su redacción.

**No se tocaron las demás plantillas.** El SRS pide la duración en el
mensaje de confirmación; agregarla al recordatorio o al aviso de
reprogramación habría sido alcance inventado. El mecanismo queda igual de
disponible para ellas si un ticket posterior lo pide.

## Alternativas descartadas

- **Cambiar sólo la constante del código** (lo que el ticket pide
  literalmente): descartada porque el arreglo no llegaría a ninguna base
  ya migrada, por la precedencia de la fila del inquilino descripta arriba.
- **Reescribir la fila de configuración incondicionalmente**: descartada
  porque pisaría el texto que la clínica hubiera personalizado, que es el
  punto mismo del mecanismo de plantillas configurables.
- **Un formateador compartido `formatDuration` que devolviera "45
  minutos"**: descartada por las razones dadas arriba —saca la unidad del
  texto personalizable y no aporta ningún formateo real.
- **Probar el criterio de aceptación sólo contra el motor de plantillas**:
  descartada por insuficiente. Un cron que pasara un parámetro con otro
  nombre que el declarado por la plantilla produce
  `MissingNotificationTemplateParamsError`, que el manejo "mejor esfuerzo"
  por turno del propio cron convierte en una línea de log y descarta, con
  lo cual el paciente simplemente no recibiría nada y ningún test con el
  motor simulado lo notaría. Por eso se agregó un test del cron que arma el
  `NotificationTemplateService` real.

## Entidades / puertos / adaptadores tocados

- Ninguna entidad nueva ni cambio de esquema. Se usa `Appointment.duration`,
  ya existente.
- `OrganizationConfig`: migración de datos sobre la clave
  `notification_template_appointment_confirmation`.
- `MessagingPort`: sin cambios; sigue recibiendo el texto ya renderizado.

## Tests y qué validan

- `appointment-confirmation.cron.spec.ts`
  - El test existente de envío verifica ahora que el cron pasa
    `durationMinutes` con la duración del turno.
  - Test nuevo, "tells the patient how long the session lasts": arma el
    cron con el `NotificationTemplateService` real (sólo la configuración
    por inquilino queda simulada, devolviendo "sin personalizar") y afirma
    que el texto entregado a `MessagingPort` contiene "45 minutos" — el
    criterio de aceptación del ticket, verificado sobre el mensaje final y
    no sobre los parámetros de un doble de prueba.
  - Refactor de apoyo: el armado del módulo de prueba pasó a una función
    `buildCron(templates)`, para que ese único test pueda sustituir el
    motor de plantillas sin duplicar el resto del andamiaje.
- `notification-template.service.spec.ts`: el ejemplo del ticket original
  (TASK-42) y la tabla de parámetros completos por plantilla incorporan
  `durationMinutes`; el texto esperado incluye "y dura 45 minutos".
- Suite unitaria completa del repo en verde (39 suites, 452 tests), más
  `tsc --noEmit` y ESLint sobre los archivos tocados.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-118-confirmation-message-duration`,
  creada desde `origin/main` recién actualizado (`git fetch origin`), misma
  precaución aplicada en TASK-81 y TASK-111. Sin commit ni push al momento
  de escribir esta entrada: la usuaria los autoriza explícitamente.
- Ticket: TASK-118 (Jira), "[CORRECCIÓN] TASK-43 — Falta la duración de la
  sesión en el mensaje de confirmación de 24h". Referencias: TASK-43
  (implementación original del cron, [[FASE-4_PROMPT-2]]), TASK-42
  (motor de plantillas y migración de siembra, [[FASE-4_PROMPT-1]]) y
  TASK-99 ([[FASE-4_PROMPT-10]], que fijó el orden guarda-antes-de-envío
  sobre el que se apoya el test nuevo).
