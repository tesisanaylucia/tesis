# Fase 2 — Pacientes (backend) — ABMC de pacientes y vínculo con profesionales (TASK-28)

## Qué se implementó

Se construyó el módulo Pacientes sobre las entidades incorporadas en la tarea
anterior: altas, bajas, modificaciones y consultas de pacientes, búsqueda por
número de documento dentro de la organización, gestión del vínculo entre un
paciente y cada profesional que lo atiende, y un punto de consulta que responde
las dos reglas que los requisitos imponen antes de reservar un turno.

Los caminos expuestos son el alta de paciente, el listado y la búsqueda, el
detalle, la modificación, la baja lógica, la vinculación con un profesional, la
actualización de la prioridad y los datos de contacto que ese profesional
mantiene, y la consulta del estado de datos del paciente. La lógica de reserva
propiamente dicha no forma parte de la tarea: el módulo expone el dato que esa
lógica necesitará, no la conversación que lo usará.

## Decisiones y por qué

**La baja es lógica y exigió una columna que el diagrama no dibuja.** El
diagrama entidad-relación no asigna al paciente ningún atributo de ciclo de
vida, pero los requisitos de la tarea piden que la baja sea lógica. Se agregó
una columna de estado activo, con el mismo criterio ya aplicado al profesional.
La alternativa, borrar la fila, es inviable por dos razones convergentes: los
vínculos de tratamiento y los consentimientos se eliminarían en cascada, y la
traza de auditoría restringe por sí misma el borrado de un paciente que
menciona. Es decir, el propio esquema ya había decidido que un paciente no se
elimina; la columna se limita a darle al módulo la forma de expresarlo. La
reactivación se resolvió como una modificación ordinaria de esa columna y no
como un alta nueva, porque un paciente que regresa tras años es la misma persona
con el mismo historial, y un alta chocaría además con la unicidad del documento.

**El contacto de emergencia y el correo electrónico se movieron al paciente.**
El diagrama entidad-relación los ubica en el vínculo paciente-profesional, y la
tarea anterior los había modelado allí por fidelidad a esa fuente. La revisión
del módulo mostró que la ubicación era incorrecta: son datos *de la persona*,
que el propio documento de requisitos enumera entre los "datos de contacto" del
paciente, y que una reserva exige con independencia del profesional con quien
sea. Alojar una copia por profesional tratante dejaba la pregunta "¿cuál es el
contacto de emergencia de este paciente?" con tantas respuestas posibles como
profesionales lo atienden y ninguna regla para elegir entre ellas, que es
exactamente el defecto que se había evitado en las demás decisiones de modelado.
Se los trasladó a la entidad Paciente mediante una migración que copia los
valores existentes antes de eliminar las columnas —conservando, cuando hay
varios vínculos con dato cargado, el del vínculo modificado más recientemente—,
de modo que la corrección no pierde información. En el vínculo permanece
únicamente lo que sí varía según quién observe al paciente: la prioridad que ese
profesional asigna, sus observaciones de uso interno, el tipo de paciente que es
para él, el indicador de primera sesión y la fecha de la última consulta.

**Los tres datos obligatorios de la reserva se exigen en el alta.** Como
consecuencia del traslado anterior, los tres datos que los requisitos exigen
para reservar —documento, fecha de nacimiento y contacto de emergencia— son
atributos del paciente y se validan en un único lugar, el alta. Sus columnas
siguen admitiendo nulos, para que la migración de pacientes preexistentes pueda
importar registros incompletos: la obligatoriedad vive en el contrato del
endpoint interactivo y no en la columna, asimetría que se documentó
explícitamente para que no se lea como un descuido. El vínculo con un profesional
queda así identificado por completo por la dirección del recurso, sin cuerpo
obligatorio.

**La vinculación es idempotente y responde 200.** El vínculo queda identificado
por completo por la URL, y quien más necesita este camino —el asistente
conversacional que toma una reserva— no puede saber de antemano si el paciente
ya estaba vinculado. Se resolvió que una segunda invocación no falle sino que
actualice los datos de contacto recibidos y devuelva el vínculo existente, con
código 200 en lugar de 201, porque la respuesta describe el estado del vínculo y
no afirma que se haya creado una fila en esa invocación. El indicador de primera
sesión se deja en el valor por defecto de la columna al crearlo y no se toca al
repetir: ese valor por defecto *es* la regla de los requisitos según la cual el
sistema debe validar si es la primera vez que el paciente se atiende con ese
profesional, y restablecerlo en el servicio le daría un segundo lugar desde el
cual divergir. La entrada de auditoría distingue de todos modos la creación de
la actualización, de modo que la traza no pierde la diferencia que la respuesta
deliberadamente no expone.

**El umbral de un año se trata como dato de la organización, no como literal.**
Los requisitos establecen que a un paciente que dejó de concurrir por más de un
año se le solicitará la actualización de sus datos de contacto. El plazo se
almacena como configuración de la organización, con el valor por defecto de doce
meses cuando no está fijado, siguiendo la convención del proyecto de tratar las
reglas de negocio como datos.

La fila con el valor por defecto se crea mediante una migración de datos, y no
únicamente en el seed. La distinción resultó relevante: el seed es dato de
desarrollo y del piloto y no se ejecuta en un entorno real, de modo que una
regla sembrada sólo allí existiría en producción exclusivamente como constante
del código, y la fila que un operador buscaría en la tabla de configuración
sencillamente no estaría. La migración inserta el valor para toda organización
existente, el seed cubre las organizaciones que se creen después y el servicio
mantiene el mismo valor como respaldo cuando la fila falta; los tres leen una
única constante, de modo que no pueden discrepar. Ni la migración ni el seed
sobrescriben un valor ya modificado por la clínica: ambos son idempotentes y
respetan la decisión registrada. Se eligió una única clave porque el mismo umbral
gobierna también la regla que vuelve a marcar al paciente como nuevo, que
corresponde a la tarea siguiente de la fase: dos claves independientes podrían
configurarse con valores distintos y producir un paciente al que se le piden
datos actualizados pero que sigue siendo tratado como recurrente. Un valor de
configuración que no sea un número entero positivo se ignora en favor del valor
por defecto, dado que la configuración se almacena como JSON y nada más se
interpone entre un error de tipeo y una fecha de corte sin sentido.

**Un profesional ve sólo a sus pacientes, y sólo su propio vínculo.** El sistema
ya restringe cada consulta a la organización del solicitante; sobre esa base, el
módulo agrega una restricción más estrecha para el rol profesional, que alcanza
únicamente a los pacientes con los que tiene vínculo de tratamiento. La
restricción se implementó como filtro del camino de lectura y no como
verificación posterior, de modo que un paciente que el profesional no atiende
resulta indistinguible de uno inexistente: responder que el acceso está
prohibido confirmaría que el registro existe, lo que ya es una revelación.
Además, en un paciente compartido por dos profesionales cada uno recibe
solamente su propio vínculo, porque la prioridad, el contacto y —cuando la tarea
correspondiente las incorpore— las observaciones de uso interno son juicio de
ese profesional y no del colega. El personal administrativo sigue viendo el
registro completo, que es lo que un legajo administrativo significa.

**La traza de auditoría registra nombres de campos, nunca valores.** Toda
mutación queda auditada dentro de la misma transacción que la produce, con la
referencia real al paciente que la clave foránea incorporada en la tarea
anterior hizo posible, de modo que una consulta de cumplimiento pueda responder
qué se hizo sobre un paciente sin interpretar identificadores genéricos. El
detalle de una modificación enumera qué campos cambiaron y nunca su contenido,
porque el contenido es dato personal y la traza no es el lugar donde deba
replicarse. Al implementarlo se detectó que enumerar las claves del objeto
recibido no produce esa lista: la biblioteca de transformación instancia todas
las propiedades declaradas, de modo que una modificación de un solo campo
generaba una entrada que declaraba haber cambiado todos. La corrección se
factorizó en una función propia, con su prueba unitaria, y la regla se incorporó
a las convenciones del repositorio.

## Alternativas descartadas

- **Conservar el contacto de emergencia y el correo en el vínculo**, por
  fidelidad al diagrama entidad-relación: descartada tras la revisión del
  módulo, por las razones expuestas más arriba. Se deja constancia de que la
  implementación se aparta aquí del diagrama de forma deliberada y documentada,
  no por omisión.
- **Responder 409 al vincular un paciente ya vinculado**, que es la lectura
  estricta del verbo de creación: descartada porque obligaría a todo cliente a
  consultar antes de vincular y a tratar el conflicto como caso normal, cuando
  lo que necesita es la garantía de que el vínculo existe y el dato de si la
  primera sesión sigue pendiente.
- **Devolver la búsqueda por documento como un recurso único con respuesta 404
  cuando no hay coincidencia**: descartada porque obligaría al cliente a
  distinguir un documento inexistente de un identificador mal formado; se
  devuelve una lista de cero o un elemento, de modo que "no está registrado" sea
  una respuesta ordinaria y no un error.
- **Ocultar sin excepción los pacientes dados de baja**: descartada porque la
  unicidad del documento alcanza también a las filas inactivas, de modo que un
  administrativo que reinscribiera a un paciente que regresa recibiría un
  conflicto contra un registro que no puede ver. El listado admite un parámetro
  explícito para incluirlos, que es lo que hace recuperable la baja.
- **Verificar la unicidad del documento leyendo antes de escribir**: descartada
  por inútil frente a la concurrencia; se deja que la restricción de la base de
  datos decida y se traduce su violación a un conflicto que nombra el documento.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma` y migración `20260721042016_add_patient_active_flag`:
  columna de estado activo del paciente, con su justificación documentada en el
  modelo.
- Migración `20260721044743_move_contact_details_to_patient`: traslado del
  correo electrónico y el contacto de emergencia del vínculo al paciente, con
  copia previa de los valores existentes.
- Migración `20260721051500_seed_patient_inactivity_config`: alta del valor por
  defecto del umbral de inactividad para toda organización existente, idempotente
  y respetuosa de un valor ya configurado.
- `prisma/seed.ts`: siembra de las reglas de negocio de alcance organizacional,
  empezando por el mismo umbral, para las organizaciones que se creen después de
  esa migración.
- `src/patients/` (nuevo): controladores de pacientes y del vínculo con
  profesionales, sus servicios, los objetos de transferencia con sus
  validaciones, el presentador que fija la forma de la respuesta y las
  constantes del módulo.
- `src/common/dates/calendar-date.ts` (nuevo): se extrajo a un único lugar la
  conversión entre fechas de calendario y los valores que la biblioteca de
  acceso a datos expone, que hasta ahora vivía dentro del servicio de ausencias,
  y se le agregaron el desplazamiento por meses —con ajuste al último día del
  mes destino— y el cálculo de edad. El servicio de ausencias pasó a consumirla.
- `src/audit/changed-fields.ts` (nuevo): cálculo de los campos efectivamente
  recibidos en una modificación, para el detalle de la traza de auditoría.
- `src/professionals/guards/professional-ownership.guard.ts` y
  `src/professionals/decorators/professional-id-param.decorator.ts` (nuevo): la
  verificación de que un profesional actúa sobre su propio registro se
  generalizó para leer el identificador del parámetro de ruta que la ruta
  declare, ya que en este módulo el parámetro principal es el paciente.
- `src/health-insurers/health-insurers.service.ts`: la validación de
  identificadores contra el catálogo de obras sociales se centralizó en el
  servicio del catálogo, del que ahora dependen tanto el módulo de
  profesionales, que registra cuáles acepta cada uno, como el de pacientes, que
  registra cuál cubre a cada paciente.
- `src/app.module.ts`, `postman/psique-backend.postman_collection.json` y
  `CLAUDE.md`: registro del módulo nuevo, colección de pruebas de la API
  actualizada y convenciones incorporadas (fechas de calendario centralizadas,
  detalle de auditoría por nombre de campo, y restricciones de visibilidad más
  estrechas que la organización).

No se tocaron puertos ni adaptadores de integración: el módulo no consume
servicios externos.

## Tests y qué validan

- `test/patients-abmc.e2e-spec.ts` (nuevo, 29 pruebas): recorre el módulo por la
  capa HTTP con credenciales reales. Valida el alta por parte del personal
  administrativo y del proceso automatizado y su rechazo para el rol
  profesional; el rechazo con error de validación cuando falta cualquiera de los
  tres datos obligatorios de la reserva, cuando el contacto de emergencia llega
  vacío, cuando el documento llega con separadores, cuando la fecha no existe en
  el calendario y cuando un campo no nulable llega explícitamente nulo; el
  carácter opcional de la obra social y el rechazo con error de validación —no
  de servidor— de una obra social inexistente; la búsqueda por documento dentro
  de la organización, su aislamiento entre organizaciones y la admisión del
  mismo documento en otra; el conflicto ante un documento repetido; que un
  profesional sólo vea a sus pacientes y sólo su propio vínculo; el registro del
  indicador de primera sesión y su preservación al repetir la vinculación; la
  vinculación independiente con dos profesionales; el rechazo de la vinculación
  con un profesional de otra organización; la asignación de
  prioridad por el profesional tratante y su rechazo para otro profesional, para
  el proceso automatizado y para el personal administrativo de otra
  organización; el rango admitido de prioridad; la respuesta del punto de
  consulta de estado de datos en los tres casos relevantes —sin vínculo, con
  concurrencia reciente y con más de un año de ausencia— y su obediencia al
  umbral configurado por la organización; la modificación de datos filiatorios
  con su traza; la baja lógica, su invisibilidad en el listado, su recuperación
  explícita y la reactivación; el aislamiento entre organizaciones en
  modificación y baja; y que las observaciones de uso interno nunca aparezcan en
  la respuesta.
- `src/common/dates/calendar-date.spec.ts` (nuevo): ajuste al último día del mes
  al desplazar meses, incluido el caso de año bisiesto; conservación del día en
  el cruce de año; y cálculo de edad en el límite del cumpleaños, que es el
  mismo límite del que depende el filtro de edad configurado por el profesional.
- `src/audit/changed-fields.spec.ts` (nuevo): omisión de las propiedades que la
  petición no trajo y conservación del nulo explícito, que sí es un cambio.
- `src/professionals/guards/professional-ownership.guard.spec.ts` (modificado):
  se agregaron los casos de la ruta que declara un parámetro distinto del
  predeterminado.
- `test/seed.e2e-spec.ts` (modificado): la siembra crea la fila de configuración
  de la organización con el valor por defecto y no la sobrescribe cuando ya fue
  modificada.
- `test/patients-entities.e2e-spec.ts` (modificado): la prueba de vínculos
  independientes pasó a fijar los datos de contacto en el paciente, que es donde
  viven tras el traslado.
- Ejecución: suite unitaria en verde (12 suites / 64 pruebas) y suite end-to-end
  en verde (16 suites / 161 pruebas) tras aplicar las migraciones, más
  compilación del proyecto sin errores. Todos los datos usados son ficticios: nombres de
  fantasía y números de documento de ejemplo, sin contenido clínico.

## Figuras pendientes

- Se registró una figura pendiente con el mapa de endpoints del módulo
  Pacientes y los permisos por rol (ver `figuras_pendientes.md`).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-28-patients-abmc` (creada a partir de
  `main`). Sin commit al momento de redactar esta entrada: los cambios quedaron
  en el árbol de trabajo a la espera de autorización.
- Ticket: TASK-28 ("P2.2 – ABMC de Pacientes"). Depende de TASK-27 (entidades
  del dominio Pacientes), TASK-16 (autenticación y roles) y TASK-17
  (auditoría), todas ya fusionadas.
