# Fase 1 — Profesionales (backend) — ABM de profesionales y matrículas (TASK-22)

## Qué se implementó

Sobre las entidades definidas en P1.1 se construyó el módulo de negocio de
Profesionales: la capa de servicios y los endpoints REST para el alta,
consulta, modificación y baja lógica de profesionales, más el ABM de sus
matrículas. Todo el módulo queda protegido por JWT, acotado por
organización y auditado en cada mutación.

Los endpoints expuestos, bajo el recurso `/profesionales`, son: creación
(solo ADMIN); listado de los profesionales activos del tenant, incluyendo
nombre y matrículas para consumo del chatbot; detalle por id con
especialidad y matrículas; edición de datos generales (ADMIN o el propio
profesional sobre su registro); y baja lógica que fija `active = false`
(solo ADMIN). El ABM de matrículas se modeló como sub-recurso anidado
—`/profesionales/:id/matriculas`— con creación, edición y eliminación, y la
validación de un máximo de tres matrículas por profesional.

El módulo se apoya íntegramente en la infraestructura transversal de las
fundaciones: el cliente de Prisma acotado por tenant (que estampa y filtra
`organizationId` automáticamente), el `AuditService`, los guards globales de
autenticación y rol, y el interceptor que propaga el tenant del request.

## Decisiones y por qué

**Autorización "ADMIN o dueño" resuelta con un guard reutilizable en lugar
de condicionales dispersos en los servicios.** El requisito distingue tres
niveles de acceso: operaciones exclusivas de ADMIN (crear, dar de baja),
operaciones que un profesional puede ejercer sobre su propio registro
(editar sus datos y sus matrículas), y consultas disponibles para cualquier
usuario autenticado del tenant. Los dos primeros niveles se expresaron de
forma declarativa: el decorador `@Roles(ADMIN)` ya existente para las rutas
solo-ADMIN, y un guard nuevo, `ProfessionalOwnershipGuard`, para las rutas
"ADMIN o dueño". El guard compara el identificador de profesional asociado
al usuario autenticado (presente en el token) con el id de profesional de
la ruta, y deja pasar incondicionalmente al ADMIN. Se optó por un guard y
no por chequeos incrustados en cada método de servicio porque la regla de
propiedad es una preocupación de autorización transversal a varias rutas
(edición de profesional y las tres operaciones de matrículas); centralizarla
en un guard evita repetir la comparación, la mantiene declarativa junto a la
ruta y la deja verificable de forma aislada. Para que el guard fuera
genérico, las rutas anidadas de matrículas exponen el id del profesional con
el mismo nombre de parámetro que la ruta de detalle, de modo que un único
guard cubre ambos controladores.

**Anclar toda operación sobre matrículas en el profesional padre.** Como se
decidió en P1.1, la matrícula no lleva `organizationId` propio y por lo
tanto queda fuera del acotamiento automático del cliente de Prisma. Para no
abrir una vía de acceso que ignore el tenant, el servicio de matrículas
nunca resuelve una matrícula por su id de forma aislada: primero verifica,
a través del servicio de profesionales, que el profesional padre pertenece
al tenant del request, y solo entonces opera sobre la matrícula filtrando
además por el id de ese profesional. Así, el aislamiento por tenant de las
matrículas se deriva del profesional —ya acotado— en lugar de replicarse.

**Límite de tres matrículas validado en dos planos complementarios.** El
tope se aplica tanto en el DTO de creación del profesional (para las
matrículas cargadas en línea junto con el alta) como en el servicio de
matrículas (para el alta incremental, contando las existentes antes de
insertar). Ambos planos comparten una única constante, evitando que el
número quede duplicado como literal en dos lugares. Superar el tope produce
un error de validación (HTTP 400).

**Datos generales acotados a P1.2; configuración de agenda excluida.** Los
DTO de alta y edición exponen únicamente el nombre, la especialidad, el tipo
de atención y la fecha de confirmación. Los atributos de duración de
consulta y franja extra (P1.4) y los de filtro por edad, aceptación de
nuevos y modalidad de reasignación (P1.5) se dejaron deliberadamente fuera
de estos DTO, para no invadir el alcance de esas tareas; conservan sus
valores por defecto de esquema hasta que sus propios endpoints los
configuren.

**Presentación explícita de la respuesta.** En lugar de devolver la entidad
de Prisma tal cual, se introdujo una función de presentación que mapea el
profesional (con su especialidad y matrículas) a un objeto de respuesta
estable, omitiendo el `organizationId` interno para que el identificador de
tenant no cruce la frontera de la API. La misma definición de qué relaciones
se cargan se comparte entre las consultas del servicio y el tipo de la
respuesta, de modo que la forma leída de la base y la forma presentada no
puedan divergir.

## Alternativas descartadas

- **Incrustar la comprobación de propiedad en cada método de servicio**:
  descartada por dispersar una misma regla de autorización en múltiples
  puntos y mezclarla con la lógica de negocio, en lugar de expresarla una
  vez de forma declarativa junto a la ruta.
- **Agregar `organizationId` a la matrícula para poder acotarla
  directamente**: descartada por contradecir el modelado de P1.1 y por
  innecesaria, dado que el anclaje en el profesional padre ya garantiza el
  aislamiento.
- **Exponer los atributos de agenda (duración, franja, filtros,
  modalidad) en el ABM**: descartada por corresponder explícitamente a las
  tareas P1.4 y P1.5.

## Entidades / puertos / adaptadores tocados

- Módulo nuevo `src/professionals/`: controladores de profesionales y de
  matrículas anidadas; servicios de profesionales y de matrículas; guard de
  propiedad; DTO de alta y edición (profesional y matrícula) validados con
  class-validator; función de presentación; y una constante compartida con
  el máximo de matrículas.
- `src/app.module.ts`: se registró el módulo de Profesionales.
- Colección de Postman: regenerada automáticamente para incluir los nuevos
  endpoints.

No se introdujeron entidades de Prisma nuevas ni migraciones: el módulo
consume el esquema creado en P1.1.

## Tests y qué validan

- `src/professionals/guards/professional-ownership.guard.spec.ts` (unitario):
  cubre la lógica del guard de propiedad —ADMIN sobre cualquiera, profesional
  sobre su propio registro, denegación sobre un registro ajeno, denegación
  sin profesional asociado y sin usuario autenticado.
- `test/professionals-abm.e2e-spec.ts` (end-to-end, sobre PostgreSQL local y
  atravesando la capa HTTP con JWT reales): valida los criterios de
  aceptación del ticket —alta por ADMIN con matrículas en línea y su
  registro de auditoría; rechazo del alta a un rol profesional; listado con
  nombre y matrículas; aislamiento por tenant (una organización recibe 404
  ante un profesional de otra); edición del propio registro por su dueño;
  rechazo (403) al editar el registro de otro profesional; tope de tres
  matrículas (400 al agregar la cuarta); y baja lógica solo-ADMIN que
  desaparece del listado activo pero permanece en la base con `active =
  false`.
- Ejecución: suites unitaria (8 suites / 28 tests) y end-to-end (10 suites /
  38 tests) completas en verde; `eslint` y `nest build` sin errores. Los
  datos usados son ficticios.

## Figuras pendientes

- Se registró una figura pendiente con el flujo de autorización del ABM
  (autenticación JWT → guard de rol → guard de propiedad) (ver
  `figuras_pendientes.md`).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-22-professionals-abm` (creada a partir
  de `main`, con P1.1 ya fusionado).
- Ticket: TASK-22 ("P1.2 – ABM de Profesionales"). Depende de TASK-21 (P1.1,
  entidades), TASK-16 (P0.5, auth/roles) y TASK-17 (P0.6, auditoría), todas
  fusionadas.
