# Fase 3 — Motor de Turnos (backend) — GET /admin/feriados no era accesible para el rol PROFESSIONAL (TASK-123, corrección a TASK-78)

## Contexto

La administración del calendario de feriados (TASK-78, [[FASE-3_PROMPT-9]]) se
implementó como cuatro rutas —alta, baja, edición de la descripción y listado
por año— restringidas en bloque al rol administrador mediante un decorador de
roles a nivel de controlador, sin ninguna excepción para el rol profesional.
La restricción en bloque tenía sentido para las tres escrituras: un
profesional gestionando su propia agenda no tiene motivo para dar de alta,
editar o eliminar un feriado que rige para todo el inquilino. Pero alcanzaba
también a la lectura, y ninguna otra ruta del sistema expone el calendario de
feriados: ni la disponibilidad de un profesional (P3.2, TASK-35) ni el
listado general de turnos (P3.9, TASK-79) devuelven una marca de "este día es
feriado", de modo que un profesional autenticado no tenía ninguna vía para
conocer el calendario de feriados de su propia organización. El documento de
requisitos prevé que la futura vista de agenda de la aplicación móvil del
profesional (P7.2, TASK-61, todavía no implementada — Fase 7 sigue "sin
iniciar" en este trabajo) marque los feriados "con color", precondición que
sin una ruta de lectura accesible no puede cumplirse. La misma auditoría de
código sobre `psique-back/main` (2026-08-14) que originó
[[FASE-3_PROMPT-27]] y [[FASE-3_PROMPT-28]] señaló este hallazgo en el módulo
de feriados, con una prueba end-to-end que ya documentaba el 403 de forma
explícita.

## Qué se implementó

- El método `findAll` de `HolidaysController` (la ruta `GET /admin/feriados`)
  incorpora su propio decorador de roles, `@Roles(ADMIN, PROFESSIONAL)`, que
  se resuelve antes que el del controlador gracias a
  `RolesGuard.getAllAndOverride`. Las tres rutas de escritura —alta, edición y
  baja— conservan el rol administrador exclusivo del controlador, que no se
  tocó.
- No se modificó el servicio: `HolidaysService.findAll` ya resolvía el
  listado a través del cliente de Prisma acotado por inquilino, de modo que
  un profesional, igual que un administrador, sólo puede ver los feriados de
  su propia organización sin ningún cambio adicional. La corrección es
  puramente de autorización en la capa de controlador.

## Decisiones y por qué

**Se ensanchó el rol de la ruta existente en lugar de incorporar el feriado a
la respuesta de disponibilidad o de turnos.** El propio ticket planteaba las
dos alternativas. Se descartó la segunda porque las respuestas de
disponibilidad y de turnos ya tienen un contrato probado por sus propias
suites end-to-end (`availability.e2e-spec.ts`,
`appointments-listing.e2e-spec.ts`) orientado a otro propósito —franjas
reservables y listado filtrable de turnos, respectivamente—, y agregarles un
campo de feriado habría acoplado una preocupación administrativa a un
contrato que no la necesita para nada de lo que hoy consume. La ruta de
feriados, en cambio, ya expone exactamente el recurso que se necesita leer;
bastaba con que el profesional pudiera alcanzarla.

**El decorador de roles se puso en el método, no se quitó del controlador.**
El patrón de anular el rol de clase con un decorador de método más permisivo
sobre un único handler ya lo usan `AppointmentsController` y
`WaitlistController` para distinguir, dentro del mismo controlador, qué
operaciones alcanza cada rol; el propio ticket señalaba que
`RolesGuard.getAllAndOverride` ya lo soporta. La diferencia con esos dos
controladores es que ninguno de ellos declara un rol de clase —cada método
lleva el suyo—, mientras que `HolidaysController` sí lo hace, de modo que
esta es la primera vez en el repositorio en que un decorador de método
efectivamente **reemplaza** uno de clase más restrictivo, en lugar de ser la
única fuente de la regla. Alternativas como retirar el rol de clase y
declararlo en las cuatro rutas, o mover el listado a un controlador propio,
se descartaron por reintroducir en cuatro lugares una regla que el mecanismo
de anulación ya resuelve en uno.

**La ruta conserva el prefijo `admin/feriados` pese a ser ahora legible por
un rol no administrativo.** El ticket ofrecía como alternativa posible un
alias sin ese prefijo. Se descartó: no hay en el repositorio un precedente de
un mismo recurso expuesto bajo dos rutas distintas según el rol que lo
consulta, y el patrón que sí existe para un recurso que ambos roles
comparten —`turnos`, `lista-espera`— es una única ruta con el rol declarado
por operación, exactamente lo que esta corrección adopta. Introducir un alias
habría sumado una segunda ruta, un segundo punto de mantenimiento y una
inconsistencia con ese patrón, a cambio de un nombre más preciso que ningún
criterio de aceptación exige. Queda como una tensión de nomenclatura menor,
registrada aquí y no resuelta: la ruta sigue nombrando "admin" un recurso que
ya no es exclusivamente suyo.

## Alternativas descartadas

- **Incorporar la lista de feriados a la respuesta de disponibilidad o de
  turnos**, la segunda opción que el propio ticket planteaba: descartada por
  acoplar una preocupación administrativa al contrato de otras dos rutas ya
  probadas, según se explica arriba.
- **Exponer el listado bajo una ruta nueva sin el prefijo `admin`**: descartada
  por no haber precedente de ese patrón en el repositorio y por duplicar una
  ruta que el mecanismo de anulación de rol ya resuelve sin ella.

## Entidades / puertos / adaptadores tocados

- `src/holidays/holidays.controller.ts` (modificado): decorador de roles de
  método agregado sólo sobre `findAll`; comentario del controlador actualizado
  para reflejar que la lectura ya no es exclusiva del administrador.

No se tocaron el esquema de la base de datos, el servicio, el presentador ni
ningún otro punto de acceso HTTP.

## Tests y qué validan

- `test/holidays.e2e-spec.ts` (modificado):
  - La prueba que antes verificaba 403 para un PROFESSIONAL en las cuatro
    rutas se redujo a las tres de escritura (alta, edición, baja), que siguen
    devolviendo 403.
  - Prueba nueva: un PROFESSIONAL puede leer el calendario de feriados de su
    propia organización (200) y el feriado recién creado por un administrador
    aparece en el listado — el criterio de aceptación del ticket.
  - Prueba nueva: un PROFESSIONAL de una organización no ve los feriados de
    otra. La fixture del archivo sólo tenía, hasta esta tarea, un
    administrador para la organización B; se agregó un profesional y su
    usuario para poder ejercitar el aislamiento por inquilino con un llamante
    de rol profesional y no sólo administrador-contra-administrador, como ya
    hacía la prueba equivalente preexistente.
- Verificación del alcance: ambas pruebas nuevas fallan si se retira el
  decorador de método agregado (la ruta vuelve a devolver 403 antes de llegar
  al aserto de contenido), de modo que su resultado positivo no es vacío.
- Ejecución: suite de feriados en verde (14 pruebas, dos más que antes de
  esta tarea) y suite de disponibilidad en verde (18 pruebas, sin cambios),
  ambas contra la instancia local de PostgreSQL; `tsc --noEmit` y `eslint`
  sobre los dos archivos modificados, sin hallazgos. Los datos usados en las
  pruebas son ficticios.

## Figuras pendientes

Ninguna nueva. La corrección amplía el alcance de un rol sobre una ruta ya
descripta en 3.2.3, sin introducir un flujo o entidad nuevos que la tesis no
documente todavía.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-123-professional-holidays-read-access`,
  creada desde `main`. Sin commit ni push al momento de redactar esta
  entrada, a pedido de la usuaria.
- Ticket: TASK-123 (Jira), "[CORRECCIÓN] TASK-78 – GET /admin/feriados no es
  accesible por rol PROFESSIONAL". Misma convención de bitácora dedicada para
  tareas puntuales dentro de la fase del ticket original que
  TASK-94/TASK-95/TASK-96/TASK-100/TASK-108/TASK-110/TASK-113/TASK-114/
  TASK-116/TASK-117 ([[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]],
  [[FASE-3_PROMPT-18]], [[FASE-3_PROMPT-19]], [[FASE-3_PROMPT-23]],
  [[FASE-3_PROMPT-24]], [[FASE-3_PROMPT-25]], [[FASE-3_PROMPT-26]],
  [[FASE-3_PROMPT-27]], [[FASE-3_PROMPT-28]]).
- Observación registrada y no resuelta: el nombre de la ruta (`admin/feriados`)
  ya no describe con precisión quién puede leerla; ver la decisión
  correspondiente más arriba.
