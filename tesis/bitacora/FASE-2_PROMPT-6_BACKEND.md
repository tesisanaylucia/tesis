# Fase 2 — Pacientes (backend) — Importador de pacientes preexistentes (TASK-32)

## Qué se implementó

Se implementó la carga masiva de pacientes preexistentes a partir de los archivos
en que la clínica conserva hoy sus registros, requisito que la especificación
enuncia como la carga de pacientes cuyos registros se encuentran en planillas de
cálculo, planillas y, en algunos casos, únicamente en papel. Se incorporó un
punto de acceso que recibe un archivo separado por comas o un libro de cálculo,
lo interpreta fila por fila, persiste las filas válidas y devuelve un informe con
las filas rechazadas, indicando en cada caso el número de fila, el campo y el
motivo.

El trabajo comprende tres piezas nuevas y una reorganización de piezas
existentes. Las nuevas son un lector de planillas independiente del dominio, la
correspondencia entre las columnas de la planilla y los campos del sistema, y el
servicio que aplica cada fila a la base de datos. La reorganización afecta a las
validaciones de los campos del paciente, que estaban repetidas en cada uno de los
cuerpos de solicitud del módulo, y al servicio de la relación paciente–profesional,
del que se separó el trabajo transaccional de las comprobaciones previas para
poder reutilizarlo. No hubo cambios de esquema ni migraciones: el modelo de datos
ya había previsto esta tarea al declarar opcionales la fecha de nacimiento, el
contacto de emergencia, el número de identificación tributaria y el teléfono
celular, precisamente porque los registros históricos carecen de ellos con
frecuencia.

## Decisiones y por qué

**La importación es parcial por diseño, y por eso responde con un informe y no
con un código de error.** Los registros que se migran son décadas de asientos
escritos a mano, y un archivo sin ninguna fila defectuosa no existe; una carga
de todo o nada significaría que la clínica no puede migrar hasta que la planilla
sea perfecta, que es exactamente lo contrario de lo que el requisito persigue.
Se intentan entonces todas las filas, las válidas se persisten y las restantes
regresan identificadas. La respuesta lleva además los totales de filas creadas,
actualizadas y rechazadas, cuya suma coincide con el total de filas del archivo,
de modo que el informe puede verificarse de un vistazo.

**La identidad de la fila es el número de documento dentro de la organización, lo
que hace la importación idempotente.** La especificación identifica al paciente
por su documento, y la base ya declara esa unicidad por organización; apoyarse en
ella significa que ejecutar dos veces el mismo archivo actualiza los mismos
registros en lugar de duplicarlos. Esa propiedad no es un detalle de eficiencia
sino la condición que vuelve seguro el ciclo natural de trabajo: cargar, leer el
informe, corregir las filas señaladas en la planilla y volver a cargarla entera.

**Sólo se escriben las celdas que el archivo trae.** Una celda vacía se
interpreta como ausencia de información y nunca como orden de borrar: la planilla
es una fotografía incompleta del pasado, no la verdad sobre el presente, y una
segunda carga no debe borrar un teléfono que el profesional cargó desde la
aplicación entre una y otra. Por la misma razón la marca de baja lógica queda
enteramente fuera del alcance del importador: dar de baja a un paciente es una
decisión de la clínica, y una planilla que enumera a todos los pacientes que
alguna vez atendió no constituye un pedido de revertirla.

**El rechazo del archivo completo se reservó a los dos casos que hablan del
archivo y no de sus filas.** Si faltan las columnas que identifican y nombran al
paciente, o si el archivo excede el tope de filas admitido, se responde con un
error único que las nombra: en el primer caso todas las filas arrojarían el mismo
error, y un informe con quinientas repeticiones del mismo mensaje comunica menos
que una sola oración. Todo lo demás —una fecha imposible, un documento vacío, una
obra social desconocida— es una equivocación de una fila y se informa como tal.

**La lectura del archivo se separó del dominio y quedó como componente común.**
El lector recibe el archivo y devuelve un encabezado y filas de texto numeradas
tal como las numera la aplicación de planillas; no sabe nada de pacientes. Qué
significa cada columna y qué valores son admisibles pertenece a quien pidió el
archivo. Esa separación es lo que permite que un error se informe como
"fila 7, fecha de nacimiento" mientras la capa de lectura sólo trata con celdas,
y deja disponible el mismo lector para cualquier otra importación futura.

**Toda celda es texto y sigue siéndolo hasta que un cuerpo de solicitud la
juzgue.** Una planilla no tiene tipos, y una biblioteca que convierte por su
cuenta transforma un documento escrito con cero inicial en un número que pierde
ese cero, y una fecha en un instante situado en la zona horaria del servidor. La
única excepción es la celda con formato de fecha, que se convierte a día
calendario a través del mismo módulo por el que el proyecto ya hace todos sus
cruces entre instante y día, leyendo exclusivamente los campos en tiempo
universal para que el día representado no se corra en la zona horaria de la
clínica.

**El separador de la planilla se detecta en lugar de suponerse.** Un archivo
exportado desde una hoja de cálculo con configuración regional en español está
separado por punto y coma, y rechazarlo sería rechazar justamente el archivo que
esta funcionalidad existe para leer. El separador se determina contando
candidatos fuera de las comillas en la primera fila, que es donde el recuento es
más confiable porque todas las columnas están presentes y ninguna vacía. Se
consideró y descartó adoptar una biblioteca general de lectura de valores
separados por comas: las cuarenta líneas que ocupa el análisis de comillas,
comillas escapadas, saltos de línea embebidos y separador son exactamente lo que
una biblioteca general cobra encareciendo el árbol de dependencias, y además la
mayoría convierte tipos por su cuenta, que es precisamente lo que aquí no debe
ocurrir. Para los libros de cálculo, en cambio, sí se incorporó una dependencia
—de sólo lectura y sin vulnerabilidades reportadas— porque el formato es un
archivo comprimido con documentos estructurados dentro y escribir un lector
propio no sería razonable.

**El formato se decide por la extensión del archivo y no por el tipo declarado.**
Un archivo separado por comas llega anunciado como texto, como hoja de cálculo o
como flujo binario genérico según el cliente y según si hay una suite ofimática
instalada, de manera que el tipo declarado no decide nada; la extensión, en
cambio, es la que la persona eligió en el cuadro de diálogo.

**Las columnas de la planilla se reconocen por un catálogo de nombres alternativos
y no por coincidencia parcial.** Los encabezados se normalizan —minúsculas, sin
acentos, sin puntuación— de modo que una misma columna se reconoce cualquiera sea
la forma en que la clínica la escribió, y el catálogo se escribe una sola vez.
La coincidencia es del nombre completo y no de una parte: "nombre" designa el
nombre de pila y "nombre completo" no, y una coincidencia parcial importaría en
silencio un dato dentro de una columna destinada a otro. Una columna que el
sistema no reconoce —un importe, una anotación administrativa— se ignora en lugar
de provocar un rechazo, porque toda planilla real trae algunas y negarse por ellas
sería negarse ante todo archivo real.

**La obra social y el profesional se reciben por nombre y se resuelven contra la
base.** Una planilla contiene el nombre del profesional, nunca un identificador
interno, y exigir que la clínica traduzca su propio archivo a claves foráneas
antes de importarlo anularía la funcionalidad. Cuando un nombre no coincide con
ninguno, o coincide con más de uno, se informa como error de esa fila en lugar de
adivinar: dos profesionales homónimos son una posibilidad real, y elegir entre
ellos adosaría la historia de un paciente al profesional equivocado. La búsqueda
incluye deliberadamente a los profesionales dados de baja, porque una planilla es
historia y un paciente atendido durante años por alguien que ya no trabaja en la
clínica no dejó de haber sido atendido por esa persona; rechazar ese vínculo
descartaría precisamente los registros más antiguos que la tarea busca rescatar.
Los nombres se normalizan con la misma función que los encabezados, de manera que
ambas normalizaciones no puedan divergir.

**Cada fila se valida con los mismos validadores que el alta interactiva.** Los
cuerpos de solicitud del módulo declaraban por separado qué constituye un
documento, un nombre, un correo o un contacto válidos, con el texto de los
mensajes repetido; se extrajeron a un módulo común de validadores compuestos que
hoy comparten el alta, la edición, la búsqueda y el importador. La consecuencia es
que un valor que el importador acepta es uno que el alta habría aceptado también,
equivalencia que impide que la migración llene la base de registros que la propia
interfaz habría rechazado. La única asimetría es deliberada y va en el sentido
opuesto: el importador exige menos campos obligatorios que el alta, porque la
fecha de nacimiento y el contacto de emergencia son requisitos de la reserva de un
turno y no del registro, y rechazar por ellos dejaría a los pacientes más antiguos
de la clínica fuera del sistema. El punto de acceso que informa qué datos le
faltan a un paciente, incorporado en una tarea anterior de esta misma fase, es lo
que después señala esa carencia cuando el paciente vuelve a atenderse.

**La validación se ejecuta de forma explícita en lugar de delegarse en el
mecanismo global.** El proyecto valida los cuerpos de solicitud mediante una
tubería global; aquí el cuerpo de la solicitud es un archivo y lo que se valida es
cada una de sus líneas, de modo que la validación se invoca sobre cada fila y su
resultado se convierte en una entrada del informe. Delegarla en el mecanismo
global habría significado responder con un error y descartar las restantes
cuatrocientas filas. Se informa una entrada por campo y no una por restricción
incumplida: un apellido vacío incumple simultáneamente tres validaciones, y tres
líneas diciendo lo mismo sobre la misma celda producen un informe que hay que
filtrar antes de poder leerlo.

**Las fechas escritas por una persona se admiten en el orden día-mes-año además
del orden normalizado, y sólo en las planillas.** Es la forma en que se escribe
una fecha en el país, y leerla al revés convertiría el 5 de junio en el 6 de mayo
sin que nada lo detectara. La conversión se limita a esa forma: cualquier otro
valor pasa intacto al validador, que lo rechaza e informa la fila. Un analizador
que reparase en silencio lo que no puede leer no informaría error alguno sobre una
fila que la persona necesita corregir. Esa tolerancia queda deliberadamente
confinada a las planillas: la interfaz de programación sigue admitiendo un único
formato, porque un cliente se escribe una vez contra un contrato mientras que una
planilla la escribe una persona.

**Cada fila se aplica dentro de su propia transacción.** El fallo de la fila
cuarenta no debe deshacer las treinta y nueve anteriores; a la vez, dentro de una
fila la escritura del paciente, el vínculo con su profesional y todas las entradas
de auditoría se confirman o se revierten como una unidad, según la regla del
proyecto que exige que una entrada de auditoría se confirme junto con la mutación
que describe.

**El tipo de paciente se deriva reutilizando el único método que ya lo decide.**
El requisito pide que los pacientes importados queden marcados como recurrentes si
tienen historial de consultas y como nuevos si no puede determinarse. La fecha de
última consulta que trae la planilla es ese historial, y el sistema ya contaba con
un método que traduce una asistencia en un tipo de paciente, una marca de primera
sesión y una fecha, escrito para el momento en que un turno se completa. Se
reutilizó en lugar de escribir las tres columnas desde el importador, de modo que
la regla siga teniendo un único lugar; una fila sin esa fecha simplemente conserva
los valores por omisión del vínculo, que son los de un paciente nuevo con su
primera sesión pendiente. Reutilizarlo obligó a una reorganización menor: tanto ese
método como el que crea el vínculo comprobaban la pertenencia del paciente sobre
el cliente exterior a la transacción, comprobación que fallaría sobre un paciente
recién creado dentro de la transacción del importador, todavía sin confirmar. Se
separó entonces en cada uno el trabajo transaccional de las comprobaciones
previas, quedando el método público como punto de entrada de las rutas —que
establece ambos extremos y luego ejecuta el trabajo— y el trabajo disponible para
un llamador que ya estableció esos extremos y aporta su propia transacción. La
fecha de última consulta queda así escrita únicamente por dos caminos: el turno
completado y esta migración.

**Se registró en la traza de auditoría el origen de cada registro.** Las entradas
nombran los campos escritos y nunca sus valores, como en todo el proyecto, y se
agregó la indicación de que la escritura provino de una importación: un registro
cargado desde una planilla y uno escrito por una persona responden de manera
distinta ante una consulta de rendición de cuentas, y la traza es donde esa
distinción debe sobrevivir.

**El punto de acceso quedó reservado al rol administrativo.** Cargar una planilla
es trabajo administrativo que se realiza una vez durante la migración, con una
persona leyendo el informe que devuelve. El proceso automatizado que atiende la
conversación registra pacientes de a uno por el punto de acceso ya existente y no
tiene por qué escribir masivamente sobre la tabla de pacientes; un profesional
queda excluido por la misma razón por la que no puede dar de alta un paciente, y
porque una importación escribe sobre toda la organización y no sólo sobre sus
propios pacientes. Los pacientes se cargan bajo la organización del usuario
autenticado, de manera que la misma planilla puede importarse de forma
independiente en organizaciones distintas.

**Los topes se declararon como constantes y no como configuración por
organización.** Siguiendo la convención del proyecto, el tamaño máximo del
archivo, la cantidad máxima de filas y la cantidad máxima de errores que el
informe enumera son topes destinados a acotar el costo de una solicitud y no
reglas de negocio: no expresan cuántos pacientes puede tener una clínica. El
recuento de filas rechazadas es siempre exacto; lo que el último tope acota son
los ejemplos que lo acompañan, y el informe declara explícitamente cuándo tuvo
que recortar la lista.

## Alternativas descartadas

- **Abortar la importación completa ante la primera fila inválida**: descartada
  por contradecir el requisito, que pide reportar los errores parciales sin
  detener la carga, y por volver impracticable la migración de registros
  históricos.
- **Una única transacción para todo el archivo**: descartada por la misma razón,
  ya que convertiría cualquier fila defectuosa en el descarte de todas las
  demás.
- **Crear siempre registros nuevos en lugar de actualizar por documento**:
  descartada porque duplicaría pacientes en la segunda ejecución y porque
  colisionaría con la unicidad del documento por organización que la base ya
  impone.
- **Interpretar una celda vacía como orden de borrar el valor almacenado**:
  descartada porque una segunda carga destruiría los datos que se hubieran
  completado desde la aplicación entre una carga y la siguiente.
- **Recibir la obra social y el profesional como identificadores internos**:
  descartada porque ninguna planilla los contiene y exigir su traducción previa
  anularía el propósito de la funcionalidad.
- **Elegir el primer profesional cuando el nombre coincide con varios**:
  descartada porque adosaría la historia de un paciente al profesional
  equivocado sin que nada lo señalara; se informa la ambigüedad como error de la
  fila.
- **Excluir a los profesionales dados de baja al resolver el nombre**:
  descartada porque descartaría los vínculos históricos, que son justamente los
  que la migración busca preservar.
- **Adoptar una biblioteca general para el análisis de valores separados por
  comas**: descartada por el costo en dependencias frente al tamaño del problema
  y, sobre todo, porque la conversión automática de tipos que esas bibliotecas
  aplican es contraria al requisito de conservar cada celda como texto.
- **Decidir el formato del archivo por el tipo declarado en la solicitud**:
  descartada porque ese valor varía según el cliente y el software instalado y no
  identifica el formato de manera confiable.
- **Reconocer las columnas por coincidencia parcial del encabezado**: descartada
  porque importaría datos dentro de columnas destinadas a otros.
- **Rechazar el archivo ante una columna desconocida**: descartada porque toda
  planilla real trae columnas que el sistema no modela.
- **Repetir los validadores de los campos del paciente en el nuevo cuerpo de
  solicitud**: descartada por ser la tercera copia de las mismas reglas, con el
  riesgo de que se endurezcan en un lugar y no en los otros; se extrajeron a
  validadores compartidos.
- **Exigir en la importación los mismos campos obligatorios que el alta**:
  descartada porque dejaría fuera del sistema a los pacientes más antiguos, que
  es lo contrario de lo que la tarea persigue; esos campos son requisito de la
  reserva de un turno y no del registro.
- **Escribir el tipo de paciente, la marca de primera sesión y la fecha de última
  consulta directamente desde el importador**: descartada por duplicar una regla
  que ya tenía un único lugar; se reutilizó el método existente.
- **Reparar en silencio una fecha que no puede leerse**: descartada porque
  ocultaría a la persona una fila que necesita corregir.
- **Extender la tolerancia de formatos de fecha a la interfaz de programación
  general**: descartada por debilitar un contrato que los clientes implementan
  una sola vez.
- **Enumerar en el informe todos los errores sin tope**: descartada porque un
  archivo íntegramente defectuoso produciría una respuesta tan extensa como él
  mismo; se acota la enumeración, se conserva exacto el recuento y se declara el
  recorte.
- **Habilitar el punto de acceso al proceso automatizado de la conversación**:
  descartada porque ese proceso registra pacientes de a uno y no tiene motivo
  para escribir masivamente sobre la organización.

## Entidades / puertos / adaptadores tocados

- `src/common/spreadsheet/spreadsheet.reader.ts` (nuevo): lectura de un archivo
  cargado —separado por comas o libro de cálculo— hacia un encabezado normalizado
  y filas de texto numeradas como las numera la aplicación de planillas; descarte
  de filas y celdas vacías y de columnas sin nombre o repetidas.
- `src/common/spreadsheet/csv.parser.ts` (nuevo): análisis de valores separados,
  con detección del separador, comillas, comillas escapadas, saltos de línea
  embebidos y marca de orden de bytes.
- `src/common/spreadsheet/spreadsheet-cell.ts` (nuevo): conversión de una celda a
  texto, normalización de etiquetas —compartida por encabezados y por los nombres
  que las filas referencian— y lectura de una fecha escrita en orden
  día-mes-año.
- `src/patients/patient-import.columns.ts` (nuevo): catálogo de nombres
  alternativos de columna, columnas obligatorias y traducción de una fila a los
  campos que representa.
- `src/patients/patient-import.row.ts` (nuevo): validación de una fila y
  conversión de sus incumplimientos en entradas del informe.
- `src/patients/patient-import.service.ts` (nuevo): aplicación de cada fila en su
  propia transacción, resolución de los nombres referenciados contra la base,
  entradas de auditoría y confección del informe.
- `src/patients/patient-import.controller.ts` (nuevo): punto de acceso de carga,
  restringido al rol administrativo, con tope de tamaño del archivo.
- `src/patients/patient-import.presenter.ts` (nuevo): forma del informe de
  respuesta.
- `src/patients/dto/import-patient-row.dto.ts` (nuevo): cuerpo de una fila
  importada.
- `src/patients/dto/patient-fields.decorators.ts` (nuevo): validadores compartidos
  de los campos filiatorios y de contacto del paciente.
- `src/patients/dto/create-patient.dto.ts`, `update-patient.dto.ts`,
  `find-patients-query.dto.ts`: reescritos sobre los validadores compartidos,
  sin cambio de comportamiento.
- `src/patients/patient-professionals.service.ts`: separación, en la creación del
  vínculo y en el registro de una consulta, del trabajo transaccional respecto de
  las comprobaciones previas, de modo que ambos puedan ejecutarse dentro de la
  transacción de un llamador que ya estableció los extremos.
- `src/patients/patients.constants.ts`: topes de tamaño de archivo, de cantidad de
  filas y de errores enumerados.
- `src/patients/patients.module.ts`: registro del controlador y del servicio
  nuevos.
- `scripts/generate-postman-collection.js`: corrección del reconocimiento del
  nombre del método cuando un decorador abarca varias líneas, y cuerpo de tipo
  formulario para la carga de archivos.
- `postman/psique-backend.postman_collection.json`: colección de pruebas de la
  API actualizada con el punto de acceso incorporado.
- `CLAUDE.md` del repositorio: registro de las convenciones nuevas sobre carga de
  archivos y lectura de planillas.

No hubo cambios de esquema ni migraciones. No se tocaron puertos ni adaptadores de
integración: la lectura de un archivo cargado no es una integración con un
servicio externo, sino un componente común sin estado.

## Tests y qué validan

- `src/common/spreadsheet/csv.parser.spec.ts` (nuevo, 11 pruebas): las formas que
  produce una hoja de cálculo real —marca de orden de bytes, fin de línea de dos
  caracteres, separador de punto y coma bajo configuración regional en español— y
  las reglas de comillas: separador contado fuera de las comillas, registro que
  abarca varias líneas, comilla escapada, comilla que no abre campo, celdas
  vacías conservadas en su posición, último registro sin salto final y archivo
  vacío.
- `src/common/spreadsheet/spreadsheet-cell.spec.ts` (nuevo, 12 pruebas): la
  conversión de una celda a texto, incluidos el caso de una columna con formato
  numérico y el de una celda con formato de fecha, que debe leerse como el día
  que representa; la normalización de etiquetas, con acentos y puntuación; y la
  lectura de fechas en orden día-mes-año, junto con la ausencia de reparación
  ante un valor ilegible.
- `src/common/spreadsheet/spreadsheet.reader.spec.ts` (nuevo, 9 pruebas): el
  encabezado normalizado y las filas indexadas por él, la numeración de filas tal
  como la muestra la aplicación de planillas, la omisión de celdas vacías, el
  descarte de filas enteramente vacías, el de columnas sin nombre y el de una
  segunda columna con el mismo nombre, y los rechazos por extensión no admitida,
  por ausencia de encabezado y por archivo que no es el libro que su extensión
  declara.
- `src/patients/patient-import.columns.spec.ts` (nuevo, 8 pruebas): la
  correspondencia entre los encabezados en español de una planilla y los campos
  del sistema, la omisión de columnas no reconocidas, la ausencia de coincidencia
  parcial, la preferencia por la columna más a la izquierda ante dos nombres
  equivalentes, la enumeración de columnas obligatorias faltantes y la
  traducción de una fila, con reescritura de las columnas de fecha.
- `src/patients/patient-import.row.spec.ts` (nuevo, 10 pruebas): la aceptación de
  una fila completa y de una fila sin fecha de nacimiento ni contacto de
  emergencia; los dos rechazos que el ticket nombra explícitamente —documento
  vacío y fecha inexistente—, junto con el de un documento escrito con puntos, el
  de una fecha ilegible y el de un correo inválido; y la forma del informe: una
  entrada por campo aunque se incumplan varias restricciones, y todos los campos
  defectuosos de una fila informados a la vez.
- `test/patients-import.e2e-spec.ts` (nuevo, 20 pruebas): por la capa HTTP, con
  credenciales reales de cada rol y con una carga de archivo real. Se cubre el
  rechazo sin autenticación y el rechazo por falta de permisos del profesional y
  del proceso automatizado; los rechazos que conciernen al archivo —solicitud sin
  archivo, extensión no admitida y ausencia de las columnas obligatorias—; la
  importación de una planilla de trece filas, diez válidas y tres defectuosas,
  verificando los recuentos, su suma, y el número de fila, campo y motivo de cada
  rechazo; la persistencia de una fila completa, incluida la fecha escrita en
  orden día-mes-año y la obra social resuelta por nombre; el paciente marcado
  como recurrente por traer fecha de última consulta y el marcado como nuevo por
  no traerla; la importación de un registro sin los datos que exige una reserva;
  la ausencia en la base de las filas rechazadas; el aislamiento por
  organización, comprobando que la organización ajena no ve ninguno de los
  registros; la entrada de auditoría, que indica el origen de importación; la
  segunda ejecución del mismo archivo, que actualiza sin duplicar ni pacientes ni
  vínculos; la conservación de los datos que el archivo no trae, cargados entre
  una ejecución y la siguiente; y la lectura de un libro de cálculo generado en
  la propia prueba, con una celda con formato de fecha y otra con la fecha
  escrita como texto en la misma columna.
- Ejecución: suite unitaria en verde (19 suites / 144 pruebas), suite end-to-end
  en verde (20 suites / 226 pruebas), compilación del proyecto sin errores y
  análisis estático sin advertencias. Todos los datos utilizados en las pruebas
  son ficticios: los documentos, nombres, teléfonos y correos son inventados, y
  ninguna fila contiene información clínica.

## Figuras pendientes

- Se registró la figura tentativa 16, con el flujo de la importación: archivo
  cargado, lector de planillas independiente del dominio, correspondencia de
  columnas a campos, validación por fila, transacción por fila con escritura del
  paciente, vínculo con el profesional y auditoría, e informe final; señalando
  los dos rechazos que conciernen al archivo completo y la bifurcación que deja
  continuar a las filas válidas. Es la pieza que el texto describe en prosa y que
  se comprende mejor de un vistazo.
- Se amplió el epígrafe de la figura tentativa 13, mapa de puntos de acceso del
  módulo con permisos por rol, para incluir la importación masiva y las
  observaciones, incorporadas en esta tarea y en la anterior.
- No se requieren figuras de entidades nuevas: la tarea no incorporó ninguna, y
  el diagrama del subdominio ya registrado en tareas anteriores cubre las que la
  importación escribe.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-32-patient-importer` (creada a partir de
  `main`). Sin commit al momento de redactar esta entrada: los cambios quedaron
  en el árbol de trabajo a la espera de autorización.
- Ticket: TASK-32 ("P2.6 – Importador de pacientes preexistentes"). Depende de
  TASK-27 (entidades del dominio Pacientes) y TASK-28 (ABMC de pacientes), ambas
  ya fusionadas. Quedan explícitamente fuera de alcance la carga de los datos
  reales de la clínica, que corresponde al piloto, y la importación de turnos
  históricos.
