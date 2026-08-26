# Fase 5 — Capa conversacional y WhatsApp (backend) — validación de edad, información de obra social y preguntas frecuentes (TASK-51)

## Qué se implementó

Se implementaron las tres reglas de negocio conversacionales que el ticket
enumera: la validación de edad para pacientes nuevos de profesionales que
sólo atienden mayores, la respuesta a consultas sobre la obra social del
profesional, y la respuesta a preguntas frecuentes con una salida
explícita para el caso en que la clínica no tenga cargada esa pregunta.

Ninguna de las tres exigió cambios de esquema. El relevamiento previo
mostró que las cuatro columnas que la tarea necesita ya existían desde
fases anteriores —el indicador de sólo mayores y el tipo de atención en
la entidad Profesional, la fecha de nacimiento en Paciente y la marca de
primera sesión en el vínculo paciente-profesional—, que la tabla de
preguntas frecuentes se había creado en TASK-47 junto con su servicio de
búsqueda, y que la aceptación de obras sociales por profesional se
modeló en la fase de Fundaciones como una entidad de vinculación contra
un catálogo global. Lo que faltaba no era estructura de datos sino
exponer esos hechos a la conversación en el momento en que el paciente
los necesita.

La validación de edad ya se aplicaba al reservar desde la fase del Motor
de Turnos, pero recién al final del flujo, cuando el paciente ya había
elegido profesional, visto los horarios libres y escogido uno. El ticket
pide en cambio que el bot valide la edad y *finalice el flujo de reserva*
cuando el paciente no cumple el requisito, lo que exige una respuesta
antes de ofrecer cualquier horario. Se agregó entonces la herramienta
`check_booking_eligibility` y, para alimentarla sin duplicar la regla, se
extrajo la regla misma a un módulo puro reutilizado por los tres puntos
que la aplican.

La información de obra social se resolvió ampliando el listado de
profesionales que el flujo de reserva ya invoca, en lugar de agregar una
herramienta nueva. La respuesta de preguntas frecuentes, que ya existía,
se completó con el texto exacto que el bot debe enviar cuando no hay
coincidencia. El manual de flujos del *system prompt* incorporó las tres
reglas como secciones propias.

## Decisiones y por qué

**La regla de los requisitos del paciente nuevo se extrajo a un módulo
puro en lugar de reimplementarse para la conversación.** Antes de esta
tarea la regla estaba escrita dos veces dentro del servicio de turnos:
una en la reserva y otra en la revalidación que ejecuta la
reprogramación, con mensajes distintos para el mismo rechazo. Agregar un
tercer punto de aplicación —la verificación previa del chatbot— habría
llevado la duplicación a tres copias de una regla que, si se desfasan,
producen exactamente el peor resultado posible en una conversación: que
el bot le diga al paciente que puede reservar y la reserva lo rechace a
continuación. La regla quedó entonces en un módulo sin estado ni acceso a
la base, al que se le pasan las dos columnas del profesional, la fecha de
nacimiento del paciente y la edad mínima del inquilino, y que devuelve el
motivo del rechazo o la ausencia de rechazo. Es el mismo tratamiento que
ya recibían la máquina de estados del turno y la regla de inactividad del
paciente. La edad mínima se recibe como parámetro y no se lee dentro de
la regla, porque es un valor de configuración por inquilino y una regla
que fuera a buscarlo necesitaría la base de datos, perdiendo justamente
la propiedad que la hace verificable de forma aislada.

**El motivo del rechazo se devuelve como un discriminante y no como un
texto.** La verificación distingue tres casos: el profesional cerró la
admisión de pacientes nuevos, el paciente no alcanza la edad mínima, y no
hay fecha de nacimiento registrada. El manual de flujos indica qué
decirle al paciente en cada uno, de modo que el rechazo se redacta
siempre igual en lugar de depender de cómo el modelo parafrasee una
oración en inglés recibida desde el servidor. El tercer caso se separó
explícitamente del segundo, que hasta ahora los trataba juntos: la fecha
de nacimiento es opcional en el esquema precisamente porque los registros
históricos que se importan desde las planillas de la clínica muchas veces
no la traen, y "no sabemos tu edad" pide una acción al paciente mientras
que "sos menor de edad" cierra la puerta. Decirle a un adulto que no
alcanza la edad mínima porque su ficha está incompleta sería un error de
información, no una restricción.

**Ante una fecha de nacimiento ausente, el bot la pide y valida con ella,
en lugar de derivar al paciente a la clínica.** El documento de requisitos
es explícito: el chatbot valida la edad "solicitando el DNI y la fecha de
nacimiento". Una primera versión de esta tarea cerraba ese caso pidiéndole
al paciente que se comunicara con la clínica, apoyándose en que la
herramienta de identificación sólo daba de alta pacientes y no
actualizaba los existentes; la usuaria lo corrigió señalando que el caso
—infrecuente, pero real en los registros importados— debe resolverse
dentro de la conversación. La herramienta de identificación pasó entonces
a tener un tercer camino: además de buscar y de crear, **completa** los
datos obligatorios de reserva que la ficha no tiene y la conversación
acaba de aportar. El manual de flujos indica en esa rama pedir la fecha,
completarla y volver a consultar la elegibilidad, de modo que la
validación se hace igual y con el dato recién obtenido.

**Sólo se escribe lo que falta, nunca lo que ya está cargado.** El filtro
es qué datos le faltan a la ficha y no qué datos mandó el modelo: si el
paciente ya tiene fecha de nacimiento registrada y el modelo envía otra,
no se escribe nada. La distinción es la que justifica apartarse de la
decisión de TASK-50 —que los datos de contacto de un paciente registrado
son el registro de la clínica y no se reescriben desde la conversación—
sin contradecirla: completar un valor nulo agrega un hecho que la ficha
nunca tuvo, mientras que pisar uno existente es la actualización
destructiva silenciosa que aquella decisión rechazó. La escritura pasa
por el mismo `PatientsService.update` auditado que usan la app y la API,
de modo que la traza registra qué campos completó la conversación y
ninguno más, y una ficha completa no paga escritura ni deja entrada de
auditoría alguna.

**La lista de datos obligatorios para reservar se extrajo a un módulo
compartido.** Estaba escrita tres veces: en la lectura de estado de datos
del paciente, en la validación previa de la reserva y, ahora, en la
herramienta que los completa. Es la misma clase de duplicación que la
regla de admisión de paciente nuevo, con la misma consecuencia
conversacional si las copias se desfasan: un bot que completa lo que se
le dijo que faltaba y una reserva que después rechaza por un campo que
nadie le pidió al paciente. El módulo devuelve además la lista en orden
fijo, para que la conversación pida los datos siempre en el mismo orden,
y la herramienta de identificación la devuelve al modelo en cada respuesta
en lugar de dejar que la infiera de qué campos vienen en nulo.

**La verificación previa es una salida temprana del flujo, nunca el
control.** La reserva sigue aplicando la misma regla al recibir el
pedido, y una prueba lo verifica explícitamente simulando un modelo que
ignora la verificación y reserva igual: el turno se rechaza. La razón es
que un paso conversacional no puede ser un control de seguridad, porque
depende de que el modelo lo ejecute; lo que aporta es que la conversación
termine con una explicación en el momento correcto en lugar de terminar
con un error después de hacerle perder tiempo al paciente.

**El predicado que define "paciente nuevo" es la marca de primera sesión
y no el tipo de paciente.** El ticket menciona el tipo del vínculo, pero
el tipo se degrada a "nuevo" cuando un paciente deja de concurrir durante
más de un año, según la regla de inactividad de la fase de Pacientes,
mientras que la marca de primera sesión permanece en falso una vez que la
primera sesión ocurrió. Un paciente que vuelve después de años no es
alguien a quien la restricción de admisión de pacientes nuevos haya sido
escrita para rechazar. Además —y esto es lo determinante— la reserva usa
la marca de primera sesión, de modo que usar el tipo en la verificación
previa habría reintroducido por otra vía la discrepancia que toda la
extracción de la regla buscaba evitar. Ambas lecturas quedaron en un
único método privado compartido.

**La consulta de obra social viaja en el listado de profesionales y no en
una herramienta propia.** El ticket la describe como una consulta
separada, pero la lectura que ya sirve el listado carga con cada
profesional las obras sociales que acepta, el flujo de reserva invoca ese
listado de todos modos, y la clínica tiene un puñado de profesionales.
Una herramienta adicional habría sido un segundo viaje a la base para
releer datos que el modelo ya tenía en el contexto. Las obras sociales se
devuelven sólo por nombre: el identificador del catálogo es interno y el
manual de flujos prohíbe mostrarle identificadores al paciente, de modo
que un campo que la herramienta no devuelve es un campo que el modelo no
puede filtrar.

**El tipo de atención se informa tal como lo modela la fuente de verdad,
con dos valores y no tres.** El ticket pide que el bot informe si el
profesional atiende "por obra social provincial, particular o ambos",
pero tanto el diagrama entidad-relación como el esquema implementado
definen el tipo de atención como un único valor entre obra social y
particular. Se optó por informar fielmente lo que el dato dice, en lugar
de agregar un tercer valor al enumerado —un cambio de esquema que la
fuente de verdad no respalda— o de inventar una combinación derivada que
ningún dato sostiene. La política de la clínica sobre obras sociales
—qué obras sociales acepta la institución en general, o si recibe
coseguro— es contenido que corresponde a la tabla de preguntas
frecuentes, que es configurable por inquilino, y el manual de flujos
deriva explícitamente ese tipo de pregunta hacia allí.

**No se informan importes, aunque el documento de requisitos los
contemple.** El SRS describe que el bot muestre el importe de la consulta
con obra social y el de la consulta particular, reservando únicamente el
copago. El ticket, en cambio, coloca los importes explícitamente fuera de
alcance y remite al guardrail de la fase anterior, que bloquea toda
respuesta con un monto. Se implementó lo que el ticket decide, y el
manual de flujos se lo indica al modelo de forma explícita, de modo que
la instrucción y el guardrail apuntan en la misma dirección en lugar de
contradecirse.

**El texto de "no tengo esa información" viaja dentro del resultado de la
herramienta.** La búsqueda en preguntas frecuentes ya devolvía la
ausencia de coincidencia como un resultado exitoso y no como un fallo,
pero sin decir qué hacer con ella. Un modelo que recibe únicamente "no
hubo coincidencia" está a un turno de responder la pregunta por su
cuenta, con lo que sea que crea saber sobre una clínica que no conoce.
El texto se devuelve entonces junto con la ausencia de coincidencia, con
la instrucción de enviarlo tal cual: es el mismo tratamiento que la fase
anterior le dio al texto de solicitud de consentimiento, y por la misma
razón. Se mantuvo como constante y no como plantilla configurable por
inquilino porque es una conducta del bot y no contenido de la clínica; lo
que cada clínica personaliza son las filas de la tabla de preguntas
frecuentes. La redacción evita deliberadamente mencionar a una secretaria,
un operador o un número de teléfono, porque el guardrail de la fase
anterior bloquea toda derivación a una persona y el documento de
requisitos es explícito en que el bot no deriva a un operador humano.

**Se corrigió una filtración detectada durante la validación: la
herramienta de identificación devolvía la entidad de la base y no la
respuesta presentada.** El criterio de aceptación sobre no volver a
pedirle la fecha de nacimiento a un paciente ya registrado obligó a mirar
qué devuelve exactamente esa herramienta, y la prueba mostró que el
modelo estaba recibiendo el identificador de organización —el dato que
todos los presentadores del sistema existen para quitar antes de cruzar
un límite— y la fecha de nacimiento como un instante en tiempo universal
en lugar del día calendario que la clínica lee. Se aplicó el mismo
presentador que usan los endpoints de pacientes, con lo que además el
modelo recibe la edad ya calculada y no tiene que hacer aritmética de
fechas, que es precisamente lo que la validación de edad determinística
existe para evitar.

## Entidades, puertos y adaptadores tocados

- Sin cambios de esquema ni migraciones. Se leen columnas ya existentes:
  el indicador de sólo mayores y el tipo de atención del profesional, la
  fecha de nacimiento del paciente, la marca de primera sesión del
  vínculo, la tabla de preguntas frecuentes y la entidad de vinculación
  entre profesional y obra social.
- Nuevo módulo puro de reglas de admisión de paciente nuevo, dentro del
  subdominio de Turnos.
- Servicio de turnos: se unificaron los dos puntos que aplicaban la regla
  y se agregó una lectura pública que responde la elegibilidad sin lanzar
  excepción, además de extraerse a métodos privados compartidos la
  lectura de la marca de primera sesión y la de las dos columnas del
  profesional.
- Nuevo módulo puro con la lista de datos obligatorios para reservar,
  dentro del subdominio de Pacientes, leído por la lectura de estado de
  datos del paciente, por la validación previa de la reserva y por la
  herramienta de identificación del chatbot.
- Catálogo de herramientas del chatbot: una herramienta nueva de
  verificación previa a la reserva; ampliación del listado de
  profesionales con tipo de atención y obras sociales; ampliación del
  resultado de la búsqueda en preguntas frecuentes con el texto de
  fallback; y, en la herramienta de identificación, aplicación del
  presentador de pacientes, un tercer camino que completa los datos
  obligatorios ausentes y la devolución de los que siguen faltando.
- Manual de flujos del *system prompt*: tres secciones nuevas y dos pasos
  agregados a los procedimientos de identificación y de reserva.

## Tests

- Pruebas unitarias del módulo puro de reglas, incluida la frontera de la
  edad —el día en que el paciente cumple la edad mínima y el día
  anterior—, la distinción entre fecha de nacimiento ausente y paciente
  menor, y la aplicación de una edad mínima distinta de la predeterminada.
  Las fechas de nacimiento se calculan a partir del día de ejecución para
  que los casos no caduquen con el paso del calendario.
- Pruebas unitarias de la lectura de elegibilidad del servicio de turnos:
  que responde en lugar de lanzar, que un paciente que ya pasó su primera
  sesión no consulta siquiera las columnas del profesional, que aplica la
  edad mínima configurada por el inquilino, y que un identificador
  inexistente sigue produciendo un 404 en lugar de un rechazo de reserva.
- Pruebas unitarias del módulo de datos obligatorios: qué falta en cada
  combinación, el orden fijo de la lista, el paciente inexistente tratado
  como carente de todo, y una cadena vacía no contada como dato cargado.
- Pruebas unitarias de las herramientas: que el rechazo llega al modelo
  como resultado exitoso y no como fallo, que el listado de profesionales
  informa el tipo de atención y las obras sociales por nombre, que un
  profesional particular se informa con una lista vacía y no con un campo
  ausente, y que la identificación de paciente devuelve la respuesta
  presentada, sin identificador de organización y con la fecha de
  nacimiento como día calendario.
- Pruebas unitarias del completado de datos: que informa qué falta sin
  escribir nada, que escribe la fecha de nacimiento ausente cuando la
  conversación la aporta, que no pisa un dato ya cargado aunque el modelo
  mande otro, y que escribe únicamente la mitad faltante cuando el modelo
  manda ambos datos.
- Pruebas de conversación extremo a extremo, con el puerto de
  inteligencia artificial simulado y todo lo demás real, para los seis
  criterios de aceptación del ticket: rechazo de un paciente de 17 años
  con un profesional de sólo mayores —verificando además que no queda
  ningún turno creado y que la reserva directa también lo rechaza—,
  reserva permitida para el mismo caso con 18 años, identificación de un
  paciente registrado que devuelve su fecha de nacimiento sin volver a
  pedirla, información de obra social por profesional, pregunta frecuente
  existente respondida con el texto del propio inquilino —con una fila
  homónima en otra organización para que una filtración entre inquilinos
  se manifieste—, y pregunta sin respuesta que devuelve el texto de
  fallback y lo entrega al paciente intacto tras atravesar los guardrails.
- Prueba de conversación del caso de la ficha sin fecha de nacimiento, de
  punta a punta y en dos turnos: la verificación previa responde que no
  puede decidir la edad, el paciente aporta la fecha, la herramienta de
  identificación la registra, la verificación vuelve a consultarse y la
  reserva se concreta; se comprueba que la fecha quedó escrita en la ficha
  y que la traza de auditoría nombra ese campo y ninguno más. Dos casos
  complementarios: una fecha aportada que deja al paciente por debajo de
  la edad mínima cierra la reserva igual que en el caso ya registrado, y
  una fecha aportada sobre una ficha que ya la tenía no la modifica.

Suite completa en verde al cierre: 641 pruebas unitarias y 478 de
integración.

## Figuras pendientes

- Diagrama de decisión de la admisión de un paciente nuevo (marca de
  primera sesión → admisión abierta del profesional → indicador de sólo
  mayores → fecha de nacimiento registrada → edad frente a la edad mínima
  del inquilino), señalando los tres motivos de rechazo y los tres puntos
  del sistema que consultan la misma regla: la verificación previa del
  chatbot, la reserva y la revalidación de la reprogramación. Sección
  4.6 Capa conversacional y WhatsApp.
- Diagrama de secuencia del flujo de reserva con la verificación previa
  intercalada, mostrando sus tres salidas: continuar hacia la
  disponibilidad, finalizar la reserva con ese profesional y ofrecer otro,
  o pedirle al paciente la fecha de nacimiento ausente, completarla y
  volver a verificar. Sección 4.6 Capa conversacional y WhatsApp.
- Diagrama del flujo de consulta general (pregunta del paciente →
  búsqueda por superposición de palabras sobre la tabla del inquilino →
  rama con coincidencia, que responde con la respuesta cargada, y rama
  sin coincidencia, que devuelve el texto fijo para enviar literalmente),
  señalando que ambas ramas son resultados exitosos. Sección 4.6 Capa
  conversacional y WhatsApp.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-51-age-validation-insurance-faq`,
  creada desde `origin/main` como pide la usuaria. Sin commit ni push al
  momento de escribir esta entrada: pendiente de autorización explícita de
  la usuaria.
- Ticket: TASK-51 (Jira), "P5.6 – Validación de edad, info de obra social/
  importes y FAQ", bajo el épico TASK-8 (Módulo 5). Fuentes de verdad
  consultadas en Drive: el documento de Especificación de Requisitos de
  Software vigente (Anexo, "Módulo turnos" y "Módulo profesionales") y el
  diagrama entidad-relación `modelo_base_de_datos.png`.
- Discrepancias registradas entre el ticket y las fuentes de verdad, y
  cómo se resolvieron: (a) el ticket pide informar "obra social
  provincial, particular o ambos", pero tanto el diagrama
  entidad-relación como el esquema definen el tipo de atención con dos
  valores excluyentes, de modo que se informa lo que el dato dice y la
  política institucional sobre obras sociales se deriva a la tabla de
  preguntas frecuentes; (b) el ticket describe la tabla de obras sociales
  como acotada por inquilino, mientras que el esquema la modela desde la
  fase de Fundaciones como catálogo global sin dueño, siendo la
  *aceptación* de una obra social —no la obra social misma— lo que
  pertenece a un profesional y, por su intermedio, a un inquilino; no se
  modificó esa decisión, ya documentada y deliberada; (c) el SRS
  contempla informar importes de consulta y el ticket los deja fuera de
  alcance remitiendo al guardrail de TASK-49, y se implementó la decisión
  del ticket.
- Dependencias declaradas: TASK-48 (P5.3, orquestador), TASK-47 (P5.2,
  herramientas) y TASK-24 (P1.4, indicador de sólo mayores en Profesional),
  todas ya fusionadas; TASK-50 (P5.5, flujos conversacionales), fusionada
  a `origin/main` antes de comenzar esta sesión.
- Fuera de alcance, declarado en el propio ticket y respetado: la carga
  inicial de filas de la tabla de preguntas frecuentes, que corresponde al
  seed del Módulo 5 o a TASK-33. Consecuencia práctica que queda
  registrada: hoy ninguna organización tiene filas cargadas fuera de las
  que crean las pruebas, de modo que en un entorno de desarrollo toda
  consulta general responde el texto de fallback hasta que ese seed
  exista.
- Corrección aplicada durante la sesión, a pedido de la usuaria y con
  respaldo en el SRS: la primera versión de esta tarea derivaba a la
  clínica al paciente ya registrado sin fecha de nacimiento que pretendía
  atenderse con un profesional de sólo mayores. El SRS dice que el
  chatbot valida la edad "solicitando el DNI y la fecha de nacimiento",
  de modo que ese caso se resolvió dentro de la conversación: la
  herramienta de identificación completa los datos obligatorios ausentes
  y el manual de flujos pide la fecha, la registra y vuelve a verificar.
  Queda registrado que ese camino escribe únicamente valores nulos y
  nunca pisa un dato ya cargado, lo que preserva la decisión de TASK-50
  sobre los datos de contacto del paciente registrado.
