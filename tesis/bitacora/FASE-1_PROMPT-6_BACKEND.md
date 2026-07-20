# Fase 1 — Profesionales (backend) — Seed del plantel del piloto y cierre de la cobertura de pruebas (TASK-26)

## Qué se implementó

La última tarea de la fase cerró el módulo de Profesionales en dos frentes. Por
un lado, se reescribió el seed de datos de desarrollo para que reproduzca el
plantel real del piloto que declara la fuente de requisitos: cuatro psiquiatras
y una psicóloga. Cada profesional se carga con su especialidad, sus dos
matrículas —provincial y profesional—, una grilla semanal de horarios de
atención y la configuración base completa, que abarca tanto los parámetros de
agenda de P1.4 como la política de admisión y reasignación de P1.5. Por otro
lado, se completó la cobertura de pruebas end-to-end del módulo con los casos
que aún no estaban ejercitados.

Los datos del seed son ficticios y deliberadamente identificables como tales;
no contienen nombres, matrículas ni datos personales reales. Los datos legales
de la clínica se cargarán cuando el cliente los provea.

Un relevamiento previo de la cobertura existente mostró que la mayor parte de lo
que la tarea enumeraba ya estaba verificado por las pruebas escritas en P1.2:
el alta, la consulta, la edición y la baja lógica de profesionales, el tope de
tres matrículas, el aislamiento por tenant en la consulta individual y las
reglas de rol y propiedad. Se optó, por tanto, por no reescribir lo ya cubierto
y concentrar el esfuerzo en los huecos reales, que resultaron ser tres: la
edición y la eliminación de una matrícula, el rechazo por propiedad sobre las
tres rutas de matrículas, y el aislamiento por tenant del listado —complemento
de la garantía ya verificada sobre la consulta individual—.

## Decisiones y por qué

**El seed se diseñó convergente, no meramente no duplicante.** El requisito
pedía idempotencia, entendida como que una segunda ejecución no duplique
registros. Se adoptó una garantía más fuerte: además de insertar o actualizar
cada fila sobre un identificador fijo, se reconcilia todo lo que el seed creó en
una ejecución anterior y ya no declara, eliminándolo. La razón es concreta y
surgió al implementar la tarea: al cambiar el esquema de identificadores de las
matrículas respecto del seed anterior, una idempotencia basada solo en insertar
o actualizar habría dejado las matrículas viejas conviviendo con las nuevas,
duplicando la colección de cada profesional.

Cabe consignar que la primera versión de esta tarea aplicó la reconciliación
únicamente a las colecciones hijas —matrículas y horarios— y no a los
profesionales mismos, pese a que su documentación afirmaba una convergencia
general. La revisión posterior del módulo detectó el error: al retirar a un
profesional del plantel declarado, su registro permanecía en la base con el
indicador de actividad en verdadero mientras sus matrículas y horarios sí se
eliminaban, produciendo un profesional que el chatbot ofrecería sin
disponibilidad alguna —un estado peor que no haber reconciliado nada—. La
corrección incorporó la reconciliación de profesionales, deliberadamente
acotada al espacio de identificadores y al rango de secuencia propios del seed,
de modo que un profesional creado a través de la API durante el desarrollo nunca
resulte eliminado. La verificación se realizó plantando dos registros —uno
dentro del espacio de identificadores del seed y otro fuera— y comprobando que
la ejecución elimina el primero y preserva el segundo.

**Los identificadores fijos se derivan en lugar de escribirse literalmente.**
El seed anterior enumeraba identificadores completos uno por uno, lo que resulta
tolerable para cuatro profesionales con pocas matrículas pero se vuelve
impracticable al sumar una grilla semanal de horarios por profesional. Se
introdujo una función que compone el identificador a partir de un espacio de
nombres por entidad y un número de secuencia derivado del profesional, con lo
que cada fila conserva un identificador estable entre ejecuciones sin necesidad
de enumerarlo. La estabilidad es necesaria porque ninguna de estas tres
entidades tiene una clave de negocio natural sobre la que insertar o actualizar.

**La configuración base se distribuyó de forma deliberadamente heterogénea.** En
lugar de asignar a los cinco profesionales los mismos parámetros, se los varió a
lo largo del plantel: duraciones de consulta que reproducen los dos ejemplos de
la fuente de requisitos, franjas extra de una y dos horas, las tres modalidades
de ubicación de la franja extra, las dos modalidades de reasignación, y al menos
un profesional con la admisión de pacientes nuevos cerrada y varios con el filtro
de edad activo. El propósito es que una base recién sembrada ejercite todas las
ramas que los módulos de agenda y de conversación deberán manejar, en lugar de
un único caso homogéneo que ocultaría errores en las ramas no representadas.

**Se retiró el profesional inactivo que incluía el seed anterior.** Existía para
comprobar que la baja lógica lo excluye del listado. Se lo eliminó porque la
fuente de requisitos fija el plantel en cuatro psiquiatras y una psicóloga, y un
sexto registro inactivo lo contradiría; la garantía que aportaba no se pierde,
puesto que la prueba end-to-end de baja lógica crea su propio profesional y no
depende del seed. Se verificó previamente que ninguna prueba dependiera de los
datos sembrados.

**No se automatizó la verificación de idempotencia del seed.** Comprobarla en una
prueba exigiría ejecutar el seed contra la base de datos compartida de
desarrollo, que las pruebas end-to-end usan creando sus propias organizaciones
aisladas; hacerlo introduciría un efecto colateral sobre datos que las demás
pruebas no controlan. La idempotencia se verificó manualmente ejecutando el seed
dos veces consecutivas y contando las filas resultantes.

## Alternativas descartadas

- **Idempotencia por inserción o actualización sin reconciliación**: descartada
  porque no resiste un cambio en los propios datos del seed, situación que se
  produjo en esta misma tarea al renumerar los identificadores de las matrículas.
- **Reemplazo completo de las colecciones hijas en cada ejecución** (borrar todo
  e insertar de nuevo): descartada porque cambiaría los identificadores en cada
  corrida, perdiendo la estabilidad que el requisito pide explícitamente.
- **Reescribir la cobertura end-to-end del módulo desde cero**: descartada tras
  relevar que la mayor parte ya estaba verificada desde P1.2; se agregaron
  únicamente los casos faltantes, evitando duplicar pruebas equivalentes.
- **Sembrar un profesional inactivo adicional**: descartada por contradecir la
  composición del plantel que fija la fuente de requisitos.

## Entidades / puertos / adaptadores tocados

- Esquema de Prisma: **sin cambios**. La tarea es de datos y de pruebas.
- `prisma/seed.ts`: reescrito. Incorpora horarios de atención y configuración
  base al sembrado de profesionales, la función de derivación de
  identificadores, y la reconciliación de matrículas y horarios.
- `package.json`: se agregó el script de sembrado que los criterios de
  aceptación nombran, que delega en el comando de sembrado del ORM ya
  configurado.
- `test/professionals-abm.e2e-spec.ts`: ampliado con los casos faltantes.
- Colección de Postman: regenerada con el script del repositorio, sin cambios,
  al no haberse modificado ninguna ruta.

## Tests y qué validan

- `test/professionals-abm.e2e-spec.ts` (end-to-end, ampliado): se agregaron la
  edición y la eliminación de una matrícula por su titular —comprobando que la
  fila desaparece de la base y deja de reportarse junto al profesional—; el
  rechazo por falta de propiedad sobre las tres rutas de matrículas cuando un
  profesional intenta operar sobre las de otro; y el aislamiento por tenant del
  listado, verificando en ambas direcciones que ninguna organización ve
  profesionales de la otra.
- Verificación del seed: ejecutado dos veces consecutivas, la segunda corrida
  deja exactamente cinco profesionales, diez matrículas y veintidós horarios de
  atención, sin duplicados y con las matrículas del esquema de identificadores
  anterior correctamente reconciliadas.
- Ejecución: suites unitaria (9 suites / 38 tests) y end-to-end (12 suites / 77
  tests) completas en verde; `eslint` sin errores y verificación de tipos sin
  errores. Los datos usados son ficticios.

## Revisión del módulo y correcciones posteriores

Cerrada la tarea, se realizó una revisión integral del módulo que detectó tres
defectos, corregidos en el mismo branch.

**La consulta de profesionales no devolvía la grilla de horarios de atención.**
El dato existía y era accesible, pero únicamente a través del sub-recurso
dedicado, de modo que el motor de agenda y la capa conversacional —que necesitan
la grilla para determinar qué turnos ofrecer— habrían tenido que consultar el
listado y luego, por cada profesional, su grilla: un patrón de consulta N+1. Se
incorporó la grilla a la definición compartida de qué relaciones se cargan junto
al profesional, ordenada por día de la semana y hora de inicio para que los
consumidores puedan confiar en ese orden. El volumen es acotado —unos pocos
bloques semanales por profesional—, por lo que el costo de transportarla es
menor que el de la consulta adicional que evita. El sub-recurso dedicado se
mantiene, ya que sigue siendo la vía para reemplazar la grilla.

**La verificación de pertenencia cargaba relaciones que descartaba.** Los
servicios de matrículas, horarios y ausencias anclaban cada operación en el
profesional padre invocando el método que lo carga con todas sus relaciones,
cuando solo necesitaban la garantía de que existe y pertenece al tenant del
solicitante. El defecto era menor hasta que la corrección anterior lo agravó: al
sumar la grilla a esas relaciones, toda operación sobre una matrícula o una
ausencia habría pasado a cargar además la grilla semanal completa. Se incorporó
un método de verificación liviano, con la misma semántica de "no encontrado"
pero sin cargar relación alguna, y se lo adoptó en los ocho puntos donde la
carga completa no se utilizaba. Es un caso ilustrativo de cómo una corrección
puede convertir una ineficiencia latente en un problema real si no se revisa el
efecto conjunto.

**La colección de Postman contenía nombres de petición repetidos.** Cuatro
controladores comparten el segmento inicial de ruta y, por lo tanto, la misma
carpeta de la colección, mientras que el generador derivaba el nombre
únicamente del nombre del método del controlador; el resultado eran varias
peticiones homónimas ("Create", "Remove", "Find All") indistinguibles en la
interfaz. Se calificó el nombre con el recurso anidado, tomándolo del prefijo
del decorador de controlador y no de la ruta del método: esa distinción es la
que evita calificar erróneamente una acción que cuelga del propio profesional,
como la de configuración, que es un método del controlador principal y no un
recurso anidado. Además, el nombre pasó a regenerarse siempre en lugar de
conservarse del archivo existente —es un dato derivado, no una personalización
del usuario, y conservarlo habría impedido que la corrección se propagara a la
colección ya versionada—, mientras que las partes efectivamente redactadas a
mano (descripciones, cuerpos de ejemplo y scripts de prueba) se siguen
preservando. Por último, se agregó al generador una verificación que aborta si
alguna carpeta queda con nombres repetidos, para que esta clase de defecto no
vuelva a pasar inadvertida.

## Segunda revisión: defectos y riesgos de concurrencia

Una revisión posterior más exhaustiva, conducida por varios revisores
independientes con enfoques distintos —aislamiento multiusuario y autorización,
conformidad arquitectónica, correctitud funcional y calidad de las pruebas—
detectó defectos que las pruebas existentes no cubrían. Se documentan aquí por
pertenecer al mismo módulo y haberse corregido en el mismo branch.

**Un valor nulo explícito en cualquier modificación parcial producía un error de
servidor.** La biblioteca de validación empleada omite todas las restantes
validaciones de un campo cuando su valor es nulo, no solo cuando está ausente, y
la configuración de la tubería de validación descarta únicamente las propiedades
sin decorar. En consecuencia, un campo declarado opcional y enviado explícitamente
en nulo atravesaba la validación intacto y alcanzaba la capa de persistencia, que
lo rechazaba contra una columna no nulable con un error no controlado. El defecto
se reprodujo empíricamente sobre el servidor en ejecución antes de corregirlo.

La corrección no podía ser global: ese mismo comportamiento era, de hecho, la
única vía por la que podía restablecerse a nulo un parámetro de agenda ya
configurado —una capacidad que funcionaba por accidente, sin estar declarada en
los tipos ni cubierta por pruebas—. Suprimir el nulo indiscriminadamente la
habría eliminado en silencio. Se introdujeron por ello dos decoradores que
obligan a cada campo opcional a declarar cuál de los dos casos es: uno rechaza el
nulo, para las columnas no nulables, y otro lo admite con el significado
explícito de "restablecer el valor", para las nulables.

**Dos invariantes que abarcan una lectura y una escritura no estaban protegidos
frente a la concurrencia.** El nivel de aislamiento por omisión de la base de
datos permite que dos peticiones simultáneas lean un estado que autoriza la
escritura y ambas escriban, produciendo un estado que ninguna de las dos habría
admitido. En el reemplazo de la grilla de horarios, la segunda transacción
eliminaba filas que la primera ya había borrado y luego insertaba las suyas, con
lo que persistía la unión de ambas grillas —solapada, es decir, exactamente el
invariante que la validación de solapamientos existe para impedir, y sin ninguna
restricción de base de datos que lo detectara después—. En el tope de tres
matrículas, dos altas simultáneas leían el mismo recuento y ambas insertaban.
Ambos casos se resolvieron ejecutando la operación completa bajo aislamiento
serializable, de modo que la transacción perdedora aborta y el conflicto se
informa como tal en lugar de corromper los datos en silencio.

**Las entradas de auditoría se escribían fuera de la transacción de la mutación
que describen.** Una mutación que se confirmaba y una auditoría que fallaba a
continuación dejaban un cambio sin registrar, y en el caso de las eliminaciones,
sin posibilidad de reconstruir quién lo efectuó. Dado que la traza de auditoría
responde a una obligación legal (Ley 25.326), el servicio de auditoría pasó a
admitir el identificador de una transacción en curso, y todas las mutaciones del
módulo lo transmiten, de forma que el registro se confirma o se revierte junto
con el cambio.

**La baja lógica de un profesional no revocaba su acceso.** La baja conserva el
registro con su indicador de actividad en falso, pero la cuenta de usuario
vinculada permanecía intacta, la autenticación no verificaba el estado del
profesional y la validación del token no consulta la base de datos. Un
profesional desvinculado podía por tanto seguir autenticándose y editando su
propia grilla de horarios, que es la que el motor de turnos consulta. Se
incorporó la verificación en el inicio de sesión; los tokens ya emitidos siguen
siendo válidos hasta su expiración, limitación que se deja consignada.

**Incoherencia en el contrato de la interfaz.** El alta de un profesional
aceptaba sus matrículas bajo una clave en castellano mientras todas las
respuestas las devolvían bajo una clave en inglés. Se unificó en inglés, por ser
la convención efectivamente vigente: el cuerpo JSON está íntegramente en inglés
y solo las rutas están en castellano.

**Documentación de arquitectura divergente del código.** La revisión constató que
la separación en capas de dominio, aplicación e infraestructura que declaraba el
documento de convenciones del repositorio no la implementa ningún módulo: el
directorio de dominio contiene únicamente puertos y el de infraestructura
únicamente adaptadores. La mitad del patrón que sí se aplica de forma consistente
es la de puertos y adaptadores en los límites de integración externa. Se optó por
corregir el documento en lugar de reestructurar el código, por dos razones: con
una única tecnología de persistencia y sin previsión de sustituirla, introducir
entidades de dominio y sus conversores sería costo sin beneficio; y un documento
que describe una realidad inexistente induce a ignorarlo por completo, incluidas
las prescripciones que sí importan, como el acotamiento por organización.

## Verificación adicional

Además de las suites automatizadas, se levantó el servidor contra la base
sembrada y se ejercitó la API con credenciales reales: la consulta de
profesionales devuelve los cinco registros del plantel con sus dos matrículas,
su grilla de horarios en el orden esperado y su configuración base, sin exponer
el identificador de organización.

## Figuras pendientes

- Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-26-professionals-seed-and-tests` (creada a
  partir de `main`, con P1.1 a P1.5 ya fusionados).
- Ticket: TASK-26 ("P1.6 – Seed y tests del módulo Profesionales"). Depende de
  TASK-21 a TASK-25. Deja fuera de alcance los datos reales de la clínica, que
  se cargarán cuando el cliente los provea, y el sembrado de pacientes
  preexistentes, correspondiente a P2.6 (TASK-32) y P2.7 (TASK-33).
