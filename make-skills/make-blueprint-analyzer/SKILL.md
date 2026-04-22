---
name: make-blueprint-analyzer
description: >
  Analiza, revisa y documenta blueprints de Make (antes Integromat) en formato JSON.
  Usa esta skill siempre que el usuario comparta un blueprint de Make (JSON exportado
  desde Make), mencione un escenario de Make, pida revisar o documentar una automatización
  de Make, o quiera entender qué hace un flujo de Make. También úsala cuando el usuario
  pida detectar bugs, proponer mejoras o generar documentación técnica o explicativa de
  un escenario Make. Aplica aunque el usuario no use la palabra "skill" o "blueprint" —
  basta con que comparta un JSON con estructura de escenario Make (campos "flow",
  "module", "mapper", "routes") o hable de módulos, routers, webhooks o automatizaciones
  de Make/Integromat.
---

# Skill: Analizador de Blueprints de Make

Genera un análisis completo de un blueprint de Make exportado en JSON. El output tiene
tres partes bien diferenciadas: revisión crítica con mejoras propuestas, documentación
técnica detallada, y documentación explicativa para cualquier persona del equipo.

---

## Flujo de trabajo

### 1. Parsear el blueprint

Extrae la estructura del JSON identificando:

- Nombre del escenario y metadatos (zona, tipo instant/scheduled)
- Todos los módulos con su `id`, `module` (tipo), nombre en designer y filtros
- La jerarquía de routers y ramas (routes, branches)
- Las variables definidas en cada `SetVariables` y leídas en cada `GetVariables`
- Las llamadas a APIs externas (URLs, métodos, autenticación)
- Los filtros de entrada de cada módulo o ruta

Para blueprints grandes usa este enfoque sistemático:
1. Primero recorre el flujo de nivel superior
2. Luego entra en cada router y mapea sus rutas
3. Dentro de cada ruta, mapea los módulos en orden
4. Para If/Else, mapea cada branch por separado

### 2. Construir el mapa de variables

Antes de escribir nada, construye internamente un mapa completo:

```
variable → definida en módulo X → leída en módulos Y, Z
```

Esto permite detectar: variables definidas pero no usadas, variables leídas antes
de ser definidas (según el orden de ejecución), y variables sobreescritas en rutas
paralelas.

### 3. Determinar el orden de ejecución real

Los routers de Make ejecutan sus rutas **en paralelo**, no en secuencia. Esto es
crítico para entender qué variables están disponibles en cada punto. El patrón
habitual para sincronizar rutas paralelas es usar `GetVariables` en una ruta
posterior que recoge lo que definieron las rutas anteriores.

Documenta explícitamente qué rutas son paralelas y cuál actúa como punto de
convergencia.

---

## Output: tres secciones obligatorias

### SECCIÓN 1 — Revisión y mejoras

Estructura como una lista priorizada. Para cada issue encontrado indica:
- **Tipo**: 🐛 Bug / ⚠️ Riesgo / 🧹 Limpieza / 💡 Mejora
- **Módulo(s) afectado(s)**
- **Descripción** del problema
- **Impacto** si no se corrige
- **Corrección sugerida** (con la expresión corregida si aplica)

Aspectos a revisar siempre:

**Bugs y riesgos:**
- Expresiones con paréntesis mal balanceados en `if()`, `contains()`, `switch()`
- Variables leídas con `GetVariables` que podrían no estar definidas si una ruta
  paralela no se ejecutó (por filtro condicional)
- Campos hardcodeados que deberían ser variables (emails, IDs, valores literales
  enterrados en módulos intermedios)
- Lógica `switch()` incompleta (casos que caen fuera de todos los predicados)
- Módulos con nombres vacíos o genéricos que dificultan la lectura del flujo

**Código residual:**
- Variables definidas en `SetVariables` que nunca aparecen en ningún `GetVariables`
  ni se referencian en ningún mapper posterior
- Módulos desconectados o con lógica duplicada por otro módulo posterior

**Mejoras de arquitectura:**
- Variables hardcodeadas que deberían estar centralizadas en un módulo de
  configuración (especialmente flags de entorno, IDs de plantillas, constantes de
  negocio que pueden cambiar)
- Oportunidades de simplificar lógica redundante

**Mejoras de mantenibilidad:**
- Módulos sin nombre en el designer
- Constantes de negocio mezcladas con datos dinámicos sin separación visual

Si no hay issues en alguna categoría, indícalo explícitamente ("No se detectaron
bugs en las expresiones").

---

### SECCIÓN 2 — Documentación técnica

Orientada a quien va a mantener o modificar el escenario. Incluye:

**Cabecera:**
- Nombre, zona, tipo (instant/scheduled), conexiones usadas, webhook si aplica

**Propósito:**
- Qué hace el escenario en una o dos frases

**Eventos que lo disparan** (si aplica):
- Tabla con evento → descripción

**Diagrama del flujo** (ASCII):
```
[ID] Nombre módulo
    └── [ID] Nombre módulo    (condición si la hay)
            ├── Ruta 1 → ...
            └── Ruta 2 → ...
```

**Módulos — detalle:**
Para cada módulo, en orden de ejecución:
- ID, tipo Make, nombre en designer
- Cuándo se ejecuta (siempre / condición)
- Tabla de variables que define (con origen/fórmula y nota si es compleja)
- Para llamadas HTTP: URL, método, autenticación
- Para If/Else: condición y qué hace cada rama

**Cálculos importantes:**
Si hay fórmulas no triviales (impuestos, fechas, detección por texto), explícalas
separadamente con la fórmula y su significado.

**Referencia de variables globales:**
Tabla con todas las variables de scope roundtrip: nombre, dónde se define,
dónde se lee, descripción.

**Estructuras de datos personalizadas:**
Si el blueprint contiene módulos `json:CreateJSON` con un `parameters.type` numérico
(ID de Data Structure), documentar cada estructura en su propia subsección. Para cada
una extraer el esquema completo del campo `expect` del módulo.

Formato de cada estructura:

```markdown
### `Nombre de la estructura` (ID: XXXXXX)
**Usada en:** Módulo N — Nombre del módulo
**Propósito:** Para qué sirve esta estructura.

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `campo` | tipo | Sí/No | descripción |

**Ejemplo de uso en este escenario:**
\`\`\`json
{
  "campo": "valor_real_del_mapper"
}
\`\`\`
```

Reglas para documentar estructuras:
- El nombre de la estructura viene del campo `restore.parameters.type.label` del módulo
- Los campos y tipos vienen del `expect` del módulo
- El ejemplo se construye con los valores reales del `mapper` del módulo, no con
  valores genéricos — así quien lo copie entiende exactamente qué se está enviando
- Si un campo del `expect` no aparece en el `mapper`, incluirlo igualmente en la tabla
  marcándolo como No requerido y con valor vacío en el ejemplo
- Documentar todas las estructuras aunque sean similares — cada una puede usarse
  de forma independiente en otro escenario
- Añadir una nota al final de la sección si alguna estructura referencia a otra
  (ej: el campo `items` de la estructura de documento usa la estructura de líneas)

---

### SECCIÓN 3 — Documentación explicativa

Orientada a alguien del equipo que no ha visto el escenario antes y necesita
entender qué hace sin conocer Make en profundidad. Sin jerga técnica innecesaria.

Estructura:

**¿Qué hace este escenario?**
Explicación en lenguaje natural de qué problema resuelve y cuándo se activa.
Máximo 3-4 frases.

**¿Qué pasa paso a paso?**
Narración del flujo completo como si fuera un proceso de negocio:
- Empieza cuando...
- Primero recoge...
- Luego comprueba si...
- Si el cliente no existe, lo crea con...
- Finalmente genera la factura con...

No mencionar IDs de módulos. Usar lenguaje de negocio.

**¿Qué productos/casos cubre?**
Si el escenario tiene lógica condicional por tipo de producto, evento, etc.,
explicar cada caso brevemente.

**¿Qué genera al final?**
El resultado concreto: qué se crea, dónde, en qué estado.

**¿Hay algo desactivado o en pruebas?**
Si hay flags de entorno o funcionalidades condicionales, explicarlas en lenguaje
llano: "Actualmente el escenario no envía el email automáticamente. Cuando esté
listo para producción, se puede activar cambiando un ajuste."

---

## Formato de entrega

Si el usuario pide un archivo (markdown, Word, etc.), genera las tres secciones
en ese formato. Si no especifica, entrégalo en markdown directamente en el chat,
con las tres secciones claramente separadas por encabezados de nivel 2.

Usa tablas para las variables siempre que haya más de 3. Usa bloques de código
para expresiones Make y para el diagrama ASCII del flujo.

Longitud: sin límite artificial. La documentación debe ser completa. Es preferible
pecar de exhaustivo que dejar huecos.

---

## Notas sobre expresiones Make

Al revisar expresiones, ten en cuenta:
- `{{if(condición; valor_true; valor_false)}}` — el tercer argumento es opcional
- `{{switch(true; cond1; val1; cond2; val2; ...)}}` — sin valor por defecto, cae en
  `undefined` si ninguna condición es true
- `contains(lower(x); "texto")` — siempre lowercase para comparaciones de texto
- `parseDate` y `formatDate` son sensibles al formato exacto; un formato incorrecto
  falla silenciosamente devolviendo null
- `emptystring` es la constante Make para string vacío, no `""`
- `39.order.future_charges[]` con `[]` itera sobre arrays; sin `[]` devuelve el
  objeto completo
- En routers paralelos, el scope `roundtrip` de `SetVariables` es compartido entre
  rutas, pero la disponibilidad depende del orden de ejecución real
