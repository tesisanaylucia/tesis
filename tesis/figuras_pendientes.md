# Figuras pendientes

Figuras mencionadas o necesarias en el texto de la tesis que todavía no
fueron generadas/insertadas. Cada fila se retira de esta lista cuando la
figura correspondiente se incorpora al capítulo con su numeración y
epígrafe definitivos (ver `guia_estilo.md`).

| N° tentativo | Epígrafe propuesto | Sección | Componente |
|---|---|---|---|
| 1 | Arquitectura hexagonal del backend (dominio / aplicación / infraestructura y dirección de las dependencias) | 3.2.0 Fundaciones | backend |
| 2 | Diagrama de secuencia del scoping multi-tenant (request → interceptor de tenant → AsyncLocalStorage → extensión de Prisma) | 3.2.0 Fundaciones | backend |
| 3 | Diagrama conceptual de function calling en un LLM conversacional (turno de conversación → solicitud de función → ejecución externa → respuesta incorporada) | 2.2 Modelos de lenguaje (LLM) e IA conversacional | — |
| 4 | Modelo de mensajería de WhatsApp Business Platform (ventana de servicio de 24 horas y disparo de plantillas fuera de ventana) | 2.3 WhatsApp Business Platform (Cloud API) | — |
| 5 | Diagrama genérico de arquitectura hexagonal (puertos y adaptadores: dominio, aplicación, infraestructura) | 2.4 Arquitectura de software | — |
| 6 | Diagrama conceptual de control de acceso IoT (cerradura BLE, gateway, plataforma cloud, sistema externo) | 2.6 IoT y control de acceso electrónico | — |
| 7 | Diagrama entidad-relación del subdominio Profesionales (profesional, especialidad, matrícula, horario de atención, ausencia; cardinalidades y acotamiento por organización) | 3.2.1 Profesionales | backend |
| 8 | Flujo de autorización del ABM de Profesionales (autenticación JWT → guard de rol → guard de propiedad "administrador o dueño") | 3.2.1 Profesionales | backend |
| 9 | Flujo del evento de ausencia registrada (registro de la ausencia → publicación por el puerto AbsenceEventsPort → punto de extensión para la reasignación de turnos en M3) | 3.2.1 Profesionales | backend |
| 10 | Diagrama entidad-relación consolidado del esquema tras la revisión de integridad y normalización (claves foráneas reales hacia Organization, claves foráneas compuestas señaladas, entidades acotadas por inquilino frente a hijas y catálogo global de obras sociales) | 3.2.0 Fundaciones | backend |
| 11 | Diagrama de recursos de la aceptación de obras sociales (catálogo global GET /obras-sociales → recurso anidado GET/PUT /profesionales/:id/obras-sociales → conjunto embebido en la respuesta del profesional) | 3.2.0 Fundaciones | backend |
| 12 | Diagrama entidad-relación del subdominio Pacientes (paciente, vínculo paciente-profesional, consentimiento y solicitud de receta; cardinalidades, claves foráneas compuestas y ubicación del identificador de organización) | 3.2.2 Pacientes | backend |
| 13 | Mapa de endpoints del módulo Pacientes con permisos por rol (alta, búsqueda por documento, detalle, modificación, baja lógica, vinculación con profesional, prioridad, observaciones, estado de datos e importación masiva; alcance de administrador, proceso automatizado y profesional tratante) | 3.2.2 Pacientes | backend |
| 14 | Diagrama de estados del tipo de paciente por vínculo con cada profesional (nuevo → recurrente al completarse la primera sesión; recurrente → nuevo al alcanzarse el umbral de inactividad, evaluado en la lectura; indicación del evento que dispara cada transición y del campo de primera sesión que se escribe junto con la promoción) | 3.2.2 Pacientes | backend |
| 15 | Diagrama de secuencia del consentimiento de tratamiento de datos (verificación previa a la reserva → solicitud únicamente cuando no hay aceptación registrada → registro con marca temporal del servidor → respuesta idempotente ante una aceptación ya existente) | 3.2.2 Pacientes | backend |
| 16 | Flujo de la importación de pacientes preexistentes (archivo cargado → lector de planillas independiente del dominio → correspondencia de columnas a campos → validación por fila → transacción por fila con escritura del paciente, vínculo con el profesional y auditoría → informe con filas creadas, actualizadas y rechazadas; señalando los dos rechazos que conciernen al archivo completo y la bifurcación que deja continuar a las filas válidas) | 3.2.2 Pacientes | backend |
