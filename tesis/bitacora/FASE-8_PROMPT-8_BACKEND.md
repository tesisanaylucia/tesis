# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — CORS explícito y helmet, cabeceras de seguridad HTTP (TASK-88, P8.1c, corrección/extensión a P8.1)

## Qué se implementó

Se implementó P8.1c ("CORS explícito y helmet"), una tarea de corrección
sobre P8.1 (TASK-66, seguridad): el punto de entrada de la aplicación
(`main.ts`) sólo configuraba el pipe de validación global y llamaba a
`app.listen`, sin ninguna cabecera de seguridad HTTP ni política CORS
propia — dependía por completo de los valores por defecto de Express/Nest,
que no incluyen cabeceras como `X-Frame-Options`/`X-Content-Type-Options` ni
restringen en absoluto qué origen puede leer una respuesta desde el
navegador.

Se agregó la dependencia `helmet`, montada antes que cualquier otra
configuración del framework, y una política CORS explícita
(`app.enableCors`) con una lista blanca de orígenes permitidos, leída de una
variable de entorno nueva (`CORS_ALLOWED_ORIGINS`, lista separada por
comas) en lugar del valor por defecto permisivo (`origin: '*'`) — decisión
obligada, no opcional, porque cada ruta de la API ya viaja con un JWT en la
cabecera `Authorization` (ver la sección de Seguridad de `CLAUDE.md`), y un
origen sin restricción podría leer una respuesta autenticada desde
cualquier script de terceros.

## Decisiones y por qué

**El montaje de helmet y CORS se extrajo a una función propia
(`applyHttpSecurity`), separada de `main.ts`, para que la prueba de extremo
a extremo pudiera aplicar exactamente la misma configuración a su propia
instancia aislada de la aplicación en lugar de reconstruirla a mano.** El
resto de las pruebas de extremo a extremo del proyecto arman su instancia
de Nest directamente sobre `AppModule` (sin pasar por el `bootstrap()` real
de `main.ts`, que Nest nunca ejecuta en ese camino), así que sin esta
extracción la única forma de probar el comportamiento real de CORS/helmet
habría sido reescribir su configuración dentro del archivo de prueba —con
el riesgo real de que ambas copias divergieran silenciosamente con el
tiempo—. Es el mismo razonamiento de "un único lugar, reutilizado en todas
partes" que ya sostienen `changedFields` (`src/audit/changed-fields.ts`) o
`latestConsultationDate` (`src/patients/patient.presenter.ts`) para otras
decisiones de negocio.

**La variable `CORS_ALLOWED_ORIGINS` es obligatoria al arrancar
(`ConfigService.getOrThrow`), con el mismo patrón de "fallar rápido" que ya
usan `JWT_SECRET`, `OPENAI_API_KEY` o `ACCESS_CODE_ENCRYPTION_KEY`, y su
propio parser (`parseCorsAllowedOrigins`) rechaza explícitamente el valor
`'*'` si aparece en la lista, incluso mezclado con orígenes reales.** No se
dejó como una variable opcional con una lista vacía por defecto: una lista
vacía silenciosa habría bloqueado a cualquier navegador sin ninguna señal
de arranque que lo explicara, mientras que fallar al iniciar el proceso
hace que un despliegue mal configurado sea imposible de pasar por alto en
lugar de sólo desalentado por convención.

**Un origen no permitido no se responde con un error explícito, sino
simplemente sin la cabecera `Access-Control-Allow-Origin`.** CORS no es un
mecanismo de bloqueo del lado del servidor: es el navegador quien se niega
a exponerle la respuesta al script que la pidió cuando esa cabecera falta o
no coincide con su propio origen. La petición HTTP en sí misma sigue
completándose normalmente contra el backend (algo que un cliente que no es
un navegador, como la propia aplicación móvil o una llamada servidor a
servidor, nunca ve restringido, ya que ninguno de los dos envía cabecera
`Origin`); replicar ese comportamiento —y no, por ejemplo, cortar la
petición con un error 403 propio— es la única forma de que el mecanismo
sea indistinguible del que exige la especificación CORS misma.

**El campo `origin` de `enableCors` se implementó como una función que
refleja el origen exacto de la petición cuando está en la lista, en lugar
de pasarle el arreglo completo de orígenes permitidos directamente a la
librería subyacente.** Con más de un origen permitido, la biblioteca `cors`
sólo sabe reflejar un origen fijo o delegar la decisión a una función — y
una función es además el único punto de extensión natural para el caso
señalado en el propio ticket como fuera de alcance: si en el futuro cada
organización necesita su propio dominio de frontend, la lista blanca hoy
global podría resolverse por organización dentro de esa misma función, sin
cambiar la forma en que se conecta al resto de la aplicación.

## Entidades / puertos / adaptadores tocados

Ninguna entidad de dominio ni tabla nueva — es exclusivamente
configuración de infraestructura HTTP, sin relación con el esquema de base
de datos.

- `package.json`: dependencia nueva `helmet`.
- `src/common/http/cors-allowlist.ts` (nuevo): `parseCorsAllowedOrigins`
  (parseo y validación de la variable de entorno) y `buildCorsOptions`
  (construcción de las opciones de CORS con la función de origen).
- `src/common/http/apply-http-security.ts` (nuevo): `applyHttpSecurity`,
  el punto único que monta helmet y CORS sobre una instancia de Nest.
- `src/main.ts`: `bootstrap()` llama a `applyHttpSecurity` antes del pipe
  de validación global y antes de `app.listen`.
- `.env.example` / `.env`: variable nueva `CORS_ALLOWED_ORIGINS`,
  documentada con su propio razonamiento.
- `CLAUDE.md`: sección de Seguridad, viñeta nueva describiendo el
  mecanismo, junto a la nota de multi-tenancy fuera de alcance.

## Tests

- `src/common/http/cors-allowlist.spec.ts` (nuevo): parseo (separación,
  recorte de espacios, entradas vacías descartadas, lista vacía rechazada,
  comodín `'*'` rechazado incluso junto a orígenes reales) y la función de
  origen en sí (permite un origen de la lista, deniega uno ausente sin
  error, permite una petición sin cabecera `Origin`), ejercitada
  directamente sin servidor HTTP real.
- `test/security-headers.e2e-spec.ts` (nuevo): las dos aserciones
  literales del ticket contra una instancia real de la aplicación —
  cabeceras estándar de helmet presentes en una respuesta real
  (`X-Content-Type-Options`, `X-Frame-Options`, `X-DNS-Prefetch-Control`,
  ausencia de `X-Powered-By`), y comportamiento de CORS de punta a punta:
  `Access-Control-Allow-Origin` presente y reflejado para un origen
  permitido, ausente para uno no permitido, ausente también para una
  petición sin cabecera `Origin`, y el mismo contraste repetido sobre el
  preflight (`OPTIONS`): 204 con cabeceras CORS para un origen permitido
  frente a la ausencia de esas cabeceras — y el consecuente 404 al no
  existir ninguna ruta que atienda `OPTIONS` — para uno no permitido.

Suite completa en verde al cierre: 78 suites unitarias (772 pruebas) y 51
suites de extremo a extremo (557 pruebas) contra PostgreSQL real,
verificación combinada de cobertura (129 suites/1329 pruebas) sin bajar del
umbral, lint y verificación de tipos sin errores.

## Figuras pendientes

Una nueva — ver `figuras_pendientes.md`: diagrama de la cadena de
middleware HTTP montada en el arranque (helmet → CORS por función de
origen contra la lista blanca → pipe de validación global → enrutamiento),
señalando la rama de origen no permitido.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-88-cors-helmet-security-headers`,
  creada desde `main` para esta tarea. Sin commitear al cierre — pendiente
  de autorización explícita de la autora antes de commitear/pushear, según
  lo indicado para esta sesión.
