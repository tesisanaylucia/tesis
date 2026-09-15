# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Plantilla en español para el aviso in-app de solicitud de receta (TASK-202, corrección a TUR-12/TASK-160)

## Qué se implementó

Se cerró un hallazgo de severidad baja de la auditoría multi-agente fechada
2026-08-28 (la misma que generó TASK-166/TASK-188/TASK-189/TASK-190, ya
documentadas en esta fase): TUR-12/TASK-160 había migrado dos de las tres
notificaciones in-app dirigidas al profesional —el aviso de cancelación de
turno y el aviso de reasignación desde lista de espera— del texto armado a
mano en `AppointmentsService`/`WaitlistReassignmentService` al motor de
plantillas configurable por tenant (`NotificationTemplateService`), pero
dejó sin convertir la tercera: `PrescriptionRequestsService.notifyProfessional`
seguía construyendo el mensaje con una cadena en inglés que citaba el
identificador interno del paciente en crudo (`` `Patient ${request.patientId}
requested a prescription` ``), inconsistente con el resto del sistema y con
el texto que efectivamente lee un profesional de habla hispana en
`GET /notificaciones`.

Se agregó la clave `PRESCRIPTION_REQUEST_NOTICE` al mismo catálogo de
plantillas (`notification-template.constants.ts`), con el texto base
`"{patientName} solicitó una receta."`, y se reescribió
`notifyProfessional` para resolverla a través de
`NotificationTemplateService.render`, exactamente el mismo mecanismo que ya
usan `APPOINTMENT_CANCELLED_NOTICE` y `APPOINTMENT_REASSIGNED_NOTICE`.

## Decisiones y por qué

**Se usó el mismo sufijo `_NOTICE` que las dos claves hermanas, en vez de
reutilizar cualquier clave patient-facing existente.** Sigue el criterio ya
establecido por TUR-12/TASK-160: un aviso profesional-facing es un evento
distinto de cualquier confirmación o pedido dirigido al paciente, aunque
describa el mismo hecho de negocio, y cada uno debe poder personalizarse por
tenant de forma independiente.

**El mensaje nombra el nombre de pila del paciente, no su identificador.**
Es el mismo patrón que ya siguen `APPOINTMENT_CANCELLED_NOTICE` y
`APPOINTMENT_REASSIGNED_NOTICE`, y preserva la restricción original del
código reemplazado: qué medicamento pide una receta nunca se registra en
este sistema (la SRS deja esa redacción al profesional, fuera de la
aplicación), así que el mensaje sigue sin nombrar contenido clínico alguno,
solo cambia de qué manera identifica al paciente.

**El cálculo del nombre de pila del paciente para una plantilla se extrajo a
una función compartida (`patientDisplayName`, en el módulo de pacientes) en
lugar de escribir una tercera copia idéntica.** `AppointmentsService` y
`WaitlistReassignmentService` ya tenían, cada uno, su propio método privado
`patientName` con exactamente la misma consulta (nombre de pila por
identificador, con reserva al propio identificador si la fila fue borrada
de forma concurrente) — código duplicado dos veces antes de esta tarea.
Agregar una tercera llamada idéntica en `PrescriptionRequestsService` habría
sido la tercera copia del mismo cálculo, así que se extrajo una función de
módulo (no un servicio con inyección de dependencias propia, para no forzar
un nuevo acoplamiento entre módulos) que toma el cliente de Prisma ya
inyectado por cada llamador y el identificador del paciente, y se
reemplazaron los tres sitios de llamada (los dos existentes más el nuevo)
para usarla. Ninguno de los dos servicios existentes necesitó un cambio de
cableado de módulos: ambos ya inyectaban `TenantScopedPrismaService` para
otros fines.

**`PatientsModule` pasó a importar `NotificationsModule`.**
`PrescriptionRequestsService` no tenía hasta ahora ninguna dependencia del
motor de plantillas; se le inyectó `NotificationTemplateService`, siguiendo
el mismo patrón que `AppointmentsModule`/`WaitlistModule` ya usan para lo
mismo.

## Entidades / puertos / adaptadores tocados

Ninguna migración de base de datos — cambio de sólo capa de aplicación.

- `src/notifications/notification-template.constants.ts`: nueva clave
  `PRESCRIPTION_REQUEST_NOTICE` en `NotificationTemplateKey` y su plantilla
  base en `DEFAULT_NOTIFICATION_TEMPLATES`.
- `src/patients/patient-display-name.ts` (nuevo): función `patientDisplayName`
  compartida, extraída de la lógica duplicada en `AppointmentsService` y
  `WaitlistReassignmentService`.
- `src/patients/prescription-requests.service.ts`: `notifyProfessional`
  renderiza el mensaje vía `NotificationTemplateService` en lugar de
  interpolar una cadena en inglés a mano; constructor recibe
  `NotificationTemplateService`.
- `src/patients/patients.module.ts`: importa `NotificationsModule`.
- `src/appointments/appointments.service.ts` y
  `src/waitlist/waitlist-reassignment.service.ts`: el método privado
  `patientName` de cada uno se eliminó, reemplazado por la función
  compartida.

## Tests agregados o modificados

`prescription-requests.service.spec.ts`: el constructor de prueba pasa a
mockear también `patient.findFirst` (para `patientDisplayName`) y
`NotificationTemplateService.render`; la prueba que verificaba el contenido
del aviso ahora comprueba que `render` se llama con la clave
`PRESCRIPTION_REQUEST_NOTICE` y el nombre de pila del paciente, y que el
mensaje persistido es el texto ya renderizado, no la cadena en inglés
anterior.

`notification-template.service.spec.ts`: se agregó la entrada
correspondiente a `PRESCRIPTION_REQUEST_NOTICE` en el mapa de parámetros
completos que cubre, por tipo, cada clave de plantilla — el propio tipo
`Record<NotificationTemplateKey, ...>` obliga a esa entrada en tiempo de
compilación.

Los specs de `AppointmentsService` y `WaitlistReassignmentService` no
necesitaron cambios: ya mockeaban `patient.findFirst` sobre el cliente de
Prisma inyectado, así que la extracción de `patientName` a una función de
módulo es transparente para ellos.

Suite completa en verde: 146 suites/1668 pruebas con `--runInBand`,
verificación de tipos y lint sin errores. Una falla preexistente no
relacionada en `appointment-reassignment.e2e-spec.ts` (booking de un slot
retenido durante la ventana MANUAL) se reprodujo de forma idéntica contra
`main` sin ningún cambio de esta tarea, confirmando que no es una regresión.

## Figuras pendientes

Ninguna — no introduce un flujo de interacción nuevo, solo cambia el idioma
y el mecanismo de generación de un texto ya existente.

## Componente y referencia

- Componente: backend.
- Branch de referencia:
  `feature/TASK-202-in-app-notifications-spanish-templates`, creada desde
  `main` para esta tarea, con pull request abierto hacia `main`.
