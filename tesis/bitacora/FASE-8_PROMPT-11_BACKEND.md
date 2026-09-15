# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Trust proxy para que el rate limiting sea realmente por IP del cliente (TASK-185, corrección a P8.1/TASK-66 y P8.1c/TASK-88)

## Qué se implementó

Se cerró un hallazgo de severidad alta de una auditoría de código contra el
anteproyecto de tesis y la especificación de requisitos (2026-08-28):
`docker-compose.prod.yml` documenta un despliegue detrás de un proxy inverso
(Nginx o Caddy) que termina TLS, pero en ningún lugar del código se
configuraba `trust proxy` de Express. `ThrottlerGuard` (el guard de
`@nestjs/throttler` que limita `POST /auth/login`) usa por defecto `req.ip`
para identificar al cliente, y sin `trust proxy`, `req.ip` resuelve siempre
a la dirección del proxy, no a la del cliente real. En la práctica, esto
colapsaba el límite "10 intentos por IP por minuto" del SRS en "10 intentos
por minuto para toda la clínica": cualquier cliente podía agotar el
contador compartido y bloquear el login de todos los usuarios reales con
diez peticiones. Se agregó `app.set('trust proxy', ...)` al mismo punto de
entrada donde ya se montaban helmet y CORS, configurable por variable de
entorno con un valor por defecto que asume la topología ya documentada.

## Decisiones y por qué

**El número de saltos de proxy confiados es configurable
(`TRUST_PROXY_HOPS`), con valor por defecto `1`, en vez de un `true` fijo o
un valor hardcodeado.** El propio hallazgo pedía "gatear por variable de
entorno" para el único despliegue donde el proxy documentado no está
presente (una prueba interna rápida contra el contenedor directamente, caso
que la propia guía de despliegue ya nombra). Un valor por defecto de `1` en
vez de `0`/sin configurar refleja que la topología recomendada y documentada
en `docs/deployment.md` — Nginx o Caddy siempre delante de `backend` — ya
asume exactamente un salto; dejar el valor por defecto en "sin proxy"
habría hecho que el caso común, no el caso raro, necesitara configuración
explícita. El valor se interpreta como un conteo de saltos, no como un
booleano, siguiendo el propio mecanismo de Express (`app.set('trust proxy',
<n>)`), para que un despliegue con más de un proxy intermedio pueda
ajustarlo sin cambiar de mecanismo.

**El parseo de la variable de entorno falla rápido ante un valor mal
formado, en vez de degradar silenciosamente a un valor por defecto
inseguro** — mismo criterio que ya seguía el validador de la lista blanca
de CORS (`parseCorsAllowedOrigins`) y el parseo de la clave de cifrado de
campo: un valor negativo o no entero se rechaza con un error explícito en
el arranque, en lugar de, por ejemplo, tratarlo silenciosamente como `0` y
dejar el propio hallazgo sin corregir sin que nadie lo note.

**La función `applyHttpSecurity` pasó a requerir `NestExpressApplication`
en vez del `INestApplication` genérico**, porque `app.set(...)` es un
método específico de la integración con Express, no parte de la interfaz
de aplicación agnóstica de plataforma que Nest expone por defecto. Esto
propaga el cambio de tipo a los tres lugares que construyen una instancia
de aplicación para pasarla a esa función — `main.ts` y los dos archivos de
prueba de extremo a extremo que ya reutilizaban `applyHttpSecurity`
(`security-headers.e2e-spec.ts`, y ahora también
`security-hardening.e2e-spec.ts`) — en vez de resolverlo con un cast
puntual dentro de la propia función, para que el tipo declarado siga
describiendo con precisión qué necesita la función para funcionar.

**Se reutilizó la prueba de extremo a extremo de rate limiting ya
existente, pasándola a través de `applyHttpSecurity` real, en vez de
probar `app.set('trust proxy', ...)` de forma aislada.** La alternativa más
simple —una prueba unitaria que verifique que Express resuelve `req.ip`
correctamente dado cierto valor de configuración— habría probado el
comportamiento de Express, no que el arranque real de la aplicación
efectivamente active ese comportamiento; siguiendo el mismo criterio que ya
documentaba `security-headers.e2e-spec.ts` ("las mismas funciones que llama
el `bootstrap()` de `main.ts`, para que la prueba no pueda desalinearse de
lo que corre en producción"), el bloque de rate limiting ahora monta la
aplicación de prueba con `applyHttpSecurity` real en vez de construir el
pipe de validación a mano únicamente, y prueba el aislamiento por cliente
enviando dos direcciones distintas por `X-Forwarded-For` contra el mismo
endpoint.

**Hallazgo colateral, corregido junto con esta tarea por tocar el mismo
archivo:** `docker-compose.prod.yml` nunca declaraba `CORS_ALLOWED_ORIGINS`
en el bloque `environment` de `backend`, a pesar de que `applyHttpSecurity`
la lee con `configService.getOrThrow` desde TASK-88 — un despliegue real
siguiendo únicamente ese archivo habría fallado al arrancar. Se agregó con
el mismo patrón `${VAR:?mensaje}` (sin valor por defecto) que ya usan los
demás secretos obligatorios del mismo bloque, y se completó también en
`.env.test-deploy` (archivo de referencia para una prueba de humo de
despliegue) para que quede coherente con el resto del stack. No es parte
del hallazgo de TASK-185 en sí — es un defecto preexistente encontrado al
editar el mismo bloque de variables de entorno para agregar
`TRUST_PROXY_HOPS`.

## Entidades / puertos / adaptadores tocados

Ninguna migración de base de datos ni cambio de esquema — es una corrección
de configuración HTTP en el punto de entrada de la aplicación.

- `src/common/http/trust-proxy.ts` (nuevo): `parseTrustProxyHops`, mismo
  patrón de fallar rápido que `parseCorsAllowedOrigins`.
- `src/common/http/apply-http-security.ts`: ahora también configura `trust
  proxy` antes de montar helmet/CORS; firma cambiada a
  `NestExpressApplication`.
- `src/main.ts`: construye la aplicación con
  `NestFactory.create<NestExpressApplication>(...)`.
- `docker-compose.prod.yml`: agrega `TRUST_PROXY_HOPS` (con valor por
  defecto `1`) y corrige la falta preexistente de `CORS_ALLOWED_ORIGINS`.
- `.env.example`, `.env.test-deploy`: documentan/agregan la variable nueva.
- `docs/deployment.md`, `CLAUDE.md`: documentan el fix y la variable.

## Tests

- `src/common/http/trust-proxy.spec.ts` (nuevo): valor por defecto sin
  configurar, valores válidos, y rechazo de valores negativos/no
  enteros/no numéricos.
- `test/security-hardening.e2e-spec.ts` (ampliado): nueva prueba en el
  bloque `rate limiting` que confirma que dos direcciones IP distintas
  llegadas por `X-Forwarded-For` se limitan de forma independiente contra
  `POST /auth/login` — se verificó manualmente que esta prueba falla
  (contador compartido) si se retira la llamada a `app.set('trust proxy',
  ...)`, antes de confirmarla en verde con el fix aplicado.
- `test/security-headers.e2e-spec.ts`: sin pruebas nuevas, solo adaptado al
  cambio de tipo de `applyHttpSecurity`.

Suite completa en verde al cierre para los archivos tocados: 92 suites
unitarias (990 pruebas), y las dos suites de extremo a extremo de
seguridad (`security-headers`, `security-hardening`, 13 pruebas) contra
PostgreSQL real, verificación de tipos y lint sin errores. Se detectó, al
correr la suite completa de extremo a extremo, una falla preexistente y no
relacionada en `test/appointments-rescheduling.e2e-spec.ts` (dos casos que
esperan 200 y reciben 400) — reproducida de forma idéntica en un
`git stash` sobre `main` sin ningún cambio de esta tarea aplicado, por lo
que queda fuera de alcance y señalada, no corregida, en la descripción del
pull request.

## Figuras pendientes

Ninguna — es una corrección de configuración sin un flujo nuevo que
amerite un diagrama propio.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-185-trust-proxy-rate-limit`, creada
  desde `main` para esta tarea, con pull request abierto hacia `main`.
