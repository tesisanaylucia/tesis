# Capítulo 2: Marco Teórico (continuación)

> Borrador. Continuación de `cap2_marco_teorico_a.md` (2.1 a 2.4). Este
> archivo cubre 2.5 a 2.8: tecnologías de implementación concretas y marco
> normativo argentino. Igual que en el archivo anterior, el contenido es
> conceptual y no describe decisiones específicas de PSIQUE.

## 2.5 Stack backend: NestJS, Prisma y PostgreSQL

NestJS es un framework para construir aplicaciones del lado del servidor
sobre Node.js utilizando TypeScript como lenguaje principal. Se distingue
de otros frameworks del ecosistema Node.js por imponer una estructura
opinionada, inspirada en convenciones de frameworks como Angular:
organización del código en módulos, un sistema de inyección de
dependencias integrado, y un conjunto de abstracciones estándar
(controladores para manejar solicitudes entrantes, proveedores para
encapsular lógica reutilizable, *guards* para autorización, interceptores
y *pipes* para transformación y validación de datos) [CITA: arquitectura y
convenciones de NestJS]. Esta estructura estandarizada facilita que
distintos módulos de una misma aplicación, o distintos desarrolladores
dentro de un equipo, sigan un mismo patrón de organización, lo que resulta
particularmente útil en aplicaciones que crecen incorporando módulos de
dominio sucesivos.

Prisma es un ORM (*object-relational mapper*) para el ecosistema
JavaScript/TypeScript que se distingue por un enfoque *schema-first*: el
modelo de datos se declara en un archivo de esquema propio, a partir del
cual Prisma genera automáticamente un cliente de acceso a datos
fuertemente tipado para el lenguaje de la aplicación, y un sistema de
migraciones que traduce los cambios sucesivos de ese esquema en sentencias
de definición de datos (DDL) versionadas y aplicables de forma ordenada
sobre la base de datos [VERIFICAR: versión de Prisma vigente al momento de
redacción]. El tipado generado automáticamente a partir del esquema permite
que errores de acceso a datos —por ejemplo, referenciar un campo que no
existe en un modelo— se detecten en tiempo de compilación en lugar de en
tiempo de ejecución, una propiedad particularmente valiosa en un lenguaje
como TypeScript, cuyo sistema de tipos es opcional y depende de que las
distintas capas de la aplicación lo aprovechen de forma consistente.

PostgreSQL es un sistema de gestión de bases de datos relacionales de
código abierto que cumple con las propiedades ACID (atomicidad,
consistencia, aislamiento y durabilidad) para las transacciones, y que
además del modelo relacional clásico ofrece soporte nativo para tipos de
datos semiestructurados (como JSONB) y un mecanismo de extensiones que
permite ampliar sus capacidades sin abandonar el modelo transaccional
[CITA: características y garantías transaccionales de PostgreSQL]. Su
soporte maduro para índices, restricciones de integridad referencial y
transacciones lo posiciona como una opción habitual para dominios de datos
estructurados con requisitos de consistencia estrictos, como los que
surgen típicamente en sistemas administrativos de salud, donde una
inconsistencia entre registros (por ejemplo, un turno que aparece
disponible para dos pacientes distintos) tiene un costo operativo directo.

La combinación de un framework de backend, un ORM y un motor de base de
datos relacional, todos con soporte de tipado estático de extremo a
extremo, permite que el mismo modelo de datos declarado una única vez se
refleje consistentemente tanto en el esquema de la base de datos como en
el código de la aplicación que la consume, reduciendo la clase de errores
que surgen del desacople entre ambos.

## 2.6 IoT y control de acceso electrónico: cerraduras inteligentes, TTLock y el rol del gateway

El control de acceso electrónico mediante Internet de las Cosas (IoT)
reemplaza la llave física tradicional por un mecanismo de apertura
gestionado remotamente: una cerradura inteligente, conectada a una red de
comunicación (habitualmente Bluetooth de bajo consumo, en algunos casos con
conectividad Wi-Fi propia), que puede recibir comandos de apertura o
códigos de acceso generados y administrados desde una plataforma en la
nube. Este modelo desplaza la gestión de accesos desde un proceso físico
—entregar y recuperar una llave o una tarjeta— hacia un proceso digital,
en el que otorgar, limitar en el tiempo o revocar un acceso es una
operación sobre un sistema de software [CITA: modelos de control de acceso
electrónico basados en IoT].

TTLock es una plataforma comercial que provee, sobre un catálogo de
cerraduras inteligentes propio, una API en la nube y un conjunto de
herramientas de desarrollo (SDK) que permiten a un sistema externo generar
códigos de acceso temporales, consultar el estado de una cerradura y
recibir notificaciones sobre eventos de apertura [VERIFICAR: alcance
exacto y límites de la API pública de TTLock vigentes al momento de
redacción]. Un código de acceso temporal es una credencial —típicamente un
código numérico— válida únicamente dentro de una ventana de tiempo
determinada y, según la configuración, utilizable una única vez o un
número limitado de veces, lo que permite otorgar acceso acotado a la
duración de un evento (por ejemplo, un turno) sin necesidad de revocar
manualmente el acceso una vez finalizado.

Muchas cerraduras de este tipo se comunican únicamente mediante Bluetooth
de corto alcance y no poseen conectividad directa a Internet, lo que hace
necesaria la presencia de un dispositivo puente, denominado *gateway*, que
sí posee conectividad a Internet (típicamente Wi-Fi) y retransmite los
comandos entre la plataforma en la nube y la cerradura. El *gateway* cumple
así una función de intermediación indispensable para que una operación
iniciada de forma remota —por ejemplo, la generación de un código temporal
desde un sistema externo— llegue efectivamente a la cerradura sin
requerir presencia física junto a ella [CITA: rol del gateway en
arquitecturas de cerraduras inteligentes Bluetooth].

Un sistema de control de acceso construido sobre este tipo de plataforma
introduce consideraciones de seguridad propias: la ventana de validez de
cada código debe ajustarse estrictamente al período en que el acceso está
autorizado, la revocación debe ser efectiva incluso si el código ya fue
comunicado al usuario, y toda operación de generación, uso o revocación de
un código debiera quedar registrada para permitir una auditoría posterior
de quién accedió a un espacio físico y cuándo [CITA: consideraciones de
seguridad y auditoría en control de acceso IoT].

## 2.7 Desarrollo de aplicaciones móviles multiplataforma: React Native y TypeScript

El desarrollo de aplicaciones móviles multiplataforma busca producir, a
partir de una única base de código, aplicaciones ejecutables tanto en iOS
como en Android, en contraposición al desarrollo nativo, que requiere
mantener dos bases de código independientes (típicamente Swift/Objective-C
para iOS y Kotlin/Java para Android). Distintos enfoques logran este
objetivo con diferentes grados de acceso a las capacidades nativas de la
plataforma: desde soluciones basadas en una vista web embebida hasta
soluciones que compilan a código nativo o que, como React Native, renderizan
componentes de interfaz de usuario verdaderamente nativos a partir de
código JavaScript [CITA: comparación de enfoques de desarrollo móvil
multiplataforma].

React Native es un framework que permite construir interfaces de usuario
móviles utilizando React —una biblioteca basada en componentes y en un
modelo declarativo de interfaz de usuario, originalmente desarrollada para
el desarrollo web— cuyo código JavaScript se ejecuta en un motor separado
y se comunica con una capa nativa que efectivamente crea y actualiza los
componentes de interfaz nativos de cada plataforma [VERIFICAR: arquitectura
de puente/motor de React Native vigente al momento de redacción]. Esto
permite que un equipo que ya trabaja con React en el desarrollo web
extienda ese mismo modelo mental y buena parte de sus herramientas al
desarrollo de aplicaciones móviles, a costa de una capa de indirección
adicional entre el código de la aplicación y las capacidades nativas del
dispositivo respecto de una aplicación completamente nativa.

La incorporación de TypeScript sobre una base de React Native aporta
tipado estático al desarrollo de la interfaz de usuario, permitiendo
detectar en tiempo de compilación errores como el uso incorrecto de las
propiedades de un componente o de la forma de los datos que este espera
recibir. Este beneficio es proporcional al tamaño de la aplicación: en una
base de código pequeña el costo de mantener las anotaciones de tipo puede
superar el beneficio inmediato, mientras que en una aplicación con múltiples
pantallas, componentes compartidos y un modelo de datos no trivial que
consume una API externa, el tipado estático reduce de forma apreciable una
clase entera de errores que de otro modo solo se manifestarían en tiempo
de ejecución [CITA: beneficios del tipado estático en el desarrollo de
interfaces de usuario a gran escala].

## 2.8 Marco normativo argentino: Ley 25.326 y Ley 26.529

La Ley 25.326 de Protección de Datos Personales, sancionada en el año 2000,
constituye el marco general argentino para el tratamiento de datos
personales por parte de organismos públicos y privados. Establece
principios que cualquier sistema que trate datos personales debe respetar,
entre ellos el principio de finalidad —los datos solo pueden utilizarse
para el propósito para el cual fueron recolectados—, el principio de
calidad de los datos, y el deber de confidencialidad y seguridad sobre la
información tratada. La ley reconoce además, en cabeza del titular de los
datos, los derechos de acceso, rectificación, actualización y supresión,
conocidos habitualmente por el acrónimo ARCO [CITA: texto vigente de la
Ley 25.326 y su reglamentación]. La ley distingue, dentro de los datos
personales, una categoría de datos sensibles —que incluye explícitamente
los datos referidos a la salud— sujeta a un estándar de protección más
estricto que el aplicable a los datos personales en general, dado el
mayor impacto potencial de una exposición indebida de este tipo de
información sobre la persona titular.

La Ley 26.529, de Derechos del Paciente, Historia Clínica y Consentimiento
Informado, sancionada en 2009, regula específicamente la relación entre el
paciente y el sistema de salud, con independencia de la ley general de
protección de datos. Reconoce, entre otros, el derecho del paciente a la
autonomía de la voluntad, el derecho a recibir información clara y
adecuada sobre su estado de salud, el requisito de consentimiento
informado para determinadas intervenciones, y el derecho del paciente a
acceder a su propia historia clínica [CITA: texto vigente de la Ley 26.529
y su reglamentación]. Esta ley introduce además el concepto de historia
clínica como documento con requisitos propios de confidencialidad, guarda
y acceso, que se superpone parcialmente con las obligaciones de
confidencialidad que ya impone la Ley 25.326 sobre cualquier dato personal.

Para un sistema que gestiona información administrativa de una institución
de salud mental —incluso cuando ese sistema esté diseñado explícitamente
para no procesar contenido clínico— ambas leyes resultan relevantes de
forma simultánea: la Ley 25.326 exige tratar con el máximo estándar de
protección cualquier dato que permita inferir que una persona es paciente
de una institución de salud mental, dado que ese solo hecho puede
constituir un dato sensible, mientras que la Ley 26.529 fija el estándar de
confidencialidad y acceso aplicable a la historia clínica propiamente
dicha, en la medida en que el sistema llegue a interactuar con ella o con
referencias a ella. La combinación de ambos marcos normativos es, en
consecuencia, uno de los factores que condiciona decisiones de diseño como
la separación estricta entre el canal administrativo y cualquier
contenido clínico, y la exigencia de trazabilidad sobre quién accede a
qué dato y cuándo [CITA: relación entre la Ley 25.326 y la Ley 26.529
aplicada a sistemas de información en salud].
