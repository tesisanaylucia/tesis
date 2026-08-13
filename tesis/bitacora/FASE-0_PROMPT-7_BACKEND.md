# Fase 0 — Fundaciones (backend) — invariante rol/profesional en `User` (TASK-93)

## Qué se implementó

Se agregó una restricción de base de datos que garantiza que una cuenta
`User` con `role = PROFESSIONAL` tenga siempre un `professionalId` cargado,
y que una cuenta con `role = ADMIN` o `role = SYSTEM` nunca lo tenga. Antes
de esta tarea nada lo impedía: ni una restricción `CHECK` en la base ni
código de aplicación, ya que TASK-16 (P0.5) solo dejó un endpoint de solo
lectura (`GET /users`) para ejercitar el guard de roles, sin ninguna vía de
creación o edición de usuarios. Una auditoría de la base de datos realizada
el 12 de agosto de 2026 confirmó, inspeccionando la definición de la tabla,
que la columna `professionalId` era simplemente anulable, sin ninguna
restricción que la ligara al valor de `role`.

Se agregó la restricción `CHECK ((role = 'PROFESSIONAL') = ("professionalId"
IS NOT NULL))` sobre la tabla `User`, mediante una migración escrita a mano
(Prisma 6.19 no admite `@@check` en `schema.prisma`), siguiendo el mismo
patrón ya usado para las restricciones `CHECK` existentes sobre `duration`,
el rango de fechas de una ausencia y el orden de la lista de espera. La
forma de igualdad de la expresión cubre los tres roles con una sola
restricción: para `PROFESSIONAL` exige `professionalId` no nulo, y para
`ADMIN`/`SYSTEM` exige que sea nulo.

Como todavía no existe un servicio de creación o edición de usuarios (fuera
de alcance de este ticket, que lo señala explícitamente como un trabajo
futuro), no hubo ninguna validación de aplicación que agregar — el ticket
contemplaba ese paso solo como condicional a que dicho servicio ya
existiera.

## Decisiones y por qué

**Una única restricción con forma de igualdad, en lugar de dos condiciones
separadas por rol.** La expresión `(role = 'PROFESSIONAL') = (professionalId
IS NOT NULL)` es verdadera exactamente cuando ambos lados coinciden: para
`PROFESSIONAL` fuerza el lado derecho a verdadero (`professionalId` no
nulo), y para cualquier otro rol lo fuerza a falso (`professionalId` nulo).
Esto evita enumerar `ADMIN` y `SYSTEM` por separado y generaliza sin cambios
si en el futuro se agregara un cuarto rol con la misma exigencia que
`ADMIN`/`SYSTEM`.

**Restricción a nivel de base de datos en lugar de solo documentación o
validación futura.** El propio ticket prioriza la restricción `CHECK` como
la entrega principal, con la validación de servicio como paso adicional
condicional a que exista un endpoint de escritura. Esto es consistente con
el resto del esquema, donde las invariantes que la base puede expresar
directamente (claves foráneas compuestas, `CHECK` sobre duraciones y
rangos) se prefieren sobre depender de que cada camino de código las
respete.

## Alternativas descartadas

No surgieron alternativas de diseño relevantes: la expresión de la
restricción está dada por el propio ticket, y el alcance excluye
explícitamente construir el CRUD de gestión de usuarios en esta tarea.

## Efecto colateral encontrado y corregido: reconciliación del seed

Al aplicar la restricción contra la base de desarrollo, la suite de
sembrado (`prisma/seed.ts`) falló: `reconcileProfessionals` — la función que
retira del roster sembrado a un profesional que ya no está declarado —
desvinculaba la cuenta asociada poniendo `professionalId` en `null` sin
tocar `role`, dejando una cuenta `PROFESSIONAL` sin profesional vinculado.
Esa combinación, tolerada hasta ahora, es exactamente la que la nueva
restricción prohíbe. Se corrigió reemplazando la desvinculación por el
borrado de la cuenta: no hay ningún otro rol al que degradar una cuenta de
inicio de sesión cuyo profesional fue eliminado, y el comentario del código
ya explicaba que el vínculo debía romperse antes de eliminar la fila del
profesional por la restricción `ON DELETE RESTRICT` de la clave foránea
compuesta — el borrado de la cuenta cumple esa misma condición.

Tres suites de pruebas end-to-end preexistentes también creaban cuentas
`PROFESSIONAL` sin `professionalId` como parte de sus datos de prueba, sin
relación con lo que cada una efectivamente verifica, y se ajustaron: la
prueba de inicio de sesión de un profesional (que solo necesitaba un login
válido) pasó a vincular un profesional real; la prueba del auto-completado
del `organizationId` por la extensión de acotamiento por tenant (donde el
rol es incidental a lo que se verifica) pasó a usar `ADMIN`; y la propia
prueba de la reconciliación del seed se actualizó a la nueva semántica de
borrado.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: comentario nuevo sobre `User.professionalId`
  documentando la restricción `CHECK` y la migración que la agrega.
- Migración nueva
  `prisma/migrations/20260813180000_user_role_professional_id_check/`:
  agrega el `CHECK` sobre `User`.
- `prisma/seed.ts`: `reconcileProfessionals` borra la cuenta vinculada a un
  profesional retirado del roster, en lugar de solo desvincularla.

## Tests y qué validan

- `test/user-role-professional-id-check.e2e-spec.ts` (nuevo): cuatro casos
  contra `PrismaService` directamente, ya que no existe ningún endpoint que
  ejercite el insert — rechaza un `PROFESSIONAL` sin `professionalId`,
  rechaza un `ADMIN` y un `SYSTEM` con `professionalId`, y acepta la
  combinación válida de ambos casos.
- `test/seed.e2e-spec.ts`: el caso de reconciliación de profesionales
  retirados se actualizó para verificar que la cuenta vinculada se elimina,
  no que queda con `professionalId` en `null`.
- `test/auth.e2e-spec.ts` y `test/tenant-scoping.e2e-spec.ts`: ajustados
  para que sus datos de prueba respeten la nueva restricción, sin cambios en
  lo que efectivamente verifican.
- Suite completa verde tras aplicar la migración vía `prisma migrate
  deploy` contra la instancia local de PostgreSQL: 38 suites / 411 tests
  unitarios y 37 suites / 432 tests end-to-end (`--runInBand`), lint y
  `tsc --noEmit` sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-93-user-role-professional-id-check`
  (creada a partir de `origin/main` fresco, `4e459b4`, con TASK-92 ya
  fusionado).
- Ticket: TASK-93 ("[CORRECCIÓN] Invariante role↔professionalId en User no
  está garantizada"), corrección sobre TASK-16 (P0.5) y TASK-72 (corrección
  previa a P0.5, que agregó el rol `SYSTEM`).
