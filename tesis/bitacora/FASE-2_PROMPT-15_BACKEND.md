# Fase 2 — Pacientes (backend) — El importador decodificaba todo `.csv` como UTF-8 sin resguardo y corrompía nombres con tildes y ñ (TASK-119, corrección a TASK-32)

## Contexto

La usuaria consultó primero, sobre este mismo repositorio y sin conocer aún el
ticket, si el importador de pacientes ([[FASE-2_PROMPT-6]]) admitía nombres con
tildes y ñ. Una verificación empírica contra el propio pipeline —generando un
libro de cálculo real con nombres acentuados y haciéndolo pasar por
`readSpreadsheet` → `mapColumns` → `validateImportRow`— mostró que el camino
`.xlsx` conserva los caracteres acentuados de forma íntegra (el formato es XML en
UTF-8 por dentro, sin ningún paso de decodificación de bytes propio del proyecto),
pero que un `.csv` guardado en Windows-1252/ANSI —el formato exacto que "CSV
(delimitado por comas)" produce en un Excel en español— llega corrompido: "José
María" se convierte en "Jos� Mar�a" y se importa igual, sin ninguna entrada en el
informe.

La causa resultó ser exactamente la que ya describía TASK-119, un ticket abierto
días antes por una auditoría de código automatizada (agente "Audit Módulo
Pacientes vs SRS", 2026-08-14) y que la usuaria pegó a continuación de su
pregunta: `spreadsheet.reader.ts` decodifica todo `.csv` como
`file.buffer.toString('utf8')`, sin resguardo alguno. Un carácter como "á", "é" o
"ñ" en Windows-1252 es un único byte que no es UTF-8 válido; `Buffer.toString`
no lanza excepción ante esa situación, sino que reescribe el byte en silencio
como U+FFFD, el carácter de reemplazo Unicode. El nombre corrompido resultante
sigue siendo una cadena no vacía, de modo que `IsString`/`IsNotEmpty` lo admiten
sin objeción y la fila se persiste tal cual, sin ninguna advertencia en el
informe de importación.

## Qué se implementó

- `hasReplacementCharacter` (`spreadsheet-cell.ts`): el chequeo heurístico que el
  propio ticket nombra como mínimo aceptable —detección de U+FFFD tras la
  decodificación— en lugar de una detección de encoding en sentido estricto.
- `SpreadsheetRow.suspectEncoding` (`spreadsheet.reader.ts`): booleano calculado
  en `toRows` sobre cualquier celda de la fila que el archivo trae, no solamente
  las que el importador de pacientes reconoce.
- `validateImportRow` (`patient-import.row.ts`): corta antes de construir el DTO
  cuando `row.suspectEncoding` es verdadero, devolviendo una entrada de fila —sin
  campo específico, igual que un documento que colisiona— con un mensaje que
  nombra la causa probable e indica la corrección.

No hubo cambios de esquema ni de contrato de otros puntos de acceso: la
corrección es interna a la lectura de planillas y a la validación de una fila.

## Decisiones y por qué

**El chequeo se hizo heurístico y no una detección de encoding en sentido
estricto.** El propio ticket ofrece las dos opciones —"detectar encoding (o, como
mínimo, agregar un chequeo heurístico post-decode: conteo de caracteres de
reemplazo Unicode)"— y se optó por la segunda. Una detección de encoding real
tendría que adivinar cuál era el encoding original a partir de una cadena que ya
perdió esa información —el reemplazo por U+FFFD descarta el byte original—, y
cualquier heurística de "encoding más probable" sobre un archivo de una sola
columna problemática es sobre todo una apuesta. El conteo de caracteres de
reemplazo, en cambio, no necesita adivinar nada: constata un hecho posterior a la
decodificación —el proceso perdió información en este punto— y dispara la
denegación sobre ese hecho, sin pretender repararlo.

**Se rechaza la fila, no se intenta recuperar el texto reintentando con otro
encoding.** Frente al mismo problema se consideró reintentar la decodificación
como Windows-1252 cuando aparece un carácter de reemplazo, ya que ese
re-decodificado siempre produce algún resultado —cada byte de un solo octeto tiene
representación en ese juego de caracteres—. Se descartó precisamente por esa
razón: "siempre produce algún resultado" no es lo mismo que "produce el resultado
correcto", y una planilla en un tercer encoding distinto de UTF-8 y de
Windows-1252 pasaría igual, generando un nombre plausible pero equivocado sin que
nada lo señalara —el mismo defecto que esta tarea corrige, desplazado un paso más
allá—. Denegar la fila y dejar que la persona la corrija en su planilla es la
opción que no arriesga introducir un dato incorrecto con apariencia de válido.

**Se evaluó cualquier celda de la fila, no sólo las columnas que el importador de
pacientes reconoce.** El defecto no es de una columna sino de la decodificación
completa del archivo: si "diagnóstico" —una columna que el importador ignora,
según ya documenta [[FASE-2_PROMPT-6]]— aparece corrompida, el resto de la fila
se decodificó con el mismo problema aunque "nombre" y "apellido" no contengan
acentos en ese registro puntual y por eso no lo delaten por sí solos. Confiar en
esas dos columnas porque no muestran el síntoma sería arbitrario frente a la causa
real, que es el archivo entero.

**El rechazo es por fila y no por archivo completo.** El módulo ya distingue
—desde TASK-32— entre los dos únicos rechazos que hablan del archivo (columnas
obligatorias ausentes, tope de filas superado) y todo lo demás, que es una
equivocación de una fila. Un archivo mayormente correcto con una sola fila
corrompida no debía perder las cuatrocientas filas restantes; se mantuvo la
consistencia con ese criterio ya establecido en lugar de agregar un tercer
rechazo de archivo completo.

**El mensaje de error es bilingüe, apartándose de la convención general del
proyecto.** `patient-import.presenter.ts` documenta explícitamente que "la
respuesta es en inglés de punta a punta, como todo contrato JSON de esta API,
aun cuando el archivo que describe es en español". El mensaje de esta entrada
incluye, sin embargo, la instrucción "Guardá el CSV como UTF-8 y volvé a subir
el archivo" en español, a pedido explícito de la usuaria durante esta misma
conversación. Es una única excepción deliberada y acotada a este mensaje, no una
reversión de la convención: la persona que lee este error específico es
administrativa y de la clínica, y la acción que debe tomar —guardar el archivo
con otra codificación desde su propio Excel— es un paso concreto sobre software
en español: se prioriza que la instrucción sea accionable de inmediato por sobre
la consistencia idiomática del contrato.

## Alternativas descartadas

- **Detección de encoding en sentido estricto** (adivinar el encoding original a
  partir del archivo): descartada por depender de información que la
  decodificación a UTF-8 ya perdió; el conteo de caracteres de reemplazo no
  necesita adivinar nada.
- **Reintentar la decodificación como Windows-1252 ante un carácter de
  reemplazo**: descartada porque siempre produce un resultado, correcto o no, y
  arriesgaría introducir un nombre incorrecto con apariencia de válido si el
  archivo estuviera en un tercer encoding.
- **Evaluar sólo las columnas que el importador de pacientes reconoce**:
  descartada porque el defecto es de la decodificación completa del archivo, no
  de una columna puntual; una fila puede no delatarse por "nombre"/"apellido" y
  sí por una columna que el importador ignora.
- **Rechazar el archivo completo ante la primera fila sospechosa**: descartada
  por rompimiento con la propiedad de importación parcial que sostiene todo el
  módulo desde TASK-32.
- **Abordar en esta misma tarea la nota secundaria del propio ticket** —que una
  columna no reconocida de la planilla (p. ej. "diagnóstico") se descarta hoy sin
  advertencia—: descartada por alcance. El ticket mismo la marca como de menor
  prioridad y no la incluye en sus criterios de aceptación; queda pendiente para
  una tarea futura si se decide abordarla.

## Entidades / puertos / adaptadores tocados

- `src/common/spreadsheet/spreadsheet-cell.ts` (modificado): `hasReplacementCharacter`.
- `src/common/spreadsheet/spreadsheet.reader.ts` (modificado): campo
  `suspectEncoding` en `SpreadsheetRow`, calculado en `toRows`.
- `src/patients/patient-import.row.ts` (modificado): corte temprano en
  `validateImportRow` cuando la fila viene marcada como sospechosa.

No se tocaron el esquema de la base de datos, los puertos ni los adaptadores de
integración: la corrección es interna a la lectura de planillas y a la
validación de una fila, sin alterar el contrato de ningún punto de acceso.

## Tests y qué validan

- `src/common/spreadsheet/spreadsheet-cell.spec.ts` (ampliado, 3 pruebas
  nuevas): `hasReplacementCharacter` encuentra el carácter dondequiera que caiga
  en la cadena, y no confunde texto acentuado normal ni texto llano con el
  síntoma que busca.
- `src/common/spreadsheet/spreadsheet.reader.spec.ts` (ampliado, 4 pruebas
  nuevas): un `.csv` construido con `Buffer.from(texto, 'latin1')` —que
  reproduce exactamente los bytes que Windows-1252 escribe para las letras
  acentuadas del español, ya que ese juego de caracteres coincide con Latin-1 en
  ese rango— marca la fila como sospechosa; la marca aparece igual cuando la
  celda corrompida está en una columna que el importador no mapea; una fila sin
  acentos no queda marcada; y una fila decodificada desde UTF-8 real tampoco.
- `src/patients/patient-import.row.spec.ts` (ampliado, 1 prueba nueva): una fila
  que el lector marcó como sospechosa se rechaza —con el mismo conjunto de datos
  que en otra prueba se acepta sin la marca— antes de construir el DTO, sin que
  el nombre corrompido llegue a persistirse.
- `test/patients-import.e2e-spec.ts` (ampliado, 1 prueba nueva, por la capa
  HTTP): una carga real de un `.csv` codificado en Windows-1252 con dos filas,
  una con nombre acentuado y otra sin acentos. El informe cuenta una creada y
  una rechazada, la entrada de error nombra la fila correcta sin campo
  específico y menciona "UTF-8"; el paciente de la fila corrompida no queda
  persistido bajo ningún nombre, y el de la fila sin acentos sí, con sus
  valores correctos —éste es el criterio de aceptación textual del ticket.
- Se comprobó además, apartando temporalmente los cambios con `git stash`, que
  seis suites end-to-end no relacionadas (`professionals-abm`,
  `professional-config`, `professional-schedules`, entre otras) ya fallaban por
  `409 Conflict` contra el estado previo de esta tarea, evidencia de datos de
  prueba compartidos entre suites en la base de desarrollo y no una regresión
  de este cambio.
- Ejecución: suite unitaria en verde (39 suites / 463 pruebas); suite end-to-end
  del importador de pacientes en verde (1 suite / 21 pruebas); compilación
  (`tsc --noEmit`) y análisis estático (`eslint`) sin errores sobre todos los
  archivos tocados. Todos los datos usados en las pruebas son ficticios.

## Figuras pendientes

Ninguna nueva. La corrección refina un paso ya cubierto por la figura tentativa
16 (flujo de la importación de pacientes preexistentes) sin introducir una rama
nueva del diagrama: el rechazo por codificación se suma al mismo punto de
validación por fila que la figura ya representa.

## Componente y referencia

- Componente: backend.
- Branch: `feature/TASK-119-csv-encoding-detection`, creada desde `main`.
  Commit `6d73e34` ("TASK-119: flag CSV rows decoded in the wrong encoding
  instead of importing corrupted names"), pusheado a
  `origin/feature/TASK-119-csv-encoding-detection`. Sin pull request abierto
  todavía al momento de redactar esta entrada.
- Ticket: TASK-119 (Jira), "[CORRECCIÓN] TASK-32 – Importador de pacientes
  asume UTF-8 sin fallback y corrompe nombres con tildes/ñ". Depende de TASK-32
  ([[FASE-2_PROMPT-6]]). Misma convención de bitácora dedicada para tareas
  puntuales dentro de la fase del ticket original que TASK-92/TASK-98/TASK-106
  ([[FASE-1_PROMPT-9]], [[FASE-2_PROMPT-13]], [[FASE-2_PROMPT-14]]).
- Pendiente explícitamente fuera de alcance: la nota secundaria del propio
  ticket sobre columnas no reconocidas de la planilla descartadas hoy sin
  advertencia en el informe (p. ej. "diagnóstico"), marcada de menor prioridad
  por el ticket mismo.
