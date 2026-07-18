# Fase 1 — Profesionales (backend) — Horarios de atención y ausencias (TASK-23)

## Qué se implementó

Sobre las entidades definidas en P1.1 y el módulo de negocio construido en
P1.2 se agregaron dos capacidades del profesional: la gestión de su grilla
semanal de horarios de atención y el ABM de sus ausencias, más el punto de
extensión (hook) para que M3/M4 consuman el evento de ausencia registrada.
No se introdujeron entidades de Prisma nuevas ni migraciones: las tablas
`WorkingHour` y `Absence` (con el enumerado `Weekday`) ya habían sido
creadas en el esquema de P1.1, y esta tarea construyó la capa de servicios,
los endpoints REST y la validación de negocio sobre ellas.

Los endpoints se modelaron como sub-recursos anidados bajo el profesional,
en coherencia con las matrículas de P1.2. Para horarios: `GET
/profesionales/:id/horarios` lista los bloques del profesional y `PUT
/profesionales/:id/horarios` reemplaza la grilla completa de forma
idempotente. Para ausencias: `GET /profesionales/:id/ausencias` lista, `POST
/profesionales/:id/ausencias` registra y `DELETE
/profesionales/:id/ausencias/:ausenciaId` cancela. Toda mutación queda
protegida por JWT, acotada por organización y auditada.

## Decisiones y por qué

**La grilla de horarios se reemplaza entera (PUT idempotente) en lugar de un
ABM bloque a bloque.** El ticket pide explícitamente que el `PUT` reemplace
toda la grilla y sea idempotente. Se implementó como un borrado de todos los
bloques del profesional seguido de la inserción del conjunto nuevo, ambos
dentro de una única transacción, de modo que un fallo no deje la grilla en
un estado parcial y que enviar el mismo payload dos veces produzca siempre
el mismo estado. Se prefirió este enfoque de reemplazo total frente a un
alta/baja individual de bloques porque simplifica la validación de
solapamientos —se valida el conjunto entrante completo de una vez, sin tener
que contrastarlo contra el estado ya persistido— y porque coincide con la
forma en que una interfaz de agenda edita una grilla semanal (se guarda la
grilla, no operaciones incrementales). Un arreglo vacío es válido y limpia
la grilla.

**La validación de solapamientos vive en el servicio y opera en memoria
sobre el payload, antes de escribir.** Las horas se almacenan como texto en
formato de reloj de pared "HH:mm" de veinticuatro horas (decisión heredada
de P1.1), lo que permite comparar horarios por orden lexicográfico de cadena
sin parsear a un tipo temporal, porque el formato con cero a la izquierda es
monótono. La comprobación agrupa los bloques por día, verifica que en cada
bloque el inicio sea estrictamente anterior al fin, y que ordenados por hora
de inicio ninguno empiece antes de que termine el anterior. Dos bloques que
apenas se tocan (uno termina exactamente cuando empieza el otro) no se
consideran solapados, lo que permite modelar mañana y tarde contiguas.
Cualquier violación produce un error de validación (HTTP 400). Se ubicó en
el servicio y no en el DTO porque es una regla de negocio sobre la relación
entre varios bloques, no una restricción de forma de un campo aislado; el
DTO solo valida el formato "HH:mm" de cada hora mediante una expresión
regular compartida como constante.

**El registro de una ausencia publica un evento a través de un puerto de
dominio, no de una llamada directa.** El ticket delimita que esta tarea solo
emite el evento y que la lógica de reasignación de turnos se implementa en
M3 (P3.7 / TASK-40). Para dejar ese punto de extensión sin acoplar el
dominio de profesionales al futuro consumidor, se definió un puerto
`AbsenceEventsPort` con un único método `absenceRegistered`, siguiendo la
convención de puertos y adaptadores ya usada para las integraciones externas
(mensajería, cerradura, IA). El servicio de ausencias depende del puerto por
inyección y publica el evento —con el id del profesional, el del tenant, el
id de la ausencia y el rango de fechas— después de persistir y auditar la
ausencia. El adaptador por defecto, `LoggingAbsenceEventsAdapter`, solo
registra en el log que el evento se emitió, sin datos clínicos; M3 lo
sustituirá por un suscriptor real que dispare la reasignación sin tocar el
dominio de profesionales.

Se optó por un puerto propio del dominio en lugar de incorporar un bus de
eventos genérico (por ejemplo, `@nestjs/event-emitter`) porque el único
evento que esta fase necesita es el de ausencia registrada; un puerto
enfocado deja el contrato explícito y tipado, evita sumar una dependencia
nueva, y es coherente con el mecanismo de extensión que el resto del sistema
ya emplea. La emisión ocurre después de la persistencia y la auditoría para
que el evento describa un hecho ya consumado y consistente.

**La ausencia bloquea el período pero no altera los horarios habituales.**
El requisito es explícito: registrar una ausencia no elimina la grilla de
horarios recurrentes. El servicio de ausencias, por tanto, solo escribe en
la entidad de ausencia y nunca toca los horarios; una prueba end-to-end fija
una grilla, registra una ausencia y confirma que la grilla permanece intacta.

**Autorización coherente con P1.2: lectura abierta al tenant, mutación
restringida al dueño o al administrador.** Consultar la grilla y las
ausencias queda disponible para cualquier usuario autenticado de la
organización, porque esa información alimentará a las capas de agenda y de
chatbot; reemplazar la grilla, registrar y cancelar ausencias se protegen
con el `ProfessionalOwnershipGuard` ya existente, reutilizado tal cual, de
modo que solo el propio profesional o un administrador puedan modificar. El
guard se aplicó a nivel de método (no de controlador) precisamente para
dejar las rutas de lectura sin restricción de propiedad. Las respuestas
pasan por funciones de presentación que omiten el `organizationId`, en
coherencia con el resto del módulo.

## Alternativas descartadas

- **ABM de horarios bloque a bloque (POST/DELETE por bloque)**: descartada
  frente al reemplazo total idempotente que pide el ticket, que además
  simplifica la validación de solapamientos al operar sobre el conjunto
  entrante completo.
- **Bus de eventos genérico (`@nestjs/event-emitter`) para publicar la
  ausencia**: descartada por sobredimensionada para un único evento; se
  prefirió un puerto de dominio enfocado y tipado, sin dependencia nueva y
  coherente con los puertos ya existentes.
- **Validar los solapamientos en el DTO con class-validator**: descartada
  por tratarse de una regla sobre la relación entre varios bloques, no sobre
  la forma de un campo; el DTO solo valida el formato de cada hora.
- **Guardar las horas como tipo temporal nativo**: ya descartada en P1.1 por
  exponer una fecha artificial; esta tarea reafirma la representación textual
  "HH:mm" al construir sobre ella la validación por comparación de cadenas.

## Entidades / puertos / adaptadores tocados

- Puerto nuevo `src/domain/ports/absence-events.port.ts`
  (`AbsenceEventsPort`, token `ABSENCE_EVENTS_PORT`, tipo del evento
  `AbsenceRegisteredEvent`).
- Adaptador nuevo
  `src/infrastructure/adapters/logging-absence-events.adapter.ts`, registrado
  y exportado en `src/infrastructure/integrations.module.ts` junto a los
  adaptadores stub existentes.
- Módulo `src/professionals/`: controladores y servicios nuevos de horarios
  (`working-hours.controller.ts`, `working-hours.service.ts`) y de ausencias
  (`absences.controller.ts`, `absences.service.ts`); DTO nuevos
  (`replace-working-hours.dto.ts` con el bloque y la grilla,
  `create-absence.dto.ts`); funciones de presentación de horario y ausencia
  agregadas a `professional.presenter.ts`; constantes de formato de hora y
  tope de bloques en `professionals.constants.ts`. El módulo pasó a importar
  `IntegrationsModule` para resolver el puerto de eventos. Se reutilizaron
  sin cambios el `ProfessionalOwnershipGuard`, el `ProfessionalsService`
  (para anclar en el profesional padre), el `AuditService` y el cliente de
  Prisma acotado por tenant.
- Colección de Postman: regenerada para incluir los cinco endpoints nuevos,
  con cuerpos de ejemplo para el reemplazo de grilla y el registro de
  ausencia.

## Tests y qué validan

- `test/professional-schedules.e2e-spec.ts` (end-to-end, sobre PostgreSQL
  local y atravesando la capa HTTP con JWT reales, con el puerto de eventos
  sustituido por un doble que captura las publicaciones): valida los
  criterios de aceptación del ticket —reemplazo idempotente de la grilla con
  varios bloques por día; listado ordenado por día y hora; rechazo (400) de
  bloques solapados, de un bloque con inicio no anterior al fin y de una hora
  mal formada; aceptación de bloques contiguos como no solapados; rechazo
  (403) al reemplazar la grilla de otro profesional; aislamiento por tenant
  (404) sobre la grilla de un profesional de otra organización; registro de
  ausencia con emisión del evento (verificando el payload: profesional,
  tenant, id de ausencia y fecha de inicio) y su auditoría; rechazo (400) de
  una ausencia con fin anterior al inicio, sin emitir evento; conservación de
  la grilla al registrar una ausencia; listado y cancelación (204) de
  ausencias; y rechazo (403) al registrar una ausencia para otro profesional.
- `src/infrastructure/integrations.module.spec.ts` (unitario): se extendió
  para resolver el nuevo `AbsenceEventsPort` por inyección de dependencias.
- Ejecución: suites unitaria (8 suites / 29 tests) y end-to-end (11 suites /
  53 tests) completas en verde; `eslint` y `nest build` sin errores. Los
  datos usados son ficticios.

## Figuras pendientes

- Se registró una figura pendiente con el flujo del evento de ausencia:
  registro de la ausencia → publicación por el puerto `AbsenceEventsPort` →
  punto de extensión para la reasignación de M3 (ver `figuras_pendientes.md`).

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-23-schedules-absences` (creada a partir
  de `main`, con P1.1 y P1.2 ya fusionados).
- Ticket: TASK-23 ("P1.3 – Horarios de atención"). Depende de TASK-21 (P1.1,
  entidades) y TASK-22 (P1.2, ABM de profesionales), ambas fusionadas. Deja
  el punto de extensión para TASK-40 (P3.7, reasignación en M3).
