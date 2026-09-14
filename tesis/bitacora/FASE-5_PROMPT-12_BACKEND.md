# Fase 5 — Capa conversacional y WhatsApp (backend) — la edición de una FAQ auditaba campos que no cambiaron (TASK-180, corrección a SEC 9)

## Contexto

Una auditoría automatizada de código contra las fuentes de verdad (SRS),
corrida el 26 de agosto de 2026, generó el hallazgo SEC 9: el método
`update` de `FaqService` construía su entrada de auditoría con
`Object.keys(dto)` en lugar de `changedFields(dto)`, el mismo patrón ya
señalado antes en `ProfessionalsService` bajo TASK-136 (hallazgos PROF-11 /
SEC-3), esta vez en un servicio distinto. `Object.keys(dto)` no es la lista
de campos que efectivamente trajo la petición: `class-transformer`
instancia todas las propiedades declaradas en `UpdateFaqDto`, así que un
`PATCH` que sólo envía `answer` produce igual un objeto con la clave
`question` (en `undefined`). Una entrada construida así afirma un cambio
que nunca ocurrió, lo que vacía de contenido la traza que la Ley 25.326
exige para reconstruir qué cambió y cuándo — el mismo razonamiento que
motivó TASK-136. Por tratarse de un hallazgo generado automáticamente, la
tarea exigía validación humana antes de intervenir: se confirmó contra el
código vigente (la línea señalada seguía existiendo tal cual la describía
el hallazgo) antes de corregir.

## Qué se implementó

Se reemplazó `Object.keys(dto)` por `changedFields(dto)` en la entrada de
auditoría que ya escribía `update()`, importando la utilidad compartida
(`src/audit/changed-fields.ts`) que ya usan `PatientsService`,
`HealthInsurersService`, `HolidaysService`, `ProfessionalsService` y
`AbsencesService` para este mismo propósito. No se escribió lógica nueva:
sólo se conectó el método al mecanismo que el resto del sistema ya usaba,
siguiendo el mismo patrón puntual — una escritura dentro de una
transacción, con el cálculo de `changedFields` insertado directamente en
la llamada a `audit.log` — que ya usa `HealthInsurersService.update`, por
ser la forma más cercana en el código existente a la de este método (sin
la lógica adicional de reintento serializable de `HealthInsurersService`,
que este método tampoco tenía antes de la corrección).

## Decisiones y por qué

**No se adoptó la optimización de "PATCH vacío no audita" de
`PatientsService.update`.** Ese servicio calcula `changedFields(dto)` antes
de la transacción y omite por completo la escritura — y la entrada de
auditoría — cuando la lista queda vacía. `FaqService.update` no tenía esa
optimización antes de esta tarea y el hallazgo tampoco la pedía: como ya se
razonó en TASK-136 ante la misma disyuntiva sobre `ProfessionalsService`,
agregarla habría sido una corrección no solicitada fuera del alcance
exacto del hallazgo SEC 9, que nombra únicamente la falta de precisión del
detalle, no la ausencia de esa optimización.

## Alternativas descartadas

- **Adoptar la optimización de "PATCH vacío no audita" de
  `PatientsService.update`**: descartada por la misma razón documentada en
  TASK-136 — el hallazgo señala únicamente la falta de detalle por campo,
  no la ausencia de esa optimización.

## Entidades / puertos / adaptadores tocados

- `src/faq/faq.service.ts`: `update()` ahora pasa
  `detail: { fields: changedFields(dto) }` a `audit.log`, importando la
  utilidad compartida `changedFields` ya usada por otros módulos.
- Ningún cambio de esquema: `AuditLog.detail` ya existía como columna JSON
  desde la implementación original de la auditoría.

## Tests y qué validan

- Se agregó una prueba unitaria nueva a `faq.service.spec.ts`,
  "excludes untouched fields even when they are present as undefined
  keys", que reproduce directamente el defecto: llama a `update` con un
  DTO que trae la clave `question` en `undefined` (la forma en que
  `class-transformer` deja un campo no enviado) y verifica que el `detail`
  de la entrada de auditoría sea exactamente `{ fields: ['answer'] }`, no
  `['question', 'answer']` — la forma directa de probar que el hallazgo
  quedó corregido, no sólo que el método sigue respondiendo 200.
- Ejecución: suite unitaria completa en verde (90 conjuntos, 976 pruebas) y
  suite de extremo a extremo completa en verde (53 conjuntos, 639 pruebas,
  `--runInBand`); análisis estático (ESLint) y verificación de tipos
  (`tsc --noEmit`) sin advertencias. Datos ficticios, sin contenido clínico
  ni datos personales reales.

## Figuras pendientes

Ninguna figura nueva; la corrección no introduce un flujo o diagrama
distinto de los ya registrados para el módulo.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-180-faq-changed-fields` (creada a
  partir de `origin/main`).
- Ticket: TASK-180 ("FaqService.update registra campos que no cambiaron"),
  hallazgo SEC 9 de una auditoría automatizada sobre `src/faq/faq.service.ts`,
  prioridad media, validado y corregido en la misma tarea.
