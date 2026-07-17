# Fase 0 — Fundaciones (backend) — eliminación del modelo `Diagnosis` (TASK-74)

## Qué se implementó

Se eliminó el modelo `Diagnosis` (originalmente `Diagnostico`, catálogo de
códigos CIE-10) del esquema de Prisma, junto con la tabla correspondiente
en la base de datos. El modelo había sido incorporado en TASK-19 (P0.8)
como catálogo auxiliar, a partir de una mención del documento de
requisitos a "tablas auxiliares: diagnósticos (DXCIE10)". Al revisar el
esquema contra el diagrama entidad-relación de la base de datos
(`modelo_base_de_datos.png`), se constató que ese diagrama no incluye
`DIAGNOSTICO` como entidad propia, y que ninguna entidad de dominio ya
implementada o planificada (turno, paciente-profesional, solicitud de
receta) declara una clave foránea hacia ella. El módulo conversacional
(capa de WhatsApp) tampoco consulta diagnósticos: sus reglas de
comportamiento (guardrails) los excluyen explícitamente del alcance de la
conversación con el paciente.

Se removió el modelo `Diagnosis` de `prisma/schema.prisma` y se agregó una
migración manual que ejecuta `DROP TABLE "Diagnosis"`, verificando
previamente que la tabla no tuviera claves foráneas dependientes (no las
tenía: la única referencia en todo el código, fuera de las migraciones
históricas, estaba en la suite de tests de catálogos). No existía ningún
seed de datos CIE-10 que remover: el catálogo nunca llegó a poblarse con
datos reales, conforme al alcance original de TASK-19 ("no hacer
todavía"). Tampoco existían un `DiagnosticoService`, un `DiagnosticoModule`
ni DTOs asociados: el modelo TASK-19 solo había definido la entidad de
Prisma y su acotamiento por tenant a través de la extensión genérica del
cliente (la misma que usan `HealthInsurer` y `OrganizationConfig`), sin
ningún servicio propio.

## Decisiones y por qué

**Eliminar la tabla en lugar de dejarla sin usar.** Se descartó la
alternativa de conservar el modelo intacto por si un futuro módulo de
diagnósticos lo necesitara, porque mantener en el esquema una entidad que
no corresponde a ninguna fuente de verdad del proyecto (ni el diagrama
entidad-relación ni el alcance vigente del documento de requisitos)
introduce una discrepancia entre el código y su especificación que
confundiría a cualquier lectura futura del esquema. Si en el futuro se
requiere un módulo de diagnósticos, el ticket deja constancia de que debe
diseñarse desde cero con su propio diagrama entidad-relación, en lugar de
reutilizar este modelo descartado.

**Migración manual de `DROP TABLE` en lugar de dejar que Prisma la genere
automáticamente.** Se siguió el mismo patrón ya establecido en las
correcciones anteriores de esta fase (TASK-72, TASK-73, TASK-75): escribir
la migración a mano, con un comentario que documenta el motivo de la
corrección, en vez de depender de `prisma migrate dev` para inferirla.

## Alternativas descartadas

- **Dejar el modelo `Diagnosis` en el esquema pero sin exponerlo a través
  de ningún servicio**, como una forma de conservar el trabajo de TASK-19
  sin comprometerse a un catálogo completo: descartada por la misma razón
  que motivó eliminarlo por completo — una tabla sin ninguna fuente de
  verdad que la respalde no aporta nada y sí genera confusión sobre el
  alcance real del sistema.

## Entidades / puertos / adaptadores tocados

- `prisma/schema.prisma`: se eliminó el modelo `Diagnosis`.
- Migración nueva `prisma/migrations/20260717190000_remove_diagnosis/`:
  `DROP TABLE "Diagnosis"`.
- `CLAUDE.md`: se agregó una aclaración explícita, junto a las
  restricciones de datos clínicos, de que no existe un catálogo de
  diagnósticos en el sistema y de que no debe reintroducirse el modelo
  eliminado.

## Tests y qué validan

- `test/catalogs.e2e-spec.ts`: se removieron los casos y las referencias a
  `Diagnosis` (creación, aislamiento por tenant y rechazo sin contexto de
  tenant), conservando los mismos casos para `HealthInsurer`, que sigue
  vigente.
- Se ejecutó la suite completa tras aplicar la migración vía
  `prisma migrate deploy` contra la instancia local de PostgreSQL: 7
  suites / 23 tests unitarios y 8 suites / 26 tests end-to-end, todos en
  verde. `prisma validate` no reporta ninguna referencia a `Diagnosis`, y
  una búsqueda en todo el repositorio (excluyendo migraciones históricas y
  artefactos de build) no encuentra ningún import ni referencia restante
  al modelo eliminado.

## Figuras pendientes

Ninguna nueva.

## Componente y referencia

- Componente: backend.
- Branch de referencia: `feature/TASK-74-remove-diagnostico` (creada a
  partir de `main`, con TASK-75 ya fusionado).
- Ticket: TASK-74 ("[CORRECCIÓN] P0.8 – Eliminar modelo Diagnostico de la
  base de datos"), corrección sobre TASK-19. Depende de TASK-19 (P0.8,
  origen del modelo) y de TASK-33 (P2.7), que en el momento de esta tarea
  no existía todavía en el repositorio, por lo que no había ningún seed de
  diagnósticos que remover.
