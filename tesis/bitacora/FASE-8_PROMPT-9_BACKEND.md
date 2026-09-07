# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — forbidNonWhitelisted en el ValidationPipe global (TASK-131, P8.1c, corrección/extensión a P8.1 y TASK-88)

## Qué se implementó

Se implementó la segunda mitad de la corrección P8.1c que TASK-88 había
dejado pendiente: el `ValidationPipe` global (`src/main.ts`) sólo
declaraba `whitelist: true` y `transform: true`, sin `forbidNonWhitelisted:
true`. Con esa configuración, un campo que el DTO de la ruta no declara no
se rechaza — `whitelist` lo descarta en silencio, y la petición se procesa
igual como si el campo nunca hubiera llegado. Se agregó
`forbidNonWhitelisted: true`, de forma que un campo no declarado ahora
responde `400`, nombrándolo en el mensaje de error, en lugar de
desaparecer sin ningún indicio para quien hizo la petición.

## Decisiones y por qué

**La construcción del pipe se extrajo a una función propia,
`createHttpValidationPipe` (`src/common/validation/http-validation-pipe.ts`),
en vez de tocar únicamente la línea de `main.ts`.** Antes de este cambio,
la app real (`main.ts`) y las treinta y dos instancias de aplicación que
arman los propios tests de extremo a extremo cada uno con su propio
`new ValidationPipe({ whitelist: true, transform: true })` construían el
pipe por separado — treinta y tres copias textuales del mismo objeto de
configuración repartidas en el repositorio. Agregar `forbidNonWhitelisted`
sólo en `main.ts` habría dejado cada instancia de prueba validando un
comportamiento que la aplicación real ya no tiene, exactamente el mismo
riesgo de divergencia silenciosa que motivó extraer `applyHttpSecurity`
para helmet/CORS en TASK-88. Se aplicó el mismo criterio: un único punto
de construcción, reutilizado por `main.ts` y por cada archivo de prueba de
extremo a extremo, de modo que ninguna copia pueda quedar desactualizada
respecto de lo que la aplicación realmente ejecuta.

**Se revisó la suite de extremo a extremo completa buscando pruebas que
dependieran del comportamiento anterior (campo desconocido descartado en
silencio) antes de asumir que el cambio era compatible hacia atrás.** Se
encontró una: `test/patient-notes.e2e-spec.ts` tenía una prueba que
enviaba a propósito un campo `notes` no declarado en el DTO de edición de
prioridad, para probar que ese campo "contrabandeado" no llegaba a
persistirse — aserción que dependía exactamente del descarte silencioso
que este ticket elimina. Se corrigió la prueba para reflejar el
comportamiento correcto: la petición completa ahora se rechaza con `400`,
y la aserción de que la nota nunca se escribió se mantiene, pero ahora
como consecuencia de que la petición entera fue rechazada, no de que el
campo sobrante fue ignorado.

**Se agregó una prueba de extremo a extremo nueva, no sólo la corrección
de la existente**, ya que ninguna prueba previa ejercitaba el nuevo
comportamiento de forma directa (todas las que tocan el pipe global lo
hacen indirectamente, a través de DTOs completos). Se incorporó al mismo
archivo que ya prueba helmet/CORS de punta a punta
(`test/security-headers.e2e-spec.ts`), no a uno nuevo, porque las tres
piezas — cabeceras HTTP, CORS y validación estricta del cuerpo — son la
misma decisión de endurecimiento del punto de entrada de la aplicación
(P8.1c), aplicadas todas antes de que cualquier ruta reciba una petición.

## Entidades / puertos / adaptadores tocados

Ninguna entidad de dominio ni tabla nueva — es exclusivamente
configuración de infraestructura HTTP, sin relación con el esquema de base
de datos.

- `src/common/validation/http-validation-pipe.ts` (nuevo):
  `createHttpValidationPipe`, el punto único que construye el
  `ValidationPipe` global.
- `src/main.ts`: `bootstrap()` usa `createHttpValidationPipe()` en lugar
  de construir el pipe en la propia línea.
- Treinta archivos de prueba de extremo a extremo (32 puntos de uso):
  reemplazado `new ValidationPipe({ whitelist: true, transform: true })`
  por `createHttpValidationPipe()`.
- `CLAUDE.md`: ampliada la viñeta de seguridad HTTP ya existente (antes
  sólo cabeceras/CORS) para cubrir también la construcción del pipe de
  validación y el nuevo comportamiento de `forbidNonWhitelisted`.

## Tests

- `test/patient-notes.e2e-spec.ts`: la prueba que probaba el descarte
  silencioso de un campo contrabandeado se corrigió para esperar `400` en
  lugar de `200`, conservando la aserción de que la nota nunca se
  persiste.
- `test/security-headers.e2e-spec.ts` (ampliado): nueva sección que prueba
  el rechazo `400` contra una ruta pública real (`POST /auth/login`) al
  enviar un campo que `LoginDto` no declara, junto con un caso de control
  que envía sólo los campos declarados y llega hasta la lógica de negocio
  (`401` por credenciales inexistentes, no por el cuerpo de la petición),
  para dejar claro que el pipe rechaza específicamente el campo sobrante y
  no la ruta entera.

Suite completa en verde al cierre: 81 suites unitarias (792 pruebas) y 51
suites de extremo a extremo (564 pruebas) contra PostgreSQL real
(`--runInBand`), verificación de tipos y lint sin errores.

## Figuras pendientes

Ninguna nueva. La Figura 55 (pendiente desde TASK-88, cadena de middleware
HTTP del arranque) ya incluía el pipe de validación global como uno de sus
pasos; se amplió su descripción en `figuras_pendientes.md` para señalar
que ese paso ahora también rechaza — no sólo transforma — un cuerpo con
campos no declarados.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-131-forbid-non-whitelisted`, creada
  desde `main` para esta tarea, con pull request abierto hacia `main`.
