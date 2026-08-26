# Fase 5 — Capa conversacional y WhatsApp (backend) — tests de conversación de punta a punta (TASK-54)

## Qué se implementó

Se agregó la suite de pruebas de integración que el propio Módulo 5 venía
dejando pendiente desde su primera tarea: una conversación entera del
chatbot dirigida de la misma forma en que llega en producción —un
`POST /webhook` firmado, como los que atiende `WhatsappWebhookController`
desde TASK-53— en lugar de invocar `OrquestadorService.procesar`
directamente, y verificada contra el espía de `MessagingPort` en vez de
contra el valor de retorno de una función. Cubre los nueve flujos que pide
el ticket: reserva completa (identificación por documento →
consentimiento → disponibilidad → reserva → confirmación), cancelación con
ofrecimiento de reprogramación, confirmación del turno a las 24 horas,
pregunta de FAQ existente en la tabla del inquilino, solicitud de receta,
guardrail de monto de copago, mensaje de urgencia fuera de alcance, edad
mínima con profesional "sólo mayores" y aislamiento completo entre dos
inquilinos.

El puerto de IA admite dos modos, elegidos por una única variable de
entorno y nada más en el archivo: simulado (por defecto, lo que corre en
integración continua, con un doble de prueba que reproduce exactamente la
secuencia de herramientas que llamaría el modelo) o real, detrás de
`USE_REAL_AI=true`, que deja conectado el `OpenAiAdapter` real de
`IntegrationsModule` en lugar de reemplazarlo. Se agregó además un grupo de
pruebas separado que reemplaza, un nivel por debajo de los puertos, tanto
el cliente del SDK de `openai` como la función `fetch` que usa
`WhatsAppCloudAdapter`, por dobles que lanzan una excepción si llegan a
invocarse — de modo que "el modo simulado no llama a ninguna API externa"
queda comprobado contra el límite real de red y no sólo confiado a que el
reemplazo de los puertos haya funcionado.

## Decisiones y por qué

**El archivo nuevo no repite lo que ya prueba `chatbot-flows.e2e-spec.ts`
(TASK-50).** Ese archivo sigue siendo la cobertura profunda por flujo sobre
el orquestador directamente —cada rama de la edad mínima, los datos
faltantes de un alta, la información de obra social—; el archivo de esta
tarea existe para probar una cosa distinta que ninguno de los dos
anteriores probaba junta: que el punto de entrada HTTP real, con firma
válida, efectivamente llega al orquestador y que la respuesta efectivamente
sale por `MessagingPort`, para cada flujo que el ticket nombra
explícitamente, incluyendo el aislamiento multi-tenant medido en ese mismo
punto de entrada.

**Sólo la mitad de cada aserción queda condicionada al modo del puerto de
IA.** El ticket pide que la misma suite corra en ambos modos "sin cambiar
el código de los tests"; como un modelo real no repite ni la redacción
exacta ni necesariamente la misma secuencia de herramientas que el doble de
prueba, cada aserción se clasificó según de qué depende. Las que dependen
de la redacción exacta o de la secuencia exacta de herramientas quedan
detrás de `if (!useRealAi)`. Las que son invariantes del dominio,
verificables con independencia de qué modelo respondió —que nunca se
reserva un turno para un menor con un profesional restringido, que el
ofrecimiento de reprogramación se agrega de forma determinística después de
una cancelación exitosa, que el corte por urgencia nunca llega a invocar al
puerto de IA— no llevan esa condición y corren igual en los dos modos. La
prueba del guardrail de copago es la única excepción declarada: forzar que
un modelo cumplidor diga un monto de dinero no es algo que un modo real
pueda hacer, así que esa prueba entera queda `it.skip` bajo
`USE_REAL_AI=true`.

**Se verificó manualmente, y no sólo se dio por sentado, que el flag
efectivamente cambia de adaptador.** Corrida sin `USE_REAL_AI`, la
suite completa pasa contra el doble de prueba; corrida con
`USE_REAL_AI=true` contra la clave de OpenAI local (una clave de prueba,
no válida), las pruebas que dependen del modelo fallan con un 401 real de
la API de OpenAI —la prueba de la urgencia, que no llama al modelo, y la
del copago, saltada, siguen en verde— lo que confirma que el flag realmente
enruta al adaptador real y no a un atajo que sólo aparenta hacerlo.

**Se descubrió y corrigió, al intentar validar todo esto, un defecto
anterior a esta tarea que habría impedido correr la suite en integración
continua real.** `openAiClientProvider` y `WhatsAppCloudAdapter` leen
`OPENAI_API_KEY` y las cuatro variables `WHATSAPP_*` con
`ConfigService.getOrThrow` al construirse, y todo proveedor del grafo de
módulos de Nest se construye al arrancar la aplicación aunque su token
esté reemplazado por un doble en una prueba puntual — el reemplazo evita
que se *llame*, no que se *construya*. El flujo de trabajo de integración
continua (`.github/workflows/ci.yml`) nunca declaraba esas cinco variables,
sólo las de base de datos y JWT, así que la aplicación nunca habría podido
arrancar ahí, en ninguna de las suites existentes, no sólo en la nueva.
Corregido agregando valores de relleno documentados, ninguno de los cuales
llega jamás a usarse para una llamada real, porque `AI_PORT` y
`MESSAGING_PORT` están siempre reemplazados por un doble en cada prueba de
integración del repositorio.

**Tres piezas que antes vivían duplicadas, o a punto de duplicarse, se
extrajeron a `test/support/`.** `ScriptedAiPort` (el doble de prueba del
puerto de IA) vivía como una clase de setenta líneas dentro de
`chatbot-flows.e2e-spec.ts`; la firma y el armado del cuerpo del webhook de
WhatsApp vivían como funciones locales dentro de `whatsapp-webhook.e2e-spec.ts`;
los constructores de profesional, paciente consentido y turno de prueba
también vivían sólo en el primero. Escribir la nueva suite iba a necesitar
las tres, así que se movieron a `test/support/` (`scripted-ai-port.ts`,
`whatsapp-webhook-payload.ts`, `chatbot-fixtures.ts`) y los dos archivos
existentes se reescribieron para importarlas en lugar de mantener su propia
copia — verificado que ninguno de los dos cambió de comportamiento
corriendo su propia suite antes y después de la extracción.

## Entidades, puertos y adaptadores tocados

Ninguno: es una tarea puramente de pruebas, sin cambios de esquema, de
migración ni de código de aplicación. El único archivo de infraestructura
tocado fuera de `test/` es `.github/workflows/ci.yml` (variables de entorno
de relleno) y `.env.example` (documentación de `USE_REAL_AI`).

## Tests

- `test/chatbot-conversation-e2e.e2e-spec.ts`, nuevo: diez casos —uno por
  cada uno de los nueve flujos del ticket, más el grupo de interceptores
  que prueba la ausencia de llamadas externas en modo simulado.
- `test/chatbot-flows.e2e-spec.ts` y `test/whatsapp-webhook.e2e-spec.ts`:
  refactorizados para usar los constructores compartidos de
  `test/support/`, sin cambios de comportamiento.

Suite completa en verde al cierre: 69 suites / 669 pruebas unitarias y 42
suites / 501 pruebas de integración (`--runInBand`), subiendo de 41/491
antes de esta tarea. Lint y verificación de tipos sin errores.

## Figuras pendientes

- Diagrama de secuencia comparando los dos modos del puerto de IA en esta
  suite (mensaje entrante por webhook → ¿`USE_REAL_AI`? → rama simulada,
  con el doble de prueba y aserciones exactas, frente a rama real, con
  `OpenAiAdapter` y las aserciones condicionadas por invariante de
  dominio), señalando que ambas ramas ejecutan el mismo código de prueba y
  sólo difieren en qué proveedor queda conectado al arrancar el módulo.
  Sección 4.6 Capa conversacional y WhatsApp.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-54-chatbot-e2e-conversation-tests`,
  creada desde `origin/main`.
- Ticket: TASK-54 (Jira), "P5.9 – Tests de conversación de punta a punta",
  bajo el épico del Módulo 5.
- Dependencias declaradas por el ticket: TASK-46 a TASK-53 (P5.1 a P5.8),
  todas ya fusionadas a `main` antes de esta sesión.
- Fuera de alcance, declarado en el propio ticket y respetado: pruebas del
  envío del código de acceso TTLock (Módulo 6) y de la app móvil del
  profesional (Módulo 7).
- Corrección de un defecto anterior encontrado al intentar validar el
  criterio de aceptación "corre verde en integración continua sin
  credenciales": variables de entorno faltantes en
  `.github/workflows/ci.yml`, sin las cuales ninguna suite de integración
  del repositorio —no sólo la de esta tarea— podría haber arrancado la
  aplicación ahí.
