# Fase 5 — Capa conversacional y WhatsApp (backend) — el bot no debe ofrecer turnos que ya pasaron (TASK-217, corrección a P5.3/P5.5)

## Contexto

Hallazgo de uso real reportado por la usuaria: "el chat ofrece turnos que ya
pasaron en el mismo día o días anteriores". El ticket (TASK-217, "No tiene
que ofrecer turnos que ya pasaron") no tiene descripción ni una conversación
de ejemplo adjunta, y el sistema no persiste el historial de las
conversaciones, de modo que la causa se determinó por inspección del código.

El cálculo de disponibilidad ya excluía los horarios pasados: desde TUR-9
(TASK-153), `AvailabilityService.getSlots` y las tres ramas de
`getNewPatientSlots` descartan todo comienzo que el reloj de la clínica
(`clinicNow()`) ya alcanzó, y `book`/`reschedule` rechazan un instante que no
sea futuro. Esa corrección tiene pruebas unitarias y de extremo a extremo y
está tanto en `development` como en `main`. El defecto no estaba, entonces,
en lo que las herramientas devuelven, sino en cómo el modelo lo leía y en qué
sabía del momento presente:

1. **Los horarios llegan con sufijo `Z`.** `scheduledAt` es la hora de pared
   de la clínica escrita en un campo UTC (TASK-150/TUR-2): un turno de las
   14:00 viaja como `...T14:00:00.000Z`. Nada en el prompt lo aclaraba, y un
   modelo que sabe que la clínica está en UTC-3 puede "convertir" ese valor
   y nombrar las 11:00 —un horario que quizá ya pasó— o, para un turno
   temprano, el día anterior.
2. **El prompt solo daba la fecha, nunca la hora.** Con "Hoy es AAAA-MM-DD"
   el modelo no tenía contra qué comparar un horario de ese mismo día.
3. **Esa fecha era la UTC.** `todayCalendarDate()` toma el día UTC, así que
   entre las 21:00 y la medianoche de la clínica el prompt ya anunciaba el
   día siguiente.
4. **Faltaba el día de la semana**, y el modelo es poco confiable para
   calcular a mano en qué día cae una fecha ("el lunes" puede terminar
   siendo un lunes ya pasado).
5. **Nada impedía reutilizar una lista vieja.** Dentro de la misma sesión
   (30 minutos de inactividad por defecto, configurable por inquilino), el
   modelo podía volver a ofrecer horarios de una llamada anterior a
   `check_availability`, alguno de los cuales ya podía haber pasado.

## Qué se implementó

- **`clinicToday()` y `clinicTimeOfDay()`** (nuevas, en
  `src/common/dates/clinic-clock.ts`): fecha `AAAA-MM-DD` y hora `HH:mm` de
  la clínica, derivadas de `clinicNow()`, de modo que cambian de día a la
  medianoche de la clínica y no a las 21:00.
- **`ChatbotConfigService.systemPrompt`** interpola ahora, además de
  `{today}`, `{weekday}` (nombre del día en español) y `{time}`, los tres a
  partir de un único `Date.now()`, para que no puedan quedar a uno y otro
  lado de la medianoche.
- **Manual de flujos** (`conversation-flows.constants.ts`):
  - El encabezado pasa a ser "Hoy es {weekday} {today} (formato AAAA-MM-DD)
    y son las {time}, hora de la clínica", con la prohibición de ofrecer
    "fechas u horarios que ya pasaron".
  - Una regla nueva: las fechas y horas que devuelven las herramientas ya
    están en la hora de la clínica aunque terminen en "Z", y se leen tal
    cual, sin conversión de zona horaria, con un ejemplo.
  - En los flujos de reserva y reprogramación: volver a llamar a
    `check_availability` cada vez que se muestran horarios, sin reutilizar
    una lista de un mensaje anterior.

## Decisiones y por qué

**La corrección es del prompt y no del motor de turnos**, a diferencia de
TASK-213 y TASK-215, donde el filtro se llevó a la herramienta. Aquí la
herramienta ya cumple: no devuelve horarios pasados y la reserva rechaza uno
pasado aunque el modelo lo pida. Lo que faltaba era que el modelo supiera qué
hora es y cómo leer lo que recibe, y esa información solo puede llegarle por
el prompt.

**Se mantuvo el formato de `scheduledAt`** en lugar de quitar la `Z` o
cambiar la forma de la respuesta de las herramientas. Esa codificación es
la de todo el sistema (TASK-150) y la misma que `book_appointment` y
`reschedule_appointment` reciben de vuelta; cambiarla solo en la capa
conversacional obligaba a traducir en ambos sentidos en cada herramienta.
Una aclaración en el prompt cubre todas las herramientas a la vez, incluidas
`list_patient_appointments` y las de confirmación.

**No se modificó `todayCalendarDate()`**, que sigue tomando el día UTC: su
propio comentario documenta que las reglas que la usan (edad, inactividad)
no dependen del lado de la medianoche en que se consulte, y cambiarla
habría alterado esas reglas sin necesidad. El prompt es el único consumidor
para el que la diferencia importa, y usa ahora las funciones nuevas del
reloj de la clínica.

**La reserva sigue siendo la red de seguridad.** Si el modelo igual
eligiera un horario pasado, `book` y `reschedule` lo rechazan desde
TASK-153; esta corrección apunta a que no se lo ofrezca al paciente.

## Alternativas descartadas

- **Agregar a la respuesta de `check_availability` campos ya formateados**
  (día de la semana, hora local legible). Descartada por ahora: resolvía
  solo esa herramienta y no las demás que devuelven `scheduledAt`, y
  sumaba una segunda representación del mismo instante que el modelo
  podría devolver mal al reservar.
- **Filtrar del historial de la sesión los resultados viejos de
  disponibilidad.** Descartada: modificar mensajes de herramientas ya
  emitidos rompe la correspondencia de `tool_call_id` que exige la API (ver
  `trimToTurnBoundary`); la instrucción de volver a consultar logra lo mismo
  sin tocar el historial.

## Entidades / puertos / adaptadores tocados

- `src/common/dates/clinic-clock.ts`: `clinicToday`, `clinicTimeOfDay`.
- `src/chatbot/chatbot-config.service.ts`: interpolación de `{weekday}` y
  `{time}`, día de la clínica en lugar del UTC.
- `src/chatbot/conversation-flows.constants.ts`: encabezado, regla sobre la
  `Z`, pasos de reserva y reprogramación.
- Ningún cambio de esquema, migración, puerto ni ruta HTTP.

## Tests y qué validan

- `src/common/dates/clinic-clock.spec.ts` (nuevo): `clinicNow`; fecha y hora
  de la clínica; a las 23:59 de la clínica (02:59 UTC del día siguiente) la
  fecha sigue siendo la de la clínica; a la medianoche de la clínica cambia
  de día.
- `src/chatbot/chatbot-config.service.spec.ts`: con el reloj fijado, el
  prompt dice "Hoy es miércoles 2026-11-11 … y son las 15:42, hora de la
  clínica" y no deja marcadores sin resolver; a las 22:30 de la clínica
  sigue diciendo el miércoles 11 y no el jueves 12; el prompt contiene la
  aclaración sobre la `Z`.
- `test/chatbot-flows.e2e-spec.ts`: el prompt armado de punta a punta
  contiene el día de la clínica.
- Resultado: pruebas unitarias en verde salvo la de selección del adaptador
  de cerradura, que falla igual sobre la rama base por la variable local
  `LOCK_ADAPTER="hardcoded"` del `.env` de la usuaria; las tres suites de
  extremo a extremo del chatbot (53 pruebas) en verde. Análisis estático y
  tipos sin errores. Datos ficticios, sin contenido clínico.
- Límite de la verificación: las pruebas comprueban lo que el prompt dice,
  no la conducta del modelo real frente a él; la confirmación final es una
  conversación de prueba por WhatsApp.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-217-no-past-slots-in-chat` (creada a
  partir de `origin/development`).
- Ticket: TASK-217, hallazgo de uso real de la usuaria, corrección al
  prompt del chatbot de P5.3 (TASK-48) y al manual de flujos de P5.5.
