# Fase 2 — Pacientes (backend) — fecha de última consulta a nivel paciente, campo agregado en la respuesta (TASK-84, corrección a TASK-29)

## Contexto

El diseño vigente registra la fecha de última consulta por cada vínculo entre
un paciente y un profesional, no a nivel del paciente, porque una sesión está
atada al profesional que la atendió y no a la persona en abstracto — la
decisión ya está fundamentada en la entrada de esta misma fase que introdujo
la columna (FASE-2_PROMPT-3, TASK-29). Sin embargo el documento de requisitos
menciona la "última consulta del paciente" en singular, un dato que hasta esta
tarea sólo podía obtenerse mediante una llamada aparte al punto de estado de
datos o calculando el máximo del lado cliente sobre la lista de vínculos que
ya trae la respuesta del paciente. La tarea, de prioridad baja, no cuestiona
el diseño por vínculo — lo confirma expresamente como correcto — sino que
agrega una proyección de conveniencia sobre la lectura.

## Qué se implementó

Se agregó un campo calculado `lastConsultationDate: Date | string | null` a
`PatientResponse`, con el mismo valor que ya respondía el punto de estado de
datos: el máximo de la fecha de última consulta entre los vínculos del
paciente con los profesionales que lo tratan. El campo no persiste nada nuevo
— no hay columna ni migración — y se recalcula en cada lectura a partir de los
mismos vínculos que la respuesta ya incluye. Un paciente sin ninguna consulta
registrada responde `null`.

## Decisiones y por qué

**El cálculo se extrajo a una función compartida en lugar de repetirse.** La
lógica del máximo ya existía, escrita para el punto de estado de datos: los
días de calendario se representan con relleno de ceros, de modo que el más
reciente es el máximo como cadena de texto, sin necesidad de convertir a
fecha. Repetir esa lógica en el mapeo de la respuesta del paciente habría
dejado dos comparaciones idénticas mantenidas por separado, expuestas a
divergir con el tiempo sin que ninguna prueba lo detectara — precisamente el
riesgo que otras partes de este módulo ya evitan mediante una única fuente de
verdad para una misma regla (la función pura de la regla del año, FASE-2_PROMPT-3).
Se extrajo la función a `patient.presenter.ts`, donde ya vive el resto de la
lógica de proyección del paciente, y tanto el mapeo de la respuesta como el
servicio de estado de datos la invocan sobre el conjunto de vínculos que cada
uno ya tiene resuelto — el paciente completo en un caso, el subconjunto
filtrado por profesional cuando se pide, en el otro.

**El campo se calcula sobre los vínculos ya narrowed para el rol del
llamante, sin reglas nuevas de acceso.** Para un llamante con rol
profesional, la lista de vínculos que la respuesta del paciente incluye ya
está acotada al propio vínculo (restricción descripta en FASE-2_PROMPT-2); el
campo agregado hereda esa restricción sin código adicional, porque opera
sobre la misma lista que ya llega recortada. No se introdujo ninguna
comprobación de autorización propia: el campo es una proyección sobre datos
que el llamante ya podía ver.

## Alternativas descartadas

- **Cambiar el modelo de datos para que la fecha de última consulta viva a
  nivel paciente**: descartada explícitamente por la propia tarea y por la
  justificación ya registrada en TASK-29 — una sesión pertenece a la relación
  con un profesional determinado, no a la persona.
- **Repetir el cálculo del máximo en el mapeo del paciente**: descartada por
  el riesgo de divergencia entre dos copias de la misma regla, señalado
  arriba.

## Entidades / puertos / adaptadores tocados

- `src/patients/patient.presenter.ts`: nueva función `latestConsultationDate`
  (el máximo compartido) y el campo `lastConsultationDate` en la interfaz
  `PatientResponse` y en `toPatientResponse`.
- `src/patients/patients.service.ts`: `getDataStatus` pasa a invocar la
  función compartida en lugar de la comparación que tenía escrita en línea.
- No se tocó el esquema de la base de datos ni el diseño por vínculo
  profesional-paciente.

## Tests y qué validan

- `test/patients-abmc.e2e-spec.ts` (nueva prueba): confirma que
  `GET /pacientes/:id` responde `null` para un paciente sin vínculos, y que
  responde la más reciente de dos fechas registradas en dos vínculos
  distintos del mismo paciente — el mismo valor que ya validaban las pruebas
  existentes del punto de estado de datos para ese caso.
- Ejecución: suite unitaria en verde (29 suites / 317 pruebas), suite
  end-to-end en verde (31 suites / 387 pruebas, `--runInBand`), compilación
  sin errores y análisis estático sin advertencias. Datos ficticios, sin
  contenido clínico.

## Figuras pendientes

Ninguna figura nueva; la tarea no introduce un flujo o diagrama distinto de
los ya registrados para el módulo.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-84-patient-last-consultation-date`
  (creada a partir de `origin/main`). Commit `b99282d`.
- Ticket: TASK-84 ("TASK-29 – Exponer fecha de última consulta a nivel
  paciente (campo agregado en PatientResponse)"), prioridad baja, dependiente
  de TASK-29 (ya fusionada).
