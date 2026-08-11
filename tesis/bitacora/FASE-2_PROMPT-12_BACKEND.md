# Fase 2 — Pacientes (backend) — punto de acceso administrativo para configurar el umbral de inactividad, con tope de doce meses (TASK-83, corrección a TASK-29)

## Contexto

TASK-29 (FASE-2_PROMPT-3) había resuelto tratar el plazo de un año de la
regla de inactividad como configuración de la organización y no como una
constante del código, guardada bajo una única clave en la tabla de
configuración por inquilino. Una auditoría posterior del código en
producción encontró que esa configurabilidad, si bien correctamente leída
por el servicio que aplica la regla, nunca había sido efectivamente
alcanzable: ningún punto de acceso invocaba la escritura de la
configuración, de modo que en la práctica todas las organizaciones
quedaban fijas en el valor por defecto sin ninguna forma de modificarlo.
Se encontró además que la lectura del valor configurado no aplicaba ningún
tope máximo, de modo que si un valor llegara a escribirse en la base por
una vía distinta al punto de acceso —por ejemplo, en forma directa—, nada
impedía que superase el "un año máximo" que el documento de requisitos
enuncia expresamente. La tarea corrige ambos puntos.

## Qué se implementó

Se agregó `PATCH /admin/configuracion/patient-inactivity-months`,
restringido al rol administrativo, que valida que el cuerpo de la
solicitud sea un entero entre 1 y 12 y, de serlo, persiste el valor bajo
la misma clave de configuración que el servicio de inactividad ya leía
desde la tarea original, y responde con el nuevo umbral y la fecha de
corte que resulta de aplicarlo. Se agregó asimismo una segunda
comprobación del tope, en la propia lectura que ya existía: un valor
almacenado que supere los doce meses se recorta a doce en lugar de
descartarse, a diferencia del valor no numérico o no positivo, que sigue
descartándose en favor del valor por defecto tal como decidió la tarea
original. Cada escritura deja además una entrada de auditoría que
identifica la clave y el nuevo valor.

## Decisiones y por qué

**El tope se defiende en dos lugares con un único origen.** La validación
del cuerpo de la solicitud impide que un valor fuera de rango llegue
siquiera a escribirse, pero el mismo límite se aplicó también en la
lectura, como defensa en profundidad ante una fila que exista por fuera de
ese único camino de escritura —una fila anterior a esta tarea, o una
escritura directa sobre la base—. Ambas comprobaciones leen la misma
constante en lugar de repetir el número doce como literal en dos lugares,
de modo que un futuro cambio del tope no pueda actualizarse en un punto y
olvidarse en el otro.

**Un valor por encima del tope se recorta y no se descarta.** La tarea
original ya había decidido que un valor no numérico o no positivo se
descarta en favor del valor por defecto, por tratarse de un dato sin
sentido alguno. Un valor de veinticuatro meses, en cambio, es un dato con
sentido que simplemente excede lo permitido; recortarlo a doce conserva la
intención más cercana a la que el valor expresa, mientras que descartarlo
por completo volvería indistinguible ese caso del de un valor
directamente inválido.

**La escritura se ubicó en el servicio existente y no en uno nuevo.** El
servicio que ya resolvía la lectura del umbral es el único lugar que
conoce la clave de configuración, la constante que gobierna el tope y la
fecha de corte que de ambas se deriva; agregar allí el método de
escritura evita que esa relación tenga que declararse una segunda vez en
otro lugar del código.

**Se registró una entrada de auditoría por cada cambio.** Se trata de la
primera invocación real del método de escritura de la configuración por
inquilino desde que existe, y una configuración que gobierna cuándo un
paciente deja de considerarse en tratamiento activo es la clase de cambio
administrativo que el proyecto ya audita en casos comparables —los feriados
del calendario administrativo, por ejemplo—, de modo que se siguió el mismo
criterio en lugar de introducir una excepción.

## Alternativas descartadas

- **Descartar también un valor por encima del tope, igual que un valor no
  numérico**: descartada porque trata dos situaciones distintas —un dato
  sin sentido y un dato válido pero excesivo— de la misma manera, perdiendo
  información que el recorte sí conserva.
- **Repetir el literal 12 en la validación del cuerpo de la solicitud y en
  la lectura**: descartada por el riesgo de que ambos lugares diverjan si
  el tope cambiara en el futuro; se prefirió una única constante compartida.
- **Fijar el plazo en 365 días, abandonando la configurabilidad por
  inquilino**: fuera de alcance según el propio ticket, que la reserva
  explícitamente como una corrección distinta si el equipo la prefiriera;
  no se tomó esa decisión aquí.

## Entidades / puertos / adaptadores tocados

- `src/patients/patients.constants.ts`: nueva constante
  `MAX_PATIENT_INACTIVITY_MONTHS`.
- `src/patients/patient-inactivity.service.ts`: `threshold()` recorta un
  valor almacenado por encima del tope; nuevo método `setThresholdMonths`
  que persiste el valor y escribe la entrada de auditoría.
- `src/patients/dto/update-patient-inactivity-months.dto.ts` (nuevo):
  valida el entero entre 1 y 12.
- `src/patients/patient-inactivity-config.controller.ts` (nuevo): expone
  `PATCH /admin/configuracion/patient-inactivity-months`, restringido al
  rol administrativo. No requiere una comprobación de alcance adicional
  más allá del rol: el inquilino nunca es un parámetro de la solicitud,
  sino que se deriva del token de quien la realiza, de modo que un
  administrador no tiene forma de dirigir la escritura a una organización
  distinta de la propia.
- `src/patients/patients.module.ts`: registra el nuevo controlador.
- No hubo cambios de esquema ni migración: la tabla de configuración por
  inquilino y su clave ya existían desde la tarea original.

## Tests y qué validan

- `src/patients/patient-inactivity.service.spec.ts` (nuevo, unitario):
  cubre el valor por defecto, el descarte de un valor no numérico o no
  positivo, el uso directo de un valor dentro del tope, el recorte de un
  valor que lo supera, y que la escritura persiste bajo la clave correcta
  y deja la entrada de auditoría esperada.
- `test/patient-inactivity-config.e2e-spec.ts` (nuevo): cubre el rechazo
  sin autenticación y con rol profesional, el rechazo de valores fuera de
  rango (cero, trece, no entero, no numérico), la aceptación del propio
  tope de doce, la entrada de auditoría, el aislamiento entre inquilinos
  —que el cambio de una organización no afecta a otra— y el criterio de
  aceptación central del ticket: un paciente cuya última consulta con un
  profesional ocurrió hace siete meses conserva su tipo bajo el umbral por
  defecto de doce meses y pasa a marcarse como nuevo una vez que el
  administrador configura el umbral en seis.
- Ejecución: suite unitaria en verde (30 suites / 323 pruebas), suite
  end-to-end en verde (32 suites / 398 pruebas, `--runInBand`), análisis
  estático sin advertencias. Datos ficticios, sin contenido clínico.

## Figuras pendientes

Ninguna figura nueva; la tarea no introduce un flujo o diagrama distinto
de los ya registrados para el módulo.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-83-patient-inactivity-months-config` (creada a partir de
  `origin/main`, posterior a TASK-84). Commit `16151a3`.
- Ticket: TASK-83 ("[CORRECCIÓN] TASK-29 – Endpoint de configuración de
  patient_inactivity_months y tope de 12 meses"), prioridad media,
  dependiente de TASK-29 (ya fusionada) y de TASK-19 (P0.8, ya fusionada).
