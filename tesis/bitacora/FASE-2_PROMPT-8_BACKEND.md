# Fase 2 — Pacientes (backend) — Revisión de código: verificación de concurrencia del consentimiento y del filtro de acceso del profesional

## Contexto

Esta entrada documenta la parte del módulo de Pacientes de la misma revisión
sistemática de código descrita en la entrada de Fase 1. La revisión no halló
defectos de comportamiento en el módulo de Pacientes: el filtrado por vínculo de
tratamiento, la restricción de las observaciones al profesional del vínculo, la
auditoría dentro de la transacción y la idempotencia del consentimiento estaban
correctamente implementados. Lo que sí identificó fue un vacío de verificación: dos
invariantes que el código garantiza no tenían prueba que los ejercitara. Esta tarea
cierra ese vacío sin modificar el comportamiento del módulo.

## Qué se implementó

Se agregaron dos pruebas, sin cambios de esquema ni de comportamiento.

La primera es una prueba de extremo a extremo de la concurrencia del registro del
consentimiento a la protección de datos personales. La regla de la especificación
—el consentimiento se solicita sólo si no fue registrado antes— es un invariante de
lectura seguida de escritura que el servicio protege registrando el consentimiento en
una transacción de aislamiento serializable con una rama idempotente. La prueba somete
ese invariante a una carrera real: dos registros del consentimiento del mismo paciente
que llegan simultáneamente deben dejar exactamente una fila y una única entrada de
auditoría.

La segunda es una prueba unitaria del filtro de acceso que restringe a un profesional a
ver únicamente los pacientes con los que tiene un vínculo de tratamiento. La prueba
verifica las dos ramas del filtro —un administrador o un proceso automático no reciben
restricción, un profesional recibe la restricción por su identificador— y, sobre todo,
la rama de denegación que ninguna prueba de extremo a extremo alcanza: una cuenta con
rol de profesional que no tiene un profesional asociado se rechaza antes de tocar la base
de datos, en lugar de degradar a un acceso sin restricción.

## Decisiones y por qué

**La concurrencia se verifica afirmando el invariante final, no un desenlace fijo.** Dos
registros simultáneos del consentimiento pueden resolverse de dos maneras legítimas: si
se serializan, el segundo encuentra el consentimiento ya registrado y lo devuelve sin
crear una fila nueva; si colisionan, la transacción perdedora aborta y se traduce en una
respuesta de conflicto. Cuál de las dos ocurre depende del planificador. Por eso la
prueba no fija cuál respuesta obtiene cada petición, sino que afirma lo que debe valer en
todos los casos: una sola fila de consentimiento, una sola entrada de auditoría y ninguna
respuesta que sea un fallo interno.

**La rama de denegación del filtro de acceso se prueba en el nivel unitario porque el de
extremo a extremo no la alcanza.** Una cuenta con rol de profesional pero sin profesional
asociado es una cuenta rota que el inicio de sesión normalmente nunca emite; construirla a
través de la interfaz pública no es posible sin forzar un estado inválido. La prueba
unitaria instancia el servicio con dependencias simuladas y comprueba que esa cuenta se
rechaza antes de consultar la base de datos —tratar la falta del identificador como acceso
sin restricción sería, precisamente, el modo en que un identificador ausente se convierte
en acceso total—. El filtrado por vínculo de tratamiento en sí mismo ya está cubierto de
extremo a extremo por tareas previas de esta fase; la prueba unitaria añade sólo lo que
aquéllas no pueden ejercitar.

## Alternativas descartadas

- **Escribir una prueba de la denegación del filtro de acceso en el nivel de extremo a
  extremo:** descartada porque exige una cuenta con rol de profesional sin profesional
  asociado, un estado que el inicio de sesión no produce y que la interfaz pública no
  permite construir; la prueba unitaria con dependencias simuladas es la que alcanza esa
  rama de forma determinista.
- **Afirmar un desenlace fijo en la prueba de concurrencia del consentimiento (por
  ejemplo, exactamente una respuesta de conflicto):** descartada por depender del
  planificador; se afirma el invariante de una sola fila y una sola auditoría, que vale en
  todos los entrelazados posibles.

## Entidades / puertos / adaptadores tocados

No se modificó ninguna entidad, servicio ni punto de acceso: la tarea es exclusivamente de
verificación.

## Tests agregados o modificados

- Prueba de extremo a extremo de concurrencia del consentimiento: dos registros
  simultáneos del mismo paciente dejan una sola fila de consentimiento y una sola entrada
  de auditoría, con respuestas de éxito o de conflicto y nunca un fallo interno.
- Prueba unitaria del filtro de acceso del servicio de pacientes: verifica la ausencia de
  restricción para el administrador y el proceso automático, la restricción por
  identificador para el profesional, y la denegación previa a toda consulta de la cuenta de
  profesional sin profesional asociado.

## Figuras pendientes

No surgen figuras nuevas de esta tarea.

## Componente y referencia

- Componente: backend.
- Rama: `main` (cambios en el árbol de trabajo, pendientes de confirmación al momento de
  redactar esta bitácora).
- Tarea: revisión de código y endurecimiento; la parte correspondiente al módulo de
  Profesionales y a la infraestructura común se documenta en la entrada de Fase 1 de esta
  misma revisión.
