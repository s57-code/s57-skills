# s57-skills

Colección de skills personalizadas para Claude, organizadas por herramienta o plataforma. Cada skill contiene las instrucciones necesarias para que Claude ejecute flujos de trabajo especializados de forma consistente y documentada.

## Qué es una skill

Una skill es un conjunto de instrucciones que extiende las capacidades de Claude para una tarea concreta. Una vez instalada en Claude.ai, se activa automáticamente cuando el contexto de la conversación lo requiere — sin necesidad de explicar cada vez cómo hacer las cosas.

## Estructura del repositorio

```
s57-skills/
└── make-skills/                        ← Skills para Make (antes Integromat)
    └── make-blueprint-analyzer/        ← Análisis, revisión y documentación de blueprints
        ├── SKILL.md
        └── README.md
```

## Skills disponibles

### Make
| Skill | Descripción |
|---|---|
| [make-blueprint-analyzer](./make-skills/make-blueprint-analyzer/) | Analiza un blueprint de Make: detecta bugs, propone mejoras y genera documentación técnica y explicativa |

## Instalación de una skill

1. Clona el repositorio
2. Genera el archivo `.skill` desde la raíz del repo:
```bash
zip -r make-blueprint-analyzer.skill make-skills/make-blueprint-analyzer/
```
3. En Claude.ai ve a **Configuración → Skills** y sube el archivo `.skill`

## Convenciones

- Cada skill vive en su propia carpeta con un `SKILL.md` y un `README.md`
- Los archivos `.skill` (zips de instalación) no se commitean — son artefactos derivados
- Las skills se organizan por herramienta (`make-skills/`, etc.)

## .gitignore

```
*.skill
```
