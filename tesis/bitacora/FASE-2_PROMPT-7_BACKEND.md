# Fase 2 — Pacientes (backend) — Seed del catálogo de obras sociales y cierre de la cobertura de pruebas (TASK-33)

## Qué se implementó

La tarea cierra el módulo Pacientes con dos piezas, sin cambios de esquema ni de
comportamiento de los puntos de acceso. La primera es la carga de datos de ejemplo
en el catálogo de obras sociales dentro del seed del inquilino del piloto, incluida
la obra social provincial, la única que la clínica acepta para las consultas según
la especificación. La segunda es la verificación de que la cobertura de pruebas de
extremo a extremo del módulo alcanza los siete escenarios que el requisito enumera,
agregando únicamente lo que esta tarea introduce de nuevo.

## Decisiones y por qué

**La obra social provincial se sembró como un nombre genérico de fantasía, como
todo otro nombre del seed.** El seed es dato de desarrollo y del piloto, y la obra
social real se carga con los datos de la clínica; nombrar aquí una entidad concreta
sería tan improcedente como nombrar profesionales reales. Se la ubicó primero en la
lista del catálogo por ser la que la especificación singulariza: sólo la provincial
se acepta para las consultas, y cualquiera para las recetas y la medicación. Modelar
esa regla pertenece al motor de turnos y a la capa conversacional, fuera del alcance
de esta fase; el seed se limita a hacer que la fila del catálogo exista.

**El catálogo se siembra de forma global y no bajo la organización del piloto.** El
requisito, redactado cuando el catálogo era todavía por inquilino, pide crearlo bajo
el inquilino de la clínica; la revisión del modelo de datos de la fase de fundaciones
convirtió la obra social en catálogo global, sin identificador de organización,
porque una obra social existe en el mundo y es cada profesional quien la acepta. En
consecuencia el seed la crea una sola vez, global, y el piloto la alcanza a través de
ese catálogo compartido en lugar de tener una copia propia. El alta se apoya en el
nombre único de la obra social, de modo que dos ejecuciones sucesivas del seed no la
dupliquen.

**No se modificó la entidad obra social.** Se evaluó completar el modelo con dos
atributos —lo que la obra social cubre y lo que otorga— que en una primera lectura
del diagrama entidad-relación parecían columnas de la entidad, y se descartó al
verificar que no lo son: corresponden a verbos de relación del diagrama, no a
atributos de la obra social, cuyo único dato propio es el nombre. La entidad quedó
por tanto intacta y el seed la puebla tal como está.

**La cobertura de pruebas se verificó y se completó sin reproducir lo ya cubierto.**
Los siete escenarios que el requisito enumera —el alta, la baja y la modificación
del paciente y su vínculo con el profesional; los campos que una reserva exige; la
regla del año que alterna el tipo del paciente entre nuevo y recurrente; el
consentimiento que no se reitera una vez aceptado; las observaciones accesibles sólo
al profesional del vínculo; el importador con archivos válidos e inválidos; y el
aislamiento entre inquilinos— ya quedaban cubiertos de extremo a extremo por las
tareas previas de esta misma fase, cada una acompañada de su prueba. Reescribirlos en
una prueba nueva de esta tarea sólo habría duplicado esa cobertura, contra la
disciplina de no redundancia del proyecto. Se agregó, en cambio, únicamente lo que
esta tarea introduce de nuevo: la prueba del propio seed, que verifica que la obra
social provincial se crea y que una segunda ejecución no la duplica.

## Alternativas descartadas

- **Completar la entidad obra social con atributos de cobertura tomados del
  diagrama**: descartada tras verificar que lo que cubre y lo que otorga son verbos
  de relación del diagrama entidad-relación y no columnas de la obra social; la
  entidad sólo tiene el nombre como dato propio.
- **Nombrar una obra social provincial real en el seed**: descartada por la misma
  razón por la que los profesionales del plantel del piloto llevan nombres de
  fantasía; la entidad real se carga con los datos de la clínica.
- **Sembrar el catálogo bajo la organización del piloto**: descartada porque la obra
  social es catálogo global desde la revisión del modelo de datos de las fundaciones,
  sin identificador de organización; se siembra una sola vez y el piloto la alcanza a
  través del catálogo compartido.
- **Reescribir los siete escenarios del requisito en una prueba nueva de esta
  tarea**: descartada por redundante, ya que las tareas previas de la fase los cubren
  de extremo a extremo; se verificó su cobertura y se agregó sólo la prueba del seed.

## Entidades / puertos / adaptadores tocados

- `prisma/seed.ts`: se incorporó la obra social provincial al catálogo de obras
  sociales, identificada por una constante exportada y ubicada primero en la lista;
  el resultado y el registro de consola del seed reflejan la obra social provincial
  creada.

No hubo cambios de esquema ni migraciones, y no se tocaron el modelo de datos, los
puntos de acceso, los puertos ni los adaptadores de integración.

## Tests y qué validan

- `test/seed.e2e-spec.ts` (una prueba nueva): tras ejecutar el seed sobre una
  organización desechable, la obra social provincial existe, y una segunda ejecución
  deja exactamente una fila, comprobando que el sembrado del catálogo es idempotente
  sobre el nombre único. Es el criterio de aceptación del requisito —que
  `npm run seed` cree la obra social provincial— ejercido sobre el código real del
  seed.
- Verificación de la cobertura de los siete escenarios del requisito, ya provista por
  las tareas anteriores de la fase: el alta, la búsqueda por documento, el detalle, la
  modificación, la baja lógica y el vínculo con el profesional en
  `test/patients-abmc.e2e-spec.ts` y `test/patients-entities.e2e-spec.ts`; los campos
  obligatorios de la reserva en `test/patients-abmc.e2e-spec.ts`; la regla del año en
  `test/patients-type-priority.e2e-spec.ts`; el consentimiento que no se reitera en
  `test/patient-consent.e2e-spec.ts`; las observaciones restringidas al profesional
  del vínculo en `test/patient-notes.e2e-spec.ts`; el importador con filas válidas e
  inválidas en `test/patients-import.e2e-spec.ts`; y el aislamiento entre inquilinos
  en esos mismos archivos y en `test/tenant-scoping.e2e-spec.ts`.
- Ejecución: suite end-to-end en verde (20 suites / 227 pruebas), compilación del
  proyecto sin errores y análisis estático sin advertencias. La ejecución de
  `npm run seed` crea la obra social provincial y, corrida por segunda vez, no la
  duplica. Todos los datos utilizados son ficticios: los nombres de las obras
  sociales, incluida la provincial, son genéricos y de fantasía, y ninguno
  corresponde a una entidad real.

## Figuras pendientes

- No se requieren figuras nuevas ni modificaciones a las ya registradas: la tarea no
  incorporó entidades ni puntos de acceso, y la obra social ya figura como catálogo
  global en las figuras del subdominio ya registradas.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-33-patients-seed-and-tests` (creada a partir de
  `main`). Sin commit al momento de redactar esta entrada: los cambios quedaron en el
  árbol de trabajo a la espera de autorización.
- Ticket: TASK-33 ("P2.7 – Seed, catálogos y tests del módulo Pacientes"). Depende de
  la creación del catálogo en la fase de fundaciones (TASK-19) y de las tareas del
  módulo Pacientes (TASK-27 a TASK-32), todas ya fusionadas. Quedan explícitamente
  fuera de alcance la carga de los datos reales de la clínica, que corresponde al
  piloto, y el seed de turnos de prueba, que corresponde al motor de turnos.
