# Fase 1 — Profesionales (backend) — entidades base del dominio (TASK-21)

## Qué se implementó

Se incorporaron al esquema de Prisma las cinco entidades base del dominio
Profesionales, tomadas del diagrama entidad-relación de la base de datos
(`modelo_base_de_datos.png`, fuente de verdad del ticket): profesional,
especialidad, matrícula, horario de atención y ausencia. La tarea abarca
únicamente el esquema y su migración; no introduce endpoints, servicios ni
validaciones de negocio, que corresponden a tareas posteriores de la misma
fase (P1.2 a P1.5).

Siguiendo la convención vigente del repositorio —los modelos de Prisma y
sus columnas se nombran en inglés, como quedó fijado en la migración de
renombrado `20260715100000_rename_to_english`— las entidades se modelaron
como `Specialty`, `Professional`, `License`, `WorkingHour` y `Absence`, con
sus campos también en inglés. Se agregaron cuatro enumerados para los
campos que el diagrama describe como conjuntos cerrados de valores:
`CareType` (obra social / particular), `ReassignmentMode` (automática /
manual), `LicenseType` (provincial / profesional) y `Weekday` (día de la
semana del horario).

`Professional` referencia a `Specialty` por clave foránea y mantiene
relaciones uno-a-muchos con `License`, `WorkingHour` y `Absence`. Los
campos que otras tareas de la fase configuran más adelante —duración de
consulta y franja extra para pacientes nuevos (P1.4)— se declararon
opcionales, de modo que un profesional pueda existir antes de que esos
valores se establezcan. Los indicadores de comportamiento (`adultsOnly`,
`acceptsNewPatients`, `reassignmentMode`, `active`) recibieron valores por
defecto razonables para no obligar a fijarlos en cada alta. La baja de un
profesional es lógica (`active`), nunca física, conforme al diagrama.

## Decisiones y por qué

**Acotamiento por tenant en cuatro de las cinco entidades, no en las
cinco.** El ticket especifica `organizationId` obligatorio en profesional,
especialidad, horario de atención y ausencia, pero no en matrícula. Se
respetó esa distinción: `License` no lleva `organizationId` y se accede
siempre a través de su `Professional`, que sí está acotado. Esta decisión
no es cosmética: la extensión multi-tenant del cliente de Prisma acota
automáticamente todo modelo que declare `organizationId`, de modo que dejar
ese campo fuera de `License` es también dejarla deliberadamente fuera del
acotamiento automático, coherente con que nunca se la consulta de forma
independiente sino como colección de un profesional ya acotado.

**Límite de matrículas en el servicio, no en el esquema.** El ticket fija
un máximo de tres matrículas por profesional, pero lo ubica explícitamente
como validación de servicio. Se evitó imponer esa regla como restricción de
base de datos (por ejemplo, un índice único por profesional y tipo), porque
el máximo de tres no se deriva de la combinación de tipos posibles y porque
la validación pertenece a la capa de aplicación de P1.2. En el esquema solo
se dejó el índice sobre la clave foránea.

**Especialidad como catálogo acotado por tenant y con nombre de texto
libre.** El diagrama menciona valores concretos (psiquiatría / psicología),
pero el ticket indica que la especialidad se modele inicialmente por tenant
para soportar organizaciones con nomenclaturas propias. Se optó por un
`String` con unicidad por `(organizationId, name)` —el mismo patrón que el
catálogo de obras sociales (`HealthInsurer`)— en lugar de un enumerado
cerrado, que habría fijado la nomenclatura de una organización en el
esquema.

**Horas de atención como texto `HH:mm`.** Los campos de hora de inicio y
fin se modelaron como cadenas en formato de reloj de pared de 24 horas, sin
componente de fecha ni de huso horario. Se descartó el tipo temporal nativo
porque el cliente de Prisma lo expone como una fecha completa con un día
epoch artificial, semántica confusa para un horario recurrente semanal que
no tiene fecha asociada; el ABM de horarios (P1.3) trabajará sobre esta
representación.

## Alternativas descartadas

- **Agregar `organizationId` también a `License`** por uniformidad con el
  resto de las entidades: descartada por contradecir la especificación
  explícita del ticket y por ser innecesaria, dado que el acotamiento por
  tenant queda garantizado a través de la relación con `Professional`.
- **Imponer el máximo de matrículas mediante restricciones de base de
  datos**: descartada porque el ticket sitúa esa validación en el servicio
  y porque una restricción de esquema no expresa naturalmente un tope de
  tres sobre dos tipos posibles.
- **Modelar la especialidad como enumerado**: descartada por cerrar la
  nomenclatura a los valores de una organización, en conflicto con el
  requisito de marca blanca.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: se agregaron los modelos `Specialty`,
  `Professional`, `License`, `WorkingHour` y `Absence`, y los enumerados
  `CareType`, `ReassignmentMode`, `LicenseType` y `Weekday`.
- Migración nueva `prisma/migrations/20260717234400_add_professional_entities/`:
  creación de los cuatro tipos enumerados, las cinco tablas, sus índices
  (unicidad de especialidad por organización; índices sobre `organizationId`
  y sobre las claves foráneas) y las claves foráneas correspondientes
  (borrado en cascada de matrículas, horarios y ausencias al eliminar el
  profesional; restricción de borrado sobre la especialidad referenciada).

No se tocaron puertos ni adaptadores: la tarea es exclusivamente de
modelado de datos.

## Tests y qué validan

- `test/professionals-entities.e2e-spec.ts` (nuevo): valida los criterios
  de aceptación del ticket contra la instancia local de PostgreSQL.
  - Inserción y lectura de un profesional junto con su especialidad y sus
    matrículas, verificando que la clave foránea a especialidad resuelve,
    que la relación uno-a-muchos con matrículas devuelve las filas
    esperadas y que los valores por defecto del esquema se aplican.
  - Rechazo de una consulta sobre una entidad acotada por tenant cuando no
    hay organización en contexto.
  - Estampado automático de `organizationId` en `Specialty` y `Professional`
    y aislamiento entre organizaciones: una organización no ve los
    profesionales de otra a través del cliente acotado.
- Ejecución: la suite end-to-end completa quedó en verde (9 suites / 29
  tests) tras aplicar la migración, y `prisma migrate status` no reporta
  migraciones pendientes. Los datos usados en los tests son ficticios
  (nombres de fantasía y números de matrícula de ejemplo).

## Figuras pendientes

- Se registró una figura pendiente con el diagrama entidad-relación acotado
  al subdominio Profesionales (ver `figuras_pendientes.md`).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-21-professional-entities` (creada a
  partir de `main`).
- Ticket: TASK-21 ("P1.1 – Entidades de Profesional"). Depende de TASK-14
  (P0.3, Prisma + PostgreSQL), TASK-15 (P0.4, multi-tenancy) y TASK-16
  (P0.5, usuarios con roles), todas ya fusionadas.
