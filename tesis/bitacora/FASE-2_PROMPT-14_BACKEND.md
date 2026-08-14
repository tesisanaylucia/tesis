# Fase 2 — Pacientes (backend) — Segmento de ruta en inglés en `.../configuracion` (TASK-106, corrección a TASK-83)

## Qué se implementó

TASK-106 fue una tarea de corrección hallada por una auditoría multi-agente
de `psique-back` sobre `main`, ángulo convenciones de CLAUDE.md, 2026-08-12.
`PatientInactivityConfigController` (agregado por TASK-83,
[[FASE-2_PROMPT-12]]) expone `@Controller('admin/configuracion')` con
`@Patch('patient-inactivity-months')`, resultando en
`PATCH /admin/configuracion/patient-inactivity-months` — una ruta que
empieza en español (`configuracion`) y termina en inglés
(`patient-inactivity-months`).

CLAUDE.md, sección "Domain naming: which surfaces are Spanish", nombra
`configuracion` explícitamente como ejemplo de ruta en español, y toda otra
ruta `.../configuracion` del código (p. ej.
`professionals.controller.ts:79`, `@Patch(':id/configuracion')`) se
mantiene en español de punta a punta. Este controller era el único que
cambiaba de idioma a mitad de ruta, rompiendo el patrón del que un
colaborador futuro infiere la convención — el mismo tipo de hallazgo que
motivó, en tareas anteriores, corregir identificadores o comentarios que se
habían desviado de una regla ya escrita en CLAUDE.md.

La corrección fue puramente léxica: el segmento final de la ruta pasó de
`patient-inactivity-months` a `meses-inactividad-pacientes`, sin tocar
lógica, DTO, ni el nombre del archivo TypeScript del controller o del DTO
(esos siguen en inglés, como corresponde a un identificador de código, no a
una ruta HTTP — CLAUDE.md distingue la superficie por *dónde* aparece el
texto, no por a qué entidad se refiere).

## Decisiones y por qué

**Alcance mínimo, sin tocar lógica de negocio.** El ticket es
explícitamente de corrección de nomenclatura: `PatientInactivityService`,
la validación del DTO y el resto del comportamiento de
`setThresholdMonths` se mantuvieron idénticos. El único cambio es el
literal pasado a `@Patch(...)`.

**Ruptura sin período de convivencia.** El ticket documenta que la ruta
vieja no tiene clientes reales fuera de la propia suite de tests (el
frontend no la consume todavía), así que se reemplazó directamente en
lugar de mantener ambas rutas activas — la alternativa (dejar la ruta
inglesa como alias) habría reintroducido la misma inconsistencia que la
tarea busca eliminar.

**El nombre en español elegido, `meses-inactividad-pacientes`, sigue el
mismo orden que el resto del glosario de dominio del proyecto** (sustantivo
seguido de sus calificadores, como `patient_inactivity_months` en su forma
inglesa de configuración interna), sin necesidad de inventar un término
nuevo: el propio ticket ya lo proponía como ejemplo consistente con el
resto de la familia `.../configuracion`.

**Los comentarios que documentaban la ruta anterior se actualizaron junto
con el código**, no solo el literal de la ruta — en `patient-inactivity.service.ts`,
`update-patient-inactivity-months.dto.ts` y el propio controller, los
comentarios citan la ruta HTTP como referencia de trazabilidad hacia el
ticket que la originó; dejarlos con la ruta vieja habría hecho que la
próxima persona buscara un endpoint que ya no existe.

## Entidades / puertos / adaptadores tocados

- `PatientInactivityConfigController`
  (`src/patients/patient-inactivity-config.controller.ts`): el decorador
  `@Patch(...)` cambia de `'patient-inactivity-months'` a
  `'meses-inactividad-pacientes'`. Sin cambios de firma, DTO ni lógica.
- Comentarios de trazabilidad actualizados en
  `src/patients/patient-inactivity.service.ts` y
  `src/patients/dto/update-patient-inactivity-months.dto.ts` (los nombres
  de archivo TypeScript, en inglés, no cambian).

## Tests y qué validan

Se actualizaron los dos literales de ruta en
`test/patient-inactivity-config.e2e-spec.ts` (el helper compartido `patch`
y el caso de acceso no autenticado) a la nueva ruta en español, junto con
el comentario introductorio del archivo. No se agregaron casos nuevos: la
cobertura de rol/tenant/validación/auditoría que TASK-83 ya había escrito
para este endpoint sigue siendo exactamente la misma, sólo que ahora
ejercita la ruta correcta — la ruta vieja en inglés dejó de existir, así
que cualquier intento de ejercitarla habría fallado con 404 y delatado un
resto sin actualizar.

Suite completa: 38 suites unitarias / 430 pruebas en verde; 37 suites e2e
/ 439 pruebas en verde (`--runInBand`). Lint limpio y verificación de tipos
(`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-106-patient-inactivity-route-spanish` (creada desde
  `origin/main` fresco, tras el merge de TASK-102). Pusheada a `origin`;
  PR abierto, no fusionado aún.
- Ticket: TASK-106 ("[CORRECCIÓN] Segmento de ruta en inglés en
  /admin/configuracion/patient-inactivity-months"), corrección a TASK-83
  (punto de acceso administrativo del umbral de inactividad,
  [[FASE-2_PROMPT-12]]), que a su vez corrige TASK-29. Misma convención de
  bitácora dedicada para correcciones pequeñas dentro de la fase del
  ticket que corrigen que TASK-98 ([[FASE-2_PROMPT-13]]) y las de Fase 1/3/4.
