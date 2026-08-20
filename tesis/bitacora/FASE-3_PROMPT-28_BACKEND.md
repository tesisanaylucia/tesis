# Fase 3 — Motor de Turnos (backend) — el tipo de paciente se cargaba en el ranking de reasignación pero nunca se comparaba (TASK-117, corrección a TASK-40)

## Contexto

El algoritmo de reasignación de una franja liberada (TASK-40,
[[FASE-3_PROMPT-7]]) recorre la lista de espera del profesional en un orden
que el propio ticket fijó con dos criterios de prioridad: la prioridad que el
profesional carga a mano sobre la relación con un paciente concreto —"un
paciente con prioridad mayor a cero sube respecto de pacientes con prioridad
cero del mismo orden"— y el tipo de vínculo —"los pacientes de tipo recurrente
tienen prioridad sobre los nuevos"—, con el orden de llegada a la lista como
desempate.

La implementación cargaba ambos datos. La estructura interna que representa a
un candidato del ranking declara los dos campos, y la consulta que los resuelve
trae los dos desde la relación paciente-profesional. Pero el comparador que
ordena a los candidatos leía únicamente la prioridad explícita, el orden de
llegada y la fecha de creación del registro: el campo de tipo se escribía y no
se volvía a leer en ninguna parte del repositorio. La consecuencia es que dos
candidatos sin prioridad explícita —el caso ordinario, ya que la prioridad es
opcional y se asigna a mano— quedaban ordenados sólo por orden de llegada, de
modo que un paciente recurrente de años no obtenía ninguna ventaja sobre uno
nuevo que se hubiera anotado antes. El criterio de aceptación del ticket
original quedaba, así, incumplido en silencio: nada fallaba, simplemente el
orden no era el prescripto. La detección proviene de la misma auditoría de
código sobre `psique-back/main` (2026-08-14) que originó
[[FASE-3_PROMPT-27]].

## Qué se implementó

- El tipo de vínculo pasa a ser un escalón del comparador del ranking, ubicado
  entre la prioridad explícita y el orden de llegada: recurrente antes que
  nuevo.
- El comparador completo —cuatro escalones— se extrajo a una función pura de
  nivel de módulo, junto a la que ya traducía la prioridad a un peso numérico,
  en lugar de permanecer como una función anónima dentro de la llamada de
  ordenamiento.

No se modificó el esquema de la base de datos: ambos campos ya existían en la
relación paciente-profesional desde TASK-34, y la corrección consiste
íntegramente en leer uno que ya se cargaba.

## Decisiones y por qué

**La prioridad de los pacientes recurrentes es incondicional; no se agregó una
opción de configuración.** El ticket señalaba una ambigüedad de la formulación
original de TASK-40, que enunciaba la regla como "los pacientes de tipo
recurrente tienen prioridad sobre los nuevos *si el profesional así lo
configuró*", y pedía resolverla con la dueña del producto antes de implementar.
El sistema no tenía —ni tiene— ningún parámetro por profesional que active o
desactive esa preferencia, de modo que la frase condicional no correspondía a
nada existente. Consultada la dueña del producto durante esta tarea, se
confirmó que la regla es incondicional. Se descartó por lo tanto agregar la
columna de configuración: el repositorio sostiene como norma que las reglas de
negocio se expresen como datos configurables por inquilino o por profesional en
lugar de condicionales fijos en el código, pero esa norma se aplica a lo que la
clínica efectivamente decide, y una opción que nadie va a cambiar es
configuración muerta —una columna, un campo del punto de acceso de
configuración, un campo de la respuesta y sus pruebas— sosteniendo una decisión
que ya está tomada.

**El tipo se ubicó como escalón intermedio y no como criterio principal.** La
prioridad que el profesional carga a mano sobre un paciente determinado es una
decisión específica sobre ese paciente; el tipo, en cambio, es un hecho general
sobre el vínculo, que el sistema deriva por sí mismo de las consultas
registradas. Ordenar primero por tipo haría que un paciente nuevo al que el
profesional marcó explícitamente como urgente quedara detrás de cualquier
recurrente sin prioridad asignada, invirtiendo el peso relativo de una decisión
deliberada frente a una condición automática. La ubicación intermedia es
además la que indica el ticket.

**El tipo comparado es el vigente, no el almacenado.** El escalón nuevo no
consulta la columna directamente, sino el vínculo tal como lo devuelve la carga
por lotes incorporada en TASK-108 ([[FASE-3_PROMPT-23]]), que aplica antes la
regla de inactividad de P2.3. Un vínculo que quedó registrado como recurrente
pero cuya última consulta es anterior al umbral del inquilino ya fue degradado
a nuevo cuando llega al comparador. Sin ese orden, la corrección habría
otorgado ventaja en la lista de espera a pacientes que el resto del sistema ya
considera nuevos. No hizo falta cambio alguno para lograrlo —la carga por lotes
ya se comportaba así—, pero sí dejar constancia de la dependencia, porque el
escalón nuevo es el primer consumidor del campo y la propiedad no es evidente
en el punto donde se compara.

**El comparador se extrajo a una función pura con nombre.** Con el escalón
agregado, la escritura anterior —una sucesión de comparaciones intermedias con
retornos tempranos dentro de una función anónima— pasaba a cinco condiciones
anidadas para expresar cuatro criterios. La regla de orden de P3.7 es la parte
del algoritmo que la tesis y el documento de requisitos describen, no depende
de ningún estado del servicio y se lee mejor entera; extraerla junto a la
función de peso de prioridad, que ya seguía ese patrón, mantiene la coherencia
del archivo y permite documentar los cuatro escalones en un único lugar.

## Alternativas descartadas

- **Agregar una columna de configuración por profesional** que active la
  preferencia por pacientes recurrentes, siguiendo la letra de la formulación
  original de TASK-40: descartada tras la confirmación de la dueña del
  producto, por las razones expuestas más arriba. El ticket contemplaba
  explícitamente ambos resultados de la consulta.
- **Ubicar el tipo como criterio principal del ranking**, por encima de la
  prioridad explícita: descartada por subordinar una decisión deliberada del
  profesional a una condición que el sistema deriva solo.
- **Ordenar los candidatos en la propia consulta a la base de datos**, en lugar
  de en memoria: descartada porque dos de los cuatro criterios —la prioridad y
  el tipo— no viven en la tabla de la lista de espera sino en la relación
  paciente-profesional, y el tipo, además, no es el valor almacenado sino el
  resultado de aplicarle la regla de inactividad. Ordenar en la base exigiría
  replicar esa regla en SQL, que es exactamente la duplicación que el
  repositorio evita por norma.

## Entidades / puertos / adaptadores tocados

- `src/waitlist/waitlist-reassignment.service.ts` (modificado): función de peso
  del tipo de paciente y comparador de candidatos extraídos a nivel de módulo,
  con el escalón de tipo incorporado; la construcción del ranking pasa a
  delegar el orden en el comparador.

No se tocaron el esquema de la base de datos, los puertos, los adaptadores ni
ningún punto de acceso HTTP: la corrección no altera el contrato de la API.

## Tests y qué validan

- `src/waitlist/waitlist-reassignment.service.spec.ts` (modificado): que entre
  dos candidatos sin prioridad explícita, el recurrente recibe la oferta antes
  que el nuevo aun cuando este último se anotó primero en la lista —el
  criterio de aceptación del ticket—; que la prioridad explícita sigue por
  encima del tipo, comprobado con un candidato nuevo de prioridad máxima frente
  a un recurrente sin prioridad —la prueba de regresión que pide el ticket—; y
  que entre dos candidatos de igual prioridad y tipo el desempate sigue siendo
  el orden de llegada, para que el escalón agregado no desplace al que ya
  existía.
- `test/appointment-reassignment.e2e-spec.ts` (modificado): el mismo criterio
  de aceptación ejercitado de punta a punta contra la instancia local de
  PostgreSQL, verificando además que el candidato nuevo sigue siendo alcanzado
  en el paso siguiente del recorrido —el tipo ordena la lista, no excluye a
  nadie de ella—. Se generalizó en esa suite el auxiliar que creaba la relación
  paciente-profesional: recibía la prioridad como parámetro obligatorio y fijaba
  el tipo en recurrente por omisión, de modo que todo caso que necesitara fijar
  un tipo debía además fijar una prioridad que no le interesaba. Ahora ambos
  campos son opcionales y se omiten con los mismos valores que la propia
  columna toma por defecto, para que cada caso declare únicamente el criterio
  que está probando.
- Verificación del alcance de las pruebas: se comprobó que tanto la prueba
  unitaria como la end-to-end del criterio de aceptación fallan si se quita el
  escalón de tipo del comparador, de modo que su resultado positivo no es
  vacío.
- Ejecución: suite unitaria en verde (39 suites / 455 pruebas) y suite
  end-to-end en verde (38 suites / 454 pruebas) en ejecución secuencial. Los
  datos usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva. La corrección completa el algoritmo de reasignación ya descripto
en 3.2.3 sin introducir un flujo que la tesis no documente.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-117-recurring-patient-priority`, creada
  desde `main`. Sin commit ni push al momento de redactar esta entrada, a
  pedido de la usuaria.
- Ticket: TASK-117 (Jira), "[CORRECCIÓN] TASK-40 – Prioridad de pacientes
  recurrentes cargada pero nunca usada en el ranking de reasignación". Misma
  convención de bitácora dedicada para tareas puntuales dentro de la fase del
  ticket original que TASK-94/TASK-95/TASK-96/TASK-100/TASK-108/TASK-110/
  TASK-113/TASK-114/TASK-116 ([[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]],
  [[FASE-3_PROMPT-18]], [[FASE-3_PROMPT-19]], [[FASE-3_PROMPT-23]],
  [[FASE-3_PROMPT-24]], [[FASE-3_PROMPT-25]], [[FASE-3_PROMPT-26]],
  [[FASE-3_PROMPT-27]]).
- Observación registrada y no resuelta: el método `getPatientPriority` del
  servicio de turnos, agregado por TASK-40 para exponer la prioridad y el tipo
  de un paciente al algoritmo de reasignación, no tiene hoy más consumidor que
  su propia prueba unitaria —el motor de reasignación resuelve los vínculos por
  la carga por lotes desde TASK-108—. Queda señalado como código muerto
  candidato a eliminación, fuera del alcance de esta tarea por tratarse de un
  método público de otro módulo.
