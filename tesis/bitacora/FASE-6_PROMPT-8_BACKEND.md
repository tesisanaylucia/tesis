# Fase 6 — Cerradura TTLock (backend) — verificación: reprogramar ya reenvía el código de acceso (TASK-188, hallazgo duplicado de TASK-166/CER 1)

## Qué se implementó

TASK-188 ("[MEDIO] Reprogramar un turno no reenvía el código de acceso al
paciente") es una tarea generada automáticamente por una auditoría
multi-agente fechada 2026-08-28, contra las mismas fuentes de verdad
(anteproyecto de tesis + SRS) que ya había producido el hallazgo CER 1 el
2026-08-26. Describe exactamente el mismo problema que TASK-166 corrigió:
`reissueAccessCode` (citada en la descripción del ticket por su ubicación
de línea previa a esa corrección, `appointments.service.ts:919-951`)
revocaba el código viejo y generaba uno nuevo al reprogramar, pero —según
el propio comentario del código que la auditoría citó— "deliberadamente"
no reenviaba `ACCESS_CODE_DELIVERY`.

Verificado contra `main` (`git fetch origin` + inspección de
`origin/main`, no una referencia local desactualizada — el mismo criterio
que ya había evitado un falso positivo idéntico en TASK-81) que la
corrección de TASK-166 ya está fusionada (PR #130, commit `d25c796`,
2026-09-11) con cobertura de prueba dedicada para los dos llamadores de
`reissueAccessCode` (reprogramación inmediata y aplicación diferida de una
`RescheduleOffer`). El comentario "deliberadamente no reenvía" que la
auditoría citó ya no existe en el código actual: fue reemplazado por el
comentario que documenta la corrección (ver `FASE-6_PROMPT-7_BACKEND.md`).
No se requirió ningún cambio de código adicional — se creó la rama de la
tarea, se corrió la suite dirigida (`appointments-rescheduling.service.spec.ts`
+ `appointments.service.spec.ts`, 126 tests, verde) para confirmar el
estado sin modificarlo, y no se abrió ningún commit ni PR de código por no
haber nada que revisar.

## Decisiones y por qué

**Se documenta como verificación, no como implementación nueva.** El
propio flujo de esta skill pide describir el qué y, cuando el porqué no es
evidente, dejar constancia en lugar de inventarlo — en este caso el qué es
"nada que corregir", así que la entrada de bitácora registra la
verificación en sí, no una corrección inexistente.

**Segundo caso de un hallazgo de auditoría duplicado/obsoleto para el
mismo problema, tras TASK-81.** TASK-81 ya había dejado precedente:
verificar contra el `origin/main` real (no un checkout stale) antes de
asumir que un hallazgo de auditoría sigue vigente, porque la ejecución de
la auditoría puede quedar desalineada con merges que ya ocurrieron entre
la fecha del hallazgo y la fecha en que se trabaja el ticket. Aquí la
brecha es más corta (dos días entre el hallazgo de TASK-166, 2026-08-26, y
el de TASK-188, 2026-08-28) — probablemente dos ejecuciones de la misma
auditoría multi-agente sobre el mismo estado del código detectaron el
mismo problema por separado, y sólo una de las dos tareas resultantes
(TASK-166) se trabajó antes de que la otra (TASK-188) llegara a esta
sesión.

## Entidades / puertos / adaptadores tocados

Ninguno — tarea de verificación, sin cambios de código.

## Tests agregados o modificados

Ninguno agregado. Se corrió la suite existente
(`appointments-rescheduling.service.spec.ts` + `appointments.service.spec.ts`,
126 tests) para confirmar que la corrección de TASK-166 sigue vigente y
verde, sin agregar cobertura nueva sobre un comportamiento ya cubierto por
`FASE-6_PROMPT-7_BACKEND.md`.

## Figuras pendientes

Ninguna nueva — ver la figura pendiente ya registrada en
`FASE-6_PROMPT-7_BACKEND.md` para la corrección real.

## Componente y referencia

Backend. Rama `feature/TASK-188-verify-reschedule-access-code-resend`,
creada desde `origin/main` (ya incluye TASK-166 y TASK-178 fusionados),
pusheada sin commits — no hubo código que cambiar. Comentario de
verificación agregado en el propio ticket de Jira (TASK-188), cerrado como
resuelto por duplicar un hallazgo ya corregido en TASK-166.
