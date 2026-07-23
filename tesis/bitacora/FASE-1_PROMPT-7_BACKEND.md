# Fase 1 — Profesionales (backend) — Revisión de código y endurecimiento (fecha de confirmación como día calendario, concurrencia de ausencias, cota de la franja extra)

## Contexto

Antes de iniciar los módulos siguientes se realizó una revisión sistemática del
código ya construido, contrastándolo contra las dos fuentes de verdad del
proyecto —el anteproyecto de tesis y el documento de especificación de requisitos
de software— y contra los tres pilares que el proyecto se impone para el modelo de
datos: integridad, normalización y relaciones lógicas. La revisión se condujo desde
varios ángulos independientes —el modelo de datos, la conformidad funcional con la
especificación y la calidad de código, la seguridad y la concurrencia— para no
depender de una única lectura.

El resultado del pilar de datos fue que el esquema cumple los tres pilares sin
requerir cambios estructurales: toda clave foránea es una clave foránea real, el
identificador de organización no está replicado en ninguna entidad alcanzable a
través de un padre que ya lo lleva, y los cruces entre entidades del mismo inquilino
usan claves foráneas compuestas que la base de datos verifica. Las correcciones que
la revisión sí produjo, y que documenta esta entrada, son de comportamiento y de
contrato antes que de esquema —con una única excepción: un cambio de tipo de una
columna, motivado por la convención de fechas del propio proyecto—.

## Qué se implementó

Se aplicaron tres correcciones en el módulo de Profesionales y una cuarta,
transversal, en el almacén de configuración por inquilino que la misma revisión puso
en evidencia.

La primera lleva la fecha de confirmación de la incorporación del profesional
(`confirmationDate`) del tipo instante con marca temporal al tipo fecha de calendario.
La columna se declaró como fecha sin hora, la validación de entrada pasó a exigir el
formato "AAAA-MM-DD" de un día que exista realmente, y la conversión entre la cadena
y la columna se canalizó por el único módulo del sistema autorizado a cruzar esa
frontera, el mismo que ya gobierna las ausencias y la fecha de nacimiento del
paciente. El cambio requirió una migración que altera el tipo de la columna.

La segunda corrige la concurrencia del alta de ausencias. La verificación de que una
ausencia nueva no se solapa con otra ya registrada es un invariante de lectura seguida
de escritura, y corría bajo el nivel de aislamiento por defecto; se lo elevó al nivel
serializable, moviendo la verificación al interior de la misma transacción que la
inserción, exactamente como ya lo hacía el reemplazo de la grilla de horarios.

La tercera acota la franja horaria extra destinada a la primera sesión de un paciente
nuevo a un máximo de dos horas, además del mínimo de cero que ya se validaba, en
correspondencia con los valores que la especificación ejemplifica (una o dos horas).

La cuarta, transversal, reemplaza en el almacén de configuración por inquilino la
secuencia de buscar y luego crear por una única operación de alta-o-actualización
sobre la restricción de unicidad de la clave dentro de la organización.

## Decisiones y por qué

**La fecha de confirmación es un día de calendario, no un instante.** Una fecha de
confirmación carece de hora y de huso horario: es el día que la clínica lee de un
calendario. Modelarla como instante reproduce el defecto que la convención de fechas
del proyecto existe para evitar —el día renderizado se desplaza en una zona horaria
al oeste de Greenwich, de modo que una fecha enviada como un día se lee de vuelta como
el día anterior—. Por coherencia con las ausencias y con la fecha de nacimiento del
paciente, que ya seguían esa convención, se unificó el tratamiento: la columna almacena
sólo el día, y ningún instante alcanza la columna ni sale hacia un cliente.

**La conversión de la fecha opcional se encapsuló en una función que preserva la
diferencia entre ausente y nulo.** El campo admite tres situaciones —ausente (no se
modifica), nulo (se borra la fecha) y presente (se fija)— sobre una columna que acepta
nulos. La expresión ingenua que convierte "valor presente a fecha, en otro caso
indefinido" colapsa el nulo de "borrar" en el indefinido de "no modificar", perdiendo
la capacidad de limpiar la fecha; por eso se introdujo una función auxiliar que
distingue los tres casos y se reutiliza donde haga falta esa semántica.

**El solapamiento de ausencias se protege con aislamiento serializable, no con una
restricción de exclusión.** El invariante "ninguna ausencia se solapa con otra" no es
expresable como una restricción de unicidad, y el motor de persistencia empleado no
modela restricciones de exclusión sobre rangos; bajo el aislamiento por defecto, dos
altas concurrentes con rangos solapados verifican cada una la ausencia de conflicto
antes de que la otra haya confirmado, y ambas insertan, dejando dos ausencias solapadas
que nada detecta después y que emitirían dos disparos de reasignación para un mismo
período. El aislamiento serializable aborta a la transacción perdedora, que se traduce
en una respuesta de conflicto para que el cliente reintente. Es la misma decisión, y
por la misma razón, que la ya tomada para el reemplazo de la grilla de horarios y para
la cota de matrículas.

**La cota de dos horas de la franja extra acota a lo que la fuente ejemplifica, sin
inventar una restricción.** La especificación ejemplifica la franja extra con los
valores de una y dos horas; se acotó el máximo a dos en lugar de admitir cualquier
entero, entendiendo la enumeración de la fuente como el rango previsto y no como una
ilustración abierta. La cota es conservadora y reversible si la clínica solicitara
franjas mayores.

**El almacén de configuración pasó a alta-o-actualización atómica.** La secuencia de
buscar y luego crear permitía que dos escrituras concurrentes de una misma clave nueva
no vieran fila alguna y ambas intentaran crearla, con lo que la perdedora chocaba contra
la restricción de unicidad como un error no controlado, que la aplicación devolvía como
un fallo interno. La operación de alta-o-actualización colapsa esa secuencia en una
única sentencia atómica, de modo que una primera escritura concurrente se resuelve como
actualización en lugar de error. Se prefirió esta forma antes que capturar el error de
unicidad y reintentar, porque expresa el invariante en una sola operación en lugar de en
un manejo de excepciones; el localizador de la fila necesita el identificador de
organización de forma explícita —la restricción de unicidad es compuesta—, que se toma
del contexto de inquilino de la petición.

## Alternativas descartadas

- **Dejar la fecha de confirmación como instante con marca temporal:** descartada por
  reintroducir el desfase de un día en zona horaria al oeste de Greenwich que la
  convención de fechas del proyecto evita en todas las demás fechas de calendario.
- **Convertir la fecha opcional con la expresión ingenua "presente a fecha, en otro
  caso indefinido":** descartada por colapsar el nulo de "borrar" en el indefinido de
  "no modificar", lo que impediría limpiar la fecha.
- **Proteger el solapamiento de ausencias con una restricción de base de datos:**
  descartada porque el invariante de no solapamiento de rangos no es una restricción de
  unicidad y el motor no expone, a través de la capa de acceso a datos empleada,
  restricciones de exclusión sobre rangos.
- **Resolver la escritura concurrente de configuración capturando el error de unicidad
  y reintentando:** descartada en favor de la operación atómica de alta-o-actualización,
  que evita el manejo de excepciones para expresar el mismo invariante.

## Entidades / puertos / adaptadores tocados

- Entidad Profesional: la columna de fecha de confirmación cambió de instante a fecha de
  calendario (migración de tipo de columna).
- Servicio de ausencias del profesional: el alta pasó a ejecutarse bajo aislamiento
  serializable, con la verificación de solapamiento dentro de la transacción.
- Servicio de configuración por inquilino: la escritura pasó a alta-o-actualización.
- Módulo común de fechas: se añadió la función auxiliar que convierte una fecha de
  calendario opcional y anulable preservando la distinción entre ausente y nulo.
- Objetos de transferencia de datos del profesional (alta, edición y configuración):
  validación de la fecha como día de calendario y cota superior de la franja extra.

## Tests agregados o modificados

- Prueba de extremo a extremo de concurrencia del reemplazo de la grilla de horarios:
  dos reemplazos simultáneos no se fusionan; sobrevive exactamente una grilla y ninguna
  respuesta es un fallo interno.
- Prueba de extremo a extremo de concurrencia del alta de ausencias: dos altas
  simultáneas con rangos solapados dejan una sola fila; la perdedora responde rechazo por
  solapamiento o por conflicto de serialización, nunca un fallo interno.
- Prueba de extremo a extremo de concurrencia de la cota de matrículas: cinco altas
  simultáneas nunca dejan más de tres filas y al menos una tiene éxito, sin fallos
  internos.
- Prueba unitaria del almacén de configuración: la escritura invoca la operación de
  alta-o-actualización con el localizador compuesto y rechaza escribir sin inquilino en
  contexto.
- Prueba de extremo a extremo del ida y vuelta de la fecha de confirmación con un valor
  no nulo: una fecha en formato de día de calendario sobrevive al alta y a la lectura sin
  alteración, se almacena como una fecha a medianoche en tiempo universal coordinado —sin
  instante que pudiera desplazar el día en la zona horaria de la clínica—, un valor con
  hora se rechaza, una edición actualiza el día y un nulo explícito lo limpia. Las pruebas
  preexistentes siguen verificando además su valor nulo por defecto, ahora sobre el tipo
  de fecha de calendario.

El método de verificación de la concurrencia es afirmar el invariante final —ausencia de
corrupción y ausencia de fallo interno— antes que un desenlace temporalmente
determinista, dado que cuál transacción gana la carrera depende del planificador y no
del comportamiento que se prueba.

## Figuras pendientes

No surgen figuras nuevas de esta tarea.

## Componente y referencia

- Componente: backend.
- Rama: `main` (cambios en el árbol de trabajo, pendientes de confirmación al momento de
  redactar esta bitácora).
- Tarea: revisión de código y endurecimiento, sin número de requisito propio en la
  especificación; se registra bajo la Fase 1 por concentrar allí la mayoría de sus
  correcciones. La parte correspondiente al módulo de Pacientes se documenta en la
  entrada de Fase 2 de esta misma revisión.
