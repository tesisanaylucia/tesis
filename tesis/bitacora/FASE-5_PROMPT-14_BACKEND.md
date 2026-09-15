# Fase 5 — Capa conversacional y WhatsApp (backend) — serialización de turnos concurrentes por sesión (TASK-192, corrección a P5.3)

## Contexto

La misma auditoría multi-agente contra las fuentes de verdad (anteproyecto
de tesis y SRS), corrida el 28 de agosto de 2026, que originó el hallazgo
crítico TASK-183 sobre el webhook de WhatsApp, generó también un hallazgo
de prioridad media (TASK-192) sobre el orquestador de conversación
(`OrquestadorService`, P5.3, TASK-48) y su store de historial
(`ConversationSessionStore`). `OrquestadorService.runTools` ya ejecuta,
desde P5.3, las herramientas de un mismo turno de forma secuencial —nunca
con `Promise.all`, precisamente para no competir por escrituras que
comparten el mismo contexto de inquilino—, pero esa secuencialidad es
*intra*-turno: nada impedía que dos llamadas a `procesar()` para el mismo
`sessionId` corrieran en paralelo. Dos escenarios concretos lo disparan: un
webhook de WhatsApp reentregado bajo un identificador de mensaje (wamid)
distinto al que TASK-183 ya deduplica (por ejemplo, dos mensajes de texto
reales enviados por el paciente segundos aparte), o un paciente tocando
enviar dos veces antes de que la primera respuesta llegue. Ambas
ejecuciones leían el mismo historial previo desde
`ConversationSessionStore.history()` antes de que ninguna hubiera guardado
el suyo, podían ejecutar herramientas con estado de forma independiente
sobre ese mismo turno —el propio hallazgo cita `book_appointment` y
`request_prescription` como ejemplos concretos— y competían en `save()`:
la última escritura en completarse ganaba y el turno completo de la otra
llamada desaparecía en silencio del historial de contexto. El hallazgo
señala explícitamente que esto agrava el propio hallazgo de idempotencia de
TASK-183, ya que aquella corrección sólo cubre la reentrega del *mismo*
wamid — no protege contra dos mensajes distintos y genuinamente
concurrentes de la misma sesión.

## Qué se implementó

Se agregó `SessionTurnQueue` (`src/chatbot/session-turn-queue.ts`), una
cola en memoria de un solo proceso, indexada por `sessionId`, con un único
método `run<T>(sessionId, turn)`. `OrquestadorService.procesar` la usa para
envolver el turno completo — no sólo `runTurn`, sino también la apertura de
`TenantContextService.run` y `ConversationContextService.run` que ya
rodeaban a `runTurn` — antes de que corra. Cada llamada a `run` encadena su
propio `turn` detrás del último turno ya encolado para ese `sessionId`, de
modo que el segundo turno de dos llamadas concurrentes sólo empieza a
ejecutarse una vez que el primero terminó de guardar (o falló) por
completo, nunca en paralelo con él. Se registró como proveedor nuevo en
`ChatbotModule`.

## Decisiones y por qué

**La cola serializa todo `procesar()`, no sólo `runTurn`.** El propio
hallazgo sugiere "serializar los turnos por sessionId ... antes de ejecutar
runTurn", lo que podría leerse como serializar únicamente esa función
interna. Se decidió envolver la llamada completa —incluidas las dos
aperturas de contexto (`TenantContextService`, `ConversationContextService`)
que hoy ya preceden a `runTurn` dentro de `procesar`— porque dejarlas fuera
de la cola no habría cerrado la ventana real de la carrera: el historial
que se lee y se guarda es responsabilidad de `runTurn`, así que en cuanto
esa función arrancara, dos turnos seguirían compitiendo por el mismo
`sessionId` exactamente igual que antes. Serializar el método público
completo es además más simple de razonar como invariante ("un turno por
sesión corre de punta a punta antes que el siguiente") que dos alcances de
serialización distintos dentro del mismo método.

**Encadenado con `.then(turn, turn)`, no con `.finally`.** `.finally` se
ejecuta tanto si la promesa anterior resolvió como si rechazó, pero no
puede cambiar qué corre a continuación — sólo observar el resultado y
dejarlo pasar (o reemplazarlo por su propio error, si el callback de
`finally` lanza). Lo que la cola necesita es exactamente lo contrario:
que el turno siguiente se ejecute *como reacción* tanto a un éxito como a
un fallo del turno anterior, sin heredar su rechazo. `.then(turn, turn)` —
pasar la misma función como manejador de éxito y de fallo— logra eso: cada
turno encolado corre exactamente una vez, sin importar cómo terminó el que
tenía adelante, y un turno que falla nunca deja atascados (*wedged*) a los
que esperan detrás suyo en la misma sesión.

**Sin TTL, a diferencia de `ConversationSessionStore` y
`ProcessedWhatsappMessageStore`.** Ambos stores hermanos necesitan una
política de expiración porque sus claves (`sessionId` en un caso, wamid en
el otro) pueden quedar en el mapa mucho después de que su turno terminó —
una sesión inactiva, un wamid que nunca vuelve a reclamarse. La cola de
turnos no tiene ese problema: la entrada que el mapa guarda para un
`sessionId` es la promesa del turno actualmente en curso (o del último
encolado detrás de él); en cuanto ese turno liquida y nada quedó encolado
detrás, la propia llamada a `run` borra la entrada dentro de su bloque
`finally`. Ninguna clave sobrevive más allá del turno que la usó, así que
no hace falta ningún barrido por tiempo.

**No se usó una dependencia externa de mutex/semáforo (p. ej.
`async-mutex`).** El problema se resuelve por completo encadenando
promesas nativas de JavaScript; agregar una dependencia nueva sólo para
esto no se justifica, con el mismo criterio de infraestructura mínima que
ya documentó `ProcessedWhatsappMessageStore` bajo TASK-183 — CLAUDE.md no
nombra ninguna infraestructura de colas o locks distribuidos para el
despliegue del piloto.

## Alternativas descartadas

- **Serializar sólo alrededor de `runTurn`, tal como la redacción literal
  del hallazgo sugiere**, dejando la apertura de `tenantContext`/
  `conversationContext` fuera de la cola: descartada porque no cierra la
  ventana real de la carrera — ver la primera decisión arriba.
- **Encadenar con `.finally(turn)`** en lugar de `.then(turn, turn)`:
  descartada porque `.finally` no puede hacer que el turno siguiente
  dependa del anterior sin heredar su rechazo; hacerlo así habría dejado
  atascada la cola de una sesión completa detrás de un único turno
  fallido.
- **Agregar una dependencia de mutex de terceros**: descartada por no
  aportar nada que una cadena de promesas nativa no resuelva ya, ver la
  última decisión arriba.
- **Reutilizar la forma de `ConversationSessionStore` (TTL + expiración
  perezosa) para esta cola**: descartada porque no hay ninguna clave que
  necesite sobrevivir más allá de su propio turno; ver la decisión sobre
  el TTL arriba.

## Entidades / puertos / adaptadores tocados

- `src/chatbot/session-turn-queue.ts` (nuevo): `SessionTurnQueue`.
- `src/chatbot/orquestador.service.ts`: `procesar` envuelve el turno
  completo en `turnQueue.run(sessionId, ...)`.
- `src/chatbot/chatbot.module.ts`: registra `SessionTurnQueue` como
  proveedor del módulo.
- Ningún cambio de esquema ni de migración: la carrera era puramente en
  memoria, sobre un store (`ConversationSessionStore`) que ya era en
  memoria antes de esta corrección.

## Tests y qué validan

- `src/chatbot/session-turn-queue.spec.ts` (nuevo): un turno único
  devuelve su resultado; un turno rechazado propaga su error a quien lo
  encoló; un segundo turno para el mismo `sessionId` queda retenido hasta
  que el primero se resuelve (con una promesa controlada por la prueba
  para probar el orden real, no sólo confiar en el orden de
  *scheduling* de microtareas); dos `sessionId` distintos corren sin
  esperarse entre sí; un turno encolado detrás de uno que rechazó igual
  se ejecuta; un turno posterior sobre una cola ya drenada corre de
  inmediato.
- `src/chatbot/orquestador.service.spec.ts`: nueva sección "concurrent
  turns on the same session (TASK-192)" con tres pruebas — la llamada de
  `AIPort` del segundo turno queda retenida hasta que el primer turno
  terminó de guardar, y cuando corre ya ve el intercambio del primer
  turno como contexto (prueba directa de que ninguna de las dos llamadas
  lee historial obsoleto); un turno encolado detrás de uno que falló
  igual se ejecuta; dos turnos en sesiones distintas no se esperan entre
  sí.
- Ejecución: suite unitaria completa en verde (93 conjuntos, 1000
  pruebas); análisis estático (ESLint) sobre el árbol completo y
  verificación de tipos (`tsc --noEmit`) sin advertencias. Datos
  ficticios, sin contenido clínico ni datos personales reales. La suite
  de extremo a extremo no se corrió en esta sesión (requiere base de
  datos disponible); no se tocó ningún archivo bajo `test/`.

## Figuras pendientes

Ninguna figura nueva; la corrección no introduce un flujo o diagrama
distinto del ya registrado para el orquestador (P5.3) — el ordenamiento
serializado es un detalle interno del componente, no un flujo nuevo
visible desde el diagrama de arquitectura.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-192-serialize-conversation-turns`
  (creada a partir de `main`).
- Ticket: TASK-192 ("[MEDIO] Race condition en ConversationSessionStore —
  dos turnos concurrentes se pisan"), hallazgo de la misma auditoría
  multi-agente del 28 de agosto de 2026 que originó TASK-183, sobre
  `src/chatbot/conversation-session.store.ts` y
  `src/chatbot/orquestador.service.ts` (`runTurn`).
