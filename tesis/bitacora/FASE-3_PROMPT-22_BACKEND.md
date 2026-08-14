# Fase 3 — Motor de Turnos (backend) — hardening de integridad sin ticket ni bitácora: `CHECK` constraints y limpieza de índices redundantes (TASK-104, trazabilidad retroactiva sobre un cambio ya fusionado)

## Qué se implementó

TASK-104 no pedía código: pedía darle trazabilidad de tesis a un cambio de
esquema que ya estaba fusionado a `main` desde el 28 de julio de 2026 sin
haber pasado nunca por el flujo de la skill `documentacion-tesis`. La
migración `20260728171035_harden_integrity_and_drop_redundant_indexes`
(rama `fix/db-integrity-hardening`, PR #39, merge `da1ae11`) se escribió y
fusionó sin ticket Jira asociado y sin entrada de bitácora — un caso
distinto de TASK-81 (verificación de un trabajo que sí tenía ticket, solo
mal reportado por una auditoría desactualizada): acá el trabajo es real y
está correctamente aplicado, pero nunca quedó documentado. Lo encontró una
auditoría multi-agente de `psique-back` sobre `main` (2026-08-12, ángulo
completitud de módulos vs. anteproyecto): `grep -rl
"db-integrity-hardening\|harden_integrity_and_drop" skill/tesis` no
devolvía ningún resultado.

El cambio en sí, según el propio mensaje de commit y el diff de
`prisma/schema.prisma`, tiene dos partes independientes:

- **Tres restricciones `CHECK`**, escritas a mano en el SQL de la migración
  porque Prisma no admite `@@check` en `schema.prisma` (versión 6.19.3 al
  momento del cambio), siguiendo el mismo patrón ya usado por el rol
  agregado en TASK-72 y, más tarde, por la restricción de rol/profesional
  de TASK-93: `Appointment.duration > 0` (una duración de cero o negativa
  no tiene sentido; la aplicación ya la deriva del lado del servidor y
  nunca la acepta del llamador, así que la restricción es un resguardo de
  base de datos, no una regla nueva), `Absence.startDate <= endDate` (la
  ausencia es un rango de días calendario inclusive, y el servicio
  correspondiente ya rechazaba un rango invertido) y `WaitlistEntry.order
  > 0` (la posición en la cola es de base uno, calculada por el propio
  servicio como `(último?.orden ?? 0) + 1`, así que cero o negativo tampoco
  tiene sentido). Las tres son un resguardo a nivel de base de datos para
  una invariante que el código de aplicación ya respetaba, no una regla de
  negocio nueva.
- **Eliminación de tres índices de una sola columna** (`AuditLog`,
  `Professional` y `User`, los tres sobre `organizationId` a secas), por
  ser redundantes: en los tres modelos ya existe un índice o restricción
  única con `organizationId` como columna más a la izquierda (el
  `@@unique([organizationId, id])` de `Professional`/`User`, y
  `@@index([organizationId, userId])` de `AuditLog`), que sirve cualquier
  consulta acotada solo por organización sin necesitar el índice aparte —
  la misma razón por la que `Patient` y `Specialty` ya evitaban ese
  duplicado desde su diseño original.

Esta entrada de bitácora documenta ambas partes retroactivamente. No se
tocó código ni se rehizo ni revirtió la migración: el propio ticket la
marca fuera de alcance, ya que está correctamente aplicada y el commit
original la verificó contra la base de datos viva (vía `psql`, sin
detectar `drift`) con la suite completa en verde (27 suites / 273 tests
unitarios, 28 suites / 349 tests end-to-end).

## Decisiones y por qué

**No se reconstruyó una justificación que el propio commit ya da.** El
mensaje de commit de `259066e` ya explica el motivo de cada uno de los seis
cambios (tres índices eliminados, tres restricciones agregadas) con el
mismo nivel de detalle que se espera de una entrada de bitácora; esta
entrada resume y organiza esa explicación en vez de inventar una nueva,
siguiendo la regla de la propia skill de no atribuir un porqué que no esté
respaldado por el código, los tests o el propio mensaje de commit.

**Se evaluó el hueco de detección del Stop-hook que permitió que este caso
pasara inadvertido, en vez de asumir que fue solo un descuido de proceso.**
El hook `back/.claude/hooks/check-tesis-pendiente.js` bloquea el cierre de
sesión comparando el timestamp del último commit que toca `src/`/`prisma/`
contra el `mtime` más reciente entre *todos* los archivos `_BACKEND.md` de
la bitácora — no contra si alguno de esos archivos describe el cambio
puntual que se acaba de hacer. En el momento en que se fusionó la
migración de hardening, el commit que la trae (`259066e`, 28 de julio) sí
era el cambio de código más reciente sobre `prisma/`; el hueco aparece más
tarde, apenas se escribe *cualquier* otra entrada de bitácora del backend
después de esa fecha — lo que ocurrió muy pronto, dado el ritmo de tareas
del proyecto —, porque desde ese momento el hook vuelve a ver
`lastBitacora > latestCodeChange` y aprueba el cierre sin haber verificado
nunca que la migración de hardening específicamente quedó documentada.
Es, en otras palabras, una comparación de *recencia* donde hacía falta una
comparación de *cobertura*. Corregir esto requiere que el hook (u otro
mecanismo) pueda determinar qué migraciones/commits ya están referenciados
en el contenido de la bitácora, no solo cuándo fue tocada por última vez —
un cambio de diseño no trivial y fuera del alcance documental de TASK-104,
tal como el propio ticket contempla. Se abre un ticket de seguimiento
dedicado para esa mejora en vez de improvisarla dentro de esta tarea (ver
sección siguiente).

**No se corrigió retroactivamente el estado de ningún otro ticket.** Al
revisar si el mismo tipo de hueco (código fusionado sin ticket ni
bitácora) se repetía en otro punto de la historia de `main`, se comparó la
lista completa de tickets con commit de merge en `main` contra los
tickets referenciados en algún archivo de `tesis/bitacora/`. Las únicas
tres discrepancias — TASK-12, TASK-13 y TASK-71 — corresponden a trabajo
fusionado el 15 y 16 de julio de 2026, antes de que el propio directorio
de tesis compartido existiera (`tesisDir` se inicializó el 16 de julio) y,
en el caso de TASK-71 específicamente, a la tarea que creó la skill
`documentacion-tesis` — no puede documentarse a sí misma antes de existir.
No son huecos de proceso equivalentes al de TASK-104: son trabajo anterior
al propio mecanismo de captura, y quedan fuera de alcance de esta
corrección.

## Alternativas descartadas

- **Fusionar esta documentación dentro de la entrada de TASK-93** (la
  corrección más cercana en el tiempo dentro de 3.2.0 que también agrega
  un `CHECK` a mano): descartada porque TASK-93 y la migración de hardening
  son cambios distintos, con commits, ramas y motivación propias —
  agruparlos habría oscurecido cuál commit corresponde a cuál ticket.
- **No abrir un ticket de seguimiento para el hueco del Stop-hook y
  dejarlo solo mencionado en el texto**: descartada porque el propio
  ticket ofrece esa opción explícitamente y el hueco es real y accionable
  (la lógica exacta a cambiar ya está identificada en
  `check-tesis-pendiente.js`); dejarlo sin ticket es exactamente el patrón
  que causó que este caso pasara inadvertido en primer lugar.

## Entidades / puertos / adaptadores tocados

Ninguno en el repo backend — el código de la migración ya estaba fusionado
antes de que este ticket existiera. Referencia:
`prisma/migrations/20260728171035_harden_integrity_and_drop_redundant_indexes/migration.sql`
y los comentarios agregados sobre `Absence.startDate`/`endDate`,
`Appointment.duration` y `WaitlistEntry.order` en `prisma/schema.prisma`.

## Tests y qué validan

No aplica — tarea documental retroactiva, sin cambios de código en esta
tarea. La verificación de la migración en sí ya había ocurrido en el
commit original (`259066e`, 28 de julio de 2026): confirmación contra la
base de datos viva vía `psql` de índices, restricciones y ausencia de
`drift`, y suite completa en verde (27 suites / 273 tests unitarios, 28
suites / 349 tests end-to-end, `--runInBand`).

## Figuras pendientes

Ninguna nueva. Las tres restricciones `CHECK` que agrega esta migración
son candidatas naturales a aparecer, junto con la de TASK-93, cuando se
produzca la Figura 10 ya pendiente ("Diagrama entidad-relación consolidado
del esquema tras la revisión de integridad y normalización",
`figuras_pendientes.md`) — no se agrega una fila nueva porque el alcance
de esa figura ya las cubre conceptualmente.

## Componente y referencia

- Componente: backend.
- Rama y PR original del cambio documentado: `fix/db-integrity-hardening`
  (PR #39, Bitbucket), fusionada a `main` el 28 de julio de 2026 (merge
  `da1ae11aa7b82949c5f5702a7974564ba2dc51bb`, commit de contenido
  `259066ea982cad07cdf280b2b625843e26dba92e`). Esta tarea (TASK-104) no
  abre rama propia en el repo backend: es puramente documental sobre la
  tesis compartida, sin cambios en `back/`.
- Ticket: TASK-104 ("[CORRECCIÓN] Migración de hardening de integridad
  (harden_integrity_and_drop_redundant_indexes) sin ticket ni bitácora"),
  trazabilidad retroactiva. Mismo patrón que las correcciones sin ticket
  previo documentadas ya en TASK-81 ([[FASE-3_PROMPT-14]]).
- Ticket de seguimiento abierto por esta tarea: TASK-112 ("[MEJORA] El
  Stop-hook `check-tesis-pendiente.js` detecta recencia de bitácora, no
  cobertura del cambio puntual"), para evaluar y eventualmente implementar
  que el hook compare contra el contenido de la bitácora más reciente y no
  solo contra su `mtime`. No implementado en esta tarea.
