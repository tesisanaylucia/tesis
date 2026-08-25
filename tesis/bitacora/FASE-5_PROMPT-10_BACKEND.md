# Fase 5 — Capa conversacional y WhatsApp (backend) — CRUD administrativo de FAQ (TASK-77)

## Qué se implementó

Se agregaron los cuatro endpoints administrativos que le faltaban al
Módulo 5 sobre la tabla `Faq`: `GET/POST/PATCH/DELETE /admin/faqs`, para que
un administrador del inquilino pueda mantener el contenido de preguntas
frecuentes que el chatbot ya consumía desde TASK-51
(`FaqService.findBestMatch`) sin que existiera ninguna vía, fuera de una
migración o del seed de TASK-33, para cargarlo o corregirlo. El modelo
`Faq` y su columna `organizationId` ya existían desde TASK-47 (P5.2); esta
tarea no tocó el esquema, sólo agregó el controlador, el servicio de
escritura y los DTO que faltaban sobre esa misma tabla.

## Decisiones y por qué

**El CRUD se agregó al módulo y al servicio ya existentes (`FaqModule`,
`FaqService`), no a uno nuevo.** El propio comentario dejado en
`faq.module.ts` por TASK-47 anotaba explícitamente "no controller yet —
nothing in this ticket's scope asks for an admin CRUD"; esta tarea es esa
extensión declarada como pendiente, así que ampliar el archivo existente
evita una segunda fuente de verdad sobre la misma tabla.

**El patrón se copió del CRUD administrativo de feriados (TASK-78,
`HolidaysController`/`HolidaysService`), no se diseñó desde cero.** Ambos
son un ABM simple, acotado por inquilino, sin relación jerárquica con
`Professional` o `Patient`, protegido por rol ADMIN vía `@Roles` de clase.
Se reutilizó la misma convención de "no encontrado en vez de prohibido"
para el aislamiento entre inquilinos (`getOwnedOrThrow` vía `findFirst`,
igual que `HolidaysService.getByDateOrThrow` y
`PatientsService.findVisibleOrThrow`): un `id` de otro inquilino es
indistinguible de uno inexistente, porque un 403 confirmaría que la fila
existe. A diferencia de feriados, `Faq` tiene una clave primaria propia
(`id`, UUID) y no una clave natural compuesta con la fecha, así que el
parámetro de ruta usa `ParseUUIDPipe` en lugar del `ParseCalendarDatePipe`
a medida de feriados.

**Los nombres de columna en el contrato JSON son `question`/`answer`, no
`pregunta`/`respuesta` como sugiere literalmente la descripción del
ticket.** El modelo `Faq`, ya implementado desde TASK-47, había elegido
esos nombres en inglés siguiendo la convención del repositorio (modelos y
contrato JSON en inglés, glosario en español sólo en comentarios); el
ticket describe la entidad en los términos del diagrama SRS/ER, no
prescribe el contrato HTTP. Mantener `question`/`answer` evita que el
mismo dato tenga dos nombres distintos según se lo lea por el chatbot o
por este CRUD.

**El DELETE devuelve `204 No Content`, no un cuerpo con un conteo de
afectados como el de feriados.** A diferencia de un feriado, borrar una
FAQ no tiene ningún efecto colateral que valga la pena reportar —el
ticket es explícito en que el borrado no debe tocar el historial de
conversaciones, y en efecto ninguna fila del chatbot referencia `id_faq`—
así que se siguió la convención más simple de `LicensesController.remove`
en lugar de inventar una respuesta para un efecto que no existe.

**El PATCH admite `question` y/o `answer` por separado, ninguno anulable
a `null`.** Ambas columnas son `NOT NULL` y una FAQ sin pregunta o sin
respuesta no es un estado parcial válido, así que se usó
`@OptionalDefined()` en los dos campos (puede omitirse cada uno, pero un
`null` explícito se rechaza) en lugar de `@OptionalNullable()`, que sólo
correspondería a una columna donde "borrar el valor" sea una operación
soportada.

**Se agregó traza de auditoría en las tres operaciones de escritura**,
siguiendo la misma convención que `HolidaysService`: cada mutación se
audita dentro de la misma transacción interactiva que la escribe
(`AuditService.log(params, tx)`), para que una falla del registro de
auditoría no deje una escritura sin rastro. El detalle de la operación de
actualización registra qué campos cambiaron (`Object.keys(dto)`), no su
valor, siguiendo la misma regla que ya aplica el resto del sistema a los
datos de paciente.

## Entidades, puertos y adaptadores tocados

- `src/faq/faq.service.ts`: se agregaron `findAll`, `create`, `update`,
  `remove` y el `getOwnedOrThrow` privado, junto con la inyección de
  `AuditService`; `findBestMatch` no cambió.
- `src/faq/faq.controller.ts`: nuevo. `GET/POST/PATCH/DELETE /admin/faqs`,
  ADMIN-only.
- `src/faq/faq.presenter.ts`: nuevo, `FaqResponse`/`toFaqResponse`.
- `src/faq/dto/create-faq.dto.ts`, `src/faq/dto/update-faq.dto.ts`: nuevos.
- `src/faq/faq.module.ts`: registra `FaqController`.

Sin cambios de esquema ni de migración: el modelo `Faq` y su índice por
`organizationId` ya existían.

## Tests

- `test/faq-admin.e2e-spec.ts`, nuevo: autenticación (401), rol PROFESSIONAL
  rechazado en las cuatro rutas (403), alta y listado, validación de
  longitud y de campo vacío (400), edición parcial, 404 sobre un id
  inexistente o de otro inquilino tanto en PATCH como en DELETE, id
  malformado (400), aislamiento de listado entre dos inquilinos, y el
  criterio de aceptación de que una FAQ borrada deja de aparecer tanto en
  el listado administrativo como en la tabla que consulta el chatbot.
- `src/faq/faq.service.spec.ts`: ampliado con `findAll`/`create`/`update`/
  `remove`, conservando las pruebas ya existentes de `findBestMatch`.

Suite completa en verde al cierre de la tarea: 69 suites / 675 pruebas
unitarias (`npx jest`) y la suite nueva de integración,
`faq-admin.e2e-spec.ts`, en 12/12 contra una base Postgres real. Lint
(`eslint --fix`) y verificación de tipos (`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva: el modelo `Faq` ya figura en la Figura 31 pendiente (DER de
`Faq`/`PrescriptionRequest` dentro del subdominio conversacional,
registrada en TASK-47/P5.2), y esta tarea no le agregó columnas.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-77-faq-admin-crud`, creada desde
  `main`.
- Ticket: TASK-77 (Jira), "P5.b – FAQ admin: endpoints CRUD para gestión de
  preguntas frecuentes por tenant", bajo el épico del Módulo 5.
- Dependencias declaradas por el ticket, ya fusionadas a `main`: TASK-51
  (P5.6, `FaqService.buscarRespuesta`/`findBestMatch`), TASK-33 (P2.7, seed
  inicial de FAQ), TASK-16/TASK-72 (guards de rol admin).
- Fuera de alcance, declarado en el propio ticket y respetado: el
  algoritmo de búsqueda de FAQ (ya implementado en TASK-51) y una pantalla
  de gestión en la app móvil del profesional.
