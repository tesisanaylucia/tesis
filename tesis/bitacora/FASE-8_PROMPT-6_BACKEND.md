# Fase 8 — Endurecimiento, cumplimiento normativo y piloto (backend) — Datos de piloto y checklist de marcha blanca (TASK-70, P8.5)

## Qué se implementó

Se implementó P8.5 ("Datos de piloto y checklist de marcha blanca") del
SRS, el último módulo de la Fase 8 y, con él, de la implementación
funcional del sistema previa a la validación con la clínica. La tarea
tiene dos entregables independientes: un script de siembra de datos
realistas para el entorno de piloto, separado del script de siembra de
desarrollo ya existente, y un documento de checklist de marcha blanca
para validar manualmente el sistema junto con el responsable de la
clínica antes de operar con datos reales.

El script de siembra de piloto (`prisma/seed-pilot.ts`, ejecutable con
`npm run seed:pilot`) crea una organización propia, "Piloto", separada de
la organización "Clínica" que el script de siembra de desarrollo ya crea
—una decisión explícita para cumplir el requisito de la tarea de que el
dato de piloto "no interfiere con otros tenants"—, reutilizando para ello
la función `seedTenant` que el script de desarrollo ya expone: la
organización, su administrador, su usuario de sistema, las
especialidades, el catálogo global de obras sociales, la configuración
de reglas de negocio del tenant y el plantel de profesionales con sus
matrículas y horarios de atención ya eran responsabilidad de esa función,
de modo que el nuevo script sólo reutiliza en lugar de reimplementar.
Sobre esa base, agrega las entidades que la tarea pide y que el script de
desarrollo no cubre: dos profesionales (un psiquiatra y una psicóloga,
frente al plantel de cuatro y uno del script de desarrollo, ya que este
conjunto de datos existe para recorrerse a mano contra el checklist, no
para ejercitar casos límite de agenda), ocho pacientes con vínculos
paciente-profesional de distinto tipo (nuevo/recurrente) y prioridad, con
consentimiento aceptado en cada caso, diez feriados —nacionales de fecha
fija más dos provinciales de San Juan marcados explícitamente como
referencia a confirmar contra el calendario oficial, ya que ese calendario
se fija por decreto cada año y no se adivina en el código—, ocho turnos
que cubren los cuatro estados que la tarea pide (reservado, confirmado,
cancelado, completado) distribuidos en los siete días siguientes a la
carga, tres entradas de lista de espera, tres preguntas frecuentes para
el chatbot, y un código de acceso activo para el turno confirmado del día
siguiente a la carga.

El script sigue el mismo criterio de convergencia que el script de
desarrollo: cada fila se actualiza o inserta según una clave natural
cuando el esquema ya ofrece una (el documento nacional de identidad del
paciente dentro de su organización, la fecha del feriado dentro de su
organización, la clave compuesta de la relación paciente-profesional), o
según un identificador determinístico derivado de un espacio de nombres
propio del script cuando la entidad no tiene clave natural (turno, entrada
de lista de espera, pregunta frecuente, código de acceso) — de modo que
ejecutar `npm run seed:pilot` más de una vez converge siempre al mismo
estado en lugar de duplicar filas. Ese espacio de nombres debió
ampliarse una vez, durante la escritura de la prueba de extremo a
extremo del propio script: el primer diseño derivaba los identificadores
de esas entidades sin clave natural de una constante fija del módulo,
sin parametrizarlos por la organización de destino, del mismo modo que sí
estaban parametrizados el profesional, la matrícula y el horario de
atención heredados de `seedTenant`. Una organización de prueba
descartable —el mismo mecanismo de aislamiento que la prueba de extremo a
extremo del script de desarrollo ya usa— terminaba entonces intentando
crear un paciente con el mismo identificador que ya existía en la
organización "Piloto" real, y la prueba fallaba con una violación de
restricción de unicidad. Se corrigió extendiendo la interfaz de espacio
de nombres para incluir un prefijo por cada una de esas cinco entidades,
pasado explícitamente a través de cada función auxiliar en lugar de leído
de una constante del módulo, exactamente el mismo patrón que ya regía
para profesional, matrícula y horario de atención — el defecto quedó
expuesto y corregido antes de fusionar el cambio, no documentado como
limitación conocida.

El código de acceso del piloto se escribe directamente con Prisma, no a
través del servicio de dominio que genera códigos de acceso en el flujo
real: ese servicio depende de un adaptador de cerradura inteligente y de
la configuración por tenant, ambos resueltos por inyección de
dependencias de Nest, contenedor que un script de siembra ejecutado fuera
de la aplicación no tiene. La fila se construye, en cambio, reutilizando
las mismas constantes de ventana de validez y la misma función de cifrado
de campo que ese servicio ya usa, de modo que el resultado tiene la misma
forma que el que el flujo real habría producido, aunque el identificador
de cerradura que guarda sea ficticio como el resto del conjunto de datos.

El checklist de marcha blanca (`docs/marcha-blanca-checklist.md`) traduce
las cuatro categorías que pide la SRS —funcional, de seguridad, de
cumplimiento— agregando una cuarta, de integración, ya que las dos
integraciones externas del sistema (WhatsApp Business API, TTLock) sólo
pueden validarse contra credenciales y hardware reales, nunca contra la
integración continua. El documento reúne dieciocho ítems verificables a
mano, tres más del mínimo de quince que pide la tarea, cada uno con el
paso concreto a ejecutar contra el conjunto de datos de piloto (por
ejemplo, iniciar sesión con las cuentas de los dos profesionales
sembrados, o confirmar que el mensaje de reserva es rechazado por el
chatbot hasta que el paciente acepta el consentimiento) y el resultado
esperado, cerrando con una constancia de aceptación con espacio de firma
para el responsable de la clínica y para el responsable técnico.

## Decisiones tomadas y su porqué

La tarea describe el requisito de aislamiento como "organizationId=`piloto`",
un identificador literal que no es compatible con el esquema real: cada
fila de `Organization` usa un identificador generado por PostgreSQL, no
una clave primaria de texto. Se interpretó ese requisito como una
referencia informal al nombre de la organización, no a su identificador
interno, y se resolvió dándole a la organización el nombre "Piloto" —el
mismo patrón que ya distingue a la organización "Clínica" del script de
desarrollo por nombre, no por un identificador fijo.

La duración de los ocho turnos sembrados se calcula duplicando la
duración de consulta configurada del profesional cuando el turno
corresponde a una primera sesión, replicando en el dato sembrado la
misma regla que el servicio de reserva de turnos ya aplica al reservar
—una primera sesión ocupa dos franjas seguidas—, en lugar de dejar todos
los turnos con la misma duración fija por simplicidad.

Los feriados nacionales sembrados se limitan a ocho fechas fijas por ley
—las que no dependen de un cálculo de fecha móvil, como Semana Santa o
Carnaval— más dos fechas provinciales de San Juan marcadas explícitamente
como valores de referencia a confirmar, en lugar de calcular las fechas
móviles o inventar fechas provinciales sin verificar. Se prefirió declarar
la incertidumbre en el propio dato, siguiendo el mismo criterio que el
script de desarrollo ya aplica a los nombres y matrículas ficticios de su
plantel de profesionales, antes que presentar una fecha no verificada
como si fuera un hecho.

## Entidades tocadas

Ninguna migración de esquema: la tarea es exclusivamente de siembra de
datos y documentación, sobre entidades ya existentes de tareas anteriores
(`Organization`, `Professional`, `License`, `WorkingHour`, `Patient`,
`PatientProfessional`, `Consent`, `Holiday`, `Appointment`,
`WaitlistEntry`, `Faq`, `AccessCode`). Se agregó y exportó una función
auxiliar (`seedUuid`) desde el script de desarrollo, ya existente pero sin
exportar, para que el nuevo script no reimplementara la misma derivación
de identificadores determinísticos.

## Tests agregados o modificados

Una prueba de extremo a extremo nueva (`test/seed-pilot.e2e-spec.ts`)
contra PostgreSQL real, siguiendo el mismo patrón de aislamiento que la
prueba del script de desarrollo (`test/seed.e2e-spec.ts`): siembra una
organización descartable con su propio espacio de nombres de
identificadores, nunca la organización "Piloto" real, y la elimina al
terminar. Verifica que el script carga cada entidad que la tarea describe
dentro del rango pedido (por ejemplo, entre cinco y diez pacientes),
que los cuatro estados de turno pedidos están efectivamente representados,
que el único código de acceso queda asociado a un turno confirmado, que
ninguna fila sembrada pertenece a otra organización, y que una segunda
ejecución del script no duplica ninguna fila (la prueba que expuso el
defecto de espacio de nombres descrito arriba). Verificado también a mano
contra la base de datos de desarrollo real, ejecutando el script dos
veces seguidas y confirmando que la segunda ejecución reporta el mismo
identificador de organización y los mismos conteos que la primera. Suite
completa verificada en verde tras el cambio: 75 suites/753 pruebas
unitarias, 49 suites/548 pruebas de extremo a extremo (`--runInBand`),
linter y verificación de tipos sin errores.

## Figuras pendientes

Ninguna nueva. La Figura 27 ya pendiente (máquina de estados de la oferta
de lista de espera, Fase 4) sigue siendo la referencia para los estados
de turno que el conjunto de datos de piloto ejercita.

## Componente y referencia

Backend. Rama `feature/TASK-70-pilot-seed-and-launch-checklist`, aún sin
fusionar a `main` al momento de escribir esta entrada.
