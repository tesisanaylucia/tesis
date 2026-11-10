# Fase 5 — Capa conversacional y WhatsApp (backend) — la disponibilidad que el bot ofrece se calcula para el paciente que pregunta (TASK-213, corrección a P5.2 sobre la regla de franja extra de P3.4)

## Contexto

Hallazgo de uso real reportado por la usuaria sobre los pacientes que se
atienden por primera vez: siendo paciente nuevo, el chat le ofrecía horarios
que después no podía tomar. El enunciado del ticket lo formula como la
conducta esperada —"si soy paciente nuevo sólo el chat me debe ofrecer los
turnos de la franja de primer paciente, los primeros turnos o últimos, no me
debe ofrecer turnos que no puedo tomar"— y no registra descripción ni
comentarios adicionales, de modo que el diagnóstico se hizo sobre el código.

La causa estaba en una asimetría entre dos superficies que el motor de
turnos (4.4) ya distinguía y que la capa conversacional no. La reserva de la
primera sesión de un paciente nuevo no se coloca en cualquier hueco libre de
la agenda: la regla de franja extra la ubica donde el profesional la haya
configurado —el primer turno del día, el último, o un par contiguo dentro de
la franja habitual— y la escritura valida el horario elegido contra
exactamente esa grilla, no contra la grilla habitual. El cálculo de esa
agenda especial existía desde P3.4 y la ruta HTTP de disponibilidad ya la
exponía bajo un parámetro propio; la herramienta de disponibilidad del bot,
en cambio, consultaba siempre la agenda habitual del profesional, sin saber
para qué paciente estaba preguntando.

La consecuencia era la que describe el ticket, y su forma importa: el
paciente nuevo era llevado a elegir un horario que el sistema le mostraba
como libre —y que de hecho lo estaba, para cualquier otro paciente— y la
reserva recién entonces lo rechazaba, con un error cuyo texto explica que el
horario no es un comienzo válido de primera sesión bajo la modalidad
configurada. Es decir, la conversación hacía recorrer al paciente un camino
que no terminaba en un turno, y el único mensaje que lo advertía llegaba
después de haber elegido. El caso extremo es el de las dos modalidades que
el ticket nombra: cuando la franja extra es el primer o el último turno del
día, el bloque que el paciente nuevo puede tomar está *fuera* de la ventana
de atención habitual, de modo que la lista que el bot le ofrecía y la lista
de lo que podía reservar no tenían un solo horario en común.

## Qué se implementó

La disponibilidad que la capa conversacional ofrece se calcula para el par
paciente/profesional, y no para el profesional solo.

- **Nuevo método de dominio** (`AppointmentsService.getBookableSlots`):
  recibe paciente, profesional y rango de fechas, y decide con qué agenda
  responder según si una reserva ahora sería la primera sesión de ese
  paciente con ese profesional. Si lo es, delega en el cálculo de la franja
  extra; si no, en la agenda habitual.
- **La herramienta de disponibilidad del bot** pasa a exigir el id del
  paciente y a delegar en ese método. No decide nada por sí misma: la
  herramienta sigue siendo un adaptador delgado, como el resto del catálogo
  de P5.2.
- **El cálculo de la franja extra acepta ahora la exclusión de un horario**
  que la agenda habitual ya aceptaba desde TUR-14: el horario que el paciente
  acaba de cancelar no debe volver a ofrecérsele. Esa exclusión no llegaba
  antes a esta agenda y ahora sí puede llegar, porque un paciente cuyo turno
  cancelado era su primera sesión sigue siendo paciente nuevo.
- **El manual de flujos** pasa el paciente en los dos lugares donde el bot
  consulta disponibilidad —la reserva y la reprogramación— y agrega la
  instrucción de ofrecer únicamente los horarios que devolvió la
  herramienta, junto con qué decir cuando la lista vuelve vacía.

## Decisiones y por qué

**La decisión de qué agenda corresponde vive en el motor de turnos y no en
la herramienta ni en el servicio de disponibilidad.** La pregunta es sobre
un paciente —sobre el vínculo paciente/profesional, que es el que registra
si la primera sesión ya ocurrió— y el servicio de disponibilidad no conoce
pacientes: computa grillas de profesionales. Ponerla en la herramienta,
por su parte, habría contradicho la regla de la fase, que es que ninguna
regla de agenda ni de reserva viva en el catálogo de herramientas.

**El predicado que decide es el mismo que usa la reserva**, el que lee del
vínculo si la primera sesión está pendiente, y no el tipo de paciente. Los
dos se mueven juntos al asistir, pero la regla de inactividad vuelve el tipo
a "nuevo" después de un año sin concurrir, mientras el indicador de primera
sesión permanece en falso: un paciente que vuelve después de años no es
alguien a quien la franja extra deba aplicarse. Reutilizar el mismo
predicado es, además, lo que garantiza que la lista que el bot ofrece y la
que la reserva acepta no puedan separarse, que es el defecto que esta
corrección repara.

**No se agregó ningún caso especial para el profesional que todavía no
configuró su modalidad de franja extra.** El cálculo de la franja extra ya
resuelve ese caso por su cuenta: devuelve la agenda habitual menos los días
que ya tienen un paciente nuevo, que es exactamente lo que la reserva
permite ahí. Un caso especial habría sido una segunda expresión de la misma
regla, con la posibilidad de divergir de la primera.

**El profesional que no acepta pacientes nuevos y la cota de un paciente
nuevo por jornada quedan cubiertos sin código adicional**, porque ambos ya
forman parte del cálculo de la franja extra. El pre-chequeo de elegibilidad
de P5.6 sigue siendo el que cierra el flujo con una explicación cuando el
profesional no toma pacientes nuevos; lo que esta corrección agrega es que,
incluso si el modelo omitiera ese pre-chequeo, la lista de horarios que
recibiría estaría vacía en lugar de ser una lista de turnos inalcanzables.

**El id del paciente es obligatorio en la herramienta y no opcional.** Una
llamada sin él no se puede responder: no habría manera de saber qué agenda
corresponde, y el valor por omisión razonable —la agenda habitual— es
precisamente la respuesta equivocada para el paciente nuevo, es decir el
defecto que se está corrigiendo. Es preferible que la llamada sea rechazada
con un error descriptivo, que el modelo puede corregir reintentando con el
dato, antes que responderla con una lista plausible y falsa.

**La exclusión del horario recién cancelado se aplica sobre los horarios de
comienzo que cada modalidad ofrece, y no sobre los bloques libres con los
que esos comienzos se construyen.** La diferencia se ve en la modalidad de
par contiguo dentro de la franja: si el paciente canceló el turno de las
nueve y media, ese comienzo debe desaparecer, pero el par que empieza a las
nueve —y que ocupa las nueve y las nueve y media— es otro turno, a otra
hora, y el bloque de las nueve y media está realmente libre. Filtrar los
bloques de entrada lo habría hecho desaparecer también, ofreciendo menos de
lo que corresponde; lo que la regla de TUR-14 prohíbe es volver a ofrecer el
turno cancelado, no todo turno que lo contenga.

**La comparación de ese horario se hace por instante y no por texto**, igual
que ya lo hacía la agenda habitual: el texto ISO que envía el llamador y la
construcción interna de la grilla pueden nombrar el mismo instante con
grafías distintas.

## Alternativas descartadas

- **Corregir sólo el manual de flujos**, instruyendo al modelo a no ofrecer
  horarios fuera de la franja cuando el paciente es nuevo. Se descartó por
  la misma razón que el resto de esta fase mantiene para cada regla que
  importa: el prompt es el control primario, pero no puede ser el único. El
  modelo no tiene de dónde deducir cuál es la franja extra de cada
  profesional, de modo que la instrucción sería incumplible además de
  inverificable, y el síntoma del ticket es precisamente que la lista con
  los horarios equivocados ya estaba a la vista del modelo.
- **Que la herramienta llamara a los dos cálculos y uniera los resultados**,
  dejando al modelo elegir. Habría conservado el defecto: la unión incluye
  los horarios que el paciente nuevo no puede tomar.
- **Agregar un parámetro booleano "paciente nuevo" a la herramienta**, como
  lo tiene la ruta HTTP equivalente. Se descartó porque obliga al modelo a
  conocer y sostener un dato que el sistema ya sabe, y porque un valor
  equivocado —en cualquiera de los dos sentidos— reproduce el defecto. La
  ruta HTTP lo expone así porque su llamador es la aplicación del
  profesional, que efectivamente elige qué vista quiere; la conversación no
  elige, sino que pregunta por un paciente concreto.
- **Cambiar la ruta HTTP de disponibilidad para que también tome el
  paciente.** Queda fuera de alcance: su llamador es la aplicación del
  profesional y el parámetro que ya tiene expresa una necesidad distinta, la
  de ver una u otra vista deliberadamente.

## Entidades / puertos / adaptadores tocados

- `src/appointments/appointments.service.ts`: el método nuevo. Ninguna regla
  de colocación se movió ni se duplicó; el método elige entre dos cálculos ya
  existentes.
- `src/availability/availability.service.ts`: el cálculo de la franja extra
  acepta el horario a excluir, con un único punto de filtrado para sus tres
  modalidades.
- `src/chatbot/tools/appointment.tools.ts`: la herramienta de
  disponibilidad, que pasa a exigir el paciente y a delegar en el motor de
  turnos; deja de depender del servicio de disponibilidad, que ya no
  consulta directamente.
- `src/chatbot/conversation-flows.constants.ts`: el manual de flujos.
- Ningún cambio de esquema ni migración. Ninguna ruta HTTP cambió de
  contrato. Los puertos de integración no se tocaron.

## Tests y qué validan

- `src/appointments/appointments.service.spec.ts`: la rama misma. Que un
  paciente nuevo recibe la agenda de franja extra y nunca la habitual; que un
  paciente que ya es del profesional recibe la habitual y nunca la otra; que
  un paciente ya vinculado pero con su primera sesión todavía pendiente
  —quien reservó y canceló— sigue siendo tratado como nuevo, que es el caso
  donde el tipo de paciente daría la respuesta contraria; que el horario a
  excluir se transmite en ambas ramas; y que un id inexistente o un paciente
  dado de baja produce el error de recurso no encontrado antes de leer
  agenda alguna, en lugar de una lista vacía que el bot leería como "no hay
  turnos".
- `src/availability/availability.service.spec.ts`: la exclusión en las tres
  modalidades de franja extra, incluido el caso que distingue filtrar
  comienzos de filtrar bloques —el par que empieza a las nueve sobrevive a la
  exclusión de las nueve y media— y la comparación por instante frente a dos
  grafías del mismo horario.
- `src/chatbot/tools/appointment.tools.spec.ts`: el contrato de la
  herramienta. Que delega en el método nuevo con el paciente por delante;
  que una llamada sin paciente es rechazada sin llegar al servicio; y que el
  esquema de entrada declara el paciente como obligatorio.
- `test/chatbot-flows.e2e-spec.ts`: la prueba de punta a punta del caso del
  ticket, con PostgreSQL real y un profesional configurado en la modalidad
  de primer turno del día, de manera que su franja extra queda fuera de la
  ventana habitual. Verifica que la conversación de un paciente nuevo recibe
  únicamente el bloque de la franja extra y ninguno de los horarios libres
  de la ventana habitual, y que el horario ofrecido es efectivamente
  reservable —la reserva ocupa los dos bloques contiguos—, que es la mitad
  del ticket que una prueba de la lista sola no cubriría. La segunda prueba
  fija la otra mitad de la regla: un paciente que ya es del profesional sigue
  recibiendo la agenda habitual y no la franja extra.
- Las suites de extremo a extremo de la fase que guionan la herramienta
  pasan a enviar el paciente, por el contrato nuevo.
- Resultado de la corrida completa: análisis estático y formato sin
  advertencias; la totalidad de las pruebas unitarias en verde salvo una
  preexistente y ajena, que depende de una variable de entorno del archivo
  local de configuración y falla igual sobre la rama base; y las suites de
  extremo a extremo en verde salvo cuatro pruebas en tres archivos,
  también preexistentes y ajenas —verificadas como tales ejecutándolas
  sobre la rama base sin estos cambios—, una de ellas por una fecha fija
  escrita en la propia prueba que el calendario ya dejó atrás. Datos
  ficticios, sin contenido clínico ni datos personales reales.

## Figuras pendientes

Se registra una figura nueva (N° 60 tentativo) que contrasta, para un mismo
día y un mismo profesional configurado en la modalidad de primer turno del
día, la agenda que el bot ofrece a un paciente que ya es suyo y la que
ofrece a un paciente nuevo, mostrando que la segunda cae fuera de la ventana
de atención habitual y que ninguno de sus horarios aparece en la primera.

## Trabajo pendiente que esta tarea deja registrado

La reasignación automática de un turno liberado (4.4) reserva el hueco para
el candidato de la lista de espera que corresponda y marca la fila como
primera sesión si el vínculo lo indica, pero no aplica la regla de franja
extra ni la cota de un paciente nuevo por jornada. Es decir, un paciente
cuya primera sesión todavía está pendiente puede terminar con un turno
simple en la ventana habitual por esa vía, que es justamente la colocación
que la reserva conversacional no admite. No se modificó: la reasignación
responde a su propia regla del documento de requisitos —ofrecer el hueco
liberado según el ranking de la lista de espera— y el alcance de este ticket
es lo que el chat ofrece. Corresponde a un ticket aparte decidir si la
franja extra debe regir también ahí.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-213-new-patient-slot-offer` (creada a
  partir de `development`), commit `7d25f5a`, integrada mediante la
  solicitud de incorporación n.º 164 del repositorio del backend hacia
  `development`.
- Ticket: TASK-213, hallazgo de uso real de la usuaria sobre los pacientes
  que se atienden por primera vez, corrección al catálogo de herramientas de
  P5.2 (TASK-47) en su consumo de la regla de franja extra de P3.4 (TASK-37)
  y de la colocación en el borde de la ventana habitual de TASK-121. El
  ticket no tiene descripción ni comentarios: el alcance se tomó del título
  y del reporte de la usuaria.
