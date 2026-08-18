# Skills & Rules by Language

Este repositorio centraliza **skills** y **rules** para asistentes de código, organizadas por lenguaje.

| Lenguaje | Skill | Rule | Estado |
|---|---|---|---|
| TypeScript | `typescript/skills/SKILL.md` | `typescript/rules/typescript-development-guide.mdc` | Skill completa, rule base presente |

## Objetivo

Mantener un catálogo por lenguaje con lineamientos reutilizables para:

1. Generación de código con tipado y arquitectura consistente.
2. Prevención de anti‑patrones y errores comunes.
3. Estandarización de decisiones técnicas por stack.

## Estructura del repositorio

```text
<language>/
  skills/
    SKILL.md
  rules/
    *.mdc
```

## Cómo agregar un nuevo lenguaje

1. Crear la carpeta del lenguaje en la raíz (por ejemplo: `python/`).
2. Agregar `skills/SKILL.md` con frontmatter y reglas de ejecución.
3. Agregar `rules/*.mdc` con convenciones operativas del lenguaje.
4. Actualizar la tabla inicial de este README.

## Convenciones recomendadas

- Una skill por dominio principal de decisión.
- Rules orientadas a ejecución (claras, verificables y accionables).
- Mantener ejemplos y anti‑patrones cerca de su regla correspondiente.