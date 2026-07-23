# Fase 3 — Motor de Turnos (backend) — servicio de disponibilidad (TASK-35)

## Qué se implementó

Se implementó el servicio de disponibilidad (`AvailabilityService.getSlots`,
`src/availability/`) que, dado un profesional y un rango de fechas, calcula
los turnos libres de su agenda, junto con el endpoint
`GET /profesionales/:id/disponibilidad?from=&to=` que lo expone. El
algoritmo sigue el procedimiento del documento de especificación de
requisitos, módulo Turnos, sección "Disponibilidad del profesional":

1. Se obtiene el horario de atención (`WorkingHour`) del profesional para los
   días de la semana dentro del rango consultado.
2. Se generan franjas desde la hora de inicio hasta la hora de fin de cada
   bloque, con paso igual a `Professional.consultationDuration` (minutos).
3. Se excluyen: los días que caen en un feriado (`Holiday`) del inquilino,
   los días cubiertos por una ausencia (`Absence`) del profesional, y las
   franjas que ya tienen un turno (`Appointment`) en estado `RESERVED` o
   `CONFIRMED` para ese profesional.
4. Se retorna la lista de franjas libres como `{ scheduledAt, duration,
   available }`.

La tarea depende de TASK-22 (horarios y ausencias del profesional, ya
fusionada a `main`) y de TASK-34 (entidades `Appointment` y `Holiday`, en
`feature/TASK-34-appointment-holiday-waitlist`, todavía no fusionada al
momento de esta tarea); la rama de esta tarea se creó a partir de la de
TASK-34 en lugar de `main`, precisamente porque las entidades que el
algoritmo consulta no existen todavía en `main`.

Deliberadamente no implementa la regla de doble franja para paciente nuevo
(placement de la franja extra de primera sesión): el propio ticket la asigna
a una tarea posterior de la misma fase (P3.4), que consumirá este servicio y
aplicará esa restricción por encima de las franjas que aquí se calculan. Los
franjas generadas siempre llegan libres al llamador, por lo que el campo
`available` de la respuesta es siempre verdadero; se conservó de todos modos
porque es la forma exacta que nombra el documento de requisitos
(`fechaHora, duracion, disponible`) y la que consumirá el chatbot (M5) sin
transformación adicional.

## Decisiones y por qué

**El cálculo trata las horas de un bloque de atención como reloj de pared en
UTC, sin husos horarios reales.** El sistema ya modela los horarios de
atención y las ausencias como valores de reloj/calendario sin huso horario
(`"HH:mm"` y fechas `DATE`), y documenta explícitamente que una dependencia
real de huso horario por inquilino queda fuera de alcance mientras no se
configure. El servicio de disponibilidad hereda la misma simplificación al
combinar un día calendario con una hora de reloj de pared para producir el
instante de un turno: interpreta ambos como si fueran directamente UTC, en
lugar de introducir aquí — de forma aislada — una conversión de huso horario
que ningún otro componente del sistema todavía tiene.

**El paso de generación es `consultationDuration`, no `slotCadence`.** El
profesional tiene dos configuraciones de agenda distintas: la duración de la
sesión y la cadencia con la que se abren franjas (por ejemplo, "atención
cada 1 hora con sesiones de 45 minutos"). El documento de requisitos, para
esta tarea puntual, especifica el paso de generación como la duración de la
consulta; `slotCadence` queda sin usar en este servicio y su aplicación, si
corresponde, se deja a una tarea posterior de espaciado de franjas.

**La ausencia de `consultationDuration` configurado se trata como un error
del pedido (400), no como una lista vacía.** El profesional puede existir sin
tener configurada su duración de consulta (P1.4 la deja anulable hasta que
se configura). Sin ese valor no hay paso con el que generar franjas, así que
el servicio distingue explícitamente "no hay franjas porque no hay horario
cargado" (lista vacía, un estado válido) de "no se puede calcular nada
porque falta un dato de configuración obligatorio para el cálculo" (rechazo
explícito), en lugar de devolver silenciosamente una lista vacía en ambos
casos.

**Se agregó un tope de rango de consulta (`MAX_AVAILABILITY_RANGE_DAYS`,
90 días) no exigido por el ticket.** El endpoint recorre un día a la vez
entre `from` y `to`; sin una cota, un rango de años terminaría iterando y
consultando ausencias/turnos sobre un período sin límite práctico. Se
modeló como una cota antiabuso sobre el costo de la consulta, no como una
regla de negocio, en la misma línea que los topes ya existentes en el
repositorio (por ejemplo, el máximo de bloques por reemplazo del horario
semanal).

**Los nombres de los parámetros de consulta y de los campos de la respuesta
son en inglés (`from`, `to`, `scheduledAt`, `duration`, `available`), aunque
el ticket los describe en español (`desde`, `hasta`, `fechaHora`, `duracion`,
`disponible`).** La convención del repositorio distingue la ruta HTTP — en
español, como el resto de las rutas anidadas bajo un profesional — del resto
del contrato, que es en inglés en su totalidad; los parámetros de consulta
ya existentes en el sistema (por ejemplo, `includeInactive` en la búsqueda de
pacientes) siguen esa misma regla para vocabulario que no es terminología
propia del dominio clínico. El nombre en español del ticket se interpretó
como la descripción funcional del endpoint en el documento de requisitos, no
como una exigencia literal de nomenclatura del contrato JSON.

**El servicio no queda anidado dentro del módulo de Profesionales, a
diferencia del horario de atención y las ausencias.** Aunque la ruta HTTP
cuelga de `/profesionales/:id/...` igual que ellas, el cálculo de
disponibilidad lee además `Holiday` y `Appointment`, entidades de otro
subdominio (Turnos), y su servicio está pensado para ser inyectado más
adelante por el motor de reasignación (M3/M4) y por la capa conversacional
del chatbot (M5) — el mismo motivo por el que la auditoría y la
configuración por inquilino son módulos propios en lugar de vivir dentro de
Profesionales. Se creó entonces un módulo `AvailabilityModule` independiente
que importa `ProfessionalsModule` sólo para reutilizar `assertOwned`, y
exporta el servicio para esos consumidores futuros.

## Alternativas descartadas

- **Devolver una lista vacía quieta ante un profesional sin
  `consultationDuration` configurado**, igual que ante un profesional sin
  horario cargado: descartada porque confundiría dos estados distintos ("no
  hay franjas hoy" frente a "falta configurar la agenda"), y el segundo es
  información que el cliente necesita para guiar al operador a completar la
  configuración, no un resultado válido de disponibilidad.
- **Usar `desde`/`hasta` y `fechaHora`/`duracion`/`disponible` tal como los
  nombra el ticket**, por fidelidad literal al documento de requisitos:
  descartada por ser inconsistente con la convención ya establecida en el
  repositorio de mantener el contrato JSON íntegramente en inglés, con la
  única excepción de vocabulario propiamente clínico/de dominio (como
  `dni`).
- **Anidar el servicio y el controlador dentro de `ProfessionalsModule`**,
  siguiendo la forma de horario de atención y ausencias: descartada porque
  el cálculo depende de entidades de otro subdominio (`Holiday`,
  `Appointment`) y se prevé que módulos futuros de otras fases lo consuman
  directamente, lo que hace más clara la dependencia como un módulo propio
  que sólo importa `ProfessionalsModule` por la verificación de pertenencia.

## Entidades / puertos / adaptadores tocados

- `src/availability/availability.service.ts` (nuevo): `AvailabilityService`
  con el algoritmo de generación de franjas.
- `src/availability/availability.controller.ts` (nuevo): controlador de
  `GET /profesionales/:id/disponibilidad`.
- `src/availability/availability.module.ts` (nuevo): módulo que importa
  `ProfessionalsModule` y exporta `AvailabilityService`.
- `src/availability/dto/availability-query.dto.ts` (nuevo): validación de los
  parámetros de consulta `from`/`to` como fechas calendario.
- `src/availability/availability.constants.ts` (nuevo): cota antiabuso del
  rango de consulta.
- `src/app.module.ts` (modificado): registro de `AvailabilityModule`.

No se tocaron puertos ni adaptadores: el servicio sólo lee entidades ya
existentes a través del cliente de Prisma acotado por inquilino.

## Tests y qué validan

- `src/availability/availability.service.spec.ts` (nuevo, 11 pruebas): el
  cliente de Prisma se simula en memoria para fijar el algoritmo puro sin
  base de datos.
  - Generación de 6 franjas para un profesional con duración de 30 minutos y
    un bloque de 3 horas (el criterio de aceptación explícito del ticket).
  - Un fin de semana sin horario configurado no genera franjas.
  - Exclusión de una franja en día feriado.
  - Exclusión de una franja en día cubierto por una ausencia.
  - Exclusión de una franja ya ocupada por un turno en estado `RESERVED` o
    `CONFIRMED`.
  - Lista vacía cuando el profesional no tiene ningún horario cargado (sin
    llegar a consultar feriados/ausencias/turnos).
  - Rechazo cuando falta `consultationDuration`.
  - Rechazo cuando `to` es anterior a `from`.
  - Rechazo cuando el rango excede la cota antiabuso, y aceptación cuando la
    alcanza exactamente.
- `test/availability.e2e-spec.ts` (nuevo, 15 pruebas): contra la instancia
  local de PostgreSQL, con dos organizaciones para probar el aislamiento por
  inquilino.
  - Rechazo del acceso sin autenticar (401).
  - Las mismas 6 franjas del caso unitario, ahora de punta a punta por HTTP.
  - Fin de semana sin horario configurado, feriado, ausencia y turno ya
    reservado/confirmado, cada uno excluyendo la franja esperada.
  - Un turno cancelado no excluye la franja que ocupaba.
  - Un feriado de otra organización, en la misma fecha, no afecta la
    disponibilidad del profesional consultado — el criterio de aislamiento
    entre inquilinos del ticket, ejercitado contra la extensión real de
    Prisma y no contra un simulacro.
  - 404 al consultar un profesional de otra organización.
  - 400 cuando falta `consultationDuration`, y en cada variante de parámetros
    de consulta inválidos (rango invertido, fecha mal formada, día
    inexistente, parámetro faltante).
- Ejecución: suite unitaria en verde (21 suites / 159 pruebas) y suite
  end-to-end en verde (22 suites / 259 pruebas). Los datos usados en las
  pruebas son ficticios (nombres de fantasía y documentos de ejemplo).

## Figuras pendientes

Ninguna nueva; la figura del diagrama entidad-relación del subdominio Turnos
ya registrada para TASK-34 (ver `figuras_pendientes.md`, entrada 17) sigue
pendiente y no requiere ampliación por esta tarea, que no modifica el
esquema.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-35-availability-service` (creada a
  partir de `feature/TASK-34-appointment-holiday-waitlist`, todavía no
  fusionada a `main` al momento de esta tarea). Commit `b42c325` al momento
  de redactar esta entrada.
- Ticket: TASK-35 ("P3.2 – Servicio de disponibilidad (generación de
  slots)"). Depende de TASK-22 (horarios/ausencias del profesional, ya
  fusionada) y de TASK-34 (entidades de Turno y Feriado, pendiente de
  fusión).
