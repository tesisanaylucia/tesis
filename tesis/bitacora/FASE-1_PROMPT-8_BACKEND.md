# Fase 1 — Profesionales (backend) — Validación mínima de matrículas para aceptar pacientes nuevos (TASK-85, corrección a TASK-21/22)

## Contexto

TASK-85 es una tarea de corrección, sin número de requisito propio en la
especificación, abierta sobre una auditoría de código posterior al cierre de
la Fase 1: la fuente de verdad de P1.1 exige que el chatbot pueda mostrar "sus
dos matrículas" de un profesional, y P1.2 implementó el tope máximo de tres
matrículas por profesional con su prueba de concurrencia, pero ninguna de las
dos tareas exigió un mínimo ni tipos obligatorios. `licenses` es opcional en
el DTO de alta y no existe restricción de unicidad por tipo en el modelo ni en
el servicio de matrículas, de modo que un profesional podía quedar creado sin
matrícula alguna, o con tres matrículas provinciales y ninguna nacional.

## Qué se implementó

Se agregó una validación de mínimo —al menos una matrícula de tipo
`PROVINCIAL` y una de tipo `PROFESSIONAL`— sobre un único punto de escritura:
`ProfessionalsService.updateConfiguration`, el método detrás de
`PATCH /profesionales/:id/configuracion` (P1.5), rechaza con 400 cualquier
intento de dejar `acceptsNewPatients` en `true` —sea activándolo o
simplemente conservándolo al enviar otro campo de la misma petición— si el
profesional no cumple el mínimo. La comprobación cuenta las matrículas
vigentes del profesional y decide antes de aplicar la escritura, dentro de la
misma transacción, ahora elevada a aislamiento serializable por la misma
razón que el tope de tres matrículas: es un invariante de lectura seguida de
escritura, y bajo el aislamiento por defecto una eliminación de matrícula
concurrente con la activación del indicador podría dejar pasar a ambas
operaciones.

No se tocó el alta ni la edición general del profesional, ni el ABM de
matrículas (creación, edición, eliminación): la validación vive
exclusivamente en el punto donde se decide si el profesional queda habilitado
para recibir pacientes nuevos.

## Decisiones y por qué

**Validación blanda sobre el indicador de aceptación de pacientes nuevos, no
validación dura en el alta o edición del profesional.** El propio ticket
plantea ambas alternativas y deja la elección al análisis técnico según el
impacto en el flujo de alta. Exigir el mínimo en el alta o en cualquier
edición del profesional se descartó por dos razones concretas, verificadas
antes de decidir y no supuestas: primero, contradice el propio diseño de
P1.2, cuyo DTO de alta documenta explícitamente que las matrículas se cargan
de forma incremental y pueden agregarse después de creado el profesional;
segundo, una inspección de la batería de pruebas de extremo a extremo del
repositorio mostró que más de veinte archivos de prueba, de módulos no
relacionados con matrículas (horarios, ausencias, turnos, lista de espera,
feriados, obras sociales), crean profesionales sin matrícula alguna porque no
es su objeto de prueba, y dependen de que `acceptsNewPatients` conserve su
valor por defecto (`true`) sin que ninguna validación lo impida. Exigir el
mínimo en el alta habría exigido modificar esa batería completa por una
tarea de corrección acotada, un costo desproporcionado al alcance del ticket.
La alternativa elegida —bloquear únicamente la activación del indicador de
aceptación de pacientes nuevos— es la que el propio ticket propone en primer
lugar y no exige tocar ningún flujo ajeno: solo se ejecutó, además, contra el
único punto de la base de código que hoy envía `acceptsNewPatients: true` a
través de la capa HTTP, lo que se confirmó por inspección exhaustiva del
repositorio antes de escribir la corrección.

**El resquicio de la eliminación de matrículas queda documentado como
limitación conocida, no cerrado.** Un profesional que ya acepta pacientes
nuevos con el mínimo justo podría, en principio, perder ese mínimo si se le
elimina o se le retipa una matrícula, sin que ninguna verificación lo
advierta —la misma clase de inconsistencia que motiva el ticket, alcanzada
por una vía distinta—. Cerrar ese resquicio exigía extender la misma
condición a `LicensesService.update` y `.remove`, y esa extensión se
implementó y se descartó tras comprobar, ejecutando la batería de pruebas
existente, que rompe un caso de la prueba de extremo a extremo del ABM de
matrículas que edita la única matrícula de un profesional para cambiarle el
tipo —una prueba ajena por completo a este invariante, que documenta un
comportamiento previo (retipar libremente una matrícula) que la tarea no
pidió alterar—. Se prefirió dejar el resquicio documentado como corrección
pendiente antes que resolverlo a costa de una prueba que no lo motivó y que
pertenece a un ticket distinto.

**Invariante de lectura seguida de escritura, protegido con aislamiento
serializable.** Se siguió el mismo criterio ya aplicado al tope de tres
matrículas y al alta de ausencias (Fase 1, revisión previa): contar y
decidir corren dentro de una única transacción serializable, de modo que una
transacción perdedora de una carrera aborte con una respuesta de conflicto en
lugar de dejar una activación inconsistente.

## Alternativas descartadas

- **Exigir el mínimo de matrículas en el alta del profesional**: descartada
  por contradecir el diseño incremental de P1.2 y por el costo de modificar
  la batería de pruebas de módulos no relacionados que dependen del valor por
  defecto del indicador.
- **Exigir el mínimo en toda edición general del profesional**
  (`PATCH /profesionales/:id`): descartada por la misma razón; ese endpoint
  no toca `acceptsNewPatients` ni matrículas.
- **Bloquear también la eliminación y la edición de matrículas de un
  profesional que ya acepta pacientes nuevos**: implementada y luego
  descartada al comprobar que rompe una prueba de extremo a extremo
  preexistente y ajena a este invariante; queda registrada como corrección
  pendiente.

## Entidades / puertos / adaptadores tocados

- `src/professionals/license-requirements.ts` (nuevo): predicado puro,
  `meetsMinimumLicenseRequirement`, que decide si un conjunto de matrículas
  cumple el mínimo de un tipo provincial y uno nacional.
- `src/professionals/professionals.service.ts`: `updateConfiguration` ahora
  ejecuta dentro de una transacción serializable (antes, una transacción
  simple) y rechaza con 400 la activación de `acceptsNewPatients` por debajo
  del mínimo.
- `src/professionals/dto/update-professional-config.dto.ts`: comentario
  actualizado sobre el campo `acceptsNewPatients` para documentar la nueva
  condición.
- No se modificó el esquema de Prisma ni se agregó ninguna migración: la
  corrección es enteramente de comportamiento de servicio.

## Tests agregados o modificados

- `src/professionals/professionals.service.spec.ts` (unitario, cliente de
  Prisma simulado): cubre que la comprobación no se ejecuta cuando
  `acceptsNewPatients` no se envía en `true`; que se rechaza con 400 sin
  matrícula alguna, con dos matrículas provinciales y ninguna nacional; y que
  se acepta con una matrícula de cada tipo.
- `test/professional-config.e2e-spec.ts` (extremo a extremo, PostgreSQL
  local): agrega los tres escenarios de aceptación del ticket sobre
  profesionales creados a tal fin —sin matrículas, con dos provinciales y
  ninguna nacional, con una de cada tipo— más un caso que confirma que
  desactivar el indicador nunca requiere matrícula alguna. El profesional
  compartido por el resto del archivo, que una prueba preexistente reactiva
  tras haberlo desactivado, se sembró con una matrícula de cada tipo en la
  configuración inicial para seguir cumpliendo el nuevo mínimo sin alterar el
  objeto de esa prueba.
- Ejecución completa: suites unitarias (29 suites / 317 tests) y de extremo a
  extremo (31 suites / 386 tests) en verde; `tsc --noEmit` y `eslint --fix`
  sin errores. Los datos usados en las pruebas son ficticios.

## Figuras pendientes

No surgen figuras nuevas de esta tarea.

## Componente y referencia

- Componente: backend.
- Rama: `feature/TASK-85-minimum-licenses` (creada desde `main`, con TASK-89
  ya fusionado).
- Ticket: TASK-85 ("[CORRECCIÓN] TASK-21/22 – Validación mínima de matrículas
  (1 provincial + 1 profesional)"). Depende de TASK-21 (P1.1) y TASK-22
  (P1.2), ambas fusionadas.
