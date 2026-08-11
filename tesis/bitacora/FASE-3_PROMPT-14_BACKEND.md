# Fase 3 — Motor de Turnos (backend) — verificación del estado real del ABM de feriados (TASK-81, verificación sobre TASK-78)

## Qué se implementó

TASK-81 no pedía código: pedía determinar con certeza si el ABM
administrativo de feriados de TASK-78 (P3.b — `GET/POST/PATCH/DELETE
/admin/feriados`) realmente existía en `main`, después de que una
auditoría automatizada del código (2026-07-28) reportara que no encontró
ni `HolidaysController` ni `HolidaysService` en la rama, solo el modelo
`Holiday` de Prisma y su lectura de sólo lectura desde
`AvailabilityService.loadHolidays()` — pese a que TASK-78 figuraba como
"Listo" en Jira.

La verificación se hizo sobre una copia recién actualizada del
repositorio (`git fetch origin` seguido de un listado del árbol de
`origin/main`), no contra una referencia local potencialmente desactualizada
— la misma precaución que ya se había dejado documentada como lección al
implementar TASK-38 (comparar contra `origin/main` recién traído, no
contra `main` local). Esa copia sí contiene `src/holidays/` completo
(controller, service, DTOs, presenter, tests unitarios y e2e) y las cuatro
rutas administrativas. El historial de `main` muestra el commit de
implementación (`c2f2714`, "Add admin CRUD endpoints for the tenant
holiday calendar") fusionado el mismo 2026-07-28 en que corrió la
auditoría (`771d81d`, PR #38). La explicación más probable es que la
auditoría se ejecutó contra una copia del repositorio anterior a esa
fusión, no que el trabajo se haya perdido o que el ticket se marcara
"Listo" sin haberse implementado.

## Decisiones y por qué

**No se modificó código de producción ni de tests.** El ABM de feriados ya
existe, funciona y está fusionado a `main`; no había nada que reimplementar
ni ningún hueco de cobertura relacionado con el hallazgo de la auditoría.

**No se corrigió el estado de TASK-78 en Jira.** El ticket ya refleja
correctamente la realidad del código ("Listo" y el código está en
`main`), así que ninguna de las tres acciones que TASK-81 contemplaba
como resultado posible (reabrir TASK-78, fusionar una rama pendiente, o
corregir un cierre prematuro) aplicaba. Se dejó en cambio un comentario en
TASK-78 documentando la verificación y el origen probable del falso
negativo, para que quede trazado por qué una auditoría posterior no debe
volver a levantar la misma alarma sin antes refrescar su copia del
repositorio.

**Se creó de todos modos una rama dedicada (`feature/TASK-81-verify-holiday-crud`,
desde `origin/main` fresco) aunque no fuera a llevar commits de código**,
siguiendo el mismo precedente que TASK-80: cada ticket de Jira, incluidos
los de verificación, queda asociado a una rama propia para trazabilidad,
independientemente de si termina modificando `src/`.

## Alternativas descartadas

- **Confiar directamente en el hallazgo de la auditoría y mover TASK-78 de
  vuelta a "Por hacer"**: descartada de inmediato al confirmarla contra el
  código real — habría revertido el estado de un ticket correctamente
  cerrado en base a un reporte obsoleto.
- **Re-ejecutar la suite e2e de feriados como parte de esta verificación**:
  descartada por no aportar información nueva sobre la pregunta que
  TASK-81 hace (¿existe el código en `main`?, no ¿funciona el código?); la
  suite de TASK-78 ya cubre ese segundo aspecto y no había motivo para
  sospechar de ella.

## Entidades / puertos / adaptadores tocados

Ninguno.

## Tests y qué validan

No aplica — tarea de verificación documental, sin cambios de código. La
verificación en sí consistió en `git fetch origin` + inspección del árbol
de `origin/main` y de su historial de commits para el módulo
`src/holidays/`.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-81-verify-holiday-crud` (creada
  desde `origin/main` fresco, sin commits propios). Pusheada a `origin`.
- Ticket: TASK-81 (Jira), "[VERIFICACIÓN] TASK-78 – Confirmar si el ABM de
  Feriados está realmente implementado en main". Referencia también a
  TASK-78 (ticket original) y a [[FASE-3_PROMPT-9]] (implementación
  original del ABM de feriados).
