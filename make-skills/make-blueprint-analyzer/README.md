# make-blueprint-analyzer

Skill para Claude que analiza, revisa y documenta blueprints de Make (antes Integromat) exportados en JSON.

## Qué hace

Dado un blueprint en JSON, genera tres outputs en un solo paso:

**1. Revisión crítica**
Detecta bugs, riesgos, código residual y oportunidades de mejora. Cada issue viene con el módulo afectado, el impacto si no se corrige y la corrección sugerida — incluyendo la expresión corregida cuando aplica.

**2. Documentación técnica**
Para quien mantiene el escenario. Incluye diagrama del flujo, detalle módulo a módulo con tablas de variables y sus fórmulas, estructuras de datos personalizadas con sus esquemas completos y ejemplos reales, y una referencia global de todas las variables.

**3. Documentación explicativa**
Para cualquier persona del equipo sin conocimientos de Make. Explica qué hace el escenario, qué pasa paso a paso en lenguaje de negocio, qué casos cubre y qué genera al final.

## Cuándo usarla

- Tienes un blueprint nuevo y quieres entender qué hace antes de tocarlo
- Acabas de terminar un escenario y quieres documentarlo para el repo
- Sospechas que hay un bug y quieres una revisión sistemática
- Quieres exportar una estructura de datos de un escenario para reutilizarla en otro

## Cómo usarla

Comparte el JSON del blueprint en el chat de Claude y pide lo que necesitas:

```
"documéntame este blueprint"
"revisa este escenario de Make y dime qué mejorarías"
"qué hace este flujo?"
"genera la documentación técnica y explicativa"
```

Claude detecta automáticamente que es un blueprint de Make y aplica la skill.

## Qué sabe revisar

- Paréntesis mal balanceados en expresiones `if()`, `contains()`, `switch()`
- Variables que podrían no estar definidas por rutas condicionales paralelas
- `switch()` sin valor por defecto (casos que caen en `undefined`)
- Campos hardcodeados que deberían ser variables de configuración
- Código residual: variables definidas pero nunca leídas, lógica duplicada
- Módulos sin nombre en el designer
- Constantes de negocio mezcladas con datos dinámicos

## Estructura de datos que extrae

Para cada módulo `json:CreateJSON` con una Data Structure asociada, documenta el esquema completo de campos con tipos y si son requeridos, y construye un ejemplo con los valores reales del escenario — listo para copiar y reutilizar en otro escenario.

## Archivos

```
make-blueprint-analyzer/
├── SKILL.md      ← instrucciones para Claude
└── README.md     ← este archivo
```

## Instalación

1. Descarga o genera el archivo `make-blueprint-analyzer.skill`
2. En Claude.ai ve a Configuración → Skills
3. Sube el archivo `.skill`

Para generar el `.skill` desde el fuente:

```bash
cd make-blueprint-analyzer
zip -r ../make-blueprint-analyzer.skill .
mv ../make-blueprint-analyzer.skill ../make-blueprint-analyzer.skill
```

O desde la carpeta padre:

```bash
zip -r make-blueprint-analyzer.skill make-blueprint-analyzer/
```
