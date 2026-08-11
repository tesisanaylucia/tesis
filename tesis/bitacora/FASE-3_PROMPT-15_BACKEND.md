# Fase 3 — Motor de Turnos (backend) — eliminación de `franja_extra_nuevos` por dato muerto (TASK-86, corrección a TASK-37/TASK-24)

## Qué se implementó

TASK-86 fue una tarea de auditoría: una revisión automatizada del código
detectó que el campo `PROFESIONAL.franja_extra_nuevos`
(`Professional.newPatientExtraSlot`, entero en horas, modelado en TASK-24 /
P1.4) se persistía y se exponía en la configuración del profesional, pero
`AvailabilityService.getNewPatientSlots` — la lógica de TASK-37 / P3.4 que
aplica la regla de paciente nuevo al generar slots — nunca lo leía. El
ticket pedía confirmar con "el dueño del motor de scheduling" si el campo
debía gatear cuánto tiempo antes/después del horario habitual se agrega el
turno extra, o si había quedado obsoleto al definirse
`modalidad_franja_nueva` (`NewPatientSlotMode`) como enum de tres modos que
ya resuelve el posicionamiento, y documentar cualquiera de las dos
decisiones que se tomara.

Un grep de todo el repo confirmó el hallazgo de la auditoría: fuera del
DTO (`UpdateProfessionalConfigDto`), el presenter (`ProfessionalResponse`),
el schema de Prisma, el seed y los tests que cubren esos tres puntos, no
había ningún otro archivo que leyera `newPatientExtraSlot`.

**Decisión: el campo quedó obsoleto y se eliminó.** Releer el propio texto
del SRS que TASK-37 cita como fuente de verdad ("Franja extra para
pacientes nuevos: el profesional configura en qué franja de su jornada
puede aceptar un paciente nuevo — primer turno del día / último turno del
día / dentro de la franja") y la definición del "doble turno" ("la reserva
de primera sesión crea DOS turnos consecutivos vinculados... duración de
cada uno = duracion_consulta del profesional") muestra que la mecánica que
efectivamente se implementó en TASK-37 no necesita ninguna cantidad de
horas: la "franja extra" para un paciente nuevo es, literalmente, un
segundo turno de duración normal, y su posición dentro de la jornada la
decide por completo el enum de tres modos. La cantidad en horas de
`franja_extra_nuevos` nunca tuvo, en el diseño que terminó implementado, un
lugar que ocupar.

## Decisiones y por qué

**Eliminar en vez de aplicar.** Aplicar el valor habría significado
inventar una regla de posicionamiento ("¿desplazar el turno extra N horas
antes/después de qué?") que ni el SRS ni TASK-37 piden en ningún lugar, y
que además competiría con los tres modos de `modalidad_franja_nueva` — que
el propio ticket marca explícitamente como fuera de alcance modificar. Con
la explicación alternativa (obsolescencia) respaldada tanto por el código
como por el propio texto de la fuente de verdad, mantener una columna sin
lector es peor que quitarla: es dato muerto que un futuro mantenedor podría
asumir, erróneamente, que todavía hace algo.

**Mismo patrón que TASK-74** (eliminación documentada de `Diagnosis`,
campo agregado fuera de alcance en P0.8): migración Prisma dedicada que
deja constancia en su propio comentario del motivo, en vez de una
migración muda; y la decisión documentada en un comentario de código (el
DTO) además de en el ticket de Jira, no solo en la bitácora de tesis.

**Se documentó la decisión como comentario en el propio ticket de Jira**
(TASK-86), tal como piden sus criterios de aceptación, además de en esta
bitácora — a diferencia de TASK-81 (verificación pura, sin código), acá sí
hay cambios de código que respaldar con contexto en el propio repo.

## Alternativas descartadas

- **Aplicar el valor como desplazamiento adicional dentro de
  `WITHIN_SCHEDULE`** (p. ej. "el doble turno debe empezar al menos N horas
  después de la apertura de la jornada"): descartada porque ninguna fuente
  (SRS, ticket de TASK-37, comentarios existentes en
  `availability.service.ts`) describe esa semántica; habría sido una regla
  inventada sin respaldo.
- **Dejar el campo sin usar pero documentado como "reservado para uso
  futuro"**: descartada por el mismo argumento que TASK-74 usó para
  `Diagnosis` — dato muerto sin ningún consumidor planeado es peor que
  eliminarlo y, si hiciera falta en el futuro, se puede volver a agregar
  con un diseño concreto en la mano.

## Entidades / puertos / adaptadores tocados

- `Professional` (Prisma): se elimina la columna `newPatientExtraSlot` vía
  migración `20260811233000_remove_new_patient_extra_slot`.
- `UpdateProfessionalConfigDto` (`src/professionals/dto/update-professional-config.dto.ts`):
  se quita el campo, su validación (`@Min(0)`/`@Max(5)`) y se reescribe el
  comentario de cabecera del archivo.
- `ProfessionalResponse` / `toProfessionalResponse`
  (`src/professionals/professional.presenter.ts`): se quita el campo de la
  interfaz y del mapeo.
- `ProfessionalsService.updateConfiguration`
  (`src/professionals/professionals.service.ts`): se quita del `data` que
  se pasa a `tx.professional.update`.
- `prisma/seed.ts`: se quita de `SeedProfessional`, del roster piloto y del
  mapeo a `Prisma.ProfessionalUpdateInput`.
- `postman/psique-backend.postman_collection.json`: se quita del cuerpo de
  ejemplo y de la descripción del endpoint `PATCH /profesionales/:id/configuracion`.

No se tocó `AvailabilityService` ni `NewPatientSlotMode`: la lógica de
posicionamiento de TASK-37 queda exactamente igual.

## Tests y qué validan

- `src/professionals/professionals.service.spec.ts`: se quita el campo de
  los fixtures (`buildProfessional`, `expectedData`) y de las aserciones
  del test parametrizado por modalidad.
- `test/professional-config.e2e-spec.ts`: se quita el campo de la interfaz
  `ProfessionalBody` y de los `send()`/aserciones de los tests que seguían
  vigentes (modalidad, actualización parcial, limpieza con `null`). Se
  eliminan dos casos que existían únicamente para cubrir este campo
  (`accepts a zero newPatientExtraSlot`, `rejects a negative
  newPatientExtraSlot (400)`) — con la columna fuera del DTO, `whitelist:
  true` simplemente descarta la propiedad en vez de rechazarla, así que la
  segunda prueba ya no tendría nada que verificar.
- `test/professionals-entities.e2e-spec.ts`: la aserción que comprobaba que
  el campo nace en `null` se reemplaza por la equivalente sobre
  `newPatientSlotMode`, que sigue siendo un campo configurado en tickets
  posteriores.
- Ejecución: 30 suites unitarias / 335 pruebas en verde; 33 suites e2e /
  409 pruebas en verde (`--runInBand`, contra Postgres local vía
  `docker-compose up -d db` + `prisma migrate deploy`). Lint limpio.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-86-remove-new-patient-extra-slot`
  (creada desde `origin/main` fresco, tras el merge de TASK-79/TASK-83 a
  TASK-85/TASK-89). Pusheada a `origin`, no fusionada aún.
- Ticket: TASK-86 ("[CORRECCIÓN] TASK-37 – franja_extra_nuevos no se aplica
  en el cálculo de slots"). Referencia también a TASK-24 (P1.4, origen del
  campo) y TASK-37 (P3.4, [[FASE-3_PROMPT-4]], regla de paciente nuevo que
  terminó sin necesitarlo). Sigue el mismo patrón documental que TASK-74
  ([[FASE-1_PROMPT-4]]) y la misma convención de bitácora dedicada para
  correcciones pequeñas que TASK-84 ([[FASE-2_PROMPT-11]]) y TASK-79/TASK-81
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]]).
