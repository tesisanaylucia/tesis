# Fase 3 — Motor de Turnos (backend) — fuga de datos entre profesionales en `GET /lista-espera/profesional/:id` (TASK-94, corrección a TASK-40)

## Contexto

Una auditoría multi-agente de `psique-back` en `main` (ángulo
multi-tenancy/seguridad, 2026-08-12) detectó que
`WaitlistService.findForProfessional` solo llamaba
`this.professionals.assertOwned(professionalId)` — un chequeo de tenant, no
de ownership dentro del tenant — a diferencia de `create`/`reorder`/`remove`
del mismo servicio, que sí aplican el método privado `assertOwnerOrAdmin`.
El controller (`GET /lista-espera/profesional/:professionalId`) tampoco
tenía `ProfessionalOwnershipGuard`, solo `@Roles(ADMIN, PROFESSIONAL)`. El
resultado: un usuario autenticado con rol `PROFESSIONAL` (profesional A)
podía pedir la lista de espera de otro profesional de su misma organización
(profesional B) y recibir el listado completo — `patientId` y orden en cola
de pacientes con los que A no tiene ningún vínculo de tratamiento. Esto
viola la sección Security de `CLAUDE.md` ("PROFESSIONAL — scoped to their
own patients/appointments") y Clinical data restrictions.

## Qué se implementó

- `WaitlistService.findForProfessional` ahora recibe el
  `AuthenticatedUser` que hace el pedido y llama al mismo
  `assertOwnerOrAdmin` privado que ya usaban `create`/`reorder`/`remove`,
  antes de `assertOwned` (tenant) y de la consulta.
- `WaitlistController.findForProfessional` inyecta `@CurrentUser()` y lo
  pasa al servicio — mismo patrón que los otros tres métodos del
  controller, que ya recibían el usuario autenticado.

## Decisiones y por qué

**Se reusó el `assertOwnerOrAdmin` existente del servicio, en vez de
agregar `ProfessionalOwnershipGuard` a la ruta.** El ticket ofrecía las dos
opciones como equivalentes. Se prefirió la primera porque
`WaitlistService` ya centraliza esa lógica de ownership en un método
privado usado por sus otras tres operaciones — repetir el chequeo con un
cuarto mecanismo (un guard tenant-blind en el controller) habría
introducido una segunda fuente de verdad para la misma regla dentro del
mismo módulo sin necesidad, cuando el patrón ya establecido resolvía el
caso con un diff mínimo.

**No se tocó ningún otro método de lectura del módulo.** El ticket pedía
revisar si otro método de lectura de `waitlist` tenía el mismo hueco. Los
únicos otros accesos a `waitlistEntry.findMany`/`findFirst` en el módulo
están dentro de `WaitlistReassignmentService` (motor de reasignación
interno, disparado por eventos del sistema, no por un
`professionalId` provisto por un llamador `PROFESSIONAL` vía HTTP) y
dentro de las transacciones internas de `create`/`reorder`/`remove`, que
ya estaban protegidas. No se encontró ningún otro endpoint expuesto con el
mismo hueco.

## Alternativas descartadas

- **`ProfessionalOwnershipGuard` en la ruta:** descartada por la razón de
  arriba (duplicaría, no reemplazaría, la lógica de ownership ya
  centralizada en el servicio).

## Entidades / puertos / adaptadores tocados

- `WaitlistService.findForProfessional`
  (`src/waitlist/waitlist.service.ts`): nueva firma
  `(professionalId, user)`, llama a `assertOwnerOrAdmin` antes de
  `assertOwned`.
- `WaitlistController.findForProfessional`
  (`src/waitlist/waitlist.controller.ts`): agrega `@CurrentUser()` y lo
  reenvía al servicio.

## Tests agregados o modificados

- `src/waitlist/waitlist.service.spec.ts`: nuevo `describe('findForProfessional', …)`
  con tres casos — ADMIN lee la lista de cualquier profesional, el
  profesional dueño lee la suya, un profesional ajeno recibe
  `ForbiddenException` y la consulta a Prisma nunca se dispara.
- `test/waitlist.e2e-spec.ts`: dos casos nuevos dentro de
  `GET /lista-espera/profesional/:professionalId` — el profesional dueño
  lee su propia lista (200, solo entradas suyas) y un profesional ajeno
  del mismo tenant recibe 403 (antes solo estaba cubierto el caso
  cross-tenant, que ya daba 404 vía `assertOwned`).

Suite completa verde tras el cambio: 38 suites unitarias / 414 pruebas; 37
suites e2e / 434 pruebas (`--runInBand`). Lint y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva — es una corrección de autorización sobre un endpoint ya
documentado (P3.7, figura de lista de espera, si existiera, no cambia de
forma).

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-94-waitlist-professional-ownership-check` (creada desde
  `origin/main` fresco, tras el merge de TASK-93). Pusheada a `origin`, PR
  abierta, no fusionada aún.
- Ticket: TASK-94 ("[CORRECCIÓN] Fuga de datos de pacientes entre
  profesionales en GET /lista-espera/profesional/:id"), corrección a
  TASK-40 (P3.7, creó `WaitlistService`/`WaitlistController`). Misma
  convención de bitácora dedicada para correcciones pequeñas dentro de la
  fase del ticket original que TASK-79/TASK-81/TASK-86
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]], [[FASE-3_PROMPT-15]]) y
  TASK-91/TASK-92 ([[FASE-4_PROMPT-7]], [[FASE-1_PROMPT-9]]).
