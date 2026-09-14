# Fase 5 — Capa conversacional y WhatsApp (backend) — corrección de importes y copago (TASK-163, corrección a TASK-51)

## Qué se implementó

Una auditoría automatizada de código contra las fuentes de verdad (SRS)
detectó el hallazgo TUR 16: el bot nunca informaba ningún importe, ni
siquiera el total que el documento de requisitos exige mostrar. La causa
era que TASK-51 (P5.6) había resuelto "no revelar el copago" colapsando esa
regla con "no revelar ningún monto en pesos", tanto en el guardrail
monetario (TASK-49) como en la instrucción del manual de flujos. El
documento de requisitos en realidad distingue dos reglas: mostrar el
importe total de la consulta —con obra social provincial, como el valor
base más "copago" (p. ej. "$20000 + copago"); particular, como un único
valor (p. ej. "$60000")— y no revelar nunca el monto del copago en sí. Esta
tarea separa ambas reglas donde antes eran una sola.

## Decisiones y por qué

**Se agregó una única columna de importe por profesional
(`consultationFee`), no dos.** Su significado depende de `careType`, el
mismo campo que ya distingue cómo se paga la consulta: para
`HEALTH_INSURANCE` es el importe base que se combina con "+ copago"; para
`PRIVATE` es el importe particular completo. Como `careType` es un valor
único por profesional y nunca ambos a la vez, una sola columna alcanza sin
introducir un segundo campo que pudiera quedar inconsistente con el
primero. El importe del copago en sí no tiene columna ni la va a tener: el
documento de requisitos es explícito en que su monto nunca se informa, así
que no hay ningún valor de copago que el sistema deba conocer.

**El campo se sumó al endpoint de configuración ya existente
(`PATCH /profesionales/:id/configuracion`), no a uno nuevo.** Sigue el
mismo patrón que `consultationDuration`/`slotCadence` (P1.4): nullable
hasta que el profesional lo carga desde la aplicación, con la misma
validación de entero estrictamente positivo y el mismo tratamiento de
`null` explícito para borrarlo. Se agrupa ahí porque es exactamente el tipo
de dato que esa configuración ya reúne —ajustes que el profesional fija una
vez y el chatbot lee después— y no un dato de alta.

**La regla de guardrail (Regla 2) se reescribió para exigir la palabra
"copago" y ya no "costo" ni "precio".** La redacción original de TASK-49
bloqueaba una respuesta si aparecía un monto junto a cualquiera de esas
tres palabras, o incluso un monto en pesos por sí solo (`DOLLAR_AMOUNT_PATTERN`
bastaba sola para bloquear). Como "costo" y "precio" son exactamente cómo
se nombra ahora el total permitido, dejaron de ser disparadores: sólo
"copago" sigue siéndolo, y un monto en pesos sin la palabra "copago" cerca
ya no alcanza para bloquear nada.

**Se reconoce y excluye la forma aditiva que el documento de requisitos
prescribe para el total ("`<importe>` + copago") antes de evaluar la
regla.** Sin esta exclusión, decir el total exactamente como el documento
de requisitos lo pide seguiría bloqueado, porque la palabra "copago" y un
monto aparecen igual de cerca en esa frase que en una filtración real del
copago. La expresión que reconoce esa forma aditiva se aplica primero,
quitando el fragmento del texto antes de buscar la palabra "copago" y un
monto en lo que queda: si el único "copago" del texto formaba parte de esa
forma aditiva, la regla ya no lo ve.

**El manual de flujos (`CONVERSATION_FLOWS_PROMPT`) pasó a informar el
importe por tipo de atención, en vez de negar cualquier cifra.** Se
reescribió la sección "Información de obra social" —ahora "Información de
obra social e importes"— para que, cuando `consultationFee` no sea nulo, el
bot arme el texto con el formato del documento de requisitos según
`careType`, y para que cuando sea nulo diga que no tiene el importe
cargado. La única prohibición que queda es la del copago, con el mismo
texto exacto que ya usa el guardrail (`COPAY_BLOCKED_MESSAGE`, importado
igual que ya se importaban los textos de fuera de alcance y urgencia), para
que lo que se le pide al modelo y lo que el sistema impone no puedan
divergir.

**Se corrigió de paso una inconsistencia preexistente, detectada al
revisar el mismo párrafo: el manual de flujos nombraba el valor de
`careType` para atención particular como "PARTICULAR".** El enum real
(`CareType`, en `schema.prisma`) y lo que la herramienta `list_professionals`
efectivamente envía al modelo es "PRIVATE". No formaba parte del hallazgo
auditado, pero se corrigió en la misma tarea por tratarse exactamente del
mismo párrafo que ya se estaba reescribiendo.

## Entidades, puertos y adaptadores tocados

- `prisma/schema.prisma`: nueva columna `Professional.consultationFee`
  (`Int?`), con migración `20260914182545_add_professional_consultation_fee`.
- `src/professionals/dto/update-professional-config.dto.ts`: nuevo campo
  `consultationFee` (`@OptionalNullable`, `@IsInt`, `@Min(1)`).
- `src/professionals/professionals.service.ts`: `updateConfiguration`
  persiste el nuevo campo.
- `src/professionals/professional.presenter.ts`: `ProfessionalResponse`
  expone `consultationFee`.
- `src/chatbot/tools/professional.tools.ts`: `ChatbotProfessional` expone
  `consultationFee`; la descripción de la herramienta menciona el importe.
- `src/chatbot/guardrail.constants.ts`: `COPAY_TRIGGER_WORD` (antes
  `COPAY_TRIGGER_WORDS`, incluía "costo"/"precio") y nuevo
  `COPAY_ADDITIVE_TOTAL_PATTERN`.
- `src/chatbot/guardrail.service.ts`: `mentionsCopayAmount` reescrito para
  excluir la forma aditiva antes de evaluar la regla.
- `src/chatbot/conversation-flows.constants.ts`: sección "Información de
  obra social e importes" reescrita; importa `COPAY_BLOCKED_MESSAGE`.
- `postman/psique-backend.postman_collection.json`: regenerado
  (`scripts/generate-postman-collection.js`) para el nuevo campo en
  `PATCH .../configuracion`, con su descripción ampliada a mano.

## Tests

- `src/chatbot/guardrail.service.spec.ts`: reescritos los casos de la Regla
  2 que asumían el bloqueo por "costo"/"precio" o por un monto en pesos por
  sí solo; agregados casos que verifican que el total particular, el total
  aditivo con obra social (con y sin signo "$") y un total dicho junto al
  otro pasan sin bloquearse, y que un copago declarado junto a un total
  aditivo legítimo en la misma respuesta igual se bloquea.
- `src/chatbot/tools/professional.tools.spec.ts`: casos nuevos para
  `consultationFee` presente y nulo en el resultado de `list_professionals`.
- `src/professionals/professionals.service.spec.ts`: casos nuevos de
  `updateConfiguration` persistiendo y limpiando `consultationFee`.
- `test/professional-config.e2e-spec.ts`: casos nuevos de configuración,
  validación (no positivo, no entero) y borrado explícito de
  `consultationFee` contra Postgres real.
- `test/chatbot-flows.e2e-spec.ts`: la prueba de tipo de atención pasa a
  verificar también `consultationFee`; casos nuevos de conversación de
  punta a punta que verifican que el total con obra social ("+ copago") y
  el total particular llegan intactos al paciente, y que un copago que el
  modelo dijera igual se reemplaza por el texto canónico del guardrail.

Suite completa en verde al cierre de la tarea: 90 suites / 969 pruebas
unitarias y 53 suites / 634 pruebas de integración (`--runInBand`, contra
Postgres real), con una única falla intermitente reproducida también contra
`main` sin cambios (`appointment-reassignment.e2e-spec.ts`, ya documentada
como flakiness preexistente e independiente de esta tarea) que desapareció
al repetir la corrida completa. Lint (`eslint --fix`) y verificación de
tipos (`tsc --noEmit`) sin errores.

## Figuras pendientes

Ninguna nueva: `Professional` ya figura en el DER general del subdominio de
profesionales: esta tarea le agrega una columna, no una entidad.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-163-consultation-fee-copay-guardrail`, creada desde `main`.
- Ticket: TASK-163 (Jira), "El bot nunca informa importes, ni siquiera el
  total permitido por el SRS" — tarea generada automáticamente por
  auditoría de código vs. SRS (hallazgo TUR 16), validada por la usuaria
  antes de implementarse, con la aclaración de que el copago sólo aplica a
  obra social provincial (única obra social que este sistema recibe para
  consultas) y que el importe particular sí debe informarse siempre.
- Corrige una decisión tomada en FASE-5_PROMPT-6 (TASK-51, P5.6): "Los
  importes quedaron fuera de alcance por decisión del ticket, alineado con
  el guardrail monetario" — la relectura del documento de requisitos contra
  ese guardrail mostró que el alcance excluido había sido mayor al que el
  documento realmente exige.
