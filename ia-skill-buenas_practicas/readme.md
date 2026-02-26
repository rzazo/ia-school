# Lo que aprendí creando Skills para Agentes de IA

> Fuente original: [mgechev/skills-best-practices](https://github.com/mgechev/skills-best-practices) — traducido y adaptado al español.

Llevo un tiempo trabajando con agentes de IA y una de las cosas que más me ha costado entender bien es **cómo escribir skills que realmente funcionen**. No scripts que "más o menos" hagan lo que quiero, sino skills profesionales, predecibles y que no contaminen el contexto del agente con información innecesaria.

En este post comparto las prácticas que más me han servido. Si quieres la documentación completa y oficial, puedes leer [la guía de Claude](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices).

---

## La estructura que uso para cada skill

Cada skill que creo sigue esta estructura de directorios:

```
skill-name/
├── SKILL.md              # Obligatorio: Metadatos + instrucciones principales (<500 líneas)
├── scripts/              # Scripts ejecutables (Python/Bash) diseñados como pequeñas CLIs
├── references/           # Contexto complementario (esquemas, cheatsheets)
└── assets/               # Plantillas o archivos estáticos usados en la salida
```

- **SKILL.md** es el "cerebro". Lo uso para navegación y procedimientos de alto nivel.
- **References**: los enlazo directamente desde `SKILL.md`. Los mantengo **a un solo nivel de profundidad**.
- **Scripts**: para operaciones frágiles o repetitivas donde cualquier variación es un bug. No meto código de librería aquí.

---

## Lo que más me costó entender: el frontmatter lo es todo

El `name` y la `description` del frontmatter de `SKILL.md` son lo **único** que el agente ve antes de decidir si activa una skill. Si no están bien escritos, la skill es invisible.

Dos reglas que no me salto:

**1. Nomenclatura estricta**
El campo `name` debe tener entre 1 y 64 caracteres, solo letras minúsculas, números y guiones (sin guiones consecutivos). Y lo más importante: **debe coincidir exactamente con el nombre del directorio padre**. Si el skill se llama `angular-testing`, debe vivir en `angular-testing/SKILL.md`.

**2. Descripciones optimizadas para el trigger**
La descripción (máximo 1.024 caracteres) es el único metadato que el agente usa para decidir qué skill cargar. Por eso la escribo **en tercera persona** y pensando en cuándo activarla... y cuándo **no**.

| ❌ Malo | ✅ Bueno |
|--------|---------|
| "React skills." | "Crea y construye componentes React con Tailwind CSS. Úsalo cuando el usuario quiera actualizar estilos o lógica de UI. No usar para proyectos Vue, Svelte o CSS puro." |

La diferencia: el bueno incluye **negative triggers**. Eso me ha ahorrado muchos falsos positivos.

---

## Cómo mantengo el contexto limpio (Progressive Disclosure)

Uno de los errores que cometía al principio era meter todo el contexto en `SKILL.md`. El agente terminaba con la ventana de contexto saturada de información que no necesitaba en ese momento.

Lo que hago ahora:

- **`SKILL.md` de menos de 500 líneas**. Solo navegación y procedimientos principales.
- **Subdirectorios planos**: un solo nivel de profundidad (`references/schema.md`, no `references/db/v1/schema.md`). Cada carpeta tiene su propósito claro:
  - `references/`: docs de API, cheatsheets, lógica de dominio
  - `scripts/`: código ejecutable para tareas determinísticas
  - `assets/`: plantillas de output, esquemas JSON, imágenes
- **Carga Just-in-Time (JiT)**: le indico explícitamente al agente cuándo leer un archivo. Hasta que no le digo que lo lea, no lo ve. Por ejemplo: *"Ver `references/auth-flow.md` para los códigos de error específicos"*.
- **Rutas relativas siempre**: con barras hacia adelante (`/`), sin importar el sistema operativo.

Y lo que **no creo** en mis skills:
- Archivos de documentación para humanos (`README.md`, `CHANGELOG.md`...)
- Lógica redundante que el agente ya maneja bien por su cuenta
- Código de librería. Los scripts deben ser pequeños y de un solo propósito

---

## Instrucciones procedurales, no prosa

Esta fue otra lección importante: las skills son para agentes, no para humanos. Eso cambia completamente cómo escribo las instrucciones.

Lo que me funciona:

- **Numeración paso a paso**: defino el flujo como una secuencia cronológica estricta. Si hay un árbol de decisión, lo mapeo explícitamente. Ejemplo: *"Paso 2: Si necesitas source maps, ejecuta `ng build --source-map`. Si no, salta al Paso 3."*
- **Plantillas concretas**: en vez de párrafos explicando cómo debe verse un JSON, pongo una plantilla en `assets/` y le digo al agente que copie su estructura. Los agentes hacen pattern-matching muy bien.
- **Imperativo en tercera persona**: *"Extrae el texto..."* en vez de *"Voy a extraer..."* o *"Deberías extraer..."*

También soy muy consistente con la terminología. Elijo un término para cada concepto y no lo cambio. Por ejemplo, en Angular siempre uso "template", nunca "html", "markup" o "view".

---

## Scripts para lo determinístico

Si el agente necesita parsear un dataset complejo o hacer consultas específicas a una base de datos, no le pido que lo genere desde cero cada vez. Le doy un script probado en `scripts/`.

Dos cosas que nunca olvido al escribir esos scripts:

1. **Manejo de errores con mensajes descriptivos**: el agente usa stdout/stderr para saber si un script funcionó. Si el mensaje de error es genérico, el agente no sabe cómo autocorregirse. Si es específico, puede.
2. **Un script = una responsabilidad**: nada de scripts que hacen diez cosas. Pequeños, enfocados, predecibles.

---

## Mi proceso de validación

Esto es lo que más me ha cambiado la forma de crear skills: **validar con LLMs antes de dar la skill por buena**.

Tener evals es crítico para asegurarse de que los cambios que haces tienen un impacto positivo y no introducen regresiones. Un benchmark que me parece útil como referencia es [SkillsBench](https://arxiv.org/abs/2602.12670).

### 1. Validación de discovery

Pruebo si el frontmatter activa la skill en los casos correctos y no en los incorrectos. Pego esto en una conversación limpia con un LLM:

> Estoy construyendo un Agent Skill basado en la especificación agentskills.io. Los agentes deciden si cargar esta skill basándose únicamente en los metadatos YAML de abajo.
>
> ```
> name: angular-vite-migrator
> description: Migra proyectos Angular CLI de Webpack a Vite y esbuild. Usar cuando el usuario quiera actualizar configuraciones de builder, reemplazar plugins de webpack por equivalentes de rollup, o acelerar la compilación de Angular.
> ```
>
> Basándote estrictamente en esta descripción:
> 1. Genera 3 prompts realistas de usuario que deberían activar esta skill con un 100% de confianza.
> 2. Genera 3 prompts que suenen similares pero que **no** deberían activarla (ej: migrar una app React a Vite, o simplemente actualizar versiones de Angular).
> 3. Critica la descripción: ¿Es demasiado amplia? Sugiere una reescritura optimizada.

Además de estos prompts, lo que también hago es lanzarle al agente asignaciones reales que espero que activen la skill, e inspeccionar su proceso de razonamiento. Intercambio mensajes con él para entender por qué eligió (o no) una skill determinada. Eso revela muchos más problemas que cualquier prompt de validación aislado.

### 2. Validación de lógica

Le paso al LLM el `SKILL.md` completo y el árbol de directorios:

> Aquí está el borrador completo de mi SKILL.md y el árbol de archivos de soporte.
>
> ```
> ├── SKILL.md
> ├── scripts/esbuild-optimizer.mjs
> └── assets/vite.config.template.ts
> ```
>
> [Pego el contenido de SKILL.md]
>
> Actúa como un agente autónomo que acaba de activar esta skill. Simula tu ejecución paso a paso ante la petición "Migra mi app Angular v17 a Vite".
>
> Para cada paso, escribe tu monólogo interno:
> 1. ¿Qué estás haciendo exactamente?
> 2. ¿Qué archivo o script estás leyendo o ejecutando?
> 3. Señala cualquier bloqueo de ejecución: indica la línea exacta donde te ves obligado a adivinar porque las instrucciones son ambiguas.

### 3. Testing de casos extremos

Le pido al LLM que ataque mi lógica:

> Ahora cambia de rol. Actúa como un QA implacable. Tu objetivo es romper esta skill.
> Hazme entre 3 y 5 preguntas específicas y desafiantes sobre casos extremos, estados de error o fallbacks que faltan en el SKILL.md. Por ejemplo:
>
> - ¿Qué pasa si `scripts/esbuild-optimizer.mjs` falla por una dependencia CommonJS legada?
> - ¿Qué pasa si el `angular.json` del usuario tiene builders Webpack muy personalizados que Vite no soporta?
> - ¿Hay suposiciones implícitas sobre el entorno Node del usuario?
>
> No arregles nada todavía. Solo hazme las preguntas numeradas y espera mis respuestas.

### 4. Refinamiento de arquitectura

Con las respuestas de los pasos anteriores, le pido que reescriba la skill aplicando Progressive Disclosure:

> Basándote en mis respuestas a las preguntas de casos extremos, reescribe el SKILL.md aplicando estrictamente el patrón de Progressive Disclosure:
>
> 1. Mantén `SKILL.md` como un conjunto de pasos de alto nivel usando comandos imperativos en tercera persona.
> 2. Si hay reglas densas, plantillas grandes o esquemas complejos en el archivo, elimínalos. Dime que cree un nuevo archivo en `references/` o `assets/`, y reemplaza el texto en `SKILL.md` con un comando estricto para leer ese archivo solo cuando sea necesario.
> 3. Añade una sección de gestión de errores al final incorporando mis respuestas sobre los fallbacks de Webpack y la resolución de CommonJS.

---

## Conclusion

Crear skills para agentes es una disciplina diferente a escribir código o documentación. El objetivo no es que un humano lo entienda, sino que el agente lo ejecute de forma **determinista y eficiente**.

Las tres cosas que más impacto han tenido en la calidad de mis skills:

1. **Frontmatter con negative triggers**: evita falsos positivos.
2. **Progressive Disclosure**: el agente solo carga lo que necesita, cuando lo necesita.
3. **Validar con LLMs antes de dar la skill por buena**: simular la ejecución con el mismo tipo de modelo que la va a usar es la forma más honesta de encontrar ambigüedades.

Si estás empezando con esto, mi recomendación es que empieces con una skill simple, la valides bien con el proceso de arriba, y luego escales. Vale mucho más una skill de 100 líneas bien escrita que diez skills de 1.000 líneas llenas de ruido.
