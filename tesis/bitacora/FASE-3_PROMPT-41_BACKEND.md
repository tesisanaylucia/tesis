# Fase 3 — Motor de Turnos (backend) — reorganización de agenda por rango de fechas y corrimiento (TASK-203, corrección a TASK-39/TASK-115)

## Qué se implementó

TASK-203 señalaba una brecha entre lo implementado en [[FASE-3_PROMPT-6]]
(TASK-39) y el documento de especificación de requisitos: éste describe la
reorganización de agenda como la "posibilidad de reprogramar todos los
turnos de un período de tiempo", pero `POST /profesionales/:id/reorganizar-agenda`
solo aceptaba un lote explícito de pares `{appointmentId, scheduledAt}`, lo
que obligaba al cliente a traer primero todos los turnos del período y
armar ese lote por su cuenta. El ticket citaba como fuente la misma
auditoría multi-agente del 2026-08-28 que originó TASK-184, y señalaba
además que el flujo estaba bloqueado de punta a punta por el defecto de
TASK-184 (puerto de confirmación de reprogramación atado a un *stub*). Esa
dependencia se verificó resuelta antes de empezar: TASK-184 ya estaba en
estado "Listo" ([[FASE-3_PROMPT-40]]), y el `RESCHEDULE_RESPONSE_PORT` real
(`RescheduleResponseAdapter`, TASK-115) está fusionado a `main` desde el
2026-09-07.

Se agregó `POST /profesionales/:id/reorganizar-agenda/rango` como variante
declarativa, siguiendo el "fix sugerido" del propio ticket: el llamador
indica un período (`from`/`to`, días calendario inclusive, misma convención
que el resto de los rangos de fecha del contrato) y un corrimiento firmado
en horas (`offsetHours`, positivo adelanta, negativo atrasa; un día es
simplemente 24), y el servicio resuelve por sí mismo qué turnos del período
mover, en lugar de exigir la enumeración explícita. Solo se seleccionan los
turnos en estado reservado o confirmado —los mismos que ya exige la
reprogramación individual—, de modo que un turno completado, cancelado o
ausente dentro del período se deja sin tocar en lugar de aparecer como un
movimiento fallido bajo un identificador que el llamador nunca nombró. El
lote resuelto se resta al mismo cupo máximo por lote que ya regía la
enumeración explícita (100 turnos): un período que resuelve a más se
rechaza pidiendo acotar el rango, en lugar de aplicar solo una parte sin
avisar.

Cada turno del lote resuelto se procesa a través de exactamente el mismo
camino que ya usaba la enumeración explícita: la misma verificación de
pertenencia al profesional de la URL, la misma reprogramación con todas sus
validaciones (grilla habitual, fecha futura, elegibilidad de paciente
nuevo), el mismo contrato de falla parcial (un movimiento que falla no
aborta los demás) y la misma exigencia de confirmación del paciente
(TASK-115): un movimiento por rango nunca escribe de forma inmediata,
siempre registra su propia oferta de reprogramación. Para lograr esa
reutilización exacta, el cuerpo del bucle por movimiento que antes vivía
dentro de `reorganizeAgenda` se extrajo a un método privado compartido
(`applyAgendaMoves`), del que ambas variantes —la enumeración explícita y
la resolución por rango— dependen por igual.

## Decisiones y por qué

**El corrimiento se expresa como un único campo firmado en horas
(`offsetHours`), no como campos separados de días y horas.** El "fix
sugerido" del ticket habla de "N horas/días" como si fueran dos unidades
distintas, pero un corrimiento de un día calendario es aritméticamente
idéntico a uno de 24 horas sobre el instante ya almacenado (que incluye
hora, no solo fecha); introducir un segundo campo solo para expresar lo
mismo que ya expresa `offsetHours = 24 * N` habría sido redundante sin
agregar ninguna capacidad real. Un valor de cero se rechaza en el propio
DTO como un corrimiento sin sentido que el llamador casi seguro no
pretendía.

**El rango que resuelve el lote se acota únicamente por estado
(reservado/confirmado), no por si el resultado excede el cupo — esa
segunda verificación ocurre después de resolver el lote, no antes.** Solo
el servicio sabe cuántos turnos caen efectivamente dentro de un período
dado; exigir el cupo a nivel del propio DTO habría sido imposible sin
consultar primero la base.

**La resolución del lote se apoya en el mismo validador de rango
(`assertValidAgendaRange`) que ya usaba la lectura de agenda del
profesional (`GET /profesionales/:id/turnos`, [[FASE-3_PROMPT-19]]), en
lugar de duplicar la verificación de "hasta no puede ser antes que desde"
y el mismo tope de amplitud de rango.** Ambos endpoints comparten la misma
noción de "rango de agenda válido"; no había ninguna razón para que
difirieran.

**Se extrajo `applyAgendaMoves` como método privado compartido entre las
dos formas de reorganización, en lugar de duplicar el bucle por
movimiento.** El contrato de falla parcial, la exigencia de confirmación y
la verificación de pertenencia al profesional de la URL son exactamente
los mismos para un movimiento nombrado explícitamente que para uno
resuelto a partir de un rango; divergir esa lógica en dos copias habría
arriesgado que una futura corrección a una de las dos formas (como ya
ocurrió con TASK-115 sobre la forma explícita) se aplicara sin querer solo
a una de ellas.

## Alternativas descartadas

- **Aceptar el rango y el corrimiento como una forma alternativa del mismo
  cuerpo que ya recibe `POST /profesionales/:id/reorganizar-agenda`** (un
  DTO con validación condicional según qué campos llegan): descartada por
  una ruta separada (`/reorganizar-agenda/rango`) con su propio DTO, más
  simple de validar con las reglas declarativas de `class-validator` y más
  clara para quien lea el contrato, sin perder ninguna capacidad respecto
  de la alternativa.
- **Rechazar en el propio movimiento por rango los turnos en un estado no
  reprogramable, reportándolos como fallidos igual que un identificador
  explícito inválido**: descartada porque el llamador nunca nombró esos
  turnos — a diferencia de la enumeración explícita, donde un identificador
  que falla es información útil sobre el pedido que el propio llamador
  armó, aquí solo indicaría que el período elegido incluye turnos que de
  todos modos no iban a moverse.
- **Agregar campos separados de días y horas para el corrimiento**:
  descartada por la razón ya expuesta en la sección de decisiones —
  redundante frente a un único campo en horas.

## Entidades / puertos / adaptadores tocados

- `src/appointments/appointments.constants.ts` (modificado): nueva
  constante `MAX_AGENDA_REORGANIZE_OFFSET_HOURS`, cota de anti-abuso sobre
  la magnitud del corrimiento (no una regla de negocio), con el mismo
  criterio que ya fijaba `MAX_AGENDA_RANGE_DAYS` para la amplitud del rango.
- `src/appointments/dto/reorganize-agenda-range.dto.ts` (nuevo):
  `ReorganizeAgendaByRangeDto` (`from`, `to`, `offsetHours`).
- `src/appointments/appointments.service.ts` (modificado): nuevo método
  público `reorganizeAgendaByRange`; el cuerpo por movimiento de
  `reorganizeAgenda` se extrajo al método privado compartido
  `applyAgendaMoves`.
- `src/appointments/agenda.controller.ts` (modificado): nueva ruta
  `POST /profesionales/:id/reorganizar-agenda/rango`, con el mismo guard de
  propiedad (`ProfessionalOwnershipGuard`) que la ruta existente.

No se modificó el esquema de la base de datos: la operación resuelve un
lote de turnos ya existentes y lo reprograma por el mismo camino ya
existente, sin ninguna entidad ni columna nueva.

## Tests y qué validan

- `src/appointments/appointments-rescheduling.service.spec.ts` (ampliado):
  resolución del lote a partir del rango con corrimiento aplicado a cada
  turno y una oferta de reprogramación registrada por movimiento;
  corrimiento negativo; rechazo cuando el rango resuelve a más turnos que
  el cupo por lote, sin registrar ninguna oferta; rechazo de un rango mal
  formado (hasta antes que desde) sin llegar a consultar turnos; resultado
  vacío sin efectos secundarios cuando ningún turno cae en el rango.
- `test/appointments-rescheduling.e2e-spec.ts` (ampliado, contra
  PostgreSQL local, con un profesional dedicado para no depender de la
  garantía de no colisión que el resto del archivo ya obtiene de un
  contador de horas estrictamente creciente, que no cubre turnos
  desplazados por un corrimiento en lugar de reservados directamente):
  resolución de todos los turnos reservados/confirmados dentro del rango
  con el corrimiento aplicado tras aceptar cada oferta, dejando sin tocar
  un turno fuera del rango; rechazo al profesional que no es dueño de la
  agenda (403); rechazo de un rango mal formado (400); rechazo de un
  corrimiento de cero horas (400).
- Ejecución: suite unitaria completa en verde (93 conjuntos / 1015
  pruebas). Suite end-to-end completa en verde (53 conjuntos / 651
  pruebas), ejecutada en serie (`--runInBand`), incluyendo las cuatro
  pruebas nuevas de esta tarea. Los datos usados en las pruebas son
  ficticios.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-203-agenda-reorganize-by-range`
  (creada desde `main` ya actualizado, con TASK-184/TASK-115 fusionados).
- Ticket: TASK-203 (Jira), "[BAJO] Reorganización masiva de turnos requiere
  enumerar cada turno individualmente, no un rango de fechas". Depende de
  TASK-184 (ya resuelto, [[FASE-3_PROMPT-40]]) y corrige a TASK-39
  ([[FASE-3_PROMPT-6]]).
