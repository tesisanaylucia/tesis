# Fase 1 — Profesionales (backend) — Endpoint de reactivación de Profesionales (TASK-125, mejora de paridad)

## Contexto

Una auditoría de código de `psique-back/main` (2026-08-14, agente "Audit
Módulo Profesionales vs SRS") señaló una asimetría entre los dos módulos de
ABMC del sistema: Pacientes expone `POST /pacientes/:id/reactivar`
(`PatientsService.reactivate`), que anula la marca de baja lógica y devuelve
al paciente a la operación normal, mientras que Profesionales solo tenía la
baja (`ProfessionalsService.deactivate`, P1.6) y ningún camino de vuelta por
API — un profesional desactivado quedaba recuperable únicamente con acceso
directo a la base de datos. TASK-125 pide agregar el endpoint simétrico,
"análogo al de pacientes, con su guard de rol correspondiente y auditoría".

## Qué se implementó

- `ProfessionalsService.reactivate(id, actorId)`: dentro de una única
  transacción, limpia `deletedAt` a `null` sobre el profesional (verificado
  primero con `assertOwned`, el mismo chequeo de tenencia que usa `update` y
  `deactivate`) y dentro de esa misma transacción registra la entrada de
  auditoría (`entity: 'Professional'`, `action: 'UPDATE'`,
  `detail: { reactivated: true }`), devolviendo el profesional con sus
  relaciones (`professionalDetailInclude`) para que el presentador pueda
  responder el objeto completo.
- `ProfessionalsController`: `POST /profesionales/:id/reactivar`, con
  `@Roles(Role.ADMIN)` a nivel de método — el mismo rol exclusivo que ya
  protege el alta y la baja — devolviendo `ProfessionalResponse` vía
  `toProfessionalResponse`.

## Decisiones y por qué

**Réplica exacta del método de `PatientsService`, no una reinterpretación.**
El ticket señala el archivo de Pacientes como patrón de referencia
explícito; la forma transaccional (verificar tenencia, escribir el `update`
y el `audit.log` dentro del mismo `$transaction`) es idéntica a
`PatientsService.reactivate`, y la del propio `ProfessionalsService.deactivate`
que ya existía en este módulo — no se introdujo ninguna variación de
comportamiento no pedida por el ticket.

**No se restituye ningún token revocado.** `deactivate` revoca, dentro de su
propia transacción, los tokens de sesión vigentes del profesional (P8.1b,
TASK-87) para que una cuenta desactivada no conserve acceso de escritura
hasta el vencimiento natural de su JWT. `reactivate` no intenta deshacer esa
revocación: se trata la readmisión como una decisión de autorización nueva y
no como la reversión exacta de la baja — el profesional reactivado
simplemente vuelve a iniciar sesión y recibe un token nuevo, sin que el
sistema tenga que reconstruir ni reemitir ninguno de los que revocó. El
módulo de Pacientes no tiene contraparte de este matiz porque un paciente no
posee cuenta ni JWT propios.

**`assertOwned`, no una variante que filtre por `deletedAt`.** El mismo
método que usan `update` y `deactivate` ya resuelve un profesional
desactivado (no filtra por `deletedAt`), que es exactamente lo que
`reactivate` necesita para encontrar a quien va a reactivar; introducir un
método de tenencia alternativo habría sido redundante.

## Alternativas descartadas

- **Agregar `Role.SYSTEM` al guard, como hace `PatientsController.reactivate`
  para el alta de pacientes:** descartada. El comentario de
  `PatientsController` deja esa extensión como decisión abierta a confirmar
  con la Clínica antes de habilitarla incluso para pacientes, que sí tienen
  un flujo conversacional que naturalmente podría dispararla (un paciente
  inactivo que vuelve a escribir al bot); un profesional no tiene ningún flujo
  equivalente iniciado por el chatbot, así que no había ninguna razón nueva
  para abrir el mismo interrogante en este módulo. El ticket tampoco lo pide:
  su alcance funcional pide únicamente "el guard de rol correspondiente", que
  en el resto del módulo (alta, baja) es siempre `Role.ADMIN` en solitario.
- **Restaurar o reemitir los tokens revocados en la reactivación:** descartada
  por la razón de seguridad explicada arriba — habría reintroducido en
  silencio el mismo riesgo que TASK-87 cerró para la baja, sin que el ticket
  lo pidiera.

## Entidades / puertos / adaptadores tocados

- Ninguna migración ni cambio de esquema: reutiliza la columna `deletedAt`
  de `Professional` ya existente desde P1.1/P1.6.
- `ProfessionalsService` (`src/professionals/professionals.service.ts`):
  método nuevo `reactivate`.
- `ProfessionalsController`
  (`src/professionals/professionals.controller.ts`): endpoint nuevo
  `POST /profesionales/:id/reactivar`.

## Tests agregados o modificados

- `src/professionals/professionals-reactivate.service.spec.ts` (nuevo,
  mismo formato que `professionals-deactivate.service.spec.ts`): Prisma/
  Audit mockeados, verifica que la escritura de `deletedAt: null` y el
  `audit.log` correspondiente ocurren sobre el mismo handle de transacción, y
  que un profesional fuera del tenant del llamador no dispara ninguna
  escritura.
- `test/professionals-abm.e2e-spec.ts`: dos casos nuevos, simétricos a los ya
  existentes para la baja — reactivación exitosa por el administrador
  (profesional de vuelta en el listado activo, `deletedAt` en `null` tanto en
  la respuesta como en la base), rechazo a un rol no administrativo (403), y
  aislamiento por tenant (404 ante el intento de un administrador de otra
  organización).

Suite completa verde tras el cambio: 79 suites unitarias / 781 pruebas; 51
suites e2e / 559 pruebas (`--runInBand`). Lint y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva — no cambia ningún diagrama de entidad ni de flujo ya
documentado, solo agrega un punto de acceso simétrico a uno existente.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-125-professional-reactivation` (creada
  desde `origin/main` fresco, tras el merge de TASK-121,
  [[FASE-3_PROMPT-31]]).
- Ticket: TASK-125 ("[MEJORA] Falta endpoint de reactivación de
  Profesionales"), mejora de paridad con Pacientes ([[FASE-2_PROMPT-2]],
  TASK-28, que introdujo `PatientsService.reactivate` como parte del ABMC de
  pacientes) sobre la baja lógica de profesionales de P1.6. Misma convención
  de bitácora dedicada para tareas de corrección/mejora puntuales dentro de
  la fase del módulo que toca, ya usada para TASK-84 ([[FASE-2_PROMPT-11]]),
  TASK-92 ([[FASE-1_PROMPT-9]]) y las demás entradas de fase listadas en
  `PROGRESO.md`.
