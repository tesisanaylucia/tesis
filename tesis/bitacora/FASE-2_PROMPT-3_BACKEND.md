# Fase 2 — Pacientes (backend) — Tipo de paciente, última consulta y prioridad (TASK-29)

## Qué se implementó

Se incorporó la lógica que faltaba sobre el vínculo entre un paciente y cada
profesional que lo atiende: la clasificación del paciente como nuevo o
recurrente, el registro de la fecha de su última consulta y la consulta de la
prioridad que el profesional mantiene. Las columnas correspondientes ya habían
sido modeladas en la primera tarea de la fase; esta tarea aporta las reglas que
las gobiernan, no el esquema.

Los requisitos establecen tres cosas sobre este vínculo. Un paciente es nuevo en
su primera sesión con un profesional determinado y pasa a recurrente una vez que
esa sesión se completa. Si deja de concurrir por más de un año, vuelve a
marcarse como nuevo. Y la prioridad la define y carga el profesional desde la
aplicación, para que el motor de turnos la consuma más adelante al reasignar.

Se agregó un punto de consulta que devuelve el tipo, la prioridad y la fecha de
última consulta de un vínculo determinado, y un servicio interno que registra
una consulta y que el módulo de turnos invocará cuando un turno pase al estado
completado. La modificación de la prioridad ya existía desde la tarea anterior y
no requirió cambios en su contrato.

## Decisiones y por qué

**La regla del año se evalúa al leer y se persiste.** La tarea descarta
explícitamente resolverla mediante un proceso programado: la evaluación ocurre
cuando se lee el paciente. Sobre esa base quedaba una decisión abierta, la de si
la reclasificación debía además escribirse en la base o calcularse en cada
lectura sin tocar la fila. Se optó por escribirla. Calcularla sin persistirla
dejaría la columna almacenada en desacuerdo permanente con lo que todos los
puntos de consulta informan, de modo que cualquier lector que no pasara por
esos puntos —una consulta directa a la tabla, un informe, el motor de turnos
leyendo el vínculo por su cuenta— vería un valor que el sistema ya considera
caduco. La escritura se realiza con una única sentencia que nombra exactamente
los vínculos leídos, y no con una condición general sobre la tabla, para que la
lectura de un paciente no modifique en silencio filas que quien consulta nunca
pidió.

La regla se ubicó en una función pura, aislada de la base de datos, y el
servicio que la rodea se limita a resolver las fechas que necesita y a persistir
lo que decide. La separación no es ceremonial: lo que la tarea pide validar es
un límite entre dos días del calendario, y una función pura permite comprobarlo
sin base de datos, mientras que las consecuencias que ninguna función pura puede
mostrar —que la regla efectivamente se ejecuta al leer, que la reclasificación
queda escrita y que todos los puntos de consulta coinciden— se validan por
separado con pruebas de integración.

**El límite del año se fijó como inclusivo, y se unificó con la regla de
actualización de datos.** Aquí apareció una discrepancia entre las fuentes. El
documento de requisitos habla de "más de un año", lo que en sentido estricto
excluye el aniversario exacto; el criterio de aceptación de la tarea, en cambio,
pide que a los trescientos sesenta y cinco días el paciente ya figure como
nuevo. La tarea anterior de la fase había implementado el mismo umbral para la
regla que solicita la actualización de los datos de contacto, y lo había hecho
con el límite excluyente.

Se consultó la decisión antes de implementarla y se resolvió adoptar el límite
inclusivo para ambas reglas, unificándolas en una única función compartida. El
criterio que ordenó la elección fue la consistencia: los requisitos enuncian las
dos reglas con la misma frase, de modo que dos comparaciones que difirieran en
un día producirían un paciente al que el sistema considera nuevo pero a quien no
le solicita datos actualizados, precisamente la clase de divergencia silenciosa
que el proyecto viene evitando en el resto del modelo. El costo asumido es que
el comportamiento de la tarea anterior se corre un día; se lo consideró
preferible a sostener dos versiones de una misma regla. La adopción del límite
inclusivo dejó además el criterio de aceptación satisfecho de forma literal.

**La fecha de última consulta sólo avanza.** Se resolvió que registrar una
consulta no pueda retroceder la fecha almacenada. La razón es que el dato existe
para responder cuánto hace que el paciente no concurre, y una sesión antigua
cargada tardíamente no vuelve más lejana la última asistencia. La condición se
expresó dentro de la propia sentencia de actualización y no como una comparación
previa en el servicio: leer el valor y decidir después dejaría que dos turnos
completados de forma concurrente leyeran ambos la fecha anterior y que el que
escribiera último la retrasara, exactamente el patrón de lectura seguida de
escritura que las convenciones del repositorio señalan como inseguro bajo el
nivel de aislamiento predeterminado.

**La promoción a recurrente y el indicador de primera sesión se escriben
juntos.** Completar la primera sesión determina dos hechos a la vez: que el
paciente ya no es nuevo para ese profesional y que su primera sesión deja de
estar pendiente. La tarea sólo menciona el primero, y un comentario previo en el
código asignaba la limpieza del indicador a una tarea posterior de la fase de
turnos. Se consultó también esta decisión y se resolvió escribir ambos campos en
el mismo momento, porque dejar el indicador de primera sesión activo junto a un
paciente ya recurrente es un estado internamente contradictorio, y es además el
campo del que dependerá el cálculo de la duración del turno: la primera sesión
ocupa dos turnos consecutivos, de modo que un indicador caduco no sería un
detalle cosmético sino una agenda mal calculada.

Ambos campos se escriben sin condicionarlos a la fecha, porque registrar una
sesión antigua sigue siendo constancia de asistencia. Si esa sesión es más
antigua que el umbral, la regla del año la reclasifica como nuevo en la lectura
siguiente. Las dos reglas se componen en lugar de contradecirse, y así se dejó
comprobado en las pruebas.

**El registro de la consulta es un servicio interno, no un punto de acceso de la
API.** La fecha de última consulta no se carga a mano: la única evidencia de
asistencia que el sistema posee es un turno que pasó a completado, y admitir que
alguien la escribiera directamente sería aceptar una afirmación sobre asistencia
que ningún turno respalda. Por eso el registro se expuso como servicio que el
módulo de turnos invocará, y no como una operación del contrato HTTP. El
servicio admite además que quien lo invoca le entregue la transacción en curso,
de modo que un turno que se confirma como completado y una consulta que nunca
queda registrada no puedan ocurrir por separado.

Se optó por una dependencia directa entre módulos y no por un puerto con su
adaptador. La distinción sigue el criterio ya fijado en el repositorio: los
puertos existen para las integraciones externas y para los casos en que el
módulo productor no debe conocer a su consumidor. Aquí la dirección es la
inversa —el módulo de turnos depende del de pacientes, que ya existe—, de modo
que no hay nada que invertir y un puerto sólo agregaría indirección.

**El punto de consulta del vínculo se restringió al profesional de la
relación.** Se le aplicó la misma autorización que a la modificación de la
prioridad: el personal administrativo accede a cualquier vínculo y un
profesional únicamente al propio. No se incluyó al proceso automatizado, y la
omisión no le quita capacidades: el asistente conversacional lee el paciente
completo, con sus vínculos incluidos, por el camino que ya tenía.

## Alternativas descartadas

- **Resolver la reclasificación con un proceso programado periódico**:
  descartada por indicación explícita de la tarea, que la sitúa en la lectura.
  Se deja constancia de que la alternativa fue considerada y desestimada por las
  fuentes, no por omisión: un proceso programado exigiría además decidir su
  frecuencia y dejaría una ventana en la que el dato almacenado sería incorrecto.
- **Calcular el tipo en cada lectura sin escribirlo**: descartada por las
  razones expuestas más arriba, esencialmente porque dejaría la columna
  almacenada en desacuerdo permanente con lo informado.
- **Sostener dos comparaciones distintas sobre el mismo umbral**, una para el
  tipo y otra para la solicitud de actualización de datos: descartada por
  producir estados incoherentes ante un mismo paciente.
- **Permitir cargar la fecha de última consulta desde la API**: descartada
  porque convertiría un hecho derivado de la asistencia en un dato de captura
  manual, sin respaldo en ningún turno.
- **Exponer el registro de consultas mediante un puerto de dominio**:
  descartada porque la dirección de la dependencia no lo justifica.

## Entidades / puertos / adaptadores tocados

- `src/patients/patient-inactivity.rule.ts` (nuevo): la regla del año como
  función pura, compartida por la clasificación del paciente y por la solicitud
  de actualización de datos de contacto.
- `src/patients/patient-inactivity.service.ts` (nuevo): resolución del umbral
  configurado por la organización y aplicación persistente de la regla sobre los
  vínculos recién leídos.
- `src/patients/patients.service.ts`: la resolución del umbral, que hasta ahora
  vivía como método privado de este servicio, se trasladó al servicio nuevo y
  dejó de estar duplicada; los caminos de lectura pasan a evaluar la regla, y la
  regla de actualización de datos pasa a usar la comparación compartida.
- `src/patients/patient-professionals.service.ts`: consulta de un vínculo,
  registro de una consulta con su traza de auditoría, y unificación en una única
  función de la verificación de existencia del vínculo, que antes se realizaba
  con una consulta adicional cuyo resultado se descartaba.
- `src/patients/patient-professionals.controller.ts`: punto de consulta del
  vínculo, con la misma autorización que su modificación.
- `src/patients/patients.module.ts`: registro del servicio nuevo y exposición
  del servicio de vínculos para el módulo de turnos.
- `postman/psique-backend.postman_collection.json`: colección de pruebas de la
  API actualizada con el punto de consulta incorporado.

No se modificó el esquema de la base de datos: las columnas que esta tarea
gobierna ya habían sido incorporadas en la primera tarea de la fase. Tampoco se
tocaron puertos ni adaptadores de integración.

## Tests y qué validan

- `src/patients/patient-inactivity.rule.spec.ts` (nuevo, 11 pruebas): el límite
  del año sobre una fecha fija, con los dos casos que el criterio de aceptación
  nombra —un día antes del año cumplido el paciente sigue siendo recurrente, y
  al cumplirse vuelve a ser nuevo—, la ausencia de reclasificación para quien
  nunca concurrió, el carácter idempotente de la regla sobre un paciente ya
  marcado como nuevo, y su obediencia a un umbral organizacional más corto que
  un año.
- `test/patients-type-priority.e2e-spec.ts` (nuevo, 18 pruebas): recorre por la
  capa HTTP lo que la función pura no puede mostrar. Valida el punto de consulta
  del vínculo y su respuesta ante un vínculo inexistente; su rechazo para un
  profesional ajeno a la relación y su admisión para el personal administrativo;
  que la reclasificación quede efectivamente escrita en la fila y no sólo
  informada; que el detalle del paciente informe el mismo tipo que el punto de
  consulta del vínculo; el registro de una consulta con la promoción a
  recurrente y la baja del indicador de primera sesión; su traza de auditoría,
  que enumera nombres de campos y ningún valor; que una consulta antigua cargada
  con posterioridad no retrase la fecha almacenada; el rechazo del registro para
  un par sin vínculo de tratamiento; la composición de ambas reglas cuando la
  única sesión registrada es anterior al umbral; y la asignación de prioridad por
  el profesional tratante, su rechazo para otro profesional, su borrado y su
  rango admitido.
- Las pruebas anclan sus fechas al umbral tal como lo deriva el servicio, y no a
  una cantidad fija de días, de modo que sigan siendo correctas con
  independencia de la fecha en que se ejecuten, incluido el cruce de un año
  bisiesto.
- Ejecución: suite unitaria en verde (13 suites / 75 pruebas), suite end-to-end
  en verde (17 suites / 179 pruebas), compilación del proyecto sin errores y
  análisis estático sin advertencias. Todos los datos usados son ficticios:
  nombres de fantasía y números de documento de ejemplo, sin contenido clínico.

## Figuras pendientes

- Se registró una figura pendiente con el diagrama de estados del tipo de
  paciente y las transiciones que lo gobiernan (ver `figuras_pendientes.md`).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-29-patient-type-and-priority` (creada a
  partir de `main`). Sin commit al momento de redactar esta entrada: los cambios
  quedaron en el árbol de trabajo a la espera de autorización.
- Ticket: TASK-29 ("P2.3 – Tipos de paciente, última consulta y prioridad").
  Depende de TASK-27 (entidades del dominio Pacientes) y TASK-28 (ABMC de
  pacientes), ambas ya fusionadas. El consumo de la prioridad para reasignar
  turnos corresponde a TASK-40, y la lista de espera a TASK-34; ninguna forma
  parte del alcance de esta tarea.
