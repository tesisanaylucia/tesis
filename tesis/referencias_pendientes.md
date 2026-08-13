# Referencias pendientes

Marcadores `[CITA: tema]` insertados en el cuerpo del texto que todavía no
tienen una fuente bibliográfica verificada asociada. Cada fila se retira de
esta tabla cuando la cita real se incorpora al texto (reemplazando el
marcador por el número de cita `[N]` correspondiente, según
`guia_estilo.md`) y se agrega a la sección 5. Referencias.

No se cargó ninguna fuente real todavía: esta tabla es exclusivamente un
registro de qué afirmaciones necesitan respaldo bibliográfico y de qué
tipo, para que las autoras completen la fuente concreta.

## Marco Teórico — Capítulo 2

| N° | Marcador (tal como aparece en el texto) | Sección | Tipo de fuente sugerida |
|---|---|---|---|
| 1 | evolución de los sistemas conversacionales en salud, de IVR a asistentes basados en LLM | 2.1 | Artículo académico / revisión sobre historia de chatbots en salud |
| 2 | distinción entre chatbots administrativos y chatbots clínicos/terapéuticos en salud mental | 2.1 | Artículo académico sobre taxonomía de asistentes conversacionales en salud mental |
| 3 | efecto de los recordatorios automatizados sobre la tasa de inasistencia a turnos en salud | 2.1 | Estudio empírico / metaanálisis sobre recordatorios y ausentismo (*no-show*) |
| 4 | riesgos de la automatización conversacional en contextos de salud mental | 2.1 | Artículo académico o guía de buenas prácticas sobre riesgos de chatbots en salud mental |
| 5 | arquitectura transformer y mecanismos de atención | 2.2 | Paper fundacional de la arquitectura transformer (p. ej. "Attention Is All You Need") o material de referencia equivalente |
| 6 | function calling / tool use en modelos de lenguaje | 2.2 | Documentación técnica o paper sobre uso de herramientas en LLM |
| 7 | modelo de webhooks de WhatsApp Business Platform | 2.3 | Documentación oficial de WhatsApp Business Platform (Meta) |
| 8 | comparación entre monolito modular y arquitectura de microservicios | 2.4 | Artículo académico o libro de referencia sobre estilos arquitectónicos de software |
| 9 | arquitectura hexagonal / puertos y adaptadores, Cockburn | 2.4 | Artículo original de Alistair Cockburn sobre arquitectura hexagonal |
| 10 | modelos de aislamiento en arquitecturas multi-tenant | 2.4 | Artículo académico o whitepaper sobre patrones de multi-tenancy |
| 11 | patrones de configuración dirigida por datos / motores de reglas de negocio | 2.4 | Libro o artículo sobre *business rules engines* / configuración como datos |
| 12 | arquitectura y convenciones de NestJS | 2.5 | Documentación oficial de NestJS |
| 13 | características y garantías transaccionales de PostgreSQL | 2.5 | Documentación oficial de PostgreSQL |
| 14 | modelos de control de acceso electrónico basados en IoT | 2.6 | Artículo académico sobre control de acceso IoT |
| 15 | rol del gateway en arquitecturas de cerraduras inteligentes Bluetooth | 2.6 | Documentación técnica de la industria de cerraduras inteligentes BLE |
| 16 | consideraciones de seguridad y auditoría en control de acceso IoT | 2.6 | Artículo académico o guía de seguridad sobre control de acceso IoT |
| 17 | comparación de enfoques de desarrollo móvil multiplataforma | 2.7 | Artículo académico o informe técnico comparando enfoques cross-platform |
| 18 | beneficios del tipado estático en el desarrollo de interfaces de usuario a gran escala | 2.7 | Artículo académico sobre impacto del tipado estático en calidad de software |
| 19 | texto vigente de la Ley 25.326 y su reglamentación | 2.8 | Texto oficial de la Ley 25.326 (InfoLEG) y decreto reglamentario |
| 20 | texto vigente de la Ley 26.529 y su reglamentación | 2.8 | Texto oficial de la Ley 26.529 (InfoLEG) y decreto reglamentario |
| 21 | relación entre la Ley 25.326 y la Ley 26.529 aplicada a sistemas de información en salud | 2.8 | Artículo académico o dictamen sobre protección de datos de salud en Argentina |
| 22 | patrones de ejecución de tareas programadas en aplicaciones backend | 2.4 | Artículo académico o libro de referencia sobre *scheduled tasks* / *cron jobs* en arquitecturas backend |
| 23 | idempotencia en el procesamiento periódico de datos | 2.4 | Artículo académico o libro sobre idempotencia y procesamiento por lotes/periódico |
| 24 | ejecución de tareas programadas en aplicaciones multi-tenant | 2.4 | Artículo académico o whitepaper sobre scheduling en sistemas multi-tenant |

## Datos volátiles marcados `[VERIFICAR: ...]` (no son citas bibliográficas)

Estos marcadores no requieren una fuente bibliográfica sino una
verificación puntual de un dato que cambia con el tiempo (versión, precio,
límite de una API) al momento de dar por cerrado el capítulo. Se listan
acá solo para no perderlos de vista; no se completan con una cifra de
memoria bajo ninguna circunstancia.

| N° | Marcador | Sección | Qué verificar |
|---|---|---|---|
| 1 | características y versión de modelo de la familia Claude vigentes al momento de redacción | 2.2 | Nombre y características del modelo de la familia Claude vigente al momento de cerrar el capítulo |
| 2 | estado y fecha de discontinuación de la On-Premises API | 2.3 | Si la On-Premises API de WhatsApp sigue discontinuada y desde cuándo |
| 3 | categorías de plantillas y modelo de costo vigentes | 2.3 | Categorías de plantillas de WhatsApp Business Platform y su modelo de costo actual |
| 4 | niveles de tier y límites de mensajería vigentes | 2.3 | Niveles de tier de mensajería de WhatsApp Business Platform y límites asociados |
| 5 | versión de Prisma vigente al momento de redacción | 2.5 | Versión de Prisma a citar en el texto |
| 6 | alcance exacto y límites de la API pública de TTLock vigentes al momento de redacción | 2.6 | Límites concretos (códigos concurrentes, tasa de solicitudes, etc.) de la API de TTLock |
| 7 | arquitectura de puente/motor de React Native vigente al momento de redacción | 2.7 | Motor/arquitectura de comunicación JS–nativo de React Native vigente (dado que este componente cambió de diseño entre versiones) |
