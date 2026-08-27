# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Empaquetado y despliegue con Docker (TASK-69, P8.4)

## Qué se implementó

Se implementó P8.4 ("Empaquetado y despliegue") del SRS: (1) una imagen
Docker del backend NestJS de dos etapas, sin secretos incluidos; (2) un
`docker-compose` de producción/staging con tres servicios (base de datos,
un contenedor de una sola ejecución que aplica las migraciones de Prisma,
y el backend), con volumen persistente y red interna; (3) una guía de
despliegue paso a paso pensada para que un tercero sin conocimiento previo
del código pueda levantar el sistema en un servidor Linux desde cero; (4)
documentación —no implementación— de una estrategia de backup periódico de
la base de datos, y de que el Gateway WiFi G2 de TTLock es un dispositivo
físico que no forma parte de este stack de contenedores.

## Decisiones y por qué

**El `docker-compose` de producción es un archivo distinto del ya
existente, no un reemplazo.** El repositorio ya tenía un `docker-compose.yml`
que solo levanta Postgres para el ciclo de desarrollo local (`npm run
start:dev` corre el backend fuera de un contenedor, para conservar la
recarga rápida al editar). Convertir ese mismo archivo en el stack
completo de producción habría forzado a reconstruir la imagen del backend
en cada guardado durante el desarrollo diario, o a mantener dos
configuraciones de servicios superpuestas dentro de un mismo archivo. Se
optó por un segundo archivo, `docker-compose.prod.yml`, que se ejecuta de
forma independiente en el servidor del cliente y nunca se combina con el
de desarrollo — ambos declaran un servicio `db`, pero sirven a entornos
que jamás corren al mismo tiempo en la misma máquina.

**La imagen se construye en dos etapas para que la etapa final no
contenga herramientas de compilación ni el código fuente en TypeScript.**
La primera etapa instala todas las dependencias —incluidas las de
desarrollo, porque tanto el compilador de TypeScript como el propio
cliente de línea de comandos de Prisma son dependencias de desarrollo—,
genera el cliente de Prisma y compila el proyecto; luego elimina las
dependencias de desarrollo del árbol de `node_modules` sin perder el
cliente de Prisma ya generado, que vive dentro de una dependencia de
producción (`@prisma/client`) y no se ve afectado por esa poda. La segunda
etapa parte de una imagen base limpia y copia únicamente ese
`node_modules` ya podado, el código compilado y el esquema de Prisma —
nunca el código fuente ni las dependencias de desarrollo—, y corre con un
usuario sin privilegios de administrador en lugar del usuario raíz que la
imagen base usa por defecto. El contenedor que aplica las migraciones en
`docker-compose.prod.yml` es la única excepción que sigue necesitando la
etapa intermedia completa, porque es la única tarea de todo el stack de
producción que todavía necesita el cliente de línea de comandos de
Prisma, ausente a propósito de la etapa final.

**Se encontró y corrigió un problema real de configuración al validar el
despliegue de punta a punta, no asumiendo que la imagen construida
funcionaría por el solo hecho de compilar sin errores.** El proyecto no
tenía fijado dónde debía quedar la raíz de la compilación de TypeScript,
de modo que el compilador, al encontrar archivos tanto dentro de la
carpeta de código fuente como fuera de ella (el script de carga inicial
de datos, ejecutado por una herramienta distinta que no pasa por esta
compilación), calculaba esa raíz como el directorio del proyecto entero
en lugar de la carpeta de código fuente, y el archivo de entrada de la
aplicación terminaba compilado en un directorio dentro del directorio de
salida en lugar de en su raíz. El efecto práctico, invisible hasta
efectivamente intentar arrancar la aplicación ya compilada, era que el
comando que la propia aplicación ya declaraba para iniciar en producción
—y el que este Dockerfile reproduce dentro del contenedor— fallaba con un
error de módulo no encontrado. La compilación en sí nunca fallaba,
así que ni la integración continua ni ningún desarrollo local lo habían
detectado hasta ahora, porque nadie había ejecutado antes ese comando de
inicio de producción de punta a punta. Se corrigió fijando esa raíz
explícitamente a la carpeta de código fuente y excluyendo de esa
compilación en particular el script de carga inicial de datos —que ya se
ejecuta por su cuenta con una herramienta distinta, sin pasar por el
código compilado—, sin afectar a ese script ni a ningún otro flujo
existente.

**Ninguna variable de entorno crítica queda con un valor por defecto en
el archivo de `docker-compose` de producción.** Cada secreto que el
código de la aplicación ya trata como obligatorio al arrancar recibe, en
este archivo, la sintaxis de variable de entorno requerida sin valor de
respaldo, en lugar de la sintaxis que provee uno: si falta alguna en el
archivo de configuración del servidor, `docker compose up` se detiene de
inmediato nombrando exactamente cuál falta, en lugar de arrancar con un
secreto vacío o de ejemplo silenciosamente incorporado a la imagen o al
contenedor en ejecución.

**Las credenciales de TTLock no se agregaron como variables de entorno,
ni siquiera como ejemplo.** El propio archivo de variables de entorno del
proyecto ya documentaba, de una tarea anterior, que esas credenciales son
propias de cada organización y se leen de la base de datos, no del
entorno del proceso — una decisión de arquitectura multi-tenant tomada
mucho antes de esta tarea. Agregarlas aquí solo para completar una lista
hubiera introducido una inconsistencia con esa decisión ya documentada,
así que la guía de despliegue explica en cambio por qué no aparecen y qué
hacer si en el futuro una misma instancia sirve a más de una organización
— nada cambia en el despliegue en ese caso, porque el archivo de entorno
ya es compartido entre organizaciones para el resto de la configuración
global, y las credenciales de la cerradura seguirían viviendo en la base
de datos, una por organización, como ya ocurre hoy.

**La estrategia de backup se documentó, no se implementó.** El alcance de
la tarea explícitamente pide documentar, no construir, un mecanismo de
backup automático — coherente con que el propio código del sistema ya
señalaba, desde la tarea de retención de datos anterior, que un trabajo
de archivado automático todavía no existe y queda para una tarea futura.
La guía de despliegue describe dos mecanismos concretos y ejecutables a
mano o por una tarea programada del sistema operativo del servidor —una
copia del volumen de Docker en frío, y un volcado lógico periódico en
caliente sin detener el sistema—, sin agregar código ni un servicio
adicional al `docker-compose`.

## Entidades / servicios tocados

Ninguna. La tarea es enteramente de empaquetado, configuración y
documentación — no se tocó ningún módulo de dominio, servicio ni prueba
existente.

- `Dockerfile` (nuevo, raíz del repositorio): build de dos etapas
  descripto arriba.
- `.dockerignore` (nuevo): excluye `.env`, `node_modules`, pruebas y
  archivos no necesarios en el contexto de build.
- `docker-compose.prod.yml` (nuevo): stack de producción/staging —
  base de datos, migraciones de una sola ejecución, backend.
- `docs/deployment.md` (nuevo): guía de despliegue paso a paso, nota
  sobre el Gateway WiFi G2, estrategia de backup, y el caso multi-tenant.
- `.env.example`: variables nuevas para las credenciales del contenedor
  de PostgreSQL, y una nota aclarando que la base de datos usa un nombre
  de host distinto según se ejecute en desarrollo local o dentro de
  `docker-compose.prod.yml`.
- `tsconfig.build.json`: corrección de la raíz de compilación descripta
  arriba — no es un cambio de comportamiento buscado por esta tarea, sino
  un defecto preexistente encontrado al validar el despliegue de punta a
  punta.
- `.gitignore`: se agregó el artefacto incremental de compilación de
  TypeScript, que no estaba ignorado.
- `README.md`, `CLAUDE.md`: sección nueva de despliegue en cada uno.

## Tests

No aplica una suite de pruebas automatizadas nueva — la validación de esta
tarea fue, en cambio, ejecutar el propio flujo de despliegue de punta a
punta contra Docker real, no solo revisar el código de los archivos de
configuración:

- Se construyó la imagen y se verificó su contenido (el `node_modules`
  final no incluye dependencias de desarrollo; el cliente de Prisma ya
  generado sí está presente).
- Se levantó el stack completo de producción desde cero
  (`docker compose up -d` sin estado previo): el contenedor de
  migraciones corrió y terminó con éxito antes de que el backend
  arrancara, y el endpoint de salud respondió con éxito.
- Se insertó una fila de prueba, se bajó el stack sin destruir volúmenes
  y se volvió a levantar: la fila siguió presente y las migraciones ya
  aplicadas no se reintentaron, confirmando el criterio de aceptación de
  persistencia de datos.
- Se corrió la suite unitaria completa y el linter tras la corrección de
  `tsconfig.build.json`, para confirmar que ese cambio no afecta ningún
  otro flujo: 75 suites (753 pruebas) en verde, linter sin errores.

## Figuras pendientes

Ninguna nueva — el diagrama de arquitectura general ya existente
(`arquitectura_general.png`) ya muestra el backend, la base de datos y el
Gateway WiFi G2 como componentes desplegados, y esta tarea no cambia esa
arquitectura, solo la empaqueta.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-69-docker-deployment`, creada desde
  `main` para esta tarea.
