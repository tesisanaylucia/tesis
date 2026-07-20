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
cada fila sobre un identificador fijo, las colecciones hijas de cada profesional
—matrículas y horarios— se reconcilian, eliminando toda fila que ya no figure en
la declaración del seed. La razón es concreta y surgió al implementar la tarea:
al cambiar el esquema de identificadores de las matrículas respecto del seed
anterior, una idempotencia basada solo en insertar o actualizar habría dejado
las matrículas viejas conviviendo con las nuevas, duplicando la colección de
cada profesional. Con la reconciliación, el seed converge siempre al estado
declarado, incluso después de que los propios datos del seed cambien.

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
