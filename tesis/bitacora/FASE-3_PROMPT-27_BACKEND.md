# Fase 3 — Motor de Turnos (backend) — el motor de reasignación de lista de espera no verificaba feriados (TASK-116, corrección a TASK-40)

## Contexto

El documento de requisitos establece que la clínica administra un calendario
de feriados y que la agenda no ofrece turnos en esas fechas. En el sistema
esa regla se concentra en el servicio de disponibilidad: tanto el cálculo de
la agenda ofrecible como la verificación puntual "¿este instante sigue
libre?" excluyen los días marcados como feriado. Las tres rutas de escritura
de turnos existentes —la reserva ordinaria, la reprogramación y la
reorganización de agenda por ausencia— pasan todas por esa verificación.

El motor de reasignación de lista de espera (TASK-40, [[FASE-3_PROMPT-7]])
es la excepción. Por una razón estructural deliberada, ese servicio escribe
directamente sobre la tabla de turnos a través del cliente con alcance de
inquilino, en lugar de delegar en el servicio de turnos: el módulo de turnos
depende de él para obtener el puerto de reasignación, de modo que la
dependencia inversa cerraría un ciclo de importación. Esa decisión, correcta
en cuanto a la arquitectura, dejó al recorrido de reasignación fuera de la
única verificación que aplica la regla de feriados: una búsqueda por la
palabra "holiday" en todo el módulo de lista de espera no arrojaba
ocurrencias.

El escenario de falla que esto habilita depende de un segundo hecho del
sistema: dar de alta un feriado no cancela los turnos ya reservados en esa
fecha. Un turno agendado con semanas de anticipación puede entonces quedar
sobre una fecha que la clínica declara feriado después. Si el paciente
original cancela ese turno, el motor de reasignación ofrece la franja
liberada a la lista de espera y, ante una aceptación, crea un turno nuevo
sobre un día en que la clínica no atiende. La detección proviene de una
auditoría de código sobre `psique-back/main` (2026-08-14, agente "Audit
waitlist/reassignment/holidays vs SRS").

## Qué se implementó

- La condición de feriado se verifica antes de ofrecer la franja liberada a
  ningún candidato. Si la fecha del turno cancelado es feriado, no se contacta
  a nadie, no se registra ninguna oferta y la retención sobre la franja se
  libera de inmediato.
- La condición se verifica una segunda vez, dentro de la misma transacción
  que crea el turno reasignado, para cubrir el caso en que el feriado se
  declare mientras una oferta ya emitida está esperando respuesta.
- En ese segundo caso la operación falla de forma explícita y libera la
  retención, en lugar de reservar en silencio.
- La verificación de feriado se extrajo del servicio de disponibilidad a un
  método propio y reutilizable, que la verificación de franja libre ahora
  consume internamente.

## Decisiones y por qué

**La verificación se ubicó en el paso de ofrecimiento, y no únicamente en el
de reserva.** El ticket admitía ambas ubicaciones. Se eligió el paso de
ofrecimiento como verificación principal porque la condición "esta fecha es
feriado" es una propiedad de la franja liberada, no del candidato: es
idéntica para todos los integrantes de la lista. Verificarla una vez por
recorrido, en lugar de una vez por candidato, evita tantas consultas como
candidatos tenga la lista y, sobre todo, evita ofrecer a un paciente un turno
que la clínica nunca podría honrar. La alternativa de verificar sólo al
reservar habría mantenido el recorrido completo: contactar al primer
candidato, esperar su ventana de respuesta de cuatro horas, fallar al
reservar, pasar al siguiente, y así con toda la lista, reteniendo entretanto
una franja inutilizable.

**La verificación en el momento de la reserva se conservó igualmente.** La
ventana de respuesta de una oferta dura cuatro horas, y el alta de un feriado
puede ocurrir dentro de ella. En ese caso la verificación del paso de
ofrecimiento ya se ejecutó y no pudo haber visto el feriado. La segunda
comprobación se ejecuta dentro de la misma transacción que crea el turno, de
modo que un feriado declarado entre la comprobación y la escritura no pueda
colarse: ambas se confirman o se revierten juntas.

**No se reutilizó la verificación de franja libre, pese a ser la que aplican
las otras rutas de escritura.** Esta fue la decisión menos evidente y merece
constancia, porque la opción aparentemente natural produce un defecto
silencioso. Esa verificación considera ocupada una franja no sólo cuando
existe un turno vivo sobre ella, sino también cuando existe un turno cancelado
bajo una retención de reasignación vigente. Ahora bien, el turno que el motor
está reasignando es exactamente eso: una fila cancelada que el propio motor
retuvo al iniciar el recorrido. Invocar allí la verificación de franja libre
hace que el turno colisione consigo mismo y que la comprobación devuelva
"ocupada" de manera determinista, no ocasional. El efecto habría sido
inutilizar por completo la reasignación automática —toda aceptación fallaría—
en lugar de corregir el defecto de feriados. Se optó por extraer la mitad que
sí corresponde, la consulta al calendario de feriados, y dejar intacta la
mitad de ocupación por turnos.

**La extracción se hizo sobre el servicio de disponibilidad y no copiando la
consulta en el módulo de lista de espera.** El ticket admitía explícitamente
"al menos un lookup directo" a la tabla de feriados. Se descartó esa opción
porque el repositorio sostiene como regla que el cálculo de la agenda y la
verificación puntual no puedan discrepar sobre qué excluye un feriado; una
tercera copia de la misma consulta, en otro módulo, es precisamente la forma
de divergencia que esa regla previene. El método extraído es ahora la única
definición y la consumen tanto la verificación de franja libre como el motor
de reasignación.

**El fallo en el momento de la reserva no reescribe la oferta como
rechazada.** La primera versión del trabajo marcaba la oferta como rechazada
y avanzaba al siguiente candidato. Se corrigió: el paciente efectivamente
aceptó, y la trazabilidad que exige la Ley 25.326 se apoya en que el registro
de auditoría diga lo que ocurrió. Consignar un rechazo que el paciente nunca
expresó introduce en el registro un hecho falso. La oferta permanece aceptada
y la falla se propaga como excepción.

**La liberación de la retención se acotó a la causa de feriado.** También en
la primera versión, el manejo de la falla capturaba cualquier error de la
transacción. Eso confundía el caso de feriado con dos casos de naturaleza
opuesta: un cambio de estado concurrente —alguien más tomó el turno, de modo
que la franja pertenece a ese otro y liberar la retención la expondría
además a la agenda del asistente— y un fallo de escritura del registro de
auditoría, que no debe quedar enmascarado. Se introdujo un tipo de error
propio para la condición de feriado, de manera que la liberación de la
retención se aplique exactamente a esa causa y todo otro fallo se propague
sin alterar el estado.

## Alternativas descartadas

- **Invocar la verificación de franja libre dentro de la reserva, tal como
  sugiere la formulación literal del ticket**: descartada por la colisión del
  turno con su propia retención, descrita más arriba. La opción no falla de
  forma visible en pruebas unitarias que sustituyan el servicio de
  disponibilidad por un doble, lo que la hace particularmente engañosa; se
  manifiesta sólo al ejercitar el recorrido contra la base de datos real.
- **Replicar la consulta al calendario de feriados dentro del módulo de lista
  de espera**: descartada por introducir una tercera definición de la misma
  regla, con el riesgo de divergencia que el repositorio evita por norma.
- **Verificar el feriado únicamente por candidato, dentro del paso de
  reserva**: descartada porque conserva el ofrecimiento de una franja
  inutilizable a toda la lista y multiplica las consultas sin obtener ninguna
  precisión adicional, dado que la condición no varía entre candidatos.
- **Cancelar los turnos ya reservados al dar de alta un feriado**, atacando la
  causa en el módulo de feriados en lugar del síntoma en el de reasignación:
  descartada por exceder el alcance del ticket y por afectar recorridos que la
  tarea no audita. Queda registrada como una carencia observada del alta de
  feriados, no resuelta por esta tarea.

## Entidades / puertos / adaptadores tocados

- `src/availability/availability.service.ts` (modificado): se extrajo la
  verificación de feriado a un método público reutilizable, que acepta el
  manejador de transacción del llamador; la verificación de franja libre pasa
  a consumirlo en lugar de ejecutar la consulta por sí misma.
- `src/waitlist/waitlist-reassignment.service.ts` (modificado): verificación
  de feriado en el paso de ofrecimiento y en el de reserva; extracción del
  cuerpo transaccional de la reserva a un método propio, y tipo de error
  específico para la condición de feriado.
- `src/waitlist/waitlist.module.ts` (modificado): importación del módulo de
  disponibilidad. Se verificó que no introduce ciclo de importación: el módulo
  de disponibilidad depende únicamente del de profesionales y ningún camino
  desde él regresa al de lista de espera, a diferencia del módulo de turnos,
  que sí depende de este último y debe permanecer unidireccional.

No se modificó el esquema de la base de datos ni se tocaron puertos ni
adaptadores. La corrección no agrega ni altera columnas.

## Tests y qué validan

- `src/waitlist/waitlist-reassignment.service.spec.ts` (modificado): que una
  franja liberada sobre un feriado no genera oferta alguna, no contacta a
  nadie y libera la retención; que la condición se consulta una vez por paso
  del recorrido y no una vez por candidato; que un feriado declarado durante
  la ventana de respuesta hace fallar la reserva de forma explícita, deja el
  turno original cancelado y no reasignado, libera la retención y conserva la
  oferta como aceptada; y que un fallo de reserva por causa distinta —un
  cambio de estado concurrente— conserva la retención y se propaga.
- `test/appointment-reassignment.e2e-spec.ts` (modificado): el criterio de
  aceptación del ticket ejercitado de punta a punta contra la instancia local
  de PostgreSQL, en las dos ubicaciones temporales del feriado. El primer caso
  reproduce el escenario descrito por el ticket: un turno agendado, un feriado
  declarado después sobre su fecha, y la cancelación del turno; se comprueba
  que no se registra ninguna oferta, que no se crea ningún turno, que el
  candidato conserva su lugar en la lista y que la franja no queda retenida.
  El segundo caso declara el feriado con una oferta ya emitida y comprueba el
  fallo explícito y la reversión completa de la transacción.
- Se agregó además limpieza de feriados entre casos de esa suite: a diferencia
  del resto del conjunto de datos de prueba, que se aísla dando a cada caso su
  propio profesional, los feriados dependen de la organización y persistían de
  un caso al siguiente, suprimiendo en silencio ofertas que otros casos
  verifican.
- Verificación del alcance de las pruebas: se comprobó que ambos casos
  end-to-end fallan si se desactivan las verificaciones introducidas, de modo
  que su resultado positivo no es vacío.
- Ejecución: suite unitaria en verde (39 suites / 451 pruebas) y suite
  end-to-end en verde (38 suites / 453 pruebas) en ejecución secuencial. En
  ejecución paralela la suite end-to-end presenta fallos por interferencia
  entre casos sobre la base de datos compartida; se verificó que esos fallos
  son previos a esta tarea, reproduciéndolos sobre el árbol sin los cambios.
  Los datos usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva. La corrección no introduce un flujo que la tesis no describa
ya: agrega una verificación al algoritmo de reasignación documentado en 3.2.3.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-116-waitlist-holiday-validation`, creada desde `origin/main`.
  Sin commit ni push al momento de redactar esta entrada, a pedido de la
  usuaria.
- Ticket: TASK-116 (Jira), "[CORRECCIÓN] TASK-40 – reserveForCandidate no
  valida feriados antes de reservar un turno reasignado". Misma convención de
  bitácora dedicada para tareas puntuales dentro de la fase del ticket
  original que TASK-79/TASK-81/TASK-86/TASK-94/TASK-95/TASK-96/TASK-100/
  TASK-108/TASK-110/TASK-113/TASK-114 ([[FASE-3_PROMPT-12]],
  [[FASE-3_PROMPT-14]], [[FASE-3_PROMPT-15]], [[FASE-3_PROMPT-16]],
  [[FASE-3_PROMPT-17]], [[FASE-3_PROMPT-18]], [[FASE-3_PROMPT-19]],
  [[FASE-3_PROMPT-23]], [[FASE-3_PROMPT-24]], [[FASE-3_PROMPT-25]],
  [[FASE-3_PROMPT-26]]).
- Observación registrada y no resuelta: el alta de un feriado no cancela ni
  reprograma los turnos ya reservados en esa fecha. Esta tarea impide que la
  reasignación agregue turnos nuevos sobre un feriado, pero los turnos
  previamente agendados sobre esa fecha permanecen.
