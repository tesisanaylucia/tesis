# Fase 2 — Pacientes (backend) — `PatientInactivityService.setThresholdMonths` audita sin transacción (TASK-98, corrección a TASK-83)

## Qué se implementó

TASK-98 fue una tarea de corrección hallada por una auditoría multi-agente
de `psique-back` sobre `main`, ángulo auditoría/fechas/reglas-como-datos,
2026-08-12. `PatientInactivityService.setThresholdMonths` — el único punto
de escritura del umbral de inactividad de pacientes, agregado por TASK-83
([[FASE-2_PROMPT-12]]) — llamaba a `ConfigTenantService.set(...)` (un
upsert autónomo) y, varias líneas después, a `AuditService.log(...)`, sin
ningún `$transaction` que envolviera a ambas llamadas y sin pasarle el
handle de transacción (`tx`) a `audit.log`. Esto reproduce, sobre una
entidad distinta, el mismo patrón que CLAUDE.md prohíbe explícitamente en
su sección "Audit trail" y que TASK-97 ([[FASE-4_PROMPT-9]]) había
corregido pocas tareas antes para `WaitlistReassignmentService.
respondToOffer`.

El caso concreto: si el proceso caía, o si la propia escritura de
auditoría fallaba, justo después de que el `upsert` de configuración ya
se hubiera confirmado, el umbral de inactividad de la organización quedaba
modificado sin ningún rastro de quién lo cambió ni cuándo. El riesgo no es
menor porque `patient_inactivity_months` es una regla que reclasifica
pacientes a NUEVO y dispara los recordatorios de actualización de
contacto — a diferencia de una simple preferencia de visualización, es una
decisión con efecto sobre el flujo clínico-administrativo que la Ley
25.326 exige poder auditar.

La corrección tuvo dos partes. Primero, `ConfigTenantService.set` pasó a
aceptar un parámetro `client` opcional (tipado `ConfigWriter`, un `Pick`
del cliente Prisma acotado por tenant a la única tabla que este método
toca) y a escribir a través de él en lugar del cliente por defecto cuando
se lo recibe — exactamente el mismo mecanismo que `AuditService.log` ya
ofrecía desde su creación con su propio parámetro `client`. Segundo,
`setThresholdMonths` envuelve ahora la llamada a `config.set(...)` y la
llamada a `audit.log(...)` en un mismo `$transaction`, pasándole el mismo
handle `tx` a las dos, de modo que la escritura del umbral y la entrada
que la describe comparten destino atómico.

## Decisiones y por qué

**Alcance mínimo, sin tocar la lógica de negocio.** El ticket es
explícitamente de corrección: la validación del rango `[1, 12]` en el DTO,
el cálculo del corte de fecha y el resto del comportamiento de
`setThresholdMonths` se mantuvieron idénticos. El único cambio real es
*dónde* corre cada escritura y que ambas reciben el mismo handle de
transacción.

**`ConfigTenantService.set` gana un parámetro opcional en lugar de un
método nuevo.** Cada llamante existente que no pasa `client` sigue
escribiendo sobre el cliente por defecto sin ningún cambio de
comportamiento observable; sólo el llamante que necesita atomicidad con
otra escritura —hoy únicamente `setThresholdMonths`— provee el handle. Es
la misma forma que `AuditService.log` ya eligió para el mismo problema, lo
que mantiene un único patrón para "esta escritura puede necesitar
transacción compartida" en todo el proyecto en lugar de dos.

**El tipo del parámetro se acota a lo que el método realmente usa.**
`ConfigWriter` es un `Pick` del cliente completo a la única propiedad
(`organizationConfig`) que `set` toca, replicando la razón ya documentada
sobre `AuditLogWriter` en `audit.service.ts`: aceptar el cliente completo
haría que cualquier objeto con la forma correcta de una sola tabla, como
el handle `tx` de una transacción interactiva, calzara sin un cast forzado
más amplio del necesario.

## Entidades / puertos / adaptadores tocados

- `ConfigTenantService.set` (`src/config-tenant/config-tenant.service.ts`):
  nuevo parámetro opcional `client: ConfigWriter`, con el tipo `ConfigWriter`
  exportado junto al servicio. Sin cambios de esquema ni de migración.
- `PatientInactivityService.setThresholdMonths`
  (`src/patients/patient-inactivity.service.ts`): la escritura de
  configuración y la entrada de auditoría pasan a correr dentro de un mismo
  `$transaction`, con `tx` propagado a ambas llamadas.

## Tests y qué validan

Se agregó a `config-tenant.service.spec.ts` un caso que verifica que,
cuando se pasa un cliente de transacción explícito, `set` escribe a través
de él y no del cliente por defecto — la mitad "mecánica" de la corrección,
independiente de quién la invoque.

Se agregaron a `patient-inactivity.service.spec.ts` dos casos nuevos sobre
`setThresholdMonths`, además de actualizar los dos ya existentes para
esperar el tercer argumento (`tx`) que `config.set` y `audit.log` reciben
ahora siempre: uno confirma que la llamada a `$transaction` ocurre
exactamente una vez y que tanto la escritura de configuración como la
entrada de auditoría se ejecutan dentro de ella; el otro hace que
`audit.log` lance una excepción y verifica que `setThresholdMonths`
propaga esa falla en lugar de silenciarla — la garantía observable, desde
un doble de prueba que no simula el estado interno de una transacción real
de Prisma, de que una auditoría fallida se lleva puesta la escritura de
configuración con ella, en vez de dejarla confirmada a medias.

Suite completa: 38 suites unitarias / 420 pruebas en verde; 37 suites e2e
/ 439 pruebas en verde (`--runInBand`, sin cambios en la suite e2e — el
comportamiento observable de la API no cambió). Lint limpio y
verificación de tipos (`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-98-patient-inactivity-audit-transaction` (creada desde
  `origin/main` fresco, tras el merge de TASK-97). Pusheada a `origin`; PR
  abierto, no fusionado aún.
- Ticket: TASK-98 ("[CORRECCIÓN] PatientInactivityService.setThresholdMonths
  audita sin transacción"), corrección a TASK-83 (punto de acceso
  administrativo del umbral de inactividad, [[FASE-2_PROMPT-12]]), que a su
  vez corrige TASK-29. Misma convención de bitácora dedicada para
  correcciones pequeñas dentro de la fase del ticket que corrigen que
  TASK-90/91/94/95/96/97 en Fase 4 y TASK-92 en Fase 1.
