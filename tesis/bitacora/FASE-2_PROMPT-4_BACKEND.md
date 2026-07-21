# Fase 2 — Pacientes (backend) — Consentimiento de tratamiento de datos, Ley 25.326 (TASK-30)

## Qué se implementó

Se incorporó el registro y la verificación del consentimiento del paciente para
el tratamiento de sus datos personales, exigido por la Ley 25.326. La entidad
correspondiente ya había sido modelada en la primera tarea de la fase, de modo
que lo que aporta esta tarea es el comportamiento: un punto de acceso que
registra la aceptación con la fecha y la hora del servidor, otro que informa si
el consentimiento está registrado y cuándo se otorgó, y un servicio interno de
verificación que la capa conversacional consultará antes de tomar una reserva.

Los requisitos enuncian la regla en una sola frase: el sistema guarda el
consentimiento con fecha y hora, y solo lo solicita si no se registró
previamente su aceptación. De esa frase se desprenden las dos propiedades que la
implementación debía garantizar: que quede constancia temporal del acto y que no
se vuelva a pedir una vez otorgado.

## Decisiones y por qué

**El registro es idempotente y responde con el estado, no con la creación.** El
punto de acceso que registra la aceptación devuelve el consentimiento ya
existente cuando lo hay, sin agregar una segunda fila, y responde con el código
de éxito ordinario en lugar del de creación. La razón es la misma que ordenó la
vinculación con un profesional en la tarea previa: quien más necesita esta
operación es el asistente conversacional en medio de una conversación, que no
puede saber si el paciente aceptó meses atrás, y volver a preguntárselo es
exactamente lo que los requisitos prohíben. Una respuesta que describe el estado
del consentimiento —si está aceptado y desde cuándo— permite además que el
cliente lea un único contrato tanto al registrar como al consultar.

**La condición de "no duplicar" se resolvió con aislamiento serializable y no
con una restricción de unicidad.** Comprobar que no hay aceptación previa y
recién entonces insertar es una lectura seguida de escritura, y bajo el nivel de
aislamiento predeterminado dos invocaciones concurrentes leerían ambas la
ausencia e insertarían ambas. La alternativa habitual, una restricción de
unicidad que la base rechazara, no es aplicable aquí: la tabla es de solo
agregado por diseño y debe admitir una segunda fila cuando la fase de
cumplimiento incorpore la revocación, de modo que una restricción que impidiera
la segunda fila impediría también aquello para lo que la tabla fue concebida. Se
utilizó, en consecuencia, el mecanismo ya adoptado en el repositorio para esta
clase de invariante: la comprobación y la inserción se ejecutan en una única
transacción serializable, y la invocación perdedora reintenta sobre la rama
idempotente.

**La marca temporal es la del servidor, y quedó reducida a una sola columna.**
El punto de acceso no admite cuerpo: lo único que registra es una aceptación
fechada en el instante de la escritura. Aceptar una fecha informada por el
cliente sería admitir una afirmación sobre cuándo se prestó el consentimiento sin
nada que la respalde, siendo que este registro es precisamente el que la clínica
exhibiría ante un organismo de control.

La primera tarea de la fase había modelado la entidad con dos marcas temporales,
la fecha del acto y la de creación de la fila, previendo un consentimiento
firmado en papel e incorporado con posterioridad conservando su fecha real.
Consultada la clínica durante esta tarea, se estableció que ese caso no existe:
todo paciente, tanto los ya cargados como los nuevos, presta el consentimiento a
través del asistente conversacional. Desaparecido el único escenario que las
distinguía, ambas columnas sólo podían contener el mismo valor, y una segunda
copia de un instante es una copia que puede discrepar sin que consulta alguna lo
detecte —el mismo argumento con el que el esquema rechaza replicar el
identificador de organización—. Se eliminó por tanto una de las dos mediante una
migración, junto con el índice que la usaba.

La que se conservó es la de creación de la fila, que es la que todo el resto del
esquema ya lleva y cuyo valor por defecto proviene del reloj de la base de datos.
Se evaluó conservar en cambio la columna con nombre de dominio propio, por su
trazabilidad al diagrama entidad-relación, y se descartó: el alta de la fila es
el momento del consentimiento, de modo que un nombre distinto para decir lo mismo
sólo agrega vocabulario, y la convención compartida con las demás entidades
resulta más simple de sostener. El nombre de dominio quedó registrado como
comentario sobre la columna, que es donde el proyecto mantiene el glosario. La
eliminación no pierde información: por construcción ambas columnas contenían el
mismo valor.

**El estado del consentimiento es la entrada más reciente, derivada en un único
lugar.** Como la tabla es de solo agregado, no existe una bandera que consultar:
la respuesta se deriva ordenando por la fecha del acto y tomando la última
entrada, ordenamiento que el índice ya existente resuelve. Esa derivación se
ubicó en la función de presentación que
ambos puntos de acceso utilizan, y el servicio de verificación lee de allí su
resultado en lugar de recalcularlo: así, la respuesta que ve el cliente y la
compuerta que consultará el asistente conversacional no pueden discrepar sobre
qué significa que el consentimiento esté aceptado.

**La revocación quedó explícitamente fuera de alcance, pero el comportamiento la
contempla.** Los derechos de acceso, rectificación y supresión corresponden a la
fase de endurecimiento y cumplimiento. No obstante, la lógica implementada no
supone que la última entrada sea siempre una aceptación: si lo último registrado
fuese una revocación, el estado informado es "sin consentimiento" y una nueva
aceptación se registra como acto nuevo, sin que la rama idempotente la absorba.
La decisión evita que la tarea futura deba rehacer esta lógica, y quedó cubierta
por pruebas para que no se pierda.

**El recurso se modeló en singular.** Aunque la tabla acumule entradas, lo que un
cliente pregunta es siempre si el paciente consiente y desde cuándo, de modo que
se definió una única dirección y ninguna entrada individual resulta
direccionable. No existe caso de uso que requiera dirigirse a una entrada
concreta, y exponer el identificador de cada fila habría sugerido operaciones de
modificación o borrado que contradicen el carácter de solo agregado.

**Los permisos se repartieron según de quién es el acto.** El registro quedó en
manos del personal administrativo y del proceso automatizado, que es donde el
paciente efectivamente lo presta; se excluyó al rol profesional, porque el
consentimiento es acto del paciente y el clínico no tiene nada que declarar en su
nombre. La consulta, en cambio, está abierta a todos los roles, con la
restricción ya vigente en el módulo: un profesional alcanza únicamente a los
pacientes que atiende, y para los demás el consentimiento responde lo mismo que
el paciente, es decir que no existe.

**El anclaje en el paciente se factorizó en lugar de duplicarse.** El
consentimiento no lleva identificador de organización, por tratarse de una
entidad con un único padre que ya lo determina, de modo que ninguna fila se
alcanza por su identificador: toda operación se ancla en el paciente, que el
servicio correspondiente resuelve dentro de la organización de quien consulta.
Para el camino de lectura hacía falta además la restricción por profesional
tratante, que la verificación existente no aplicaba. Se agregó una segunda
verificación junto a la anterior y se extrajo a un método común la consulta y el
mensaje de error que ambas comparten, de manera que la diferencia entre ellas sea
únicamente el filtro de alcance y no dos formulaciones distintas de la misma
respuesta.

## Alternativas descartadas

- **Una restricción de unicidad en la base para impedir el consentimiento
  duplicado**: descartada porque impediría también la revocación y las
  aceptaciones posteriores, que son el motivo por el cual la tabla es de solo
  agregado.
- **Admitir la fecha del consentimiento en el cuerpo de la solicitud**:
  descartada porque convertiría el registro legal en un dato de captura manual
  sin respaldo, y porque no existe caso de uso que lo requiera una vez
  establecido que todo consentimiento se presta por el asistente conversacional.
- **Conservar las dos marcas temporales por si el consentimiento en papel
  apareciera más adelante**: descartada por sostener una réplica sin caso de uso
  que la justifique. Si ese escenario surgiera, la columna puede reintroducirse
  con la migración correspondiente, mientras que mantener dos valores idénticos
  entretanto sólo ofrece la posibilidad de que discrepen.
- **Conservar la columna con nombre de dominio propio en lugar de la de creación
  de la fila**: descartada en favor de la convención que el resto del esquema ya
  sigue, dado que ambas designan el mismo instante; el término del diagrama
  entidad-relación se preservó como comentario del glosario sobre la columna.
- **Una bandera de consentimiento sobre el paciente**: descartada porque
  destruiría el historial de qué se aceptó y cuándo, que es lo que una norma
  sobre rendición de cuentas exige conservar.
- **Exponer el consentimiento como colección con entradas direccionables**:
  descartada por no existir caso de uso que lo requiera y por sugerir
  operaciones incompatibles con una tabla de solo agregado.
- **Un puerto de dominio para la verificación que consumirá la capa
  conversacional**: descartada por el mismo criterio ya aplicado en la tarea
  anterior, dado que la dirección de la dependencia no lo justifica.

## Entidades / puertos / adaptadores tocados

- `src/patients/consents.service.ts` (nuevo): registro idempotente de la
  aceptación en transacción serializable, consulta del estado y verificación
  para la capa conversacional.
- `src/patients/consents.controller.ts` (nuevo): los dos puntos de acceso
  anidados bajo el paciente, con sus permisos por rol.
- `src/patients/patient.presenter.ts`: tipo de respuesta del consentimiento y
  derivación de su estado a partir de la entrada más reciente.
- `src/patients/patients.service.ts`: verificación de alcance para caminos de
  lectura, y unificación de ambas verificaciones de pertenencia en un método
  común.
- `src/patients/patients.module.ts`: registro del servicio y del controlador
  nuevos, y exposición del servicio de consentimiento para la fase conversacional.
- `prisma/schema.prisma` y migración `consent_single_timestamp`: eliminación de
  la segunda marca temporal, que quedó sin caso de uso, y traslado del índice de
  lectura a la columna que se conserva.
- `test/patients-entities.e2e-spec.ts`: las pruebas de la entidad, escritas en la
  primera tarea de la fase, se ajustaron a la única marca temporal.
- `postman/psique-backend.postman_collection.json`: colección de pruebas de la
  API actualizada con los dos puntos de acceso incorporados.

La entidad y su índice ya habían sido incorporados en la primera tarea de la
fase; el único cambio de esquema es la eliminación de la columna redundante
descripta más arriba. No se tocaron puertos ni adaptadores de integración.

## Tests y qué validan

- `src/patients/consents.service.spec.ts` (nuevo, 10 pruebas): que la aceptación
  se escriba sin marca temporal alguna, dejándola al valor por defecto de la
  columna y no a un valor externo, que la traza
  de auditoría se emita dentro de la misma transacción que la escritura, que una
  segunda invocación devuelva la aceptación existente sin escribir ni auditar
  nada, que una aceptación posterior a una revocación sí se registre, que el
  estado se derive de la entrada más reciente, y que la verificación responda
  correctamente ante una aceptación, ante una revocación y ante la ausencia de
  registro.
- `test/patient-consent.e2e-spec.ts` (nuevo, 11 pruebas): por la capa HTTP y con
  credenciales reales de cada rol, el registro con fecha y hora comprendida entre
  la solicitud y su respuesta, su presencia efectiva en la base como fila única,
  la traza de auditoría referida al paciente, el registro por parte del proceso
  automatizado, su rechazo para el rol profesional, la idempotencia comprobada
  tanto en la respuesta como en el recuento de filas y de entradas de auditoría,
  el flujo completo en el que el sistema deja de solicitar el consentimiento una
  vez otorgado, la lectura por parte del profesional tratante, su ocultamiento
  para un profesional ajeno y para otra organización, y el rechazo de la
  operación sin credenciales.
- Ejecución: suite unitaria en verde (14 suites / 85 pruebas), suite end-to-end
  en verde (18 suites / 190 pruebas), compilación del proyecto sin errores y
  análisis estático sin advertencias. Todos los datos usados son ficticios y no
  contienen información clínica.

## Figuras pendientes

- Se registró una figura pendiente con el diagrama de secuencia del
  consentimiento —verificación previa, solicitud únicamente cuando no hay
  registro, y comportamiento idempotente ante una aceptación ya existente— (ver
  `figuras_pendientes.md`).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-30-consent` (creada a partir de `main`).
  Sin commit al momento de redactar esta entrada: los cambios quedaron en el
  árbol de trabajo a la espera de autorización.
- Ticket: TASK-30 ("P2.4 – Consentimiento Ley 25.326"). Depende de TASK-27
  (entidades del dominio Pacientes) y TASK-28 (ABMC de pacientes), ambas ya
  fusionadas. La integración al flujo conversacional corresponde a TASK-50 y los
  derechos de acceso, rectificación y supresión a TASK-67; ninguna forma parte
  del alcance de esta tarea.
