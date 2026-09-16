<div align="center">
  <img src="./public/icon-transparent.png" width="180" alt="Icono del catálogo Skills & Rules" />
  <br />
  <br />

  <h1>Skills & Rules</h1>

  <p>Catálogo reutilizable de instrucciones para asistentes de código</p>

  <a href="#inicio-rápido">Inicio rápido</a>
  <span>&nbsp;•&nbsp;</span>
  <a href="#estructura">Estructura</a>
  <span>&nbsp;•&nbsp;</span>
  <a href="#cómo-contribuir">Contribuir</a>
</div>

<br />

<div align="center">
  <img alt="Markdown Badge" src="https://img.shields.io/badge/Markdown-000000?logo=markdown&logoColor=fff&style=flat" />
  <img alt="MDC Badge" src="https://img.shields.io/badge/Rules-.mdc-4B32C3?style=flat" />
  <img alt="Documentation Badge" src="https://img.shields.io/badge/Documentation-reusable-2EA44F?style=flat" />
</div>

## 📝 Sobre el proyecto

Este repositorio reúne **skills**, **rules** y comandos reutilizables para mejorar la generación, revisión y mantenimiento de código con asistentes de desarrollo.

Cada módulo concentra el conocimiento de un dominio concreto: un lenguaje, un framework, una herramienta de infraestructura o una decisión arquitectónica. El objetivo es mantener instrucciones claras, accionables y fáciles de incorporar a nuevos proyectos.

## ✨ ¿Qué contiene?

- 📚 **Skills:** guías detalladas para razonar y trabajar dentro de un dominio técnico.
- ⚙️ **Rules:** reglas operativas que pueden aplicarse automáticamente al trabajar con determinados archivos o tecnologías.
- 🧱 **Arquitecturas:** criterios para Clean Architecture y Screaming Architecture.
- 🛠️ **Herramientas y frameworks:** instrucciones específicas para Docker, Prisma, Next.js, Astro, Laravel, NestJS, Tailwind CSS y más.
- 🧭 **Comandos OpenSpec:** flujos documentados para explorar, proponer, aplicar y archivar cambios.

## 🗂️ Módulos disponibles

| Módulo | Enfoque | Recursos |
|---|---|---|
| [Architectures](architectures/) | Clean Architecture y Screaming Architecture | Skills y rules |
| [Astro](astro/) | Desarrollo con Astro | Skill y rules |
| [Docker](docker/) | Contenedores, imágenes y Compose | Skill y rules |
| [Git](git/) | Flujos y buenas prácticas de Git | Skill |
| [Laravel](laravel/) | Desarrollo con Laravel | Módulo en desarrollo |
| [NestJS](nest/) | Desarrollo con NestJS | Módulo en desarrollo |
| [Next.js](next/) | App Router y React Server Components | Skill y rules |
| [OpenSpec](openspec/) | Flujos de trabajo especificados | Comandos |
| [PHP](php/) | Convenciones de PHP | Módulo en desarrollo |
| [Prisma](prisma/) | Modelado, consultas y migraciones | Skill y rules |
| [Tailwind CSS](tailwind%20css/) | Diseño y utilidades CSS | Skill y rules |
| [TypeScript](typescript/) | Tipado, arquitectura y desarrollo seguro | Skill y rules |

## 🧩 Estructura

```text
<módulo>/
├── skills/
│   └── SKILL.md
├── rules/
│   └── *.mdc
└── commands/
    └── *.md
```

No todos los módulos necesitan incluir las tres carpetas. Cada uno utiliza únicamente la estructura que corresponde a su propósito.

## 🚀 Inicio rápido

### Usar una skill

1. Abre la carpeta del dominio que necesitas.
2. Lee su archivo `skills/SKILL.md`.
3. Sigue sus convenciones y flujo de trabajo en la tarea correspondiente.

### Usar una rule

Las rules se encuentran en `rules/*.mdc` y están pensadas para integrarse en el sistema de instrucciones del editor o asistente de código. Revisa el encabezado `alwaysApply` y el alcance definido por cada archivo antes de incorporarlo.

### Usar comandos OpenSpec

Consulta los comandos disponibles en [`openspec/commands/`](openspec/commands/) para conocer el flujo de exploración, propuesta, implementación y archivo de cambios.

## 🧭 Principios del repositorio

- Mantener las instrucciones específicas y verificables.
- Preferir ejemplos concretos sobre explicaciones abstractas.
- Documentar patrones recomendados junto a sus anti-patrones.
- Evitar duplicar reglas entre módulos.
- Mantener una separación clara entre conocimiento conceptual y reglas de ejecución.

## 🤝 Cómo contribuir

### Agregar un módulo

1. Crea una carpeta con el nombre de la tecnología o dominio.
2. Agrega `skills/SKILL.md` si necesitas documentar un flujo de trabajo completo.
3. Agrega `rules/*.mdc` si necesitas definir reglas aplicables durante el desarrollo.
4. Incluye ejemplos, límites y anti-patrones relevantes.
5. Actualiza la tabla de módulos de este README.

### Revisar una contribución

Antes de finalizar, comprueba que:

- Los enlaces del README apunten a rutas existentes.
- Los ejemplos sean coherentes con la tecnología documentada.
- Las reglas sean claras, accionables y no contradictorias.
- No se hayan agregado referencias o contenido duplicado innecesariamente.