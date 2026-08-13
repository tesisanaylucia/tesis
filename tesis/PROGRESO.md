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
| 2.4 Arquitectura de software (monolito modular, hexagonal, multi-tenancy, reglas como datos, ejecución de tareas programadas) | borrador | Redacción Marco Teórico completo (2026-07-16); ampliada con la extensión de scheduling en FASE-4_PROMPT-6 (TASK-76) | — |
| 2.5 Stack backend (NestJS, Prisma, PostgreSQL) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.6 IoT y control de acceso (TTLock) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.7 App móvil (React Native) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |
| 2.8 Marco normativo (Ley 25.326 y 26.529) | borrador | Redacción Marco Teórico completo (2026-07-16) | — |

## Capítulo 3: Solución PSIQUE

| Subsección | Estado | Último prompt | Componente |
|---|---|---|---|
| 3.1 Arquitectura general | sin iniciar | — | — |
| 3.2.0 Fundaciones | en progreso | FASE-0_PROMPT-7 (TASK-93, corrección a TASK-16/TASK-72: restricción `CHECK` que ata `professionalId` al rol en `User`, con corrección de un efecto colateral en la reconciliación del seed) | backend |
| 3.2.1 Profesionales | completo (pendiente de revisión) | FASE-1_PROMPT-9 (TASK-92, corrección a TASK-21/22: restricción de unicidad `(professionalId, type, number)` sobre `License`, en reemplazo del índice simple por profesional, con traducción de la violación a un 400 legible en alta y edición) | backend |
| 3.2.2 Pacientes | completo (pendiente de revisión) | FASE-2_PROMPT-12 (TASK-83: punto de acceso administrativo para configurar el umbral de inactividad, con tope de doce meses aplicado también en la lectura) | backend |
| 3.2.3 Motor de Turnos | en progreso | FASE-3_PROMPT-18 (TASK-96, corrección a TASK-40: `DELETE /lista-espera/:id` podía tirar 500 por violación de FK cuando la entrada tenía una `WaitlistOffer` REJECTED/EXPIRED todavía apuntándole, ya que sólo el camino de aceptación de una oferta limpiaba esa referencia; corregido generalizando el mismo mecanismo — anular la referencia dentro de la misma transacción, antes del borrado — al camino de baja explícita); FASE-3_PROMPT-17 (TASK-95, corrección a TASK-39: `rescheduleCore` no reseteaba `confirmationRequestedAt`/`reminderSentAt` al reprogramar un turno, desincronizando las guardas de idempotencia de los crons de auto-cancelación y recordatorio respecto de la fecha nueva; corregido reseteando ambos campos en el mismo punto de escritura); FASE-3_PROMPT-16 (TASK-94, corrección a TASK-40: `GET /lista-espera/profesional/:id` sólo chequeaba tenant, no ownership dentro del tenant; fuga de datos de pacientes entre profesionales de la misma organización, corregida reusando `assertOwnerOrAdmin`) | backend |
| 3.2.4 Notificaciones y Scheduler | en progreso | FASE-4_PROMPT-2 (TASK-43, P4.2: cron de confirmación a 24h — ventana de detección 23h-25h, columna `confirmationRequestedAt` dedicada a la idempotencia en vez de reutilizar `confirmedAt`, envío vía MessagingPort stub y registro en auditoría); FASE-4_PROMPT-3 (TASK-44, P4.3: cron de autocancelación a las 4h sin respuesta — motivo NO_CONFIRMATION agregado al enum de cancelación, sin extensión del plazo por fin de semana, disparo de ReassignmentPort); FASE-4_PROMPT-4 (TASK-45, P4.4: cron de recordatorio configurable por inquilino — columna `reminderSentAt` dedicada, solo turnos CONFIRMADO — y andamiaje del futuro job de expiración de códigos TTLock, módulo nuevo `access-codes` con un placeholder cada 15 minutos hasta que la entidad AccessCode exista en M6); FASE-4_PROMPT-5 (TASK-82, P4.5: timeout real de 4h para la oferta de turno en lista de espera bajo reasignación AUTOMÁTICA — tabla nueva `WaitlistOffer`, WaitlistResponsePort reemplaza su stub síncrono por un adaptador real, el recorrido de la lista de espera pasa a ofrecer a un candidato por vez y se detiene, `WaitlistOfferTimeoutCron` nuevo cada 15 min vence las ofertas vencidas y avanza el recorrido); FASE-4_PROMPT-6 (TASK-76, P7.b: canal de notificaciones in-app para el profesional — modelo `Notification` sin columna de organización propia, anclado a través del profesional; `GET /notificaciones`, marcar una o todas como leídas, y el listado administrativo por profesional; disparado desde la cancelación de turno y desde la reserva por reasignación; solicitud de receta y error de cerradura modelados sin disparador real todavía); FASE-4_PROMPT-7 (TASK-91, corrección a TASK-82: índice faltante `@@index([organizationId, waitlistEntryId])` en `WaitlistOffer` para respaldar su tercera FK compuesta, sin cambios de lógica); FASE-4_PROMPT-8 (TASK-90, corrección a TASK-76: `Notification.appointmentId`/`.prescriptionRequestId` pasan de FK simple a FK compuesta contra `professionalId` — la columna que su propio padre determina, ya que `Notification` no lleva `organizationId` propio —, con `ON DELETE CASCADE` y los dos índices de soporte faltantes) | backend |
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
