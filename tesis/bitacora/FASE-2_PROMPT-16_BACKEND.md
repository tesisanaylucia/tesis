# Fase 2 — Pacientes (backend) — La prioridad del vínculo paciente-profesional dejó de admitir edición por ADMIN (TASK-122, corrección a TASK-29)

## Contexto

La misma auditoría de código sobre `psique-back/main` (2026-08-14, agente
"Audit Módulo Pacientes vs SRS") que originó [[FASE-3_PROMPT-27]],
[[FASE-3_PROMPT-28]] y [[FASE-3_PROMPT-29]] señaló un punto en el módulo de
Pacientes que no calificó como defecto sino como pregunta abierta: desde
TASK-29 ([[FASE-2_PROMPT-3]]), `PATCH /pacientes/:id/profesionales/:profId`
—el punto de acceso que edita la prioridad del vínculo— usa la misma
comprobación que su lectura, `ProfessionalOwnershipGuard`, que deja pasar a
cualquier ADMIN además del profesional del vínculo. El comportamiento estaba
documentado y era deliberado desde su origen (así lo registra
[[FASE-2_PROMPT-3]]: "se le aplicó la misma autorización que a la
modificación de la prioridad"), de modo que la auditoría no lo reportó como
un error de código sino como un desacuerdo potencial con el documento de
requisitos, que atribuye la carga de la prioridad específicamente al
profesional. El ticket, abierto como tarea de verificación, planteaba dos
desenlaces posibles: si el acceso administrativo era intencional (por
ejemplo, para soporte operativo), documentarlo explícitamente donde
correspondiera; si los requisitos exigían en cambio exclusividad del
profesional, restringir la comprobación para ese único punto de acceso. Para
cuando esta tarea comenzó, la usuaria ya había editado la descripción del
ticket dejando registrada la segunda alternativa como decisión.

Antes de implementar, se verificó esa decisión contra la fuente de verdad —el
documento de Especificación de Requisitos de Software en Drive, en su
variante con fecha de modificación más reciente (03/08/2026)— en lugar de
darla por sentada. La sección de Módulo Pacientes del anexo enumera: "Prioridad
del paciente: la define y carga el profesional desde la app; se utiliza para
la reasignación de turnos", sin mención alguna de una vía administrativa. La
verificación confirmó la decisión ya registrada en el ticket.

## Qué se implementó

Se reemplazó, únicamente en el método `update` (la escritura de la prioridad)
de `PatientProfessionalsController`, el guard `ProfessionalOwnershipGuard`
por `ProfessionalSelfGuard` — el mismo par de comprobaciones que
[[FASE-2_PROMPT-5]] (TASK-31) ya había introducido para las observaciones del
profesional, reutilizado aquí sin ninguna modificación propia. La lectura del
vínculo (`GET /pacientes/:id/profesionales/:profId`) no se tocó y sigue
detrás de `ProfessionalOwnershipGuard`: un ADMIN continúa pudiendo *leer* la
prioridad de cualquier vínculo, y deja de poder *escribirla*. Se actualizaron
además los comentarios de los tres métodos del controlador (`find`, `update`,
`updateNotes`) para que la división lectura/escritura quede explícita, y el
comentario de clase de `ProfessionalSelfGuard` para citar la prioridad junto
a las observaciones como los dos casos que hoy exigen esa comprobación
estricta.

No hubo cambios de esquema, de servicio ni de contrato de datos: la
prioridad ya se validaba y persistía igual que antes de esta tarea; lo único
que cambió es quién puede alcanzar esa escritura.

## Decisiones y por qué

**Se reutilizó el par de guards existente en lugar de introducir una
comprobación nueva.** El proyecto ya distinguía, desde TASK-31, entre una
comprobación permisiva (`ProfessionalOwnershipGuard`, que exime al rol
administrativo) y una estricta (`ProfessionalSelfGuard`, que no exime a
nadie salvo al profesional nombrado en la ruta) sobre la misma base común
(`ProfessionalRouteGuard`). La prioridad pasó a necesitar exactamente la
comprobación estricta que ya existía para un caso hermano dentro del mismo
controlador, de modo que la corrección fue swapear el decorador de un método
sin escribir código de autorización nuevo — el escenario que la extracción de
TASK-31 hizo posible sin que en ese momento se supiera que un segundo punto
de acceso lo necesitaría.

**Se restringió la escritura y no la lectura.** El texto de requisitos verificado
habla de quién "define y carga" la prioridad — un verbo de escritura — y no
dice nada sobre quién puede leerla, a diferencia de la frase que rige las
observaciones ("de manejo exclusivo del profesional... la carga y la
visualiza", que sí nombra la lectura explícitamente y por eso [[FASE-2_PROMPT-5]]
restringió ambas direcciones). Extender la restricción a la lectura habría
sido una decisión de producto no pedida por el ticket ni respaldada por el
documento de requisitos, además de romper una prueba end-to-end ya existente
desde TASK-29 que cubre expresamente "un ADMIN lee cualquier vínculo" como
comportamiento válido.

**La verificación contra la fuente de verdad se hizo antes de programar, no
después.** El ticket llegó con una decisión ya registrada por la usuaria, pero
las instrucciones del repositorio son que Drive es la fuente de verdad del
proyecto; se releyó el documento de Especificación de Requisitos vigente para
confirmar la cita exacta ("la define y carga el profesional desde la app")
en lugar de confiar en la paráfrasis que el propio código ya tenía en un
comentario desde TASK-29. La cita coincidió con la decisión registrada, sin
discrepancia que resolver.

## Alternativas descartadas

- **Documentar el acceso administrativo como intencional, sin restringir el
  guard**: era la otra rama que el propio ticket contemplaba: descartada tras
  verificar que el documento de requisitos atribuye la carga de la prioridad
  exclusivamente al profesional, sin prever ninguna vía administrativa.
- **Expresar la restricción dentro del servicio en lugar del enrutamiento**:
  descartada por el mismo criterio que ya fijó [[FASE-2_PROMPT-5]] — dispersaría
  la decisión de acceso entre dos capas en lugar de dejarla resuelta,
  declarativamente, en la ruta.
- **Restringir también la lectura del vínculo**: descartada por no tener
  respaldo en el texto de requisitos verificado (que sólo habla de quién
  carga la prioridad, no de quién puede leerla) y por remover una capacidad
  —lectura administrativa de cualquier vínculo— que una prueba end-to-end ya
  vigente desde TASK-29 cubre como válida y que el ticket no pedía tocar.

## Entidades / puertos / adaptadores tocados

- `src/patients/patient-professionals.controller.ts` (modificado): el método
  `update` pasa de `@UseGuards(ProfessionalOwnershipGuard)` a
  `@UseGuards(ProfessionalSelfGuard)`; comentarios de `find`, `update` y
  `updateNotes` actualizados para reflejar la división lectura/escritura.
- `src/professionals/guards/professional-self.guard.ts` (modificado): sólo el
  comentario de clase, ampliado para citar también la prioridad.
- No se tocaron `professional-ownership.guard.ts`, `professional-route.guard.ts`
  ni `patient-professionals.service.ts` — el cambio es puramente de
  enrutamiento/autorización, sin lógica nueva.

## Tests y qué validan

- `test/patients-type-priority.e2e-spec.ts` (ampliado, 1 prueba nueva):
  "forbids an ADMIN from setting the priority (403)", espejo de la prueba ya
  existente que confirma que un ADMIN sí puede *leer* cualquier vínculo,
  verificando además que la prioridad almacenada no cambia.
- `test/patients-abmc.e2e-spec.ts` (modificado, 3 pruebas ajustadas): la
  prueba "lets the treating professional set the priority, and nobody else"
  reemplazó su caso de ADMIN de otra organización (que antes documentaba 404
  por el chequeo de tenant del servicio, ya que el guard anterior dejaba
  pasar a cualquier ADMIN sin mirar el inquilino) por un ADMIN de la misma
  organización, que ahora debe recibir 403 directamente del guard, sin llegar
  al servicio; las dos pruebas que usaban un token de ADMIN incidentalmente
  para llegar al punto de acceso (validación de enum fuera de rango, 404 de
  un vínculo inexistente) pasaron a usar el token del profesional
  correspondiente, ya que el ADMIN ya no alcanza a esas rutas para que esas
  pruebas sigan verificando lo que se proponían.
- `test/patient-notes.e2e-spec.ts` (modificado, 2 pruebas ajustadas): las dos
  pruebas de P2.5 que usaban el punto de acceso de prioridad como control
  —para confirmar que las observaciones no viajan en su respuesta— pasaron de
  un token de ADMIN a uno del profesional del vínculo, sin cambiar lo que la
  prueba verifica.
- Ejecución: suite unitaria en verde (40 suites / 468 pruebas); suite
  end-to-end completa en verde (38 suites / 458 pruebas, `--runInBand`);
  compilación (`tsc --noEmit`) y análisis estático (`eslint`) sin errores
  sobre todos los archivos tocados. Todos los datos usados en las pruebas son
  ficticios.

## Figuras pendientes

Ninguna nueva. La corrección es de autorización sobre un punto de acceso ya
existente y no introduce un flujo distinto del que el diagrama de entidades
del módulo, ya pendiente de tareas anteriores, cubre.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-122-patient-priority-professional-exclusive`
  (creada a partir de `main`). Sin commit al momento de redactar esta entrada:
  los cambios quedaron en el árbol de trabajo a la espera de autorización.
- Ticket: TASK-122 ("[VERIFICACIÓN] Confirmar si un ADMIN debe poder editar
  la prioridad del paciente, no solo el profesional"). Depende de TASK-29
  ([[FASE-2_PROMPT-3]]) y reutiliza el guard introducido por TASK-31
  ([[FASE-2_PROMPT-5]]), ambas ya fusionadas.
