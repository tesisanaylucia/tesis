# Fase 4 — Notificaciones y Scheduler (backend) — la ventana de recordatorio coincidía con la de confirmación, y su texto asumía siempre "mañana" (TASK-158, corrección a TASK-45)

## Qué se implementó

TASK-158 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, validada por la usuaria
antes de implementarse. Reunía dos hallazgos relacionados sobre el trabajo
programado de recordatorio de turno (`AppointmentReminderCron`, sección 4.5):

- **TUR-6**: el valor por defecto de la ventana de recordatorio (24 horas) y
  el ancho de banda de detección de ±1 hora alrededor de ese valor
  reproducían exactamente la misma banda de 23 a 25 horas que ya usa el job
  de confirmación (P4.2). Para cualquier organización que nunca personalizara
  esa configuración —el caso esperable, no un caso extremo— un turno recibía
  el pedido de confirmación y, dentro de la misma hora en que el paciente
  respondía, un recordatorio redundante sobre algo que ya acababa de
  confirmar.
- **TUR-7**: el texto base de la plantilla de recordatorio decía
  "...mañana a las {hora}" de forma fija, aunque la ventana de recordatorio
  es explícitamente configurable por inquilino a cualquier cantidad positiva
  de horas de anticipación. Un inquilino configurado a 2 horas recibía
  "mañana" el mismo día; uno configurado a 48 horas recibía "mañana" sobre un
  turno que en realidad era pasado mañana.

La corrección cambió el valor por defecto de la ventana de recordatorio de
24 a 2 horas, y agregó un marcador de posición `{relativeDay}` a la plantilla
base, calculado en el propio trabajo programado a partir de la brecha real
en días de calendario entre el instante en que se envía el recordatorio y el
horario del turno — no a partir del valor de configuración de horas de
anticipación — de modo que el texto sea correcto sin importar qué ventana
produjo ese envío: "hoy" si el turno cae el mismo día de calendario que el
envío, "mañana" si cae exactamente un día después, o la fecha literal
("el DD/MM/YYYY") para cualquier brecha mayor.

## Decisiones y por qué

**El marcador se calcula a partir de la brecha real de calendario entre el
envío y el turno, no a partir del valor configurado de horas de
anticipación.** La alternativa más simple —mapear directamente el valor de
`hoursBefore` a una palabra ("≤ 20h → hoy", "20h-30h → mañana", etc.)— habría
acoplado la redacción del mensaje a la configuración vigente en el momento en
que se escribió el código, en lugar de al hecho real que el mensaje describe.
Además, el ancho de banda de detección de ±1 hora ya introduce un margen
entre "cuándo se configuró que se enviara" y "cuándo efectivamente se envía";
derivar el texto de la brecha real de calendario entre el envío y el turno
evita que ese margen —o un valor de configuración atípico, como 20 o 30
horas— produzca una palabra incorrecta que un mapeo fijo sobre `hoursBefore`
sí podría dar mal. `formatRelativeDay` recibe por eso dos fechas (el
`scheduledAt` del turno y el instante de envío), no un número de horas.

**El instante de envío que recibe `formatRelativeDay` es el mismo
`clinicNow()` con el que el lote de candidatos fue seleccionado, no un
`clinicNow()` nuevo tomado al procesar cada turno.** Tomar la hora actual de
nuevo en el momento de armar el mensaje habría podido, en el margen de un
lote con muchos turnos, hacer que `{relativeDay}` se calculara contra un
instante distinto al que decidió que ese turno era candidato — una
inconsistencia menor pero evitable pasando el mismo valor ya calculado.

**El valor por defecto pasó a 2 horas, no a un punto intermedio entre las
opciones que la propia auditoría sugería (2 u otro valor menor a 23).** El
documento de requisitos no fija un valor para esta ventana más allá de "por
defecto X horas antes del turno, configurable por tenant"; 2 horas separa con
margen amplio la ventana de recordatorio de la banda fija de 23-25 horas que
usa la confirmación, sin acercarse a otro límite de negocio existente (el
umbral de cancelación automática por falta de confirmación, de 4 horas).

**La refactorización de `notification-template.format.ts` extrajo el
formateo de fecha ("DD/MM/YYYY") a una función privada compartida por
`formatScheduledAt` y `formatRelativeDay`,** en lugar de que esta última
repitiera su propio relleno de ceros: ambas ya necesitaban exactamente el
mismo formato de fecha —una ya lo usaba como mitad de "DD/MM/YYYY HH:mm", la
otra lo necesita entero para su caso "más de un día"— y esa duplicación es
precisamente el patrón que una auditoría anterior de esta misma sección ya
había señalado y corregido una vez (helper de hora en reloj de pared,
documentado más arriba en esta sección).

## Entidades / puertos / adaptadores tocados

- `src/appointments/appointments.constants.ts`:
  `DEFAULT_APPOINTMENT_REMINDER_HOURS_BEFORE` pasó de 24 a 2, con el
  comentario actualizado explicando por qué 24 era el valor incorrecto
  (TUR-6). Sin cambio de esquema — la clave de configuración por inquilino
  (`appointment_reminder_hours_before`) no cambió, solo el valor que se
  aplica cuando el inquilino nunca la definió.
- `src/notifications/notification-template.format.ts`: nueva función
  `formatRelativeDay(scheduledAt, sentAt)`; el formateo de fecha
  ("DD/MM/YYYY") de `formatScheduledAt` se extrajo a una función privada
  `formatDate` reutilizada por ambas.
- `src/notifications/notification-template.constants.ts`: el texto base de
  `APPOINTMENT_REMINDER` cambió de "...mañana a las {time}." a
  "...{relativeDay} a las {time}.".
- `src/appointments/appointment-reminder.cron.ts`: `sendReminder` ahora
  recibe también el `now` (en milisegundos, el mismo `clinicNow()` usado
  para seleccionar el lote) y pasa `relativeDay: formatRelativeDay(...)`
  como parámetro adicional al renderizar la plantilla.

No se necesitó ninguna migración de datos: a diferencia de un cambio de
texto base de plantilla (que si ya existe sembrado en la configuración de
cada organización necesita una migración que lo actualice, como en la
corrección de duración de sesión documentada más arriba en esta sección),
este cambio agrega un marcador nuevo al texto base, no reemplaza uno
existente — cualquier organización que no haya personalizado esta plantilla
sigue leyendo el texto base actualizado directamente por el mecanismo normal
de "prioriza la fila de configuración, si no hay ninguna usa la constante
del código"; una organización que sí personalizó su propio texto no tiene
`{relativeDay}` y sigue sin tenerlo, la misma decisión ya tomada para la
corrección de duración de sesión.

## Tests y qué validan

- `notification-template.format.spec.ts` (nuevo): cobertura unitaria directa
  de `formatRelativeDay` — "hoy" para el mismo día de calendario (incluido un
  caso cerca de medianoche), "mañana" para exactamente un día de diferencia,
  la fecha literal para más de un día, y el caso defensivo de una fecha de
  turno anterior al instante de envío. También cubre `formatScheduledAt` y
  `formatTime` de forma directa, sin un caso dedicado previo.
- `appointment-reminder.cron.spec.ts`: actualizado el turno de referencia por
  defecto de 24 a 2 horas de anticipación (y la prueba de "fuera de la
  ventana" de 26 a 4 horas), y el caso central ahora verifica que
  `relativeDay: 'hoy'` viaja en los parámetros de renderizado. Se agregaron
  dos pruebas dedicadas a TUR-7: una ventana configurada a 26 horas (cruza
  exactamente un día de calendario) renderiza "mañana", y una configurada a
  50 horas (cruza dos días) renderiza la fecha literal en lugar de "mañana".
- `notification-template.service.spec.ts`: el fixture de parámetros
  completos para `APPOINTMENT_REMINDER` incluye ahora `relativeDay`, para
  que la prueba parametrizada de "renderiza sin marcadores sin reemplazar"
  siga cubriendo esta plantilla con su forma real.

Suite completa sin regresiones nuevas: 90 suites unitarias / 961 pruebas en
verde. De la suite e2e completa (53 suites / 627 pruebas), 52 suites pasan;
la única que falla (`patients-entities.e2e-spec.ts`, "admits the same DNI in
a different organization") se verificó — antes de atribuirle nada a este
cambio, siguiendo la lección ya documentada sobre no confiar en un
diagnóstico sin refrescar contra el estado real— que reproduce de forma
idéntica en un `git stash` contra `origin/main` sin ningún cambio de este
ticket: una tercera organización con datos ya presentes en la base de datos
local aparece donde la prueba espera solo dos, un problema de estado
preexistente en el módulo de Pacientes ajeno a Notificaciones/Scheduler y no
introducido por esta corrección. `lint` y `tsc --noEmit` sin errores sobre
los archivos tocados.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-158-reminder-window-and-relative-date` (creada desde
  `origin/main` fresco, tras el merge de TASK-157).
- Ticket: TASK-158 ("El recordatorio puede coincidir con la confirmación y
  su texto no se adapta a la ventana configurada"), tarea de auditoría
  automática (hallazgos TUR-6 y TUR-7) validada por la usuaria antes de
  implementarse. Misma convención de bitácora dedicada para una corrección
  puntual que TASK-157 ([[FASE-4_PROMPT-15]]) y TASK-129
  ([[FASE-4_PROMPT-14]]).
