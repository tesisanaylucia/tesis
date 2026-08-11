# Fase 3 — Motor de Turnos (backend) — listado y filtro de turnos reservados (TASK-79, corrección a P3.9)

## Qué se implementó

Se agregó `GET /turnos`, el punto de acceso de lectura general —por
profesional, por paciente, por rango de fechas y por estado, en cualquier
combinación— que el documento de requisitos describe en P3.9 y que ningún
ticket previo del módulo de Turnos había implementado: una auditoría del
código sobre `main` confirmó que el servicio de turnos no tenía ningún
método de listado (sólo creación y las transiciones de estado) y que
ningún controlador exponía una operación `GET` sobre turnos, más allá de
la consulta de franjas libres —que responde una pregunta distinta— y de
la agenda propia del profesional agregada por TASK-89 (`GET
/profesionales/:id/turnos`), que sólo admite filtrar por rango de fechas
y exige nombrar al profesional en la propia ruta.

El nuevo punto de acceso admite cuatro filtros, todos opcionales:
identificador de profesional, identificador de paciente, fecha de inicio
y fecha de fin, y estado del turno. Cada resultado incluye, además de los
mismos campos que ya devuelve la reserva de un turno, los datos mínimos
del paciente que el documento de requisitos pide —nombre y apellido—, de
modo que un cliente no necesite una segunda consulta por fila sólo para
mostrar con quién es el turno. La respuesta queda paginada mediante
`limit` y `offset`, con un tamaño de página por defecto y un tope máximo
tratados como límites contra abuso y no como reglas de negocio, en la
misma línea que el resto de los topes de esta clase ya presentes en el
módulo (el rango máximo de la agenda propia del profesional, el tamaño
máximo del lote de reorganización).

## Decisiones y por qué

**Un profesional queda acotado a sus propios turnos de forma
incondicional, no sólo validada contra el filtro de profesional cuando
éste se recibe.** Si el filtro de profesional se omite, se completa con
la identidad propia del profesional autenticado en lugar de dejar la
consulta sin acotar; si se recibe un filtro de paciente sin filtro de
profesional, el acotamiento a la identidad propia igual se aplica antes
de combinarse con el filtro de paciente. De esta manera una consulta
sólo por paciente, o sin filtros más allá del rango de fechas, no puede
hacer aparecer turnos de otro profesional por la vía de omitir el filtro
que se estaría validando. Nombrar explícitamente el identificador de
*otro* profesional responde con prohibido (403), tal como pide el propio
criterio de aceptación del ticket, replicando el mismo mecanismo que ya
usan las transiciones de estado de un turno individual (confirmar,
cancelar, completar, marcar ausente, registrar pago u orden,
reprogramar) para el mismo tipo de comprobación sobre la misma entidad,
en lugar de la respuesta "no encontrado" que usa el resto del sistema
para un recurso fuera del alcance del inquilino: a diferencia de esos
casos, el profesional ya conoce su propio identificador, así que negarlo
explícitamente no revela nada que una solicitud legítima no supiera de
antemano.

**El filtro por paciente se valida con la misma comprobación de
visibilidad que usa el resto de las lecturas de pacientes por
profesional, no con la comprobación de pertenencia al inquilino sin
más.** Un profesional que nombra un paciente con el que no tiene vínculo
de tratamiento recibe "no encontrado", igual que si ese paciente no
existiera — la razón documentada en el resto del sistema para esa
distinción (una restricción más angosta que el inquilino responde con
"no encontrado", no con "prohibido", porque el llamador no tiene ya
conocimiento previo de qué pacientes pertenecen a otro profesional) se
aplicó aquí sin cambios, a diferencia del caso anterior sobre el
identificador de profesional.

**Se exige al menos un identificador de profesional o de paciente cuando
quien pregunta es administración, para evitar un recorrido sin acotar de
los turnos del inquilino.** El propio documento de requisitos lo
recomienda sin exigirlo; se decidió exigirlo igualmente por administración
—devolviendo un error de solicitud incorrecta en su ausencia— en la misma
línea que el resto de los topes contra abuso del módulo. La exigencia no
llega nunca a manifestarse para un profesional, porque el acotamiento
incondicional a su propia identidad ya provee ese identificador antes de
que la validación se ejecute.

**El rango de fechas es opcional en su conjunto, pero no puede recibirse
a medias.** A diferencia de la agenda propia del profesional, donde
ambos extremos son obligatorios, aquí una consulta sin rango de fechas es
legítima —queda acotada por el filtro de profesional o de paciente en su
lugar—, pero recibir sólo el inicio o sólo el fin no tiene un significado
inequívoco y se rechaza como solicitud incorrecta.

## Alternativas descartadas

- **Aceptar una consulta sin ningún filtro para cualquier rol**: descartada
  porque habría permitido a administración recorrer el listado completo de
  turnos del inquilino en una sola solicitud, exactamente el recorrido sin
  acotar que el propio documento de requisitos advierte evitar.
- **Validar el filtro de paciente contra el inquilino sin más, como se
  hace con el filtro de profesional**: descartada porque un profesional
  que nombra un paciente ajeno no tiene, a diferencia del caso del
  identificador de profesional, conocimiento previo de si ese paciente
  existe en el inquilino; responder "no encontrado" en lugar de
  "prohibido" evita revelar esa información, siguiendo la misma
  distinción ya establecida en el resto del sistema para las lecturas de
  pacientes.
- **Defaults de página (`limit`/`offset`) declarados en el propio
  decorador de validación del cuerpo de la consulta**: descartada en
  favor de aplicarlos en el servicio, siguiendo el mismo criterio ya
  fijado para el filtro por año del listado de feriados — completar un
  valor por defecto es responsabilidad del servicio, no de la validación
  de forma.

## Entidades / puertos / adaptadores tocados

- `src/appointments/dto/list-appointments-query.dto.ts` (nuevo): valida
  los cinco filtros y los dos parámetros de paginación de la consulta.
- `src/appointments/appointments.service.ts` (modificado): nuevo método
  `findAll`, con el acotamiento por rol, las comprobaciones de
  pertenencia/visibilidad, la validación del rango opcional y la consulta
  paginada.
- `src/appointments/appointments.controller.ts` (modificado): nueva
  operación `GET /turnos`, restringida a los roles administración y
  profesional.
- `src/appointments/appointment.presenter.ts` (modificado): selección y
  forma de respuesta nuevas para el listado (turno más datos mínimos del
  paciente), y el sobre de paginación (`items`, `total`, `limit`,
  `offset`).
- `src/appointments/appointments.constants.ts` (modificado): tamaño de
  página por defecto y tope máximo, como límites contra abuso.

No se modificó el esquema de Prisma: la entidad `Appointment` y su
acotamiento por inquilino ya existían desde TASK-34.

## Tests y qué validan

- `src/appointments/appointments.service.spec.ts` (ampliado, 24 pruebas
  nuevas sobre el método `findAll`; 60 pruebas en total en el archivo):
  el cliente de Prisma se simula en memoria.
  - Administración sin filtro de profesional ni de paciente recibe
    solicitud incorrecta (400), y la consulta ni se ejecuta.
  - Administración puede filtrar por profesional o por paciente por
    separado; el filtro por paciente se comprueba mediante la misma
    verificación de visibilidad que usa el resto del sistema.
  - Un profesional sin filtro de profesional queda acotado a su propia
    identidad; con su propio identificador explícito, la consulta se
    permite igual; con el identificador de otro profesional, se rechaza
    con prohibido (403) y la consulta no se ejecuta.
  - El filtro de estado se traslada intacto a la condición de la
    consulta.
  - Un rango de fechas a medias (sólo inicio o sólo fin) se rechaza; un
    fin anterior al inicio también.
  - El tamaño de página y el desplazamiento por defecto se aplican
    cuando no se reciben, y un valor personalizado de ambos se traslada
    a la consulta y se refleja en la respuesta junto con el total
    informado por el conteo.
  - Cada elemento devuelto incluye los datos mínimos del paciente
    embebidos.
- `test/appointments-listing.e2e-spec.ts` (nuevo, 11 pruebas): contra la
  instancia local de PostgreSQL, con dos organizaciones y dos
  profesionales de la misma organización.
  - Rechazo de una consulta de administración sin filtro de profesional
    ni de paciente (400).
  - Filtro por profesional, ordenado por fecha y hora, sin incluir
    turnos de otro profesional de la misma organización.
  - Filtro por paciente a través de varios profesionales.
  - Filtro por estado y por rango de fechas, cada uno por separado.
  - Rechazo de un rango de fechas a medias (400).
  - Paginación con `limit`/`offset` y verificación del total informado.
  - El profesional dueño obtiene su propia agenda sin nombrar su
    identificador, y sin que aparezcan turnos de otro profesional de la
    misma organización con el mismo paciente.
  - Rechazo (403) de un profesional que intenta listar los turnos de
    otro.
  - Un identificador de profesional de otra organización responde "no
    encontrado" (404).
  - Rechazo del acceso sin autenticar (401).
- Ejecución: suite unitaria completa en verde (30 suites / 335 pruebas).
  Suite end-to-end completa en verde en modo serie (33 suites / 409
  pruebas), siguiendo la misma recomendación ya registrada en la entrada
  de TASK-78 de preferir el modo serie para una señal confiable frente a
  la intermitencia preexistente del entorno de pruebas en modo paralelo.
  Verificación de tipos y análisis estático sin errores. Los datos
  usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-79-appointments-list-endpoint`
  (creada a partir de `main` ya actualizado con TASK-83/TASK-84).
  Commit `d2b1b57` al momento de redactar esta entrada. Pusheada a
  `origin`, pendiente de Pull Request en Bitbucket.
- Ticket: TASK-79 ("[CORRECCIÓN] P3.9 – Endpoint GET /turnos (listado y
  filtro de turnos reservados)"). Depende de TASK-34 (entidad turno),
  TASK-38 (estados) y TASK-16 (roles), todas ya fusionadas a `main`.
