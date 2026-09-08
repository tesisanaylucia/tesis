# Fase 1 — Profesionales (backend) — la edición de un profesional auditaba sin indicar qué campos cambiaron (TASK-136, corrección a PROF-11/SEC-3)

## Contexto

Una auditoría automatizada de código contra las fuentes de verdad (SRS),
corrida el 20 de agosto de 2026, generó el hallazgo PROF-11 —confirmado de
forma independiente por una segunda auditoría bajo el identificador SEC-3,
el mismo hallazgo detectado dos veces—: el método `update` de
`ProfessionalsService` construía su entrada de auditoría sin el detalle de
qué campos había cambiado la petición, a diferencia del resto del código
—`PatientsService`, `PatientProfessionalsService`, el importador de
pacientes y, dentro del propio módulo de turnos, la edición de un
feriado—, que ya usa `changedFields(dto)` para ese propósito. La
consecuencia concreta que señaló el hallazgo: dos ediciones distintas sobre
el mismo profesional —por ejemplo, cerrar la aceptación de pacientes nuevos
frente a cambiar el filtro de edad— dejaban entradas de auditoría
idénticas, indistinguibles entre sí, lo que vacía de contenido la traza
exigida por la Ley 25.326 para reconstruir qué cambió y cuándo. Por
tratarse de un hallazgo generado automáticamente, la tarea exigía
validación humana antes de intervenir: se confirmó contra el código
vigente, se comprobó que seguía aplicando y se determinó su alcance exacto
antes de corregir.

## Qué se implementó

Se agregó `detail: { fields: changedFields(dto) }` a la entrada de
auditoría que ya escribía `update()`, siguiendo al pie de la letra el mismo
patrón que ya usa `HolidaysService.update` —una escritura de un único
objeto envuelta en una transacción, con el cálculo de `changedFields`
insertado directamente en la llamada a `audit.log`—, por ser la forma más
cercana en el código existente a la de este método. `changedFields` ya
existía como utilidad compartida (`src/audit/changed-fields.ts`) desde
antes de esta tarea: no se escribió lógica nueva, solo se conectó el método
al mecanismo que el resto del sistema ya usaba.

## Decisiones y por qué

**No se modificó `updateConfiguration`, el otro método de escritura del
mismo servicio, pese a compartir una carencia parecida.** Ese método ya
escribe un `detail` propio, `{ configuration: true }`, que distingue una
edición de configuración de una edición general del profesional pero
tampoco nombra qué campo puntual cambió dentro de esa configuración. El
hallazgo que originó esta tarea nombra explícitamente solo `update()` y
las líneas exactas de ese método; extender el alcance a
`updateConfiguration` sin un hallazgo propio que lo pidiera habría sido
una corrección no solicitada sobre un método que, a diferencia de
`update()`, sí deja alguna distinción en su traza. Queda documentado aquí
como una carencia real pero fuera del alcance de este ticket, a la espera
de un hallazgo o tarea que la nombre.

**Se prefirió el patrón de `HolidaysService.update` al de
`PatientsService.update`.** Este último calcula `changedFields(dto)` antes
de la transacción y omite por completo la escritura —y la entrada de
auditoría— cuando la lista queda vacía, una optimización deliberada para
no acumular entradas de `UPDATE` sin efecto. `ProfessionalsService.update`
no tenía esa optimización antes de esta tarea y el hallazgo tampoco la
pedía, así que agregarla habría sido, otra vez, una corrección no
solicitada; se limitó el cambio a la única carencia señalada.

## Alternativas descartadas

- **Extender también `updateConfiguration` a `changedFields(dto)`**:
  descartada por no formar parte del hallazgo que originó la tarea; ver
  arriba.
- **Adoptar la optimización de "PATCH vacío no audita" de
  `PatientsService.update`**: descartada por la misma razón — el hallazgo
  señala únicamente la falta de detalle por campo, no la ausencia de esa
  optimización.

## Entidades / puertos / adaptadores tocados

- `src/professionals/professionals.service.ts`: `update()` ahora pasa
  `detail: { fields: changedFields(dto) }` a `audit.log`, importando la
  utilidad compartida `changedFields` ya usada por otros módulos.
- Ningún cambio de esquema: `AuditLog.detail` ya existía como columna JSON
  desde la implementación original de la auditoría.

## Tests y qué validan

- Se amplió la prueba de extremo a extremo ya existente "lets the owning
  professional edit their own record" (`test/professionals-abm.e2e-spec.ts`)
  para leer la entrada de auditoría que la edición deja y verificar que su
  `detail` sea exactamente `{ fields: ['name'] }`, el campo que la petición
  de la prueba efectivamente envía — la forma directa de probar que el
  hallazgo quedó corregido, no solo que el método sigue respondiendo 200.
- Ejecución: suite unitaria y de extremo a extremo completas en verde (83
  conjuntos unitarios, 809 pruebas; 51 conjuntos de extremo a extremo, 572
  pruebas, `--runInBand`), análisis estático sin advertencias. Datos
  ficticios, sin contenido clínico ni datos personales reales.

## Figuras pendientes

Ninguna figura nueva; la corrección no introduce un flujo o diagrama
distinto de los ya registrados para el módulo.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-136-professional-update-audit-fields`
  (creada a partir de `origin/main`, ya con TASK-135 integrada).
- Ticket: TASK-136 ("La edición de profesional audita sin indicar qué
  campos cambiaron"), hallazgo PROF-11 (= SEC-3) de dos auditorías
  automatizadas independientes sobre
  `src/professionals/professionals.service.ts`, prioridad media, validado y
  corregido en la misma tarea.
