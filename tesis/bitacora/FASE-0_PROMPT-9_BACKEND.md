# Fase 0 — Fundaciones (backend) — el `detail` de la traza de auditoría dejaba de nombrar campos y pasaba a guardar valores en tres servicios (TASK-130, corrección a TASK-17/P0.6)

## Qué se implementó

Se estandarizó el `detail` de la entrada de auditoría en tres puntos del
código —`ProfessionalHealthInsurersService.replaceForProfessional`,
`PatientInactivityService.setThresholdMonths` y las tres operaciones de
`HolidaysService` (alta, edición y baja)— que registraban valores nuevos o
de configuración en lugar de limitarse a nombrar qué cambió, en
contradicción con la regla ya vigente en el proyecto desde P0.6 ("el
`detail` de una entrada nombra campos, nunca valores") y con
`WorkingHoursService.replaceForProfessional`, una operación
estructuralmente idéntica a la de obras sociales aceptadas (borrado y
recreación completa del vínculo) que ya seguía la regla correctamente,
registrando solo la cantidad de filas escritas.

Una tarea de auditoría de código automatizada contra el documento de
requisitos, ejecutada el 20 de agosto de 2026, señaló la inconsistencia
(hallazgo SEC-4) comparando ambos pares de servicios estructuralmente
equivalentes. No existía ningún lint ni prueba que hiciera cumplir la
regla de forma transversal —cada servicio la respetaba o no según el
criterio de quien lo escribió—, así que la corrección se limitó a alinear
los cuatro puntos señalados con el patrón ya correcto, sin tocar
`AuditService` ni la regla misma.

## Decisiones y por qué

**Reemplazar la lista completa de identificadores por su cantidad, en
`ProfessionalHealthInsurersService`.** El vínculo entre un profesional y
las obras sociales que acepta se reemplaza por completo en cada
operación, igual que la grilla de horarios habituales; ninguna de las dos
operaciones tiene un "antes" y un "después" que reconciliar campo por
campo, así que la cantidad de filas escritas —ya el patrón elegido para
horarios— es la información equivalente y suficiente para saber qué tan
grande fue el cambio sin reproducir su contenido en la traza.

**Retirar el valor de meses del `detail` de
`PatientInactivityService.setThresholdMonths`, conservando solo la clave
de configuración modificada.** A diferencia de una edición de paciente,
donde varios campos pueden cambiar a la vez y `changedFields(dto)` ya
resuelve cuáles, esta operación modifica un único valor de configuración
por tenant; nombrar la clave que cambió alcanza para saber qué se tocó, y
el valor numérico en sí no aporta nada a una auditoría de responsabilidad
—quién cambió la política de inactividad y cuándo— que el valor mismo no
sea indispensable para reconstruir de todos modos, ya que la fila vigente
en `OrganizationConfig` siempre lo tiene disponible.

**Diferenciar, dentro de `HolidaysService`, entre las tres operaciones en
lugar de aplicar una única corrección uniforme.** La creación de un
feriado no tiene otro campo no sensible que registrar aparte del
identificador ya presente como entidad afectada —fecha y descripción son
el contenido íntegro del recurso creado—, así que se optó por no escribir
ningún `detail` en ese caso, en lugar de inventar una lista de campos que
no dice más que "se creó". La edición sí tiene un campo distinguible del
resto —únicamente la descripción es editable, la fecha es la clave
natural del recurso— y se alineó con el patrón ya usado en el resto del
código para actualizaciones parciales (`detail: { fields:
changedFields(dto) }`), en lugar de seguir registrando la fecha, que
además no es lo que cambia en una edición. La baja conservó la cantidad de
turnos afectados por la liberación del día, ya un conteo legítimo bajo la
misma regla que permite el de horarios, y solo se le retiró la fecha
redundante con el identificador de la entidad ya presente en la entrada.

## Alternativas descartadas

Se consideró mantener el identificador de la obra social o de la
configuración junto con su cantidad o clave, replicando el patrón de
`WorkingHoursService`, que sí repite el identificador del profesional pese
a que ya figura como entidad afectada. Se descartó porque en ese caso el
identificador repetido es el ancla del vínculo (a qué profesional
pertenece la grilla u obra social), mientras que el valor retirado aquí
—la lista de identificadores de obra social, la cantidad de meses, la
fecha del feriado— es el contenido del cambio, no su ancla; conservarlo no
habría corregido la inconsistencia que motivó la tarea.

Se consideró también escribir una prueba o regla de lint genérica que
impida a futuro pasar un valor no permitido dentro de `detail` en
cualquier llamada a `AuditService.log`. Se descartó por quedar fuera del
alcance del hallazgo puntual —una convención automatizable existe, pero
decidir su forma (lista blanca de claves, tipo de dato, expresión
regular) es una tarea de diseño propia que el hallazgo no pedía resolver
— y quedó fuera de esta corrección.

## Entidades / puertos / adaptadores tocados

- `src/professionals/professional-health-insurers.service.ts`:
  `replaceForProfessional` registra `{ professionalId, count }` en lugar
  de la lista completa de identificadores de obra social.
- `src/patients/patient-inactivity.service.ts`: `setThresholdMonths`
  registra solo la clave de configuración modificada, sin el valor nuevo.
- `src/holidays/holidays.service.ts`: `create` deja de escribir `detail`;
  `update` registra `{ fields: changedFields(dto) }`; `remove` conserva
  `affectedAppointments` y retira la fecha redundante con el
  identificador de la entidad.

## Tests y qué validan

- `src/holidays/holidays.service.spec.ts` y
  `src/patients/patient-inactivity.service.spec.ts` (existentes,
  actualizados): las aserciones que verificaban la forma anterior de
  `detail` —incluyendo un título de prueba que decía explícitamente "a qué
  valor" cambió algo— se ajustaron a la nueva forma, sin agregar
  cobertura nueva: el comportamiento que prueban (que la entrada se
  escribe en la misma transacción que la mutación) no cambió, solo el
  contenido del campo.
- `test/patient-inactivity-config.e2e-spec.ts` (existente, actualizado):
  la aserción contra una base de datos real sobre el `detail` persistido
  se ajustó de la misma manera.
- Suite completa verde: 132 suites / 1354 tests con `--runInBand` contra
  PostgreSQL local (la corrida en paralelo por defecto mostró fallas
  intermitentes en suites de turnos y configuración de profesionales no
  relacionadas con este cambio, por la misma contención sobre la base de
  datos compartida entre workers documentada en tareas previas;
  desaparecen al correr secuencial). Lint y `tsc --noEmit` sin errores en
  cada archivo tocado.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-130-audit-detail-consistency`
  (creada a partir de `origin/main` fresco, tras la fusión de TASK-129).
- Ticket: TASK-130 ("El detail del audit log a veces guarda valores nuevos
  en vez de solo nombres de campo"), tarea de corrección generada a partir
  de la auditoría de código vs. SRS del 20 de agosto de 2026 (hallazgo
  SEC-4), sin un ticket de implementación previo específico al que
  corregir — los tres servicios tocados pertenecen a los módulos de
  Profesionales (obras sociales aceptadas), Pacientes (umbral de
  inactividad) y feriados administrativos (P3.b, TASK-78), documentados en
  las secciones 4.2, 4.3 y 4.4 respectivamente.
