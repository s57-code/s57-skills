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

Genera un análisis completo de un blueprint de Make exportado en JSON y produce
un archivo `.md` listo para subir a GitHub. El output tiene diez secciones:
revisión crítica con mejoras, documentación técnica, documentación de JSONs con
diagrama de dependencias, conexiones requeridas, documentación explicativa,
checklist de instalación parametrizada, tabla de variables por cliente, protocolo
de verificación post-instalación, y dos secciones de estructura vacía para que
el autor complete: decisiones técnicas y errores conocidos.

---

## Flujo de trabajo

### 0. Preguntar qué se necesita

Antes de analizar nada, usar `ask_user_input_v0` para presentar un menú
interactivo con botones. Esto permite generar solo lo que se necesita en
cada momento en lugar de producir siempre el documento completo.

```
Pregunta: "¿Qué quieres hacer con este blueprint?"
Tipo: multi_select (puede elegir varias opciones)
Opciones:
  - 📋 Documentación completa (todas las secciones)
  - 🔍 Solo revisión y mejoras
  - 🔐 Solo auditoría de seguridad
  - 📦 Solo documentación de JSONs y dependencias
  - 🚀 Solo checklist de instalación para cliente nuevo
  - 📖 Solo documentación explicativa (para el cliente)
```

Según la selección, ejecutar solo las secciones correspondientes:

| Opción seleccionada | Secciones a generar |
|---|---|
| Documentación completa | 1 → 10 |
| Revisión y mejoras | Sección 1 |
| Auditoría de seguridad | Subsección 🔐 de Sección 1 |
| JSONs y dependencias | Subsección JSONs de Sección 2 |
| Checklist instalación | Secciones 6 + 7 + 8 |
| Documentación explicativa | Sección 3 |

Si el usuario selecciona "Documentación completa" o múltiples opciones que
cubran todo el documento, generar el `.md` completo en outputs. Si selecciona
una o pocas secciones concretas, entregar el resultado directamente en el chat
sin crear archivo, salvo que el usuario lo pida explícitamente.

---

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

## Output: diez secciones obligatorias

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

**🔐 Seguridad y conexiones:**

Revisar siempre estos patrones — son los más frecuentes al migrar escenarios entre
cuentas y los más difíciles de detectar a simple vista:

- Variables en `SetVariables` cuyos nombres sugieren credenciales: buscar `token`,
  `key`, `api`, `secret`, `password`, `bearer`, `auth` (case insensitive). Si
  contienen valores reales en texto plano, marcar como riesgo de seguridad y
  recomendar moverlas al keychain.
- Módulos `http:MakeRequest` que tienen `authenticationType: apiKey` en parameters
  pero además un header `Authorization` manual en el mapper — el keychain no está
  bien configurado, el token está duplicado y expuesto en el blueprint.
- Módulos `http:MakeRequest` con `authenticationType: noAuth` que llaman a URLs
  de APIs externas que típicamente requieren autenticación.
- Headers con valores que empiezan por `Bearer ` hardcodeados directamente en el
  mapper en lugar de gestionarse por keychain.
- Variables definidas en un módulo CONFIG que deberían vivir en el keychain.

**Patrón crítico — token en CONFIG + header manual que lo inyecta:**

Este es el patrón de mayor riesgo y el más difícil de detectar porque
el keychain puede parecer correcto a primera vista. Tiene dos partes que
hay que cruzar:

*Paso 1 — Detectar el token expuesto en el módulo CONFIG:*
En el `mapper.variables` de cualquier `util:SetVariables` cuyo nombre
de diseñador contenga "CONFIG" o "config" o "ajuste", buscar variables
cuyos nombres incluyan `token`, `key`, `api`, `secret`, `password`,
`bearer` o `auth` (case insensitive) **y cuyo valor sea texto plano**
(no una referencia `{{...}}` a otro módulo). Esos valores son
credenciales hardcodeadas que quedan expuestas en el blueprint exportado.

*Paso 2 — Detectar que ese token se inyecta como header manual:*
Para cada variable credencial encontrada en el paso 1, buscar en todos
los módulos `http:MakeRequest` si algún header del `mapper.headers[]`
referencia esa variable mediante `{{ID_modulo.nombre_variable}}`. Si
lo hace, el token no solo está expuesto en el CONFIG — también se está
usando para sobreescribir o duplicar la autenticación del keychain.

*Cuando se confirma el patrón, reportar así:*
```
🔐 [SEGURIDAD CRÍTICA] Token expuesto en texto plano + inyectado como header manual
Módulos afectados:
  - Módulo X (CONFIG): variable `nombre_variable` = "VALOR_REAL_DEL_TOKEN"
  - Módulo Y (http:MakeRequest): header Authorization = "Bearer {{X.nombre_variable}}"

Situación: El módulo Y tiene un keychain configurado correctamente
(apiKeyKeychain: ID_KEYCHAIN, label: "Nombre del keychain"), pero el
header manual del mapper sobreescribe la autenticación del keychain con
el token en texto plano. El token queda visible en cualquier exportación
del blueprint.

Corrección: eliminar la variable del módulo CONFIG y eliminar el header
Authorization del mapper del módulo HTTP. El keychain ya está configurado
y gestionará la autenticación. Si el keychain aún no tiene el token
correcto, actualizarlo en Connections antes de eliminar el header manual.
```

*Distinguir si el keychain ya existe o no:*
- Si `parameters.apiKeyKeychain` tiene un ID numérico real → el keychain
  **ya existe**, solo hay que eliminar el header manual y la variable CONFIG.
- Si `parameters.apiKeyKeychain` está vacío o no existe → el keychain
  **no está configurado**, hay que crearlo primero antes de eliminar el
  header manual (si se elimina antes, la autenticación dejará de funcionar).

Para cada issue de seguridad detectado, incluir instrucciones exactas de
configuración del keychain:

```
Connections → Add connection → API Key
  Key name:               [Nombre descriptivo]
  Key:                    Bearer TU_TOKEN   ← incluir prefijo "Bearer " si la API lo requiere
  API key placement:      In the header
  API Key parameter name: Authorization     ← o el header específico que use la API
```

> Advertencia clave: Make inyecta el valor del campo "Key" tal cual como valor
> del header. Si la API espera `Authorization: Bearer TOKEN`, el campo Key debe
> contener `Bearer TOKEN` completo — no solo el token. Sin el prefijo, la
> autenticación falla silenciosamente y parece que el keychain no funciona, lo
> que lleva a añadir el token como header manual (exponiéndolo en el blueprint).

Si no hay issues en alguna categoría, indícalo explícitamente ("No se detectaron
bugs en las expresiones").

---

### SECCIÓN 2 — Documentación técnica

Orientada a quien va a mantener o modificar el escenario. Incluye:

**Conexiones requeridas:**
Antes del detalle de módulos, listar todas las conexiones que necesita el escenario.
Detectarlas buscando en el JSON:
- `__IMTCONN__` en `parameters` → conexión nativa (Holded, Gmail, etc.) — mostrar
  la label del `restore.parameters.__IMTCONN__.label`
- `apiKeyKeychain` en `parameters` → conexión API Key — mostrar la label del
  `restore.parameters.apiKeyKeychain.label`

Para cada conexión indicar:
- Nombre de la conexión en Make
- Qué módulos la usan
- Cómo crearla — especialmente para API Keys, incluir los campos exactos y
  advertencias de configuración (ej: el prefijo `Bearer ` para APIs que lo requieren)

**Cabecera:**
- Nombre, zona, tipo (instant/scheduled), webhook si aplica

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

**Estructuras de datos personalizadas (json:CreateJSON):**

Documentar **todos** los módulos `json:CreateJSON` del escenario, tengan o no
Data Structure registrada en Make. Para cada uno:

**Propósito inferido:** Determinar qué hace este JSON analizando:
- El módulo inmediatamente posterior que lo consume (¿se envía a un endpoint HTTP?
  ¿se parsea? ¿alimenta otro JSON?)
- Los campos que contiene (si tiene `contactId`, `items[]`, `date` → es cuerpo de
  creación de documento; si tiene `name`, `email` → es creación de contacto, etc.)
- El nombre del módulo en el designer si lo tiene

Formato de cada JSON:

```markdown
### JSON N — `Nombre del módulo en designer` (Módulo ID)
**Propósito:** Descripción en una frase de qué representa este JSON y por qué
existe (no solo qué contiene, sino para qué se necesita construirlo así).
**Consumido por:** Módulo ID+1 — nombre — [tipo de consumo: enviado a API /
parseado / alimenta otro JSON]
**Endpoint destino** (si aplica): MÉTODO https://url/del/endpoint
**Data Structure ID en blueprint:** [ID numérico del campo `parameters.type`] —
[Nombre de la Data Structure si figura en `restore.parameters.type.label`]

| Campo | Tipo Make | Valor / Expresión | Origen |
|---|---|---|---|
| `campo` | Text / Number / Boolean / Array / Collection | `{{expresión o literal}}` | Módulo X / hardcoded / CONFIG |

**JSON de muestra para recrear la Data Structure:**
Pegar en Make → módulo json:CreateJSON → Data structure → Generator:
‣ Si el JSON raíz es un objeto:
  ```json
  {
    "campo1": "valor_ejemplo",
    "campo2": 123,
    "campo3": false,
    "campoArray": ["item1"],
    "campoObjeto": { "subcampo": "valor" }
  }
  ```
‣ Si el JSON raíz es un array (estructura items[]):
  ```json
  [
    {
      "campo1": "valor_ejemplo",
      "campo2": 123
    }
  ]
  ```

**Notas de reinstalación:** indicar si la Data Structure puede importarse
automáticamente al importar el blueprint o si hay que recrearla manualmente, y
en qué orden crearlas cuando hay dependencias entre JSONs del mismo escenario.
```

**Reglas para generar el JSON de muestra:**
- Usar valores de ejemplo realistas, del mismo tipo que los reales (string → texto
  corto, number → número plausible, boolean → false, array → un elemento de muestra)
- La **forma del JSON raíz** debe coincidir exactamente con la forma del mapper:
  - Si el mapper raíz es un objeto (`{ "campo": ... }`) → el JSON de muestra es un objeto
  - Si el mapper raíz es un array (`[ { ... } ]`) → el JSON de muestra es un array
  - ⚠️ Un mapper con un campo `items: [...]` dentro de un objeto raíz NO es un array
    raíz — es un objeto. Error frecuente: pegar un array raíz cuando el mapper es
    `{ "items": [ {...} ] }` genera una DS incorrecta que rompe el ParseJSON posterior.
- Si un campo es un array de strings (ej: `taxes: ["s_ipsi4"]`), representarlo como
  `["valor_ejemplo"]`
- Si un campo es un array de objetos (ej: `items: [{...}]`), representarlo con un
  objeto completo dentro del array
- Si un campo es un timestamp Unix (ej: `date`), usar un número entero de 10 dígitos
- Nunca poner `null` como valor de muestra — Make no infiere el tipo desde null
- Nunca omitir campos opcionales que estén en el mapper, aunque estén vacíos en el
  blueprint: ponerlos como `""` para que Make los incluya en la Data Structure

**⚠️ Error crítico frecuente — Array of Text en lugar de Array of Collection:**

Cuando el Generator de Make infiere una DS que contiene un array de objetos
(ej: `items: [ { "sku": "", "desc": "", ... } ]`), puede tipar el array como
`Array → item type: Text` en lugar de `Array → item type: Collection`. Esto
hace que al serializar el JSON final, Make envuelva cada objeto en comillas
escapadas, produciendo un array de strings en lugar de un array de objetos:

```
❌ "items": ["{\"sku\":null,\"desc\":\"Pago único\",...}"]   ← string escapado
✅ "items": [{"sku": null, "desc": "Pago único", ...}]       ← objeto correcto
```

La API destino recibe un string donde espera un objeto y falla silenciosamente
o crea el registro con datos incorrectos.

**Cómo detectarlo:** abrir la DS del módulo afectado → campo `items` → si
"Array Item Specification → Type" es `Text`, está mal.

**Cómo corregirlo:**
1. Editar la DS: campo `items` → Array Item Specification → Type → cambiar
   `Text` a `Collection`
2. Añadir los subcampos de la colección uno a uno con su tipo correcto
3. Guardar la DS
4. Volver al módulo `json:CreateJSON` que la usa y verificar que el mapping
   de `items` sigue apuntando a la variable correcta (Make puede resetearlo
   al modificar la DS)

**En la documentación:** para cada campo que sea `Array of Collections`,
documentar explícitamente los subcampos con su tipo Make en la tabla, y añadir
una nota en el JSON de muestra indicando que Make puede inferirlo mal y que hay
que verificar el tipo del array tras crear la DS con el Generator.

---

**Aviso sobre Data Structures al migrar a cuenta nueva:**

Make **no exporta las Data Structures** al exportar un blueprint. Al importarlo
en otra cuenta, todos los módulos `json:CreateJSON` aparecerán con la Data
Structure en rojo ("Data structure not found"). Hay que recrearlas manualmente
antes de que el escenario pueda ejecutarse.

Incluir siempre este bloque en la documentación cuando hay `json:CreateJSON`:

```markdown
### ⚠️ Recrear Data Structures tras importar el blueprint

Make no exporta las Data Structures con el blueprint. Al importarlo en una
cuenta nueva, los módulos json:CreateJSON aparecerán con error. Recrearlas
en este orden (respetar el orden si hay dependencias entre JSONs):

1. **[Nombre DS 1]** → módulo [ID]
   - json:CreateJSON → campo "Data structure" → Add → Generator
   - Pegar el JSON de muestra de la sección JSON 1 → Save
   - Nombre: `[Nombre exacto]`
   - ⚠️ Si la DS contiene arrays de objetos, verificar que "Array Item
     Specification → Type" es `Collection` y no `Text`. Si Make lo infiere
     como Text, corregirlo manualmente añadiendo los subcampos de la colección.

2. **[Nombre DS 2]** → módulo [ID]
   ...

> El orden importa cuando un json:ParseJSON intermedio alimenta otro
> json:CreateJSON: si la primera DS no está creada, el ParseJSON no
> tiene schema y el siguiente CreateJSON no puede mapear sus campos.
```

**Diagrama de dependencias entre JSONs:**

Cuando hay múltiples `json:CreateJSON` en el escenario, generar siempre un
diagrama ASCII que muestre cómo se relacionan entre sí y con los endpoints:

```
[M21] json:CreateJSON "Líneas de factura"
    │  Construye array items[]
    ▼
[M27] json:ParseJSON   ← necesita DS de M21 para tener schema
    │  items[] tipado
    ▼
[M26] json:CreateJSON "Cuerpo factura"  ←── [M23] GetVariables (contactId, date)
    │  Embebe items[] + metadatos
    ▼
[M20] http → POST /invoices (Holded)
```

Reglas del diagrama:
- Mostrar el ID y nombre corto de cada módulo JSON
- Indicar qué dato fluye por cada flecha
- Incluir los módulos que alimentan cada JSON (GetVariables, SetVariables)
- Incluir el endpoint final al que llega el JSON construido
- Si un JSON no llega directamente a un HTTP sino que alimenta otro JSON,
  mostrarlo con una flecha lateral
- Marcar explícitamente los `json:ParseJSON` intermedios, porque son los que
  crean la dependencia de orden al recrear Data Structures

**Por qué se construyen los JSONs a mano:**
Si el escenario usa `json:CreateJSON` para llamar a una API que tiene módulo
nativo en Make (ej: Holded), inferir y documentar el motivo probable:
- El módulo nativo no expone todos los campos necesarios
- La estructura requiere arrays anidados que el módulo nativo no soporta
- Se necesita control exacto sobre campos opcionales/nulos
Marcar como `[inferido]` si no hay evidencia explícita en el blueprint.

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

### SECCIÓN 6 — Checklist de instalación en cliente nuevo

Generar una checklist parametrizada y accionable. No genérica — usar los nombres
reales de conexiones, módulos y variables del escenario.

Estructura:

```markdown
## Checklist de instalación — [Nombre del escenario]

### Antes de importar el blueprint
- [ ] Obtener del cliente: [listar exactamente qué credenciales/datos necesitas,
      ej: "API Key de ThriveCart" / "credenciales de Holded" / "ID de plantilla de email"]

### Conexiones a crear (en este orden)
- [ ] **[Nombre conexión 1]** — Tipo: [API Key / OAuth / nativa]
      Cómo crearla: Connections → Add → [pasos específicos]
      ⚠️ [Advertencia si aplica, ej: incluir prefijo "Bearer " en el campo Key]
- [ ] **[Nombre conexión 2]** ...

### Recrear Data Structures (si hay módulos json:CreateJSON)
- [ ] Abrir cada módulo `json:CreateJSON` con Data Structure en rojo
- [ ] Para cada uno: Data structure → Add → [nombre] → Generator → pegar JSON de muestra (ver sección JSONs) → Save
- [ ] Respetar el orden indicado en la sección JSONs si hay dependencias entre estructuras

### Configurar módulo CLIENT (si existe)
- [ ] Campo `[nombre]`: [qué valor poner y cómo obtenerlo]

### Configurar módulo CONFIG
- [ ] `[variable]`: [valor por defecto] → cambiar a [qué] para este cliente
- [ ] `[flag]`: false → cambiar a true cuando [condición]

### Webhook
- [ ] Copiar URL del webhook de Make
- [ ] Configurarlo en [plataforma origen] → [ruta exacta en esa plataforma]

### Verificación (ver Sección 8)
- [ ] Ejecutar evento de prueba
- [ ] Confirmar resultado esperado
- [ ] Activar escenario en producción
```

---

### SECCIÓN 7 — Qué cambia por cliente vs qué es fijo

Tabla que separa claramente las dos categorías para instalaciones rápidas:

```markdown
## Variables por instalación

| Variable / Ajuste | Módulo | Valor por defecto | Qué poner en cada cliente |
|---|---|---|---|
| `[variable]` | CONFIG | `[valor]` | [instrucción concreta] |

## Constantes del escenario (no tocar)

| Elemento | Módulo | Valor | Por qué es fijo |
|---|---|---|---|
| `[constante]` | [módulo] | `[valor]` | [motivo] |
```

Inferir qué es fijo vs variable analizando:
- Variables del módulo CONFIG → candidatas a cambiar por cliente
- IDs hardcodeados en mappers intermedios → probablemente fijos del escenario
- URLs de endpoints → fijas (son de la API) salvo que haya entornos staging/prod
- Flags booleanos de entorno → variables (controlan comportamiento en prod vs test)

---

### SECCIÓN 8 — Protocolo de verificación post-instalación

Instrucciones concretas para confirmar que el escenario funciona antes de
entregarlo al cliente. Inferir el protocolo del tipo de trigger y del flujo:

```markdown
## Protocolo de verificación

### Evento de prueba
- **Qué lanzar:** [descripción del evento de prueba — ej: "completar una compra
  de prueba en ThriveCart con el producto X en modo sandbox"]
- **Dónde lanzarlo:** [plataforma / ruta]

### Qué debe ocurrir en Make
| Paso | Módulo | Resultado esperado | Cómo verificarlo |
|---|---|---|---|
| 1 | [Webhook] | Recibe el payload | Ver "Input" del módulo en el historial |
| 2 | [CONFIG] | Variables inicializadas | Ver "Output" → comprobar valores |
| ... | | | |

### Resultado final esperado
- **En [sistema destino]:** [qué debe haberse creado/modificado y dónde verlo]
- **Confirmación visual:** [qué pantalla abrir y qué buscar]

### Si algo falla
- Primero comprobar: [el punto más frecuente de fallo en este escenario]
- Ver Sección 10 (Errores conocidos) para diagnóstico detallado
```

---

### SECCIÓN 9 — Decisiones técnicas *(completar manualmente)*

Generar la estructura vacía. No inventar contenido — dejar las filas en blanco
para que el autor las rellene con su conocimiento del caso.

```markdown
## Decisiones técnicas

> Esta sección recoge el "por qué está construido así". Rellénala justo después
> de terminar la instalación, cuando el razonamiento está fresco.

| Decisión | Alternativa descartada | Motivo |
|---|---|---|
| Se construye el JSON de factura manualmente con `json:CreateJSON` en lugar de usar el módulo nativo de Holded | Módulo nativo `holded:createDocument` | |
| | | |
| | | |

> Añade una fila por cada decisión de diseño no obvia del escenario.
```

Regla: pre-rellenar la columna "Decisión" con las decisiones técnicas que sí
son visibles en el blueprint (ej: uso de JSON manual cuando existe módulo nativo,
rutas paralelas en lugar de secuenciales, variables roundtrip en lugar de
pasarlas por mapper...). Dejar "Alternativa descartada" y "Motivo" en blanco.

---

### SECCIÓN 10 — Errores conocidos y resolución *(completar manualmente)*

Generar la estructura vacía con columnas relevantes para el tipo de escenario.

```markdown
## Errores conocidos y resolución

> Documenta aquí cada error que encuentres durante instalaciones o en producción.
> Esta tabla es tu base de conocimiento de soporte para este escenario.

| Síntoma | Cuándo ocurre | Causa probable | Solución | Módulo afectado |
|---|---|---|---|---|
| | | | | |
| | | | | |

> Ejemplos de síntomas a documentar: "El escenario se ejecuta pero no crea la
> factura", "Error 401 en el módulo HTTP", "El contacto se duplica en Holded",
> "El webhook no recibe el payload de ThriveCart".
```

---

## Formato de entrega

El output **siempre** es un archivo `.md` guardado en `/mnt/user-data/outputs/`
con el nombre `[nombre-escenario]-docs.md`. Nunca entregar el output solo en
el chat — siempre crear el archivo.

Estructura del archivo:

```markdown
# [Nombre del escenario]
> Generado: [fecha]  |  Blueprint: [nombre del archivo JSON]

---
## Índice
1. [Revisión y mejoras](#revisión-y-mejoras)
2. [Documentación técnica](#documentación-técnica)
3. [JSONs — estructura y dependencias](#jsons--estructura-y-dependencias)
4. [Conexiones requeridas](#conexiones-requeridas)
5. [Documentación explicativa](#documentación-explicativa)
6. [Checklist de instalación](#checklist-de-instalación)
7. [Qué cambia por cliente](#qué-cambia-por-cliente)
8. [Protocolo de verificación](#protocolo-de-verificación)
9. [Decisiones técnicas](#decisiones-técnicas)
10. [Errores conocidos](#errores-conocidos)
---
[secciones 1-10]
```

Usa tablas para las variables siempre que haya más de 3. Usa bloques de código
para expresiones Make y para los diagramas ASCII. Sin límite de longitud —
la documentación debe ser completa. Es preferible pecar de exhaustivo que dejar huecos.

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
