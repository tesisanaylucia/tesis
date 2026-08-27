# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Suite de tests integral y cobertura (TASK-68, P8.3)

## Qué se implementó

Se implementó P8.3 ("Suite de tests integral y cobertura") del SRS, la
tarea que cierra el módulo de tests consolidando lo que las tareas de
extremo a extremo de cada módulo anterior ya habían construido por
separado: (1) configuración de cobertura de Jest con reporte en formato
`lcov` y texto en consola, y un umbral mínimo del 80% de líneas y ramas
sobre la capa de servicios de dominio, verificado en la integración
continua; (2) verificación (sin tests nuevos) de que el aislamiento
multi-tenant ya está probado en varios niveles distintos, documentando
cuál test cubre cada uno; (3) verificación (sin tests nuevos) de que las
pruebas de seguridad básicas — 401 sin token, 403 con rol equivocado,
límite de tasa sobre el inicio de sesión — ya existen; (4) verificación de
que la integración continua ejecuta la suite de extremo a extremo antes de
cualquier fusión a la rama principal, y su ampliación para que también
ejecute y verifique la cobertura.

## Decisiones y por qué

**El umbral de cobertura se mide sobre la suite completa (unitaria + de
extremo a extremo) en una sola corrida, no solo sobre la suite unitaria.**
Antes de decidir esto se midió la cobertura real de la capa de servicios
usando solo la suite unitaria: 76% de líneas, 67% de ramas — por debajo
del umbral pedido, y de forma reveladora, no por falta de pruebas sino
porque una parte considerable de los servicios de este sistema (el de
pacientes, el de profesionales, el de matrículas, el de ausencias, entre
otros) se valida exclusivamente mediante pruebas de extremo a extremo
contra PostgreSQL real, nunca con una prueba unitaria sobre un cliente de
base de datos simulado — una práctica ya establecida en tareas anteriores
y documentada como tal en las convenciones del repositorio. Medir el
umbral solo sobre la suite unitaria habría exigido escribir una cantidad
sustancial de pruebas unitarias nuevas únicamente para satisfacer un
número, sin agregar ninguna confianza real que la suite de extremo a
extremo ya existente no aportara. Se optó, en cambio, por unificar ambas
suites bajo una sola configuración de Jest con dos proyectos (antes
repartida entre una configuración incluida en `package.json` para las
pruebas unitarias y un archivo de configuración aparte para las de
extremo a extremo) y ejecutar ambos proyectos juntos exclusivamente para
medir la cobertura combinada — Jest combina de forma nativa la cobertura
de todos los proyectos que corrieron en una misma invocación en un solo
reporte, sin necesitar ninguna herramienta externa de fusión de reportes.
Con ambas suites combinadas la cobertura real de la capa de servicios
resultó 97% de líneas y 83% de ramas, ampliamente por encima del umbral.
Los dos comandos que ejecutan cada suite por separado para el trabajo
cotidiano (la corrida rápida solo unitaria, y la de extremo a extremo
contra PostgreSQL real) siguen existiendo sin cambios de comportamiento,
seleccionando cada uno su propio proyecto de Jest.

**El umbral se limita a la capa de servicios, y excluye a los adaptadores
de integración externa sin necesitar una lista de exclusión explícita.**
La especificación pide medir la cobertura de "los servicios de dominio" y
excluir explícitamente a los adaptadores de integraciones externas (la
cerradura inteligente, WhatsApp, el proveedor de inteligencia artificial),
que se prueban con dobles de prueba. Ambos límites coinciden exactamente
con una convención de nombres que el propio código ya sigue desde tareas
anteriores: cada servicio de dominio se llama `*.service.ts`, y cada
adaptador de integración externa vive en un directorio separado y se
llama `*.adapter.ts`. Se aprovechó esa coincidencia en lugar de mantener
una lista de archivos excluidos a mano: la cobertura se recolecta
únicamente sobre los archivos que terminan en `.service.ts`, de modo que
ningún adaptador entra nunca al cálculo sin que haga falta nombrarlo, y
la exclusión no puede desactualizarse si se agrega un adaptador nuevo en
el futuro.

**El umbral se aplica como un promedio agregado sobre toda la capa, no
archivo por archivo — una decisión que exigió corregir un supuesto
incorrecto sobre el propio motor de pruebas.** El mecanismo de Jest para
fijar un umbral sobre un subconjunto de archivos (una clave de
configuración que acepta un patrón en lugar de la palabra reservada que
fija el umbral general) no promedia la cobertura del conjunto de archivos
que ese patrón selecciona: la verificación real, confirmada ejecutando la
suite con un umbral así configurado, exige que **cada archivo
individual** que coincide con el patrón supere el umbral por separado —
un comportamiento distinto al que la documentación de la herramienta
sugiere a primera lectura, y que hubiera exigido llevar cada servicio
existente, uno por uno, por encima del 80%, en lugar de medir la
capa como un conjunto. Se resolvió sin necesidad de esa clave especial:
como la recolección de cobertura ya está restringida únicamente a los
archivos de servicio (ver la decisión anterior), la palabra reservada que
fija el umbral general pasó a operar, de hecho, como el promedio agregado
buscado sobre exactamente esa capa y ninguna otra.

**El aislamiento multi-tenant y las pruebas de seguridad básicas no
necesitaron una prueba nueva: la tarea consistió en verificar la cobertura
existente y dejarla documentada explícitamente, no en escribirla de
nuevo.** Se recorrió la suite de extremo a extremo buscando cada uno de
los casos que pide la tarea y se confirmó que ya existen, en capas
distintas y complementarias: el mecanismo de aislamiento en sí (una
prueba desde la tarea que introdujo la extensión de Prisma que filtra por
organización), el caso literal que pide la tarea — un paciente creado
bajo el token de una organización no es accesible con el token de otra —
(una prueba del módulo de pacientes), el aislamiento a nivel del motor de
turnos y lista de espera (una prueba de la tarea de integración de ese
motor), y el aislamiento en el propio webhook de WhatsApp cuando dos
organizaciones reutilizan el mismo número de teléfono configurado (una
prueba de las conversaciones de extremo a extremo del chatbot). De igual
manera, la respuesta 401 sin token ya tiene un caso canónico y varios
repetidos por módulo, la respuesta 403 por rol equivocado y el límite de
tasa sobre el inicio de sesión ya tienen su propia prueba dedicada, ambas
de la tarea de endurecimiento de seguridad anterior. No inventar una
prueba redundante donde ya existía una equivalente seguía el mismo
criterio que esa tarea anterior había aplicado sobre los guards de
autorización. Lo nuevo de esta tarea fue registrar, en un solo lugar de
la documentación del repositorio, cuál prueba cubre cada uno de estos
casos — antes esa cobertura existía pero no estaba señalada como tal en
ningún sitio.

**La integración continua ejecuta la suite de extremo a extremo dos veces
por corrida, una decisión deliberada y no un descuido.** La verificación
de cobertura, al necesitar ejecutar ambas suites combinadas para medir el
número agregado, vuelve a correr la suite de extremo a extremo además de
la corrida ya existente que la ejecuta sola. Se consideró evitar esa
repetición reutilizando el resultado de la primera corrida, pero se
descartó: mantener la corrida rápida y sin instrumentar como primera señal
de fallo, separada de la corrida con instrumentación de cobertura —más
lenta por la instrumentación en sí—, se valoró por encima del ahorro de
minutos de cómputo que evitar la repetición hubiera dado, sobre todo
porque este es un proyecto de tesis y no un sistema con una cadencia de
fusiones que vuelva ese costo significativo.

## Entidades / servicios tocados

- `jest.config.ts` (nuevo, raíz del repositorio): configuración única de
  Jest con dos proyectos (`unit`, `e2e`), reemplaza la configuración
  incluida en `package.json` y el archivo `test/jest-e2e.json` (eliminado).
  Define la recolección y el umbral de cobertura sobre `src/**/*.service.ts`.
- `package.json`: los comandos de prueba pasan a seleccionar su propio
  proyecto de Jest (`--selectProjects`); `test:cov` ejecuta ambos proyectos
  juntos con cobertura.
- `tsconfig.build.json`: excluye `jest.config.ts` de la compilación de
  producción, siguiendo el mismo patrón ya usado para excluir los archivos
  de prueba.
- `.github/workflows/ci.yml`: nuevo paso que ejecuta la suite combinada
  con cobertura después de los pasos ya existentes de pruebas unitarias y
  de extremo a extremo.
- `CLAUDE.md`: nueva sección de tests y cobertura, documentando la
  configuración, el porqué del umbral agregado sobre servicios, y qué
  prueba existente cubre cada caso de aislamiento multi-tenant y de
  seguridad básica que la tarea pedía verificar.
- `README.md`: se agregó `test:cov` a la tabla de comandos.

No se modificó ningún servicio de dominio ni ninguna prueba existente —
la tarea es enteramente de configuración y documentación.

## Tests

No se agregaron pruebas nuevas: la tarea verificó que la cobertura y los
casos de aislamiento/seguridad que pedía ya existían, y configuró la
medición de cobertura sobre la suite existente. Suite completa en verde al
cierre: 75 suites unitarias (753 pruebas) y 48 suites de extremo a extremo
(546 pruebas) contra PostgreSQL real — 1299 pruebas en total corriendo
juntas sin interferencia en una sola invocación de Jest —, cobertura de la
capa de servicios en 97% de líneas y 83% de ramas (umbral: 80% en ambas),
lint y verificación de tipos sin errores, compilación de producción sin
errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-68-test-suite-and-coverage`, creada
  desde `main` para esta tarea (ya incluye TASK-66 fusionada; TASK-67
  seguía en su propia rama sin fusionar al momento de crear esta). Sin
  commitear al cierre — pendiente de autorización explícita de la autora
  antes de commitear/pushear, según lo indicado para esta sesión.
