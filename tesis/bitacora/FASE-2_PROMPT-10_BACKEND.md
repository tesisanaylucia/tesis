# Fase 2 — Cierre (backend) — Refinamiento del modelo de datos: baja lógica como marca temporal, retiro de un campo sin uso y documentación de la redundancia deliberada en la auditoría

## Contexto

A partir de una revisión de tres campos del esquema surgida de preguntas sobre su
sentido, se introdujeron tres ajustes al modelo de datos, ninguno de los cuales
altera el comportamiento observable del sistema: se documentó de forma explícita
una decisión de diseño ya vigente en la tabla de auditoría, se cambió la
representación de la baja lógica de un valor booleano a una marca temporal, y se
retiró un campo del profesional que se almacenaba sin que ninguna regla lo
consumiera.

## Qué se implementó

**Documentación de la redundancia deliberada entre el puntero genérico y la clave
foránea de paciente en la auditoría.** La tabla de auditoría identifica el registro
afectado por cada acción mediante un puntero genérico —el nombre del tipo de
entidad como texto y el identificador de la fila, que para entidades de clave
compuesta es una cadena compuesta—, porque una única tabla registra acciones sobre
entidades de tipos distintos y no puede tener una clave foránea real hacia cada una.
Además de ese puntero genérico, cuando la acción concierne a un paciente se registra
también una clave foránea real hacia el paciente. Esa duplicación es intencional: el
puntero genérico por sí solo obligaría a una consulta de cumplimiento —"todo lo que
se hizo sobre este paciente", que es la pregunta que la traza existe para responder
en materia de protección de datos personales— a interpretar y comparar cadenas de
texto a través de todos los tipos de entidad, mientras que la clave foránea real
permite resolverla como una unión indexada directa y hace que la base garantice que
el vínculo es válido y de la misma organización. La decisión ya estaba tomada en el
código; esta tarea la deja escrita de forma completa junto a la definición de la
tabla, para que el porqué de mantener ambos punteros no se pierda.

**Baja lógica representada por una marca temporal en lugar de un booleano.** El
indicador de actividad de profesionales y pacientes era un valor booleano. Se lo
reemplazó por una marca temporal anulable: la ausencia de valor significa activo, y
una fecha y hora registran el instante en que se produjo la baja. La reactivación
vuelve a dejar el campo sin valor. La migración correspondiente preserva el estado
existente: las filas que estaban inactivas reciben una marca temporal, de modo que
ninguna baja previa se pierde en la conversión.

**Retiro del campo de fecha de confirmación del profesional.** El profesional tenía
un campo de fecha de confirmación de su incorporación que se podía cargar y editar,
se almacenaba y se devolvía en la respuesta, pero que ninguna regla de negocio
consumía. Se lo retiró del esquema, de los objetos de transferencia de alta y
edición, del servicio y de la representación de respuesta, junto con su migración de
eliminación, por no aportar función alguna en el estado actual del sistema.

## Decisiones y por qué

**La baja lógica es una marca temporal porque el instante de la baja es información
que el sistema debe poder rendir.** Un booleano responde únicamente si el registro
está o no dado de baja; una marca temporal responde además cuándo ocurrió, que es un
dato auditable en un dominio regido por la protección de datos personales y por los
derechos del paciente. La reactivación se expresa naturalmente como el borrado de esa
marca, sin necesidad de un segundo campo. Se descartó conservar el booleano y agregar
por separado una fecha de baja, porque serían dos columnas que deben mantenerse
coherentes entre sí —una de ellas derivable de la otra— y esa coherencia es
precisamente lo que una sola marca temporal vuelve imposible de violar.

**Un campo que nada consume se retira en lugar de conservarse por si acaso.** Mantener
un campo que se almacena y se expone sin que ninguna regla lo lea agrega superficie
—validaciones, conversión de tipos, presencia en el contrato de la API— sin
contrapartida, e induce a error sobre su relevancia. Si en el futuro se necesitara un
dato equivalente, se lo diseñará entonces con el uso concreto que lo justifique.

**La redundancia en la auditoría se documenta en vez de eliminarse.** Podría verse la
clave foránea de paciente como una duplicación a normalizar, dado que el puntero
genérico ya identifica el registro. Se decidió lo contrario y se dejó constancia del
motivo: la clave foránea real es lo que convierte la consulta central de cumplimiento
en una unión indexada validada por la base, en lugar de un análisis de cadenas de
texto; la redundancia compra esa garantía y esa eficiencia, y por eso se mantiene y se
explica.

## Alternativas descartadas

- **Conservar el booleano de actividad y sumar una fecha de baja separada:** descartada
  por requerir la coherencia entre dos columnas donde una sola marca temporal ya
  expresa ambos hechos sin posibilidad de contradicción.
- **Normalizar la auditoría eliminando la clave foránea de paciente por considerarla
  redundante con el puntero genérico:** descartada porque esa clave foránea es la que
  hace eficiente y verificable la consulta de cumplimiento; la redundancia es
  deliberada y quedó documentada como tal.

## Entidades / puertos / adaptadores tocados

- Esquema: en Profesional y Paciente, el indicador booleano de actividad se reemplazó
  por una marca temporal de baja lógica; en Profesional se eliminó la fecha de
  confirmación; en la tabla de auditoría se documentó la relación entre el puntero
  genérico y la clave foránea de paciente. Dos migraciones acompañan estos cambios
  —la conversión de la baja lógica, que preserva las filas inactivas, y la eliminación
  del campo sin uso—.
- Servicios de Pacientes y de Profesionales, servicio de usuarios (verificación de
  actividad del profesional en el inicio de sesión), objetos de transferencia de alta
  y edición del profesional, y las representaciones de respuesta de ambos módulos.

## Tests agregados o modificados

No se agregaron pruebas nuevas de comportamiento. Se ajustaron las existentes al nuevo
nombre y forma del campo de baja lógica —tanto en las respuestas de la API como en las
lecturas directas de la base—, y se retiró la prueba de extremo a extremo dedicada a la
fecha de confirmación, por corresponder a un campo que ya no existe. La suite completa
—pruebas unitarias y de extremo a extremo— quedó en verde.

## Figuras pendientes

No surgen figuras nuevas de esta tarea.

## Componente y referencia

- Componente: backend.
- Rama: `main` (cambios en el árbol de trabajo, pendientes de confirmación al momento de
  redactar esta bitácora).
- Tarea: refinamiento del modelo de datos derivado de la revisión de tres campos del
  esquema (auditoría, baja lógica y fecha de confirmación).
