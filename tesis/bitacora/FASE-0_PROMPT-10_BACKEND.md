# Fase 0 — Fundaciones (backend) — alta administrativa de usuarios: `POST /users` para dar de alta el login de un profesional (TASK-205)

## Qué se implementó

Una auditoría de código multi-agente contra el documento de requisitos,
ejecutada el 28 de agosto de 2026, había señalado que ninguna ruta del
código permitía crear el login de un profesional: `bcrypt.hash` solo
aparecía en el script de siembra (`prisma/seed.ts`, `prisma/seed-pilot.ts`)
y en las pruebas; el alta de un profesional creaba únicamente el registro
`Professional`, sin ninguna cuenta `User` vinculada; y el controlador de
usuarios exponía solo una lectura (`GET /users`), sin ningún equivalente
de escritura en todo el código fuente. El hallazgo se había marcado como de
prioridad baja y a confirmar, no como un hueco a resolver de inmediato,
porque el propio texto del documento de requisitos de una revisión anterior
sugería que el alta de usuarios podía estar pensada para vivir fuera de
este sistema, delegada a un sistema de seguridad corporativo externo que
administraría usuarios y roles.

Antes de implementar, se releyó la fuente de verdad vigente —el documento
de requisitos compartido en Drive, en su revisión más reciente
("documento reunión ACTUALIZADO")— y se confirmó que esa hipótesis ya no
aplica: esa revisión incorpora explícitamente el "Módulo Seguridad –
Gestión de Roles de Usuarios" entre los módulos que el sistema sí
administra, con perfiles, roles, facultades y usuarios estructurados
dentro de la propia plataforma, en lugar de figurar entre los módulos que
el sistema no administrará. Confirmado esto, la falta de una acción
administrativa auditada para dar de alta un login dejó de ser una decisión
de alcance pendiente de confirmar y pasó a ser, simplemente, la
funcionalidad faltante que la auditoría había detectado: en el estado
anterior del código, poner en producción el login de un profesional nuevo
exigía acceso directo a la base de datos o al script de siembra, en lugar
de una acción administrativa auditada dentro de la propia aplicación.

Se agregó `POST /users`, restringida al rol ADMIN, que crea una cuenta de
acceso para un profesional ya existente (vinculándola a su
`professionalId`) o para otro administrador. El cuerpo de la solicitud
admite únicamente los roles ADMIN y PROFESSIONAL —nunca SYSTEM, reservado
para procesos internos y sin ningún camino de inicio de sesión manual,
como ya establecía `AuthService.login`—, exige `professionalId` cuando el
rol es PROFESSIONAL y lo prohíbe en cualquier otro caso, reproduciendo a
nivel de aplicación la misma restricción `CHECK` que ata ambos campos en
la base de datos (TASK-93, documentada en FASE-0_PROMPT-7). La contraseña
se cifra con `bcrypt` con el mismo factor de costo que usa el resto del
código; un correo electrónico duplicado se traduce en un `409`, y la
creación de la cuenta junto con su entrada en la traza de auditoría se
escriben dentro de la misma transacción, siguiendo el mismo patrón que ya
usa el resto de las operaciones de escritura del sistema.

## Decisiones y por qué

**Verificar la existencia y pertenencia del profesional consultando
directamente la tabla `Professional` a través del cliente acotado por
tenant, en lugar de invocar al servicio de profesionales.** El módulo de
profesionales ya importa el módulo de usuarios —lo necesita para revocar
tokens al dar de baja a un profesional (P8.1b, TASK-87)—, así que una
dependencia en sentido inverso habría cerrado un ciclo entre ambos
módulos. Se optó por duplicar localmente la misma consulta mínima que ya
usa `UsersService.isLinkedProfessionalActive` para comprobar si un
profesional vinculado sigue activo, en lugar de forzar una dependencia
cruzada solo para reutilizar una verificación de una línea.

**No exigir que el profesional esté activo para poder crear una cuenta a
su nombre.** Un profesional dado de baja ya no puede iniciar sesión de
todos modos, porque `AuthService.login` rechaza esa situación de forma
independiente; exigir que estuviera activo en el momento de la creación
de la cuenta habría impedido preparar el alta de un profesional que está a
punto de reincorporarse, sin ninguna ganancia real de seguridad, dado que
el control efectivo ya ocurre en el inicio de sesión.

**Traducir el correo electrónico duplicado a un `409`, no a un `400`.**
El correo es una clave natural única a nivel global, no solo dentro de un
tenant, y una colisión detectada recién al escribir es una condición de
carrera legítima entre dos altas concurrentes, no una entrada mal
formada. Se siguió el mismo criterio que ya distingue, en el resto del
código, entre una colisión de este tipo —el DNI de un paciente, tratado
como `409`— y una violación de unicidad que ocurre enteramente dentro de
una única solicitud, como las matrículas duplicadas dentro de la misma
alta de un profesional, tratada como `400` por no ser una carrera sino un
cuerpo de solicitud redundante en sí mismo.

## Alternativas descartadas

Se consideró incorporar la creación de la cuenta directamente dentro de
`ProfessionalsService.create`, de modo que un único alta pudiera crear el
profesional y su login en la misma solicitud. Se descartó porque el alta
de un profesional y el alta de su cuenta de acceso resultaron ser, en la
práctica ya observada en el propio código —el alta de un profesional
señalado por la auditoría como caso concreto sin cuenta vinculada—, dos
decisiones administrativas independientes con su propio momento: nada en
el documento de requisitos ni en el código existente sujeta la creación
de una cuenta al momento exacto del alta del profesional, y una ruta
separada permite dar de alta o reemplazar un login sin reabrir el resto
de los datos generales del profesional.

## Entidades / puertos / adaptadores tocados

- `src/users/dto/create-user.dto.ts` (nuevo): `CreateUserDto`, con el rol
  restringido a ADMIN/PROFESSIONAL y el `professionalId` condicionalmente
  requerido según el rol.
- `src/users/users.constants.ts` (nuevo): longitud mínima de contraseña.
- `src/users/user.presenter.ts` (nuevo): proyección de `User` a la
  respuesta pública, sin `passwordHash` ni `tokenVersion`.
- `src/users/users.service.ts`: método `create`, con la verificación de
  pertenencia del profesional, el cifrado de la contraseña, la traducción
  del correo duplicado a `409` y la escritura conjunta con su entrada de
  auditoría dentro de una transacción.
- `src/users/users.controller.ts`: `POST /users`, restringida a ADMIN;
  `GET /users` se reescribió sobre el mismo presentador nuevo en lugar de
  su mapeo manual anterior, para que ambas rutas no puedan divergir en qué
  campos exponen.

## Tests y qué validan

- `src/users/users.service.spec.ts` (ampliado): cobertura unitaria de
  `create` —alta de una cuenta PROFESSIONAL vinculada a un profesional
  propio del tenant con su entrada de auditoría, alta de una cuenta ADMIN
  sin `professionalId`, rechazo cuando el profesional no pertenece al
  tenant, rechazo cuando se envía `professionalId` junto a un rol que no
  es PROFESSIONAL, traducción de la violación de unicidad de correo a
  `409`, y que cualquier otro error se repropaga sin modificar— además de
  verificar que el valor cifrado nunca coincide con la contraseña en
  texto plano y sí verifica contra ella.
- `test/users-admin-provisioning.e2e-spec.ts` (nuevo): contra PostgreSQL
  real, con dos organizaciones separadas. Cubre el flujo completo —alta
  de la cuenta de un profesional previamente sin vincular y inicio de
  sesión exitoso con las credenciales nuevas—, el rechazo sin
  autenticación (`401`) y con un rol distinto de ADMIN (`403`), los
  rechazos de validación (`professionalId` faltante, `professionalId`
  junto a un rol no PROFESSIONAL, rol SYSTEM, contraseña por debajo del
  mínimo), el rechazo de un `professionalId` de otra organización —
  indistinguible de uno inexistente, según la convención ya vigente en el
  resto del código— y el `409` por correo duplicado.
- Suite completa verde: 93 suites/1026 pruebas unitarias y 54 suites/663
  pruebas de extremo a extremo (`--runInBand`, contra PostgreSQL local).
  Lint y formateo sin errores en cada archivo tocado.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-205-professional-user-provisioning`
  (creada a partir de `origin/main` fresco).
- Ticket: TASK-205 ("[BAJO] Confirmar modelo operativo: no existe alta de
  usuarios (login) por API"), generado a partir de la auditoría de código
  multi-agente del 28 de agosto de 2026 contra el anteproyecto de tesis y
  el documento de requisitos. Se reabrió como tarea de implementación, no
  solo de confirmación, tras verificar contra la revisión vigente del
  documento de requisitos en Drive que el Módulo Seguridad sí está dentro
  del alcance de este sistema.
