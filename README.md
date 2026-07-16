# Tesis PSIQUE (directorio compartido)

Este directorio contiene los borradores de la tesis de PSIQUE (Trabajo Final
de Grado, Licenciatura en Ciencias de la Computación, UNSJ). Es un
directorio **compartido** entre los dos repos de código del proyecto:

- `back/` — backend NestJS.
- `front/` — app móvil React Native.

Ninguno de los dos repos escribe la tesis dentro de sí mismo. En cambio,
cada repo tiene una skill de Claude Code (`documentacion-tesis`, idéntica en
ambos repos salvo por un archivo de configuración) que, al cerrar cada tarea
de implementación validada, escribe o amplía los borradores de este
directorio. Ver `back/.claude/skills/documentacion-tesis/SKILL.md` (o su
equivalente en `front/`) para el detalle del flujo.

## Estructura

- `tesis/bitacora/` — una entrada por prompt/tarea validada, nombrada
  `FASE-X_PROMPT-Y_BACKEND.md` o `FASE-X_PROMPT-Y_MOVIL.md`.
- `tesis/capitulos/` — el texto de la tesis en sí, dividido en un archivo
  por capítulo o sección larga (p. ej. `cap2_marco_teorico.md`,
  `cap3_desarrollo.md`).
- `tesis/figuras_pendientes.md` — figuras que faltan generar/insertar.
- `tesis/PROGRESO.md` — estado de avance por subsección.

Este directorio se versiona con git de forma independiente de los repos de
código (no es un submódulo de ninguno de los dos). Los commits acá son
manuales: la automatización de ambos repos solo escribe en el working tree,
nunca hace commit ni push de la tesis.
