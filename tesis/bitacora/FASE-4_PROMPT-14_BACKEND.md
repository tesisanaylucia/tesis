# Fase 4 — Notificaciones y Scheduler (backend) — un inquilino que falla aborta la corrida de los trabajos programados para el resto (TASK-129, corrección a TASK-109)

## Qué se implementó

TASK-129 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, validada por la usuaria
antes de implementarse. El hallazgo apuntaba al bucle sobre organizaciones
de `runForEachOrganizationAsSystem` (`src/common/scheduling/`, TASK-109):
no tenía ningún `try`/`catch` propio, mientras que cada trabajo programado
individual sólo protegía su propio bucle de candidatos, un nivel más
adentro. Si la lectura masiva de una organización lanzaba una excepción
—por ejemplo, un timeout del pool de conexiones— la excepción se propagaba
fuera del `for` que recorre las organizaciones, y ninguna de las
organizaciones siguientes se procesaba en esa corrida. Para el trabajo de
autocompletado semanal de turnos vencidos (sección 4.4), eso significaba
una semana entera perdida para el resto de las clínicas por la falla de
una sola.

La corrección movió el `try`/`catch` al nivel del propio bucle de
organizaciones, envolviendo tanto la resolución del usuario SYSTEM como la
llamada al `job` de cada cron: una organización cuya iteración lanza una
excepción, en cualquiera de esos dos pasos, queda registrada con
`logger.error` (incluyendo la traza) y la corrida continúa con la
organización siguiente, en lugar de abortar el resto. El caso ya cubierto
—una organización sin usuario SYSTEM se omite con `logger.warn` y sigue— no
cambió: ese camino nunca lanza una excepción, así que sigue resolviéndose
antes de llegar al nuevo `catch`.

## Decisiones y por qué

**El `try`/`catch` envuelve la llamada completa a `tenantContext.run`, no
sólo el `job`.** El hallazgo de la auditoría señalaba en particular "la
lectura masiva de un tenant", que en este helper ocurre en dos lugares
distintos dentro del mismo callback: la consulta del usuario SYSTEM
(`prisma.user.findFirst`) y la llamada al `job` de cada cron. Acotar la
protección sólo al `job` habría dejado sin cubrir exactamente el primer
paso que el hallazgo nombra como ejemplo — una consulta de base de datos
que puede fallar por la misma clase de error (timeout de pool) tanto si
ocurre resolviendo el usuario SYSTEM como si ocurre dentro del trabajo del
cron en sí. Envolver la llamada entera a `tenantContext.run` cubre ambos
sin duplicar el bloque `try`/`catch`.

**`logger.error`, no `logger.warn`, y con la traza.** El caso ya existente
de organización sin usuario SYSTEM es un estado válido y esperado del
sistema —una migración pendiente, un dato de siembra incompleto— y se
sigue registrando con `warn`. Una excepción no capturada por ningún nivel
inferior es, en cambio, una falla real que un operador necesita poder
diagnosticar: se promovió a `error` e incluye el `stack` de la excepción
cuando está disponible, siguiendo el mismo patrón de extracción de mensaje
(`error instanceof Error ? error.message : String(error)`) que ya usan los
bloques "mejor esfuerzo" de `AppointmentAutoCancellationCron` y sus pares
de esta misma sección, para no introducir una segunda convención de
formateo de errores en el mismo módulo.

**Sin cambio de comportamiento para el caso feliz ni para el caso ya
cubierto.** La corrección es puramente aditiva sobre el camino de error no
capturado que el hallazgo describe; no toca la resolución del usuario
SYSTEM, la firma del helper, ni el contrato que cada uno de los cinco
crons consumidores ya tenía con él.

## Entidades / puertos / adaptadores tocados

- `src/common/scheduling/run-for-each-organization-as-system.ts`: el
  `try`/`catch` se movió del nivel de cada cron individual (que seguía
  intacto, un nivel más adentro) al nivel del `for` sobre organizaciones,
  como pedía la solución propuesta por el hallazgo. Ningún consumidor
  (`AppointmentConfirmationCron`, `AppointmentAutoCancellationCron`,
  `AppointmentReminderCron`, `AppointmentAutoCompletionCron`,
  `WaitlistOfferTimeoutCron`) cambió su propio código: todos siguen
  llamando al helper con la misma firma.

## Tests y qué validan

Se agregó `run-for-each-organization-as-system.spec.ts`, primera prueba
unitaria dedicada al helper en sí — hasta ahora sólo estaba cubierto de
forma indirecta a través de la prueba "walks every organization" de cada
cron consumidor. Se centralizó aquí porque el helper es compartido por
cinco crons: una prueba directa sobre el propio helper evita repetir el
mismo caso "el fallo de una organización no aborta el resto" cinco veces,
una por cada archivo de prueba de cron. Casos cubiertos:

- recorre todas las organizaciones, cada una con su propio usuario SYSTEM
  resuelto y pasado al `job`;
- una organización sin usuario SYSTEM se omite con `logger.warn` y las
  siguientes se siguen procesando (comportamiento preexistente, no
  modificado por esta tarea);
- una excepción lanzada dentro del `job` de una organización no impide que
  las organizaciones siguientes se procesen, y queda registrada con
  `logger.error` nombrando la organización que falló (el caso central del
  hallazgo);
- una excepción lanzada al resolver el usuario SYSTEM —no sólo dentro del
  `job`— también queda aislada de la misma manera, confirmando que la
  protección cubre ambos pasos y no sólo el callback del cron.

Suite completa sin regresiones: 81 suites unitarias / 792 pruebas en
verde (+5 nuevas), 51 suites e2e / 562 pruebas en verde (`--runInBand`),
lint y `tsc --noEmit` sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-129-cron-tenant-isolation` (creada
  desde `origin/main` fresco, tras el merge de TASK-127). Pusheada a
  `origin`; PR abierto, no fusionado aún.
- Ticket: TASK-129 ("Un tenant que falla aborta la corrida de los crons
  para el resto"), tarea de auditoría automática (hallazgo
  [[FASE-4_PROMPT-11]], TASK-109) validada por la usuaria antes de
  implementarse. Misma convención de bitácora dedicada para una corrección
  puntual que TASK-121 ([[FASE-3_PROMPT-31]]) y TASK-84
  ([[FASE-2_PROMPT-11]]).
