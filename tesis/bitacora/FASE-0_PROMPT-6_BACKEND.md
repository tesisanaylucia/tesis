# Fase 0 — Fundaciones (backend) — Revisión del modelo de datos: integridad referencial, normalización y relaciones lógicas

## Qué se implementó

Antes de continuar con los módulos siguientes se realizó una revisión integral
del esquema de base de datos, con el objetivo de asentarlo sobre tres criterios
explícitos: **integridad** (datos exactos y verificables por la propia base),
**normalización** (ausencia de redundancia) y **relaciones lógicas** (conexiones
declaradas entre tablas). La revisión no agrega funcionalidad: corrige el modelo
sobre el que se construirán los módulos de pacientes, turnos, capa
conversacional y control de acceso.

Se encontraron y corrigieron tres clases de problema.

**Ninguna referencia a la organización era una clave foránea real.** Todas las
columnas `organizationId` —en usuarios, profesionales, especialidades,
configuración por tenant, auditoría, eventos de cerradura, catálogo de obras
sociales, horarios de atención y ausencias— eran identificadores universales sin
restricción alguna, al igual que la columna que vincula una cuenta de usuario
con su profesional. La base aceptaba, por tanto, filas que apuntaban a
organizaciones inexistentes. Que esto no fuera hipotético quedó demostrado al
aplicar las nuevas restricciones: una suite de pruebas end-to-end existente
creaba usuarios con un identificador de organización inventado, que ninguna fila
respaldaba, y pasaba en verde.

**El identificador de organización estaba replicado en entidades que ya lo
alcanzan a través de su padre.** Horarios de atención y ausencias lo llevaban
pese a pertenecer a un profesional que lo lleva a su vez; los eventos de
cerradura lo llevaban pese a que su organización se deriva del turno que los
origina; y el catálogo de obras sociales lo llevaba pese a que una organización
no tiene obras sociales.

**No existía forma de impedir un vínculo entre organizaciones distintas.** Un
profesional podía referenciar una especialidad de otra organización y una cuenta
podía vincularse a un profesional de otra organización: la base no lo impedía y
solo el filtrado de la capa de servicios se interponía.

## Decisiones y por qué

**Se optó por normalización e integridad frente al acotamiento sin uniones.** La
convención vigente hasta esta tarea replicaba deliberadamente el identificador
de organización en las tablas hijas, con el argumento de que así toda consulta
se acota sin necesidad de unir tablas. Se revirtió esa decisión. El costo de la
réplica es un valor que puede discrepar del de su padre, y esa discrepancia es
una corrupción que ninguna consulta detecta: la fila resulta simultáneamente de
una organización según su propia columna y de otra según su padre. El beneficio,
en cambio, es una unión evitada sobre una tabla indexada, en un sistema cuyo
volumen previsto es el de una clínica —no el de centenares de clínicas y
millones de pacientes—. Se dejó asentado que una desnormalización controlada
podrá evaluarse si aparece un problema de rendimiento *medido*, y no de forma
preventiva.

Cabe señalar que la normalización no es incompatible con la multi-tenancy: el
patrón habitual en sistemas multi-inquilino consiste precisamente en navegar la
relación —de turno a profesional y de profesional a organización— con índices
sobre las claves foráneas involucradas.

**Se distinguió el identificador de organización propio del replicado.** El
criterio adoptado, y registrado como norma para toda entidad nueva, es que
`organizationId` vive en una entidad que pertenece al inquilino *directamente* y
no tiene padre a través del cual alcanzarse —usuarios, profesionales,
especialidades, configuración, auditoría—, y nunca en una que sí lo tiene. Las
primeras conservan el acotamiento automático que la extensión del cliente de
Prisma aplica al descubrirlas por esa columna; las segundas se alcanzan siempre
anclando la operación en la verificación de pertenencia del padre, que es
entonces lo único que contiene la solicitud. Ese anclaje ya existía en los
servicios de horarios y ausencias, de modo que la corrección del esquema no
debilitó el aislamiento: se limitó a hacer explícito que la verificación del
padre es la garantía real.

**Las claves foráneas compuestas resuelven el caso en que el identificador de
organización debe permanecer.** Cuando una entidad conserva legítimamente su
`organizationId` y además apunta a otra entidad acotada por inquilino, la clave
foránea se declara sobre el par `(organizationId, idDestino)` contra una
restricción de unicidad `(organizationId, id)` en el destino. La base pasa
entonces a rechazar por sí misma el vínculo entre organizaciones distintas, en
lugar de confiar en que cada camino de servicio recuerde comprobarlo. Se aplicó
a las tres relaciones donde el caso se presenta: profesional a especialidad,
usuario a profesional y auditoría a usuario. Esta construcción es lo que
convierte la permanencia del identificador en una decisión segura y no meramente
cómoda.

En el vínculo de usuario a profesional, la columna del profesional es opcional
—una cuenta administrativa no tiene profesional detrás—; el comportamiento por
omisión de PostgreSQL para claves foráneas compuestas (`MATCH SIMPLE`) deja la
restricción sin verificar mientras alguna de sus columnas es nula, que es
exactamente el caso deseado. Se declaró además la acción de borrado como
restrictiva y no como anulación, porque anular una clave compuesta intentaría
anular también el identificador de organización, que es una columna obligatoria.

**El catálogo de obras sociales pasó a ser global y la aceptación, del
profesional.** Una organización no tiene obras sociales: las obras sociales
existen con independencia de ella, y lo que sí es propio del dominio es qué
obras sociales acepta cada profesional. El catálogo por inquilino almacenaba la
misma entidad una vez por organización y, aun así, dejaba sin responder la
pregunta relevante. Se lo convirtió en catálogo global con nombre único y se
incorporó una relación opcional de muchos a muchos entre profesional y obra
social, cuya clave primaria es el par —de modo que una obra social no puede
vincularse dos veces al mismo profesional— y que se elimina en cascada con el
profesional sin afectar el catálogo.

Se mantuvo, en cambio, el catálogo de especialidades acotado por organización:
a diferencia de las obras sociales, la nomenclatura de especialidades es propia
de cada clínica.

**El registro de auditoría conserva su identificador de organización.** Es el
único caso en que la permanencia no es un atajo. La traza debe poder responderse
por organización con independencia de la entidad que cada fila describa, y el
par entidad/identificador que la fila almacena es polimórfico: no hay unión que
lo resuelva de forma genérica. Se lo dotó de clave foránea real hacia la
organización y de clave foránea compuesta hacia el usuario actuante, ambas con
borrado restrictivo, lo que convierte el carácter apendicular de la traza en una
garantía de la base y no en una mera convención.

**Los eventos de cerradura quedan en un estado transitorio documentado.** Se les
quitó el identificador de organización porque la organización de un evento se
deriva del turno que lo origina, y una copia aquí podría contradecir a ese turno
en cuanto la clave foránea exista. La consecuencia se asume explícitamente: la
tabla es de solo escritura hasta que existan las entidades de turno y de código
de acceso, sin ninguna vía de lectura por organización, y esa lectura deberá
resolverse entonces uniendo a través del turno, nunca reintroduciendo la
columna. La instrucción quedó registrada tanto en el esquema como en el servicio
y en el documento de convenciones del repositorio, junto con la forma exacta que
deberán tomar las claves foráneas pendientes.

Como efecto secundario, la escritura de un evento de cerradura dejó de requerir
una organización en el contexto de la solicitud. Ese requisito era un artefacto
de la columna eliminada, y su desaparición es coherente con el origen real de
estos eventos: reintentos y notificaciones del proveedor de la cerradura, que no
ocurren dentro de una solicitud de usuario.

## Alternativas descartadas

- **Conservar la réplica del identificador de organización en las tablas hijas**
  (convención vigente hasta esta tarea): descartada por introducir un valor que
  puede discrepar del de su padre sin que ninguna consulta pueda detectarlo, a
  cambio de evitar uniones que, al volumen previsto y con los índices
  existentes, no constituyen un problema demostrado.
- **Agregar únicamente las claves foráneas, sin normalizar**: descartada porque
  dejaría en pie la redundancia; una clave foránea sobre una columna replicada
  garantiza que la organización existe, no que coincida con la del padre.
- **Normalizar sin claves compuestas**, confiando el control de coherencia entre
  organizaciones a la capa de servicios: descartada por ser exactamente el tipo
  de garantía que la base puede sostener sin costo y el código no puede sostener
  sin disciplina permanente.
- **Eliminar el identificador de organización también del registro de
  auditoría**: descartada por el carácter polimórfico de la referencia a la
  entidad auditada, que impide reconstruir la organización por unión.
- **Conservar el catálogo de obras sociales por inquilino**: descartada por
  replicar la misma entidad por organización sin responder a qué profesional la
  acepta.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: se antepuso al esquema la enunciación de las tres
  reglas —integridad, normalización y claves compuestas— que toda entidad nueva
  debe declarar cuál aplica. Se agregaron claves foráneas reales hacia
  `Organization` en `User`, `Professional`, `Specialty`, `OrganizationConfig` y
  `AuditLog`; claves compuestas en `Professional → Specialty`,
  `User → Professional` y `AuditLog → User`; se eliminó `organizationId` de
  `WorkingHour`, `Absence`, `LockLog` y `HealthInsurer`; y se incorporó la
  entidad de vínculo `ProfessionalHealthInsurer`.
- `prisma/migrations/20260720213000_normalize_tenancy_and_add_foreign_keys/`:
  migración redactada a mano, con las sentencias de saneamiento previo
  necesarias para que las nuevas restricciones puedan crearse sobre los datos de
  desarrollo existentes.
- `src/professionals/absences.service.ts`: el identificador de organización que
  acompaña a los eventos de ausencia pasó a tomarse del contexto de la
  solicitud, dado que la fila ya no lo almacena.
- `src/professionals/working-hours.service.ts` y `src/lock-log/lock-log.service.ts`:
  se retiró el marcado por inquilino de las escrituras, que dejó de
  corresponder.
- `prisma/seed.ts`: el sembrado del catálogo de obras sociales pasó a ser global
  e idempotente entre organizaciones.
- `CLAUDE.md`: se reescribió la convención de multi-tenancy, que hasta ahora
  prescribía la réplica, y se agregó la sección que enuncia las tres reglas.
- `src/health-insurers/` (nuevo módulo): catálogo global de solo lectura.
- `src/professionals/professional-health-insurers.controller.ts` y
  `.service.ts` (nuevos): recurso anidado de lectura y reemplazo de la
  aceptación por profesional.
- `src/professionals/dto/replace-professional-health-insurers.dto.ts` (nuevo).
- `src/professionals/professional.presenter.ts`: se incorporó el conjunto de
  obras sociales aceptadas a la respuesta del profesional, con el mismo
  razonamiento de evitar N+1 que ya se aplicaba a la grilla de horarios.
- `src/app.module.ts`: registro del nuevo módulo de catálogo.
- `postman/psique-backend.postman_collection.json`: regenerada.

## Tests y qué validan

- `test/catalogs.e2e-spec.ts` (reescrito): verifica que el catálogo de obras
  sociales es legible sin organización en contexto, que el nombre es único de
  forma global, que la aceptación se registra por profesional y se alcanza
  navegando desde él, y que se elimina en cascada con el profesional dejando el
  catálogo intacto.
- `test/lock-log.e2e-spec.ts` (reescrito): verifica que un evento se persiste
  sin organización en contexto y sin columna de organización, y que no genera
  entradas en la traza de auditoría. La verificación de aislamiento se
  reformuló contra el evento concreto en lugar de contra un recuento global,
  porque las suites comparten base de datos y se ejecutan concurrentemente.
- `test/auth.e2e-spec.ts`: se corrigió la preparación de datos, que creaba
  usuarios contra un identificador de organización inventado. Es la evidencia
  directa del hueco de integridad que la clave foránea cierra.
- `test/seed.e2e-spec.ts`: se ajustó el orden de limpieza al exigido por las
  nuevas restricciones —cuentas antes que profesionales, profesionales antes que
  especialidades, todo antes que la organización— y se dejó de eliminar del
  catálogo global de obras sociales, que ya no pertenece a ninguna organización.
- Ejecución completa en verde: suite unitaria (10 suites / 50 tests) y
  end-to-end (13 suites / 105 tests), `eslint` sin errores y verificación de
  tipos sin errores. La migración se aplicó sobre la base de desarrollo y el
  ORM reportó el esquema sincronizado sin divergencias, lo que confirma que la
  migración escrita a mano se corresponde exactamente con el esquema declarado.

## Segunda etapa: endpoints sobre la relación profesional–obra social

La corrección del esquema dejó la aceptación de obras sociales por profesional
como una entidad de la base sin ninguna vía de acceso desde la API. Se cerró
ese hueco en la misma rama, exponiéndola con el mismo patrón que ya usan las
matrículas y la grilla de horarios: un recurso anidado bajo el profesional.

**El catálogo global se expuso aparte, sin anidarlo bajo profesionales.** Dado
que `HealthInsurer` no pertenece a ningún profesional ni organización —es la
razón misma de la corrección de la primera etapa—, se lo sirve desde
`GET /obras-sociales`, en un módulo propio e independiente del de
Profesionales, coherente con el estilo de módulo plano que rige el resto del
backend. Sin esta lista, un cliente no tendría forma de saber qué
identificadores son válidos para el recurso anidado.

**La aceptación se modeló como reemplazo completo, no como altas y bajas
individuales.** A diferencia de las matrículas, donde cada fila tiene campos
propios que editar, aceptar una obra social es un hecho binario: se acepta o
no. Se adoptó por ello el mismo patrón que la grilla de horarios semanal —un
`PUT` idempotente que declara el conjunto completo, con el arreglo vacío como
valor válido para un profesional que solo atiende de forma particular— en
lugar de exponer altas y bajas de a una. A diferencia de la grilla, aquí no
hay una invariante que dependa de una lectura previa para decidir si el nuevo
conjunto es válido — la validez de cada identificador es independiente del
estado anterior—, por lo que la operación corre en una transacción simple y no
bajo aislamiento serializable: dos reemplazos concurrentes pueden dejar un
resultado impredecible pero nunca uno corrupto, a diferencia del solapamiento
de horarios que sí exige serializable.

**Los identificadores desconocidos se validan antes de escribir, no se dejan
fallar contra la clave foránea.** La tabla de vínculo ya tiene una clave
foránea real hacia el catálogo (correctora de la primera etapa), que por sí
sola bastaría para rechazar un identificador inexistente. Se prefirió, sin
embargo, verificarlo explícitamente antes de la escritura para que el rechazo
llegue como un error de validación (400) con los identificadores desconocidos
nombrados, en lugar de la violación de restricción que la base devolvería sin
ese paso.

**La lista de obras sociales aceptadas se incorporó a la respuesta del
profesional, no solo al recurso anidado.** Se siguió el mismo razonamiento que
llevó a incorporar la grilla de horarios a esa respuesta en una tarea previa:
la capa conversacional necesita filtrar profesionales por obra social
aceptada, y sin este campo tendría que consultar el listado y luego, por cada
profesional, su conjunto de obras sociales — una consulta N+1. El conjunto es
acotado por el tamaño del catálogo, por lo que el costo de transportarlo es
menor que el de la consulta adicional que evita.

## Tests y qué validan (segunda etapa)

- `test/professional-health-insurers.e2e-spec.ts` (nuevo): cubre el catálogo
  global (listado ordenado por nombre, rechazo sin autenticación), el ciclo
  completo del recurso anidado (profesional recién creado sin obras sociales,
  reemplazo, reemplazo idempotente que sustituye el conjunto anterior en lugar
  de sumarse a él, vaciado con arreglo vacío), el rechazo de un identificador
  inexistente y de identificadores duplicados en el mismo cuerpo, las reglas de
  propiedad y rol ya vigentes en el módulo (el dueño puede modificar su propio
  conjunto, no el de otro profesional; un ADMIN puede modificar el de
  cualquiera), el aislamiento por tenant, y que la respuesta del profesional
  incluye el conjunto aceptado.
- Se actualizó la colección de Postman con el script del repositorio y se
  completó a mano el cuerpo de ejemplo del reemplazo, que el generador deja en
  blanco.
- Ejecución: suite unitaria (10 suites / 50 tests) y end-to-end (14 suites /
  117 tests) completas en verde tras agregar el módulo nuevo; `eslint` y
  verificación de tipos sin errores.

## Figuras pendientes

- Diagrama entidad-relación actualizado del esquema, con las claves foráneas
  reales y las claves compuestas señaladas. Corresponde a la subsección 3.2.0.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `refactor/db-integrity-normalization`, creada a partir de
  `main`. Sin commits al momento de redactar esta entrada, por pedido expreso de
  revisión previa.
- Deja pendiente, documentado en el esquema y en `CLAUDE.md`: convertir en
  claves foráneas reales las referencias a turno y paciente del registro de
  auditoría y a turno y código de acceso de los eventos de cerradura, cuando las
  fases correspondientes incorporen esas entidades. La aceptación de obras
  sociales por profesional, pendiente al cierre de la primera etapa, se expuso
  en la segunda etapa de esta misma entrada.
