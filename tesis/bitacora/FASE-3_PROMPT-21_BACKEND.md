# Fase 3 — Motor de Turnos (backend) — edad mínima de la modalidad "solo_mayores" hardcodeada en vez de configuración por tenant (TASK-102, corrección a TASK-37)

## Contexto

Una auditoría multi-agente de `psique-back` en `main` (ángulo auditoría/
fechas/reglas-como-datos, 2026-08-12) detectó que la constante
`MINIMUM_AGE_YEARS = 18` (`src/appointments/appointments.service.ts`) se
comparaba directamente contra la edad del paciente en dos puntos del
código que aplican la modalidad "solo_mayores" (P3.4, TASK-37): el
propio flujo de reserva (`AppointmentsService.book`) y la
revalidación de las reglas de paciente nuevo al reprogramar
(`AppointmentsService.assertNewPatientRulesStillApply`, P3.6).

`CLAUDE.md` establece que las reglas de negocio (a diferencia de los
topes anti-abuso) se tratan como dato por tenant, no como condicional
hardcodeado en el código. La regla hermana en el mismo archivo,
`cancellationMinHours()` (que respalda la ventana mínima de cancelación,
P3.5), ya seguía ese patrón correctamente: lee
`APPOINTMENT_CANCELLATION_MIN_HOURS_KEY` desde `ConfigTenantService`, con
una constante como valor por defecto si la fila no existe. El umbral de
edad de la modalidad "solo_mayores" es una regla de negocio real (la
mayoría de edad, no un límite arbitrario del sistema), pero no tenía ese
mismo respaldo en `OrganizationConfig`: un tenant white-label, o uno en
una jurisdicción con una mayoría de edad distinta, no podía configurar
el valor sin un cambio de código y un redeploy.

## Qué se implementó

- Se agregó la clave `appointment_minimum_age_years` a
  `appointments.constants.ts`
  (`APPOINTMENT_MINIMUM_AGE_YEARS_KEY`/`DEFAULT_APPOINTMENT_MINIMUM_AGE_YEARS = 18`),
  siguiendo el mismo patrón de nombres y comentario que
  `APPOINTMENT_CANCELLATION_MIN_HOURS_KEY`.
- Se agregó `AppointmentsService.minimumAge()`, un método privado con la
  misma forma exacta que `cancellationMinHours()`: lee la clave vía
  `ConfigTenantService.get`, y sólo acepta el valor configurado si es un
  número entero positivo — cualquier otra cosa (ausente, no numérico, no
  entero, no positivo) cae al valor por defecto de 18, con el mismo
  razonamiento que la constante hermana: `OrganizationConfig` guarda JSON
  sin tipar, así que esa validación es lo único que distingue una fila
  mal cargada de un umbral real.
- Se reemplazaron las dos comparaciones hardcodeadas
  (`age < MINIMUM_AGE_YEARS`) por `age < await this.minimumAge()`, en
  `book` y en `assertNewPatientRulesStillApply`; se eliminó la constante
  módulo-nivel `MINIMUM_AGE_YEARS`.
- Se agregó una migración de datos
  (`20260814013744_seed_appointment_minimum_age_config`) que siembra el
  valor 18 para toda organización existente, `ON CONFLICT DO NOTHING`
  (no pisa un valor ya configurado), con el mismo razonamiento que la
  migración equivalente de `appointment_cancellation_min_hours`: el seed
  de `prisma/seed.ts` es dato de desarrollo/piloto y no corre en un
  entorno productivo, así que sin la migración la regla sólo existiría
  como constante de código para cualquier organización creada antes de
  este cambio.
- Se agregó la misma clave a `TENANT_CONFIG` en `prisma/seed.ts`, para
  que toda organización creada después de este cambio la reciba desde el
  seed.

## Decisiones y por qué

**Mismo patrón exacto que `cancellationMinHours()`, sin variaciones.**
El ticket señala explícitamente esa función como el patrón de
referencia a seguir; no había ninguna razón funcional para desviarse
(mismo tipo de dato — un entero positivo —, misma clave de
`OrganizationConfig`, mismo mecanismo de fallback), así que replicarlo
literalmente evita introducir una segunda forma de resolver el mismo
tipo de regla en el mismo archivo.

**Las dos comparaciones se corrigieron, no sólo la de `book`.** El
cuerpo del ticket describe el hallazgo centrado en la reserva ("al
reservar"), pero el propio texto de la tarea pide reemplazar "las dos
comparaciones hardcodeadas" y cita ambas líneas explícitamente. Dejar la
revalidación de `assertNewPatientRulesStillApply` (P3.6, disparada al
reprogramar un turno de un paciente nuevo) leyendo la constante vieja
habría dejado ese segundo camino desincronizado del nuevo valor
configurado por el tenant — el mismo tipo de inconsistencia que motivó
el ticket en primer lugar.

## Alternativas descartadas

Ninguna: el patrón de referencia estaba explícitamente indicado por el
propio ticket y no había ambigüedad de diseño que resolver.

## Entidades / puertos / adaptadores tocados

- `appointments/appointments.constants.ts`: clave y default nuevos.
- `appointments/appointments.service.ts`: nuevo método privado
  `minimumAge()`; los dos call sites de la modalidad "solo_mayores"
  (`book`, `assertNewPatientRulesStillApply`) ahora lo consultan en vez
  de comparar contra una constante; se elimina `MINIMUM_AGE_YEARS`.
- `prisma/migrations/20260814013744_seed_appointment_minimum_age_config/`
  (nueva): siembra el valor por defecto para organizaciones existentes.
- `prisma/seed.ts`: la clave se agrega a `TENANT_CONFIG`.

## Tests agregados o modificados

- `src/appointments/appointments.service.spec.ts` (suite `book`): se
  expone el mock de `ConfigTenantService.get` como una variable
  (`configGet`) en vez de un `jest.fn()` anónimo, para poder
  parametrizarlo por test — sigue devolviendo `undefined` por defecto en
  `beforeEach`, igual que antes. Dos casos nuevos:
  - Con `configGet` devolviendo 21 y un paciente de 19 años (adulto bajo
    el default de 18, pero menor bajo este tenant), la reserva se
    rechaza con `BadRequestException`.
  - Con `configGet` devolviendo `undefined` (sin configurar) y el mismo
    paciente de 19 años, la reserva se acepta — prueba explícitamente que
    el comportamiento por defecto (18) se preserva cuando la
    organización no configuró la clave.

  No se agregó cobertura unitaria dedicada para
  `assertNewPatientRulesStillApply` (el camino de reprogramación): ese
  método no tenía cobertura unitaria propia antes de este cambio
  tampoco — se limitó el alcance de los tests nuevos a los criterios de
  aceptación explícitos del ticket, que hablan sólo de la reserva.

Suite completa verde tras el cambio: 38 suites unitarias / 430 pruebas
(8 nuevas: las 2 propias de este ticket más los 6 tests ya existentes de
la suite `book` que ejercitan indirectamente el nuevo fallback); 37
suites e2e / 439 pruebas (sin cambios — no se agregó cobertura e2e
dedicada, la ya existente de "solo_mayores"
(`test/appointments-booking.e2e-spec.ts`) sigue pasando sin
modificación porque no configura la clave y por lo tanto ejercita el
mismo camino de fallback a 18 que antes). `--runInBand` en ambas
suites. Lint y verificación de tipos (`tsc --noEmit`) sin errores.
Migración aplicada contra Postgres local (`prisma migrate deploy`).

## Figuras pendientes

Ninguna nueva — corrección puntual sobre una regla ya documentada
(P3.4, modalidad "solo_mayores"), que ahora sigue el mismo mecanismo de
configuración por tenant que P3.5 (ventana de cancelación) ya tenía
documentado.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-102-minimum-age-config` (creada
  desde `origin/main` fresco, tras el merge de TASK-101).
- Ticket: TASK-102 ("[CORRECCIÓN] Edad mínima para reserva (18 años)
  hardcodeada en vez de configuración por tenant"), corrección a TASK-37
  (P3.4, gates de paciente nuevo). Misma convención de bitácora dedicada
  para correcciones pequeñas dentro de la fase del ticket original que
  TASK-79/TASK-81/TASK-86/TASK-94/TASK-95/TASK-96/TASK-100/TASK-101
  ([[FASE-3_PROMPT-12]], [[FASE-3_PROMPT-14]], [[FASE-3_PROMPT-15]],
  [[FASE-3_PROMPT-16]], [[FASE-3_PROMPT-17]], [[FASE-3_PROMPT-18]],
  [[FASE-3_PROMPT-19]], [[FASE-3_PROMPT-20]]).
