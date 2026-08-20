# Fase 3 — Motor de Turnos (backend) — la cadencia de turnos no llegaba a la generación de la agenda (TASK-114, corrección a TASK-35)

## Contexto

El documento de requisitos distingue explícitamente dos parámetros de agenda
del profesional, en la enumeración del Anexo, Módulo Profesionales:
*"Duración de la consulta configurable: cada profesional define la duración
de sus turnos desde la app (por ejemplo, atención cada una hora con sesiones
de 45 minutos, o cada 30 minutos con sesiones de 20 minutos)"*. Cada ejemplo
nombra dos magnitudes distintas: cada cuánto se abre un horario ofrecible —la
cadencia— y cuánto dura la sesión que ocupa ese horario.

TASK-24 ([[FASE-1_PROMPT-4]]) había separado ambos datos en el modelo, tras
una revisión que detectó que la configuración inicial los colapsaba en uno
solo: se agregó `Professional.slotCadence` junto a
`Professional.consultationDuration`, anulable hasta configurarse, y se lo
expuso en `PATCH /profesionales/:id/configuracion`. La bitácora de esa tarea
lo registra como un dato "destinado a alimentar la futura generación de
agenda".

Esa generación de agenda es TASK-35 ([[FASE-3_PROMPT-2]]), y no llegó a
consumirlo. Ambos recorridos que producen instantes de turno en
`AvailabilityService` —el que arma la agenda ordinaria y el que arma la
grilla sobre la que se apoya la regla de doble franja para paciente nuevo—
avanzaban con `minutes += duration`, es decir, siempre con la duración de la
sesión. La propia bitácora de TASK-35 asentó la decisión como deliberada
("El paso de generación es `consultationDuration`, no `slotCadence`"),
atribuyéndola al documento de requisitos; la relectura de la fuente de verdad
para esta corrección no la respalda: el documento no especifica un paso de
generación distinto de la cadencia que él mismo introduce, y el ejemplo que
da es precisamente el caso que el código no podía producir. El efecto
observable era que un profesional configurado con cadencia de sesenta minutos
y sesiones de cuarenta y cinco —el ejemplo textual de la fuente de verdad—
seguía viendo horarios ofrecidos cada cuarenta y cinco minutos, y la
configuración que el sistema le permitía guardar no cambiaba nada.

Ningún test del repositorio mencionaba la cadencia. La detección proviene de
una auditoría de código sobre `psique-back/main` (2026-08-14, agente "Audit
Profesionales vs SRS").

## Qué se implementó

- La cadencia pasa a ser el paso entre inicios de turno consecutivos, y la
  duración de la consulta sigue siendo la longitud de la sesión. Un
  profesional sin cadencia configurada conserva exactamente la agenda
  anterior, porque el paso cae de nuevo en la duración.
- La condición de corte de cada bloque de atención sigue exigiendo que la
  *sesión completa* entre antes del fin del bloque, no la cadencia: un bloque
  de 09:00 a 17:00 con cadencia sesenta y sesiones de cuarenta y cinco ofrece
  de 09:00 a 16:00 y no un inicio a las 17:00.
- El campo `duration` de cada franja devuelta sigue siendo la duración de la
  consulta. Es el dato que el asistente comunica al paciente al confirmar, y
  la cadencia no debía filtrarse a la respuesta.
- La regla de doble franja para paciente nuevo pasa a emparejar turnos
  separados por una cadencia, no por una duración. El documento de requisitos
  pide "dos turnos consecutivos de su agenda", y en una agenda con cadencia el
  turno siguiente empieza una cadencia después.
- La reserva de una primera sesión coloca la segunda mitad del turno doble una
  cadencia después de la primera, no una duración después.
- Se agregó una invariante de configuración: la cadencia no puede ser menor
  que la duración de la consulta.

## Decisiones y por qué

**Las dos configuraciones se leen y se interpretan en un único lugar
compartido.** La causa del defecto no fue una fórmula equivocada sino una
duplicación: existían dos recorridos que generaban instantes de turno, cada
uno con su propia copia del paso, y la corrección de uno solo habría dejado
la agenda ordinaria y la grilla de paciente nuevo produciendo instantes
distintos. Se introdujo entonces un módulo nuevo, `slot-schedule.ts`, que
concentra la lectura del par de columnas, la validación de que la duración
está configurada y la derivación del paso efectivo; el servicio de
disponibilidad y el de turnos lo consumen ambos. La consecuencia buscada es
que los instantes que la agenda ofrece y los instantes que una reserva ocupa
no puedan discrepar, porque provienen de una única definición.

**Los dos recorridos de generación se unificaron en uno.** Además de
compartir la interpretación de la configuración, se reemplazaron los dos
loops por un único método que devuelve la grilla completa de instantes por
día calendario. La agenda ordinaria es esa grilla menos los instantes ya
ocupados; la regla de doble franja necesita la grilla sin esa resta, porque
debe poder distinguir "el primer turno del día está tomado" —caso en el que
no ofrece nada— de "ese turno no existe en la grilla", distinción que la
lista de franjas libres no puede hacer, ya que ambos casos se le presentan
igual, como ausencia. Mantenerlos como dos recorridos separados era
exactamente lo que permitió que la cadencia no llegara a ninguno de los dos.

**La segunda mitad del turno doble se coloca una cadencia después, y esta
consecuencia no era opcional.** La reserva de una primera sesión validaba el
instante de inicio contra la grilla de paciente nuevo y luego derivaba el
segundo instante sumando la duración de la sesión. Con cadencia sesenta y
sesiones de cuarenta y cinco, eso habría escrito un turno a las 09:45: un
instante que la agenda no ofrece, que ninguna consulta de disponibilidad
puede volver a mostrar, y que además se solapa con el turno de las 10:00, que
sí sigue ofreciéndose porque la comparación de ocupación es por instante
exacto. Es decir, hacer efectiva la cadencia sin corregir también este punto
habría creado un camino de sobreventa que antes no existía, no porque el
cálculo fuera correcto antes, sino porque la cadencia no llegaba a producir
la discrepancia.

**Se agregó la restricción de que la cadencia no sea menor que la duración de
la consulta.** Mientras la cadencia no se usaba, el par no podía contradecirse
en ningún efecto observable. Al hacerla efectiva, una cadencia menor que la
sesión abre el turno siguiente mientras la sesión anterior todavía transcurre:
la agenda ofrece dos horarios solapados y, como la verificación de
disponibilidad compara instantes exactos y no intervalos, ambos son
reservables. La restricción se ubicó en el servicio y no en el objeto de
transferencia de datos porque la comprobación abarca el par tal como quedará
*almacenado*: la actualización es parcial y puede traer una sola de las dos
columnas, de modo que una validación que sólo mirara el cuerpo del pedido se
satisfaría enviando las dos mitades en dos pedidos separados. Por el mismo
motivo es una invariante de lectura-y-escritura y se ejecuta dentro de la
transacción serializable que ese endpoint ya abría por la verificación de
matrículas: dos actualizaciones concurrentes, una fijando la duración y otra
la cadencia, podrían de otro modo leer cada una un estado que permite su
propia escritura y dejar entre ambas un par que ninguna habría aceptado.
Valores iguales son la agenda consecutiva de siempre y siguen siendo válidos.

**La restricción no se expresó como restricción `CHECK` en la base de
datos.** Ambas columnas son anulables hasta que la fase de configuración las
fija, y la mitad no configurada no puede contradecir a la otra; una
restricción de tabla que las compare debería además tolerar los tres estados
intermedios, y el mensaje de error que produciría no distingue cuál de las
dos mitades el operador acaba de mover. Se dejó constancia de la ubicación
elegida en el comentario de la columna en el esquema, para que la regla sea
visible desde el modelo y no sólo desde el servicio.

## Alternativas descartadas

- **Corregir sólo el paso de los dos loops, tal como enuncia el ticket, sin
  tocar el emparejamiento de la regla de paciente nuevo ni la reserva del
  turno doble**: descartada porque el resultado sería una configuración que
  el sistema acepta y que rompe una funcionalidad ya entregada. Con cadencia
  y duración distintas, la comprobación de adyacencia —que exige que dos
  franjas consecutivas disten exactamente una duración— no encontraría ningún
  par, y la modalidad de paciente nuevo dejaría de ofrecer horarios en
  silencio; y la reserva colocaría la segunda mitad fuera de la grilla, con la
  sobreventa descrita más arriba.
- **Derivar el paso efectivo en cada punto de consumo, en vez de en un módulo
  compartido**: descartada por ser la forma exacta del defecto que se
  corrige. Dos derivaciones independientes de la misma regla es lo que
  permitió que ninguna de las dos la aplicara.
- **Interpretar la cadencia como duración del turno y no como espaciado**,
  con lo que el campo `duration` de la respuesta pasaría a valer la cadencia:
  descartada por contradecir la fuente de verdad, que pide informar al
  paciente la duración de la sesión ("al confirmar el turno se informará al
  paciente la duración de la sesión (por ejemplo, 45 minutos)"), y por
  cambiar el significado de un campo del contrato ya consumido.
- **Permitir una cadencia menor que la duración y resolver el solapamiento en
  la verificación de disponibilidad**, comparando intervalos en vez de
  instantes exactos: descartada por desproporcionada. Sostener una
  configuración que la fuente de verdad no contempla —sus dos ejemplos tienen
  siempre la sesión contenida en la cadencia— obligaría a reescribir la
  definición de ocupación que comparten la lectura y las dos rutas de
  escritura, cuando la configuración en sí no describe ninguna agenda que la
  clínica pueda atender.

## Entidades / puertos / adaptadores tocados

- `src/professionals/slot-schedule.ts` (nuevo): lectura e interpretación
  compartida del par duración/cadencia, con la selección de columnas de
  Prisma que ambos servicios reutilizan.
- `src/availability/availability.service.ts` (modificado): los dos recorridos
  de generación se unifican en uno solo, parametrizado por el par; el
  emparejamiento de la regla de doble franja pasa a usar la cadencia.
- `src/appointments/appointments.service.ts` (modificado): la reserva lee la
  configuración a través del módulo compartido y coloca la segunda mitad del
  turno doble una cadencia después.
- `src/professionals/professionals.service.ts` (modificado): invariante de
  configuración cadencia ≥ duración, dentro de la transacción serializable ya
  existente.
- `src/professionals/dto/update-professional-config.dto.ts`,
  `prisma/schema.prisma`, `postman/psique-backend.postman_collection.json`
  (modificados): documentación de la regla en el contrato, en el glosario del
  modelo y en la colección de pruebas manuales.

No se modificó el esquema de la base de datos: la corrección no agrega ni
altera columnas, sólo el comentario de glosario de la columna existente. No
se tocaron puertos ni adaptadores.

## Tests y qué validan

- `src/availability/availability.service.spec.ts` (modificado): con horario
  de 09:00 a 17:00, duración cuarenta y cinco y cadencia sesenta, los
  horarios ofrecidos caen en horas exactas de 09:00 a 16:00 y cada uno
  informa cuarenta y cinco minutos de duración —el criterio de aceptación
  explícito del ticket—; y, como regresión, un profesional sin cadencia
  configurada conserva el paso por duración. Dos casos más cubren el
  emparejamiento de la regla de paciente nuevo con cadencia, en la modalidad
  dentro de franja y en la de primer turno del día.
- `src/appointments/appointments.service.spec.ts` (modificado): la segunda
  mitad de un turno doble se coloca una cadencia después de la primera, y
  cada mitad conserva la duración de la sesión.
- `src/professionals/professionals.service.spec.ts` (modificado): rechazo de
  una cadencia menor que la duración enviada junto a ella y menor que la ya
  almacenada, aceptación de cadencia igual a la duración y del borrado
  explícito de la cadencia, y comprobación de que la invariante no consulta
  la fila cuando el pedido no toca ninguna de las dos columnas.
- `test/availability.e2e-spec.ts` (modificado): un profesional configurado
  con el ejemplo textual de la fuente de verdad, de punta a punta por HTTP
  contra la instancia local de PostgreSQL.
- `test/professional-config.e2e-spec.ts` (modificado): la invariante del par
  ejercitada por HTTP, incluido el caso de enviar las dos mitades en pedidos
  separados.
- Ejecución: suite unitaria en verde (39 suites / 447 pruebas) y suite
  end-to-end en verde (38 suites / 451 pruebas). Los datos usados en las
  pruebas son ficticios.

## Figuras pendientes

Ninguna nueva. La corrección no introduce un flujo que la tesis no describa
ya: ajusta el paso del algoritmo de disponibilidad documentado en 3.2.3 y la
regla de doble franja documentada en la misma subsección.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-114-slot-cadence-drives-slot-generation` (creada desde
  `origin/main`; la rama `main` local estaba dos commits atrás y no incluía
  todavía la fusión de TASK-113). Sin commit ni push al momento de redactar
  esta entrada, a pedido de la usuaria.
- Ticket: TASK-114 (Jira), "[CORRECCIÓN] TASK-35 – La cadencia de turnos
  (slotCadence) no se usa en la generación de slots". Misma convención de
  bitácora dedicada para tareas puntuales dentro de la fase del ticket
  original que TASK-79/TASK-81/TASK-86/TASK-94/TASK-95/TASK-96/TASK-100/
  TASK-108/TASK-110/TASK-113 ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]],
  [[FASE-3_PROMPT-15]], [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]],
  [[FASE-3_PROMPT-18]], [[FASE-3_PROMPT-19]], [[FASE-3_PROMPT-23]],
  [[FASE-3_PROMPT-24]], [[FASE-3_PROMPT-25]]).
- Fuente de verdad consultada: SRS "Secretaria Virtual — PSIQUE
  NEUROCIENCIAS" (Google Drive, *PROPUESTA PSIQUE - documento reunion
  ACTUALIZADO*), Anexo → Módulo profesionales → "Duración de la consulta
  configurable"; y Anexo → Módulo turnos → "Aviso de duración de la sesión" y
  "Pacientes nuevos / primera sesión".
