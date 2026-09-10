# Fase 3 — Motor de Turnos (backend) — el cron de auto-cancelación puede cancelar retroactivamente turnos inminentes o ya pasados (TASK-152, corrección a TASK-44/TASK-155)

## Contexto

TASK-152 fue una tarea generada automáticamente por la auditoría de código
contra las fuentes de verdad (SRS) del 2026-08-20, la misma auditoría que dio
origen a TASK-150 ([[FASE-3_PROMPT-36]]), TASK-151 ([[FASE-3_PROMPT-37]]) y
TASK-153 ([[FASE-3_PROMPT-38]]), validada por la usuaria antes de
implementarse. El detalle completo está en el artefacto
`https://claude.ai/code/artifact/5c2ee112-c426-4d6e-a8e2-499ea7bbbc74`
("auditoria-psique-back.md"), hallazgo TUR-4.

TUR-4 señalaba que la consulta de candidatos de
`AppointmentAutoCancellationCron` (P4.3, cancelación por falta de
confirmación a las 4h) no tenía ningún piso sobre `scheduledAt`: un proceso
detenido varias horas —un despliegue, un reinicio, una caída— podía, al
reanudarse, encontrar turnos cuyo plazo de respuesta ya estaba vencido según
el cálculo de la fecha límite, sin importar si el turno mismo ya había
empezado o estaba por empezar. El escenario citado por la auditoría: el
proceso cae el domingo a la noche y se recupera el lunes a la mañana, con un
turno de las 09:00 ya con el paciente en camino; el cron lo cancela y dispara
la cascada de reasignación de lista de espera sobre un horario que, por
tener solo una hora de margen, no puede completar ni la primera oferta de
las cuatro horas de ventana antes de que el turno original ya haya pasado.

## Qué se implementó

Se agregó `scheduledAt: { gt: ahora }` a la consulta Prisma de
`cancelUnconfirmed` (usando `clinicNow()`, no `Date.now()`, por la misma
razón que TASK-150/TUR-2 ya documenta: `scheduledAt` es la hora de pared de
la clínica reetiquetada como UTC, no un instante real). Con este piso, un
turno cuyo propio horario ya llegó deja de ser candidato a auto-cancelación
sin importar cuán vencido esté su plazo de no-respuesta — no se inventó un
camino nuevo de "revisión manual": el turno queda `RESERVED` y es el barrido
semanal ya existente de `AppointmentAutoCompletionCron` el que termina
resolviéndolo a `COMPLETED` una vez que efectivamente finaliza, el mismo
desenlace que recibe cualquier turno que nadie resolvió a tiempo por
cualquier otro motivo — este cron no tiene forma de distinguir si el
paciente asistió o no.

Como consecuencia directa de ese piso, el tope `Math.min(deadline,
scheduledAt)` que `noResponseDeadline` calculaba desde TASK-155 (corrección
TUR-1, no documentada hasta esta entrada — ver nota más abajo) quedó
inalcanzable: una vez que todo candidato de la consulta cumple
`scheduledAt > ahora`, ese tope solo podía devolver un valor igual a
`scheduledAt`, y en ese caso `ahora < deadline` es una consecuencia lógica
de la propia condición de la consulta, de modo que la rama capada nunca
podía volver a disparar una cancelación. Se eliminó el tope en la misma
corrección en lugar de dejarlo como código muerto, y `noResponseDeadline`
volvió a depender únicamente de `confirmationRequestedAt`, sin necesitar
`scheduledAt` como parámetro.

## Decisiones y por qué

**Piso en la consulta, no un tope adicional sobre el plazo calculado.** La
auditoría proponía textualmente agregar `scheduledAt: { gt: ahora }` al
`where`; se siguió esa forma en vez de, por ejemplo, capar el resultado de
`noResponseDeadline` a un valor por debajo de `scheduledAt` (como ya hacía el
código de TASK-155), porque un piso a nivel de consulta excluye el turno de
la candidatura por completo, mientras que un tope sobre el plazo calculado
solo movería el momento en que la comparación en memoria lo descarta —sin
resolver el caso de un turno cuyo plazo, ya vencido, se evalúa por primera
vez varias horas después de que su propio horario llegó.

**No inventar una vía de "incidencia manual".** El hallazgo ofrecía dos
destinos alternativos para un turno vencido: completarlo o encaminarlo a una
revisión manual. No existe en el código ningún mecanismo de incidencia
manual para turnos, y crear uno solo para este caso habría sido una pieza de
infraestructura nueva sin precedente ni necesidad, dado que
`AppointmentAutoCompletionCron` ya resuelve exactamente ese vacío desde
TASK-89: cualquier turno `RESERVED`/`CONFIRMED` cuyo horario más su duración
ya pasó, y que nadie marcó como completado o ausente, se completa
automáticamente. Excluir el turno de la candidatura de este cron es
suficiente para que ese mecanismo ya existente lo alcance.

**Eliminar el tope redundante en vez de dejarlo.** Se consideró dejar el
`Math.min` sin tocar, ya que no cambia ningún resultado observable una vez
agregado el piso de la consulta. Se descartó esa opción porque deja código
sin ningún camino de ejecución real, que un lector futuro podría interpretar
como una regla vigente y no como el efecto colateral de una corrección
anterior — se prefirió simplificar `noResponseDeadline` a su única
responsabilidad restante (el ajuste de fin de semana de TUR-1) y actualizar
el comentario cruzado de `business-day.ts` que documentaba esa
responsabilidad.

## Punto no evidente: dos tareas previas de esta misma fase (TASK-155/TUR-1,
TASK-156/TUR-3) no tenían entrada de bitácora

Al inspeccionar el código vigente de `appointment-auto-cancellation.cron.ts`
antes de esta corrección, ya estaban presentes tanto el ajuste de fin de
semana del plazo de no-respuesta (TUR-1) como la notificación al profesional
y al paciente al cancelar (TUR-3) — ninguno de los dos introducido por esta
tarea. Revisando el historial de `git log` se confirmó que ambos corresponden
a tareas ya fusionadas a `main` antes de esta rama: TASK-155 ("weekend rule
for the no-response cancellation deadline (TUR-1)") y TASK-156
("auto-cancellation cron notifies professional and patient (TUR-3)"). Ninguna
de las dos tiene entrada de bitácora en este directorio, y la entrada de
[[FASE-3_PROMPT-38]] (TASK-153) cita por error "TASK-152" como ya fusionada
en su branch de referencia, cuando el commit que cita (`43256ab`) corresponde
en realidad a TASK-155/TASK-156. Se deja constancia aquí en lugar de
corregir esa entrada anterior o inventar retroactivamente el contenido de
las dos tareas no documentadas, ya que esta tarea no tuvo visibilidad directa
de las decisiones tomadas al implementarlas — queda como pendiente para la
usuaria decidir si se documentan retroactivamente.

## Entidades / puertos / adaptadores tocados

- `src/appointments/appointment-auto-cancellation.cron.ts` (modificado):
  `cancelUnconfirmed` (agrega `scheduledAt: { gt: ahora }` al `where`, mueve
  el cálculo de `ahora` antes de la consulta); `noResponseDeadline` (pierde
  el parámetro `scheduledAt` y el tope `Math.min`).
- `src/common/dates/business-day.ts` (modificado, solo comentario): la
  referencia cruzada a dónde se aplica el tope de `scheduledAt` se actualiza
  para apuntar al piso de la consulta en vez de al tope ya eliminado de
  `noResponseDeadline`.
- Sin cambios de esquema: ningún modelo nuevo, ninguna migración — la
  corrección es exclusivamente de la condición de selección de candidatos.

## Tests y qué validan

- `src/appointments/appointment-auto-cancellation.cron.spec.ts` (unitaria):
  el mock de `findMany` pasa a aplicar también el predicado `scheduledAt`
  que la consulta real usa, para que las pruebas ejerciten el filtro nuevo
  y no solo la lógica en memoria. Se reemplazó la prueba que fijaba como
  correcto cancelar exactamente en el instante en que el turno comienza (el
  propio comportamiento que TUR-4 corrige) por dos pruebas que verifican lo
  contrario: un turno no se selecciona una vez que su horario ya llegó
  (mismo escenario domingo/lunes de TASK-155, ahora con el resultado
  invertido), y un turno no se selecciona si su horario ya pasó por
  completo. Se agregó además una prueba que confirma que la cancelación
  normal —con margen suficiente antes de `scheduledAt`— sigue funcionando
  sin cambios.
- Ejecución: suite unitaria completa en verde (896 pruebas, 87 suites);
  `eslint` y `tsc --noEmit` sin hallazgos. Suite de extremo a extremo no
  ejecutada contra Postgres real en esta corrección (ningún spec e2e
  existente ejercita este cron); los datos usados en las pruebas son
  ficticios.

## Figuras pendientes

Ninguna figura nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-152-auto-cancellation-scheduled-at-floor`,
  creada desde `origin/main` fresco (`1870a14`, con TASK-150/TASK-151/
  TASK-153/TASK-155/TASK-156 ya fusionadas).
- Ticket: TASK-152 ("El cron de auto-cancelación puede cancelar
  retroactivamente turnos inminentes o ya pasados"), tarea de auditoría
  automática (hallazgo TUR-4) validada por la usuaria antes de
  implementarse. Misma convención de bitácora dedicada para una corrección
  puntual dentro de la fase del ticket original que TASK-150/TASK-151/
  TASK-153 ([[FASE-3_PROMPT-36]]/[[FASE-3_PROMPT-37]]/[[FASE-3_PROMPT-38]]).
