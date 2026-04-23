# make-blueprint-analyzer

Skill para Claude que analiza, revisa y documenta blueprints de Make (antes Integromat) exportados en JSON. Genera un archivo `.md` listo para subir a GitHub con todo lo necesario para instalar el escenario en un cliente nuevo.

## Qué hace

Al recibir un blueprint, primero pregunta qué necesitas mediante un menú interactivo con botones. Puedes pedir una sección concreta o la documentación completa.

**1. Revisión crítica**
Detecta bugs, riesgos, código residual y oportunidades de mejora. Cada issue viene con el módulo afectado, el impacto si no se corrige y la corrección sugerida — incluyendo la expresión corregida cuando aplica.

**2. Auditoría de seguridad**
Detecta credenciales expuestas en texto plano en módulos CONFIG, tokens inyectados como headers manuales en módulos HTTP, y conflictos entre keychain configurado y autenticación manual duplicada. Para cada issue indica los pasos exactos de corrección y si el keychain ya existe o hay que crearlo primero.

**3. Documentación técnica**
Para quien mantiene el escenario. Incluye diagrama del flujo, detalle módulo a módulo con tablas de variables y sus fórmulas, referencia global de todas las variables, y documentación completa de conexiones requeridas con instrucciones de creación.

**4. Documentación de JSONs y dependencias**
Para cada `json:CreateJSON` del escenario: propósito inferido por contexto, tabla de campos con origen de cada valor, y un diagrama ASCII que muestra cómo fluyen los JSONs entre sí hasta llegar al endpoint final. Incluye nota sobre por qué se construyen a mano cuando existe módulo nativo.

**5. Documentación explicativa**
Para cualquier persona del equipo sin conocimientos de Make. Explica qué hace el escenario, qué pasa paso a paso en lenguaje de negocio, qué casos cubre y qué genera al final.

**6. Checklist de instalación parametrizada**
No genérica — usa los nombres reales de conexiones, módulos y variables del escenario. Lista exactamente qué credenciales pedir al cliente, cómo crear cada conexión, qué configurar en el módulo CONFIG y cómo configurar el webhook.

**7. Qué cambia por cliente vs qué es fijo**
Tabla que separa las variables que hay que ajustar en cada instalación de las constantes del escenario que no se tocan.

**8. Protocolo de verificación post-instalación**
Qué evento de prueba lanzar, qué debe ocurrir módulo a módulo y cómo confirmar el resultado en el sistema destino antes de entregar al cliente.

**9. Decisiones técnicas** *(estructura vacía para completar manualmente)*
Tabla con las decisiones de diseño no obvias pre-identificadas del blueprint. El autor completa el motivo de cada una justo después de terminar la instalación.

**10. Errores conocidos** *(estructura vacía para completar manualmente)*
Tabla para acumular síntomas, causas y soluciones de errores encontrados en instalaciones o en producción. Se convierte en la base de conocimiento de soporte del escenario.

## Cuándo usarla

- Acabas de terminar un escenario y necesitas documentarlo para el repo
- Vas a instalar un escenario en un cliente nuevo y quieres el checklist y las instrucciones
- Sospechas que hay un bug o un problema de seguridad y quieres una revisión sistemática
- Quieres exportar la estructura de un JSON para reutilizarla en otro escenario
- Tienes un blueprint que no has tocado en meses y necesitas entender qué hace

## Cómo usarla

Comparte el JSON del blueprint en el chat de Claude. La skill pregunta qué quieres hacer mediante un menú con botones:

```
📋 Documentación completa (todas las secciones)
🔍 Solo revisión y mejoras
🔐 Solo auditoría de seguridad
📦 Solo documentación de JSONs y dependencias
🚀 Solo checklist de instalación para cliente nuevo
📖 Solo documentación explicativa (para el cliente)
```

También puedes pedirlo directamente en texto:

```
"documéntame este blueprint"
"revisa este escenario de Make y dime qué mejorarías"
"auditoría de seguridad de este blueprint"
"genera el checklist de instalación para cliente nuevo"
"qué hace este flujo?"
```

Si pides documentación completa, genera un `.md` listo para subir a GitHub.
Si pides una sección concreta, entrega el resultado directamente en el chat.

## Qué sabe revisar

**Bugs y riesgos:**
- Paréntesis mal balanceados en expresiones `if()`, `contains()`, `switch()`
- Variables que podrían no estar definidas por rutas condicionales paralelas
- `switch()` sin valor por defecto (casos que caen en `undefined`)
- Campos hardcodeados que deberían ser variables de configuración

**Seguridad:**
- Credenciales en texto plano en módulos CONFIG (`token`, `key`, `api`, `secret`...)
- Token en CONFIG inyectado como header manual en módulo HTTP — incluso cuando el keychain está configurado
- Headers `Authorization: Bearer ...` hardcodeados en el mapper
- Módulos `http:MakeRequest` con `noAuth` llamando a APIs que requieren autenticación

**Calidad:**
- Código residual: variables definidas pero nunca leídas, lógica duplicada
- Módulos sin nombre en el designer
- Constantes de negocio mezcladas con datos dinámicos

## Formato de salida

Siempre en `.md`. Si se pide documentación completa, crea el archivo en outputs con el nombre `[nombre-escenario]-docs.md` listo para subir al repo. Incluye índice con anclas a las diez secciones.

## Archivos

```
make-blueprint-analyzer/
├── SKILL.md      ← instrucciones para Claude
└── README.md     ← este archivo
```

## Instalación

Desde la carpeta padre del repo:

```bash
zip -r make-blueprint-analyzer.skill make-blueprint-analyzer/
```

En Claude.ai: Configuración → Skills → subir el archivo `.skill`.
