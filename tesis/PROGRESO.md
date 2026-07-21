# Progreso de la tesis

Estado de avance por subsección. Se actualiza como parte del paso 6 del
flujo descripto en `documentacion-tesis/SKILL.md`, cada vez que una tarea
validada amplía una subsección.

Estados posibles: `sin iniciar`, `borrador` (texto conceptual escrito, con
citas y datos volátiles pendientes de verificación), `en progreso`,
`completo (pendiente de revisión)`.

## Capítulo 1: Introducción

| Subsección | Estado | Último prompt | Componente |
|---|---|---|---|
| 1.1 Antecedentes | sin iniciar | — | — |
| 1.2 Formulación del Problema y Justificación | sin iniciar | — | — |
| 1.3 Objetivos | sin iniciar | — | — |
| 1.4 Metodología | sin iniciar | — | — |

## Capítulo 2: Marco Teórico

| Subsección | Estado | Último prompt | Componente |
|---|---|---|---|
| 2.1 Secretarías virtuales / asistentes conversacionales en salud | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.2 LLM e IA conversacional (function calling) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.3 WhatsApp Business Platform (Cloud API) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.4 Arquitectura de software (monolito modular, hexagonal, multi-tenancy, reglas como datos) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.5 Stack backend (NestJS, Prisma, PostgreSQL) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.6 IoT y control de acceso (TTLock) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.7 App móvil (React Native) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.8 Marco normativo (Ley 25.326 y 26.529) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |

## Capítulo 3: Solución PSIQUE

| Subsección | Estado | Último prompt | Componente |
|---|---|---|---|
| 3.1 Arquitectura general | sin iniciar | — | — |
| 3.2.0 Fundaciones | en progreso | FASE-0_PROMPT-6 (revisión del modelo de datos: integridad referencial, normalización y claves foráneas compuestas) | backend |
| 3.2.1 Profesionales | completo (pendiente de revisión) | FASE-1_PROMPT-6 (TASK-26, seed del plantel del piloto y cierre de la cobertura de pruebas) | backend |
| 3.2.2 Pacientes | en progreso | FASE-2_PROMPT-3 (TASK-29, tipo de paciente, última consulta y prioridad por vínculo) | backend |
| 3.2.3 Motor de Turnos | sin iniciar | — | backend |
| 3.2.4 Notificaciones y Scheduler | sin iniciar | — | backend |
| 3.2.5 Capa conversacional y WhatsApp | sin iniciar | — | backend |
| 3.2.6 Cerradura TTLock | sin iniciar | — | backend |
| 3.2.7 App móvil del profesional | sin iniciar | — | movil |
| 3.2.8 Endurecimiento, cumplimiento y piloto | sin iniciar | — | backend |
| 3.3 Presentación de la solución | sin iniciar | — | — |

## Capítulo 4: Resultados, Conclusiones y Trabajo Futuro

| Subsección | Estado | Último prompt | Componente |
|---|---|---|---|
| Resultados | sin iniciar | — | — |
| Conclusiones | sin iniciar | — | — |
| Trabajo futuro | sin iniciar | — | — |

## Notas

- "FASE-0_PROMPT-EJEMPLO" (backend) es la entrada de muestra creada al
  configurar la skill `documentacion-tesis`, no una tarea real numerada;
  sirve solo para validar el formato. Ver
  `tesis/bitacora/FASE-0_PROMPT-EJEMPLO_BACKEND.md`.
- El Capítulo 2 completo (2.1 a 2.8) está en `tesis/capitulos/cap2_marco_teorico_a.md`
  (2.1–2.4) y `cap2_marco_teorico_b.md` (2.5–2.8). Se marca `borrador` y no
  `completo` porque el texto contiene marcadores `[CITA: ...]` sin fuente
  verificada (ver `tesis/referencias_pendientes.md`) y marcadores
  `[VERIFICAR: ...]` sobre datos volátiles (versiones, precios, límites de
  API) todavía no confirmados. Pasa a `completo (pendiente de revisión)`
  recién cuando ambos tipos de marcador se resuelvan.
