# 🤖 Curso Guiado: Construyendo Agentes con el OpenAI Agents SDK

> Una reescritura de la Semana 2 del curso, pensada para **leerse y entenderse sin video**.
> Aquí yo (Claude) hago de profesor: te explico el *qué*, el *cómo* y —sobre todo— el *por qué*.

---

## 👋 Antes de empezar: cómo usar este curso

Este no es un resumen. Es el **curso completo en texto**, escrito para que puedas releer, subrayar y volver atrás cuando algo no te cuadre — eso que en un video se te escapa.

Cada módulo sigue siempre la misma estructura, para que sepas qué esperar:

| Sección | Qué encontrarás |
|---|---|
| 🎯 **Objetivo** | Qué sabrás hacer al terminar |
| 🧠 **La idea** | La intuición antes que el código |
| 💻 **El código, explicado** | Cada línea, con su *por qué* |
| ⚠️ **Errores comunes** | Las trampas en las que cae todo el mundo |
| 🧪 **Pruébalo tú** | Pequeños retos para fijar lo aprendido |
| 📌 **Para llevar** | Las 3-4 ideas que no debes olvidar |

> 💡 **Consejo de estudio:** lee el módulo entero una vez sin tocar el teclado. Luego vuelve al principio y ve ejecutando el código en un notebook a la vez que relees. Aprender se aprende haciendo, pero entender se entiende leyendo dos veces.

---

## 🗺️ El mapa del curso

```
                    OpenAI Agents SDK
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   1 agente           varios agentes      agentes que
   solo               colaborando         usan el mundo real
        │                  │                  │
   ┌────┴────┐      ┌───────┴───────┐    ┌─────┴─────┐
   │ Módulo 1│      │   Módulo 2    │    │ Módulo 4  │
   │ Tu      │      │ Tools +       │    │ Deep      │
   │ primer  │      │ Handoffs      │    │ Research  │
   │ agente  │      └───────┬───────┘    └───────────┘
   └─────────┘              │
                       ┌────┴────┐
                       │ Módulo 3│
                       │ Modelos,│
                       │ outputs │
                       │ guardr. │
                       └─────────┘
```

| # | Módulo | Aprenderás a... | Concepto estrella |
|---|--------|-----------------|-------------------|
| [00](00_introduccion.md) | **Introducción** | Pensar en "agentes" sin marearte | El bucle agéntico |
| [01](01_tu_primer_agente.md) | **Tu primer agente** | Crear y ejecutar un agente en 3 líneas | `Agent` · `Runner` · `trace` |
| [02](02_workflows_tools_handoffs.md) | **Workflows, Tools y Handoffs** | Hacer que varios agentes colaboren | `@function_tool` · `as_tool` · handoffs |
| [03](03_modelos_structured_outputs_guardrails.md) | **Modelos, Structured Outputs y Guardrails** | Cambiar de modelo, forzar formato y poner barreras de seguridad | Pydantic · `input_guardrail` |
| [04](04_deep_research.md) | **Deep Research** | Construir un investigador autónomo que busca en la web | `WebSearchTool` · pipeline |
| [05](05_patrones_y_chuleta.md) | **Patrones y chuleta** | Tener todo a mano de un vistazo | Glosario + cheat sheet |

---

## ✅ Requisitos previos

Antes de tocar el primer módulo, asegúrate de tener:

1. **El entorno montado** (desde la raíz del repo):
   ```sh
   uv sync
   ```
2. **Tu `.env`** en la raíz del repo con, como mínimo:
   ```
   OPENAI_API_KEY=sk-...
   ```
   Opcionales (los usaremos en módulos concretos, te avisaré):
   ```
   GOOGLE_API_KEY=...      # Módulo 3 (Gemini)
   DEEPSEEK_API_KEY=...    # Módulo 3 (DeepSeek)
   GROQ_API_KEY=...        # Módulo 3 (Llama 3.3)
   SENDGRID_API_KEY=...    # Módulos 2 y 4 (enviar emails) — OPCIONAL
   ```
3. **El kernel correcto** en tu notebook: selecciona `.venv (Python 3.12)`.

> 🔍 **¿Algo no arranca?** En la raíz del repo tienes `setup/diagnostics.py` y `setup/troubleshooting.ipynb`. Son tu primera parada ante cualquier error de entorno.

---

## 🧭 Filosofía de este curso

El OpenAI Agents SDK tiene una virtud que vamos a repetir hasta el cansancio: **es ridículamente ligero**. Donde antes escribías 50 líneas de JSON a mano para describir una herramienta, ahora pones un decorador. Donde antes orquestabas llamadas manualmente, ahora pasas una lista.

Mi trabajo aquí no es enseñarte a memorizar la API. Es que entiendas **el modelo mental**: una vez que "ves" cómo piensa un agente, el SDK se vuelve obvio.

Empecemos. → [Módulo 00: Introducción](00_introduccion.md)

---

<sub>📚 Material basado en la Semana 2 del curso *"Master AI Agentic Engineering"* de Edward Donner, reescrito y ampliado en formato guiado por Claude.</sub>
