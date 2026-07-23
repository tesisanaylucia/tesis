# Fase 2 — Cierre (backend) — Revisión de código multi-ángulo contra las fuentes de verdad y correcciones derivadas

## Contexto

Concluida la implementación de los módulos de Profesionales y Pacientes sobre las
fundaciones, se realizó una segunda revisión sistemática de código —esta vez
organizada en varios ángulos de análisis independientes— cuyo objetivo era
verificar en qué medida el código satisface, hasta el estado actual, las dos
fuentes de verdad del proyecto: el anteproyecto de tesis y el documento de
especificación de requisitos de software. La revisión abarcó la totalidad de lo
implementado en la rama principal: el módulo de Pacientes, el de Profesionales, la
capa transversal de fundaciones (autenticación, multi-tenencia, auditoría, fechas
calendario, concurrencia y puertos de integración) y el esquema de base de datos
con sus migraciones.

El resultado global fue favorable: no se hallaron defectos de severidad alta. La
multi-tenencia, el manejo de fechas calendario, la auditoría dentro de la
transacción y el reciente refactor de integridad y normalización de la base
resultaron sólidos. La revisión sí identificó diez observaciones —seis de
severidad media y cuatro baja— entre defectos de conformidad con la
especificación, un vacío en la garantía de aislamiento entre organizaciones,
imprecisiones en la traza de auditoría y cuestiones de calidad. Esta tarea aplica
las diez correcciones. Por su naturaleza transversal, los cambios tocan código de
las tres fases ya implementadas (fundaciones, Profesionales y Pacientes).

## Qué se implementó

**Baja y alta del paciente como decisiones administrativas en sus propios puntos de
acceso.** El atributo de actividad del paciente dejó de ser un campo editable del
punto de modificación general. La baja lógica permanece en su punto de acceso
restringido al rol administrativo, y la reactivación pasó a un punto de acceso
propio, también restringido al rol administrativo. De este modo, un proceso
automático —el asistente conversacional— o un profesional ya no pueden dar de baja
ni reactivar a un paciente deslizando el atributo dentro de una modificación
ordinaria, y cada operación queda registrada en la traza de auditoría como el alta
o la baja que es, y no como una modificación genérica de campos.

**Cierre de un vacío en el filtrado automático por organización.** La extensión que
aplica el filtro de tenencia a cada consulta no contemplaba dos operaciones que la
biblioteca de acceso a datos ofrece —la actualización múltiple con retorno de filas
y la creación múltiple con retorno de filas—. La primera no falla de forma segura
ante la ausencia de filtro: sin la corrección, una consulta de ese tipo habría
podido leer y modificar filas de todas las organizaciones. Ambas operaciones se
incorporaron al conjunto que la extensión filtra y estampa. En la misma extensión
se corrigió además un caso en que una consulta sin argumentos sobre un modelo
alcanzado por la tenencia provocaba un error en tiempo de ejecución en lugar de
devolver el resultado ya filtrado.

**Contrato del puerto de la cerradura preparado para la integración real.** El
puerto que abstrae la cerradura electrónica identificaba cada código por el valor
del PIN y no permitía indicar sobre qué cerradura operar. Como la plataforma de
cerradura real elimina y consulta un código por un identificador propio del
proveedor —no por sus dígitos— y una instalación puede tener más de una cerradura,
ese contrato no habría permitido invalidar el código anterior al reprogramar o
cancelar un turno, que es exactamente lo que la especificación del módulo de
cerradura exige. El contrato se rediseñó para que la creación de un código devuelva
un identificador opaco junto con el PIN, y para que las tres operaciones reciban el
identificador de la cerradura sobre la que actúan.

**Configuración de agenda del profesional completada con la cadencia de turnos.** La
especificación distingue dos datos que la configuración anterior colapsaba en uno
solo: la cadencia con que se abren los turnos en la agenda —cada cuánto tiempo hay
un horario ofrecible— y la duración de la sesión, que es el dato que el asistente
comunica al paciente al confirmar el turno. Se agregó la cadencia como un dato
propio de cada profesional, distinto de la duración de la sesión, de modo que un
profesional pueda, por ejemplo, abrir turnos cada sesenta minutos con sesiones de
cuarenta y cinco. El límite superior del tramo horario adicional para pacientes
nuevos, que la especificación deja abierto, dejó de ser un tope estrecho para
pasar a ser una cota amplia de contención antes que una regla de negocio.

**Precisión de la traza de auditoría ante operaciones sin efecto.** Se corrigieron
dos casos en que la traza registraba cambios que no habían ocurrido: el registro de
una consulta atendida nombraba siempre los mismos tres campos como modificados, aun
cuando la fecha de última consulta no avanzaba, y una modificación de cuerpo vacío
—de un paciente o de un vínculo de tratamiento— dejaba una entrada de auditoría de
una operación que no cambiaba nada. En ambos casos la operación de escritura y su
entrada de auditoría se omiten cuando no hay ningún campo que efectivamente cambie,
y el registro de la consulta nombra únicamente los campos que realmente se
movieron.

**Correcciones de calidad y de convención.** Se unificó la definición duplicada de
la representación de una obra social, que existía por partida doble en la capa de
presentación de Profesionales y en la de Obras Sociales, importándola desde su
lugar canónico. Se cerró un canal lateral de tiempo en el inicio de sesión que
permitía distinguir un correo registrado de uno inexistente por la latencia de la
respuesta, igualando el trabajo de ambos caminos. Y se actualizó el documento de
convenciones del repositorio para reflejar que los métodos de los puertos de
integración se nombran en inglés, como el resto de los identificadores del código,
dejando el término del glosario en español en un comentario sobre cada puerto.

## Decisiones y por qué

**El atributo de actividad se retira del punto de modificación en lugar de validarse
allí.** Se consideró conservar el campo en la modificación general y rechazar por
validación cualquier valor que intentara desactivar al paciente, pero eso deja la
regla dependiendo de una comprobación que hay que recordar mantener. Separar el alta
y la baja en puntos de acceso propios convierte la restricción en un hecho de
enrutamiento —qué rol puede alcanzar cada punto— en vez de una condición dispersa en
la lógica, y hace que cada operación se audite con la semántica que le corresponde. La
reactivación se restringió, por ahora, al rol administrativo, en simetría con la baja;
queda registrada como opción abierta la posibilidad de habilitar también al proceso
automático a reactivar a un paciente inactivo que vuelve a solicitar turno en el flujo
conversacional, a confirmar con la institución antes de habilitarla, dado que —a
diferencia de la baja— la reactivación puede surgir naturalmente durante una reserva.

**El aislamiento entre organizaciones se garantiza en la extensión, no en cada
consumidor.** Las dos operaciones omitidas no tienen hoy ningún invocador en el
código, pero la extensión es precisamente la barrera única en la que descansa la
garantía de que una petición de una organización nunca alcanza los datos de otra;
dejar una operación fuera de esa barrera es un riesgo latente que se materializaría
en cuanto alguien la usara. Por eso la corrección se aplica en la extensión y cubre
todas las formas del dato de entrada, no en los puntos de uso.

**El contrato del puerto de la cerradura se corrige antes de construir la
integración, no después.** Aunque la integración real con la plataforma de cerradura
pertenece a una fase posterior y hoy sólo existe un adaptador de prueba, el contrato
del puerto es lo que condiciona esa fase futura. Un contrato que identifica el código
por su valor y no sabe sobre qué cerradura opera obligaría a rehacer el puerto —y
todo lo construido contra él— al llegar la integración. Corregirlo ahora, mientras el
único implementador es el adaptador de prueba, es la intervención de menor costo.

**La cadencia y la duración de la sesión son dos datos porque cumplen dos funciones
distintas.** La cadencia determina los horarios de turno que el sistema puede ofrecer
y alimenta la futura generación de agenda; la duración de la sesión es información que
se comunica al paciente al confirmar. Colapsarlas en un único valor impide expresar el
propio ejemplo de la especificación —atención cada una hora con sesiones de cuarenta y
cinco minutos— y confunde dos conceptos que la fuente de verdad mantiene separados.

**La traza de auditoría afirma sólo lo que ocurrió.** Registrar como modificados
campos que no cambiaron degrada el valor de la traza, que es el registro contra el
cual se rinde cuentas en materia de protección de datos personales. Se optó por
derivar los campos efectivamente modificados —de la cantidad de filas afectadas por
la escritura condicional y del estado previo del vínculo— y por omitir por completo la
escritura y su entrada cuando la operación no cambia nada.

## Alternativas descartadas

- **Validar el atributo de actividad dentro de la modificación general** en lugar de
  retirarlo: descartada porque mantiene la regla dependiendo de una comprobación
  recordada, en vez de un hecho de enrutamiento por punto de acceso y rol.
- **Conservar el filtro de tiempo del inicio de sesión sólo en el mensaje idéntico de
  error**: descartada porque el mensaje ya era idéntico y la diferencia observable era
  la latencia; sólo igualar el trabajo de ambos caminos cierra el canal.
- **Renombrar los métodos de los puertos al español** para satisfacer la convención tal
  como estaba redactada: descartada en favor de actualizar la convención, ya que los
  puertos son interfaces ordinarias del lenguaje y el resto de los identificadores del
  código están en inglés; el término del glosario en español se conserva en un
  comentario sobre cada puerto.

## Entidades / puertos / adaptadores tocados

- Esquema: se agregó la cadencia de turnos como columna del profesional, con su
  migración correspondiente; el dato es nulo hasta que el profesional lo configura,
  como la duración de la sesión.
- Puerto de la cerradura y su adaptador de prueba: nuevo tipo de retorno con
  identificador opaco y parámetro de cerradura en las tres operaciones.
- Extensión de tenencia: cobertura de las operaciones múltiples con retorno de filas y
  del caso de consulta sin argumentos.
- Servicios de Pacientes y de vínculos paciente–profesional, servicio de Profesionales,
  servicio de autenticación y capas de presentación de Profesionales y Obras Sociales.

## Tests agregados o modificados

No se agregaron pruebas nuevas; se ajustaron las existentes para acompañar los cambios
de contrato, y se ejecutó la suite completa para verificar que las correcciones no
introdujeran regresiones. Se actualizó la prueba de resolución del puerto de la
cerradura al nuevo contrato de retorno e identificación; la prueba de extremo a extremo
del alta, baja y reactivación del paciente, para ejercitar el nuevo punto de acceso de
reactivación y la imposibilidad de que un profesional lo alcance; y las pruebas de la
configuración del profesional, para incorporar la cadencia de turnos. La suite completa
—pruebas unitarias y de extremo a extremo— quedó en verde.

## Figuras pendientes

No surgen figuras nuevas de esta tarea.

## Componente y referencia

- Componente: backend.
- Rama: `main` (cambios en el árbol de trabajo, pendientes de confirmación al momento de
  redactar esta bitácora).
- Tarea: revisión de código multi-ángulo contra las fuentes de verdad y aplicación de las
  diez correcciones derivadas, que cierra la etapa de implementación de los módulos de
  Profesionales y Pacientes sobre las fundaciones.
