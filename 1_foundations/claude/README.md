# 🧱 Curso Guiado: Fundamentos (sin framework todavía)

> Reescritura de la **Semana 1** del curso, en formato guiado para **leerse y entenderse sin video**.
> Yo (Claude) hago de profesor: el *qué*, el *cómo* y el *por qué*.

---

## 👋 Por qué esta semana es la más importante

Aquí **no** usamos ningún framework de agentes. Y es a propósito. Antes de que CrewAI, LangGraph o el OpenAI Agents SDK te escondan la maquinaria, quiero que la veas desnuda: una llamada a un LLM, una lista de mensajes, un bucle, un `if`. Todo lo demás del curso son **azucarillos** encima de esto.

> 💡 Si entiendes de verdad esta semana, las otras cinco te parecerán "ah, esto es lo de la Semana 1 pero más bonito". Esa es la idea.

---

## 🗺️ El mapa del curso

```
   Semana 1: de "llamar a un LLM" a "un agente que usa herramientas"

   1 llamada  ──▶  muchos modelos  ──▶  un chatbot   ──▶  herramientas +
   simple          + un juez          con memoria        notificaciones
      │               │                 y autocrítica       (¡un agente!)
   Módulo 1        Módulo 2           Módulo 3            Módulo 4
```

| # | Módulo | Aprenderás a... | Concepto estrella |
|---|--------|-----------------|-------------------|
| [00](00_introduccion.md) | **Introducción** | Entender qué es (y qué no) un LLM | Mensajes y roles |
| [01](01_primera_llamada.md) | **Tu primera llamada** | Llamar a un LLM y encadenar llamadas | El cliente OpenAI |
| [02](02_muchos_modelos_y_el_juez.md) | **Muchos modelos y un juez** | Comparar LLMs y que uno juzgue a los otros | LLM as a judge |
| [03](03_chatbot_personal_y_evaluador.md) | **Chatbot personal + evaluador** | Un chatbot con Gradio que se autocorrige | Patrón evaluator-optimizer |
| [04](04_tools_y_notificaciones.md) | **Tools y notificaciones** | Dar herramientas al LLM y desplegar la app | Function calling + el bucle |
| [05](05_patrones_y_chuleta.md) | **Patrones y chuleta** | Tener todo a mano | Glosario + cheat sheet |

---

## ✅ Requisitos previos

1. **Entorno montado** (desde la raíz del repo): `uv sync`
2. **`.env`** en la raíz del repo. Esta semana el curso usa **OpenRouter** (un único proxy que te da acceso a muchos modelos con una sola clave) y/o OpenAI:
   ```
   OPENROUTER_API_KEY=...     # recomendado: una clave, muchos modelos
   OPENAI_API_KEY=sk-...      # usado por app.py y algunos labs
   ```
   Opcionales según el módulo (te avisaré):
   ```
   PUSHOVER_USER=...          # Módulo 4 (notificaciones al móvil)
   PUSHOVER_TOKEN=...
   ```
   Y opcionalmente **Ollama** para correr modelos locales gratis (Módulo 2).
3. **Kernel** `.venv (Python 3.12)` en tus notebooks.

> 🔍 ¿Algo no arranca? `setup/diagnostics.py` y `setup/troubleshooting.ipynb` en la raíz son tu primera parada.

---

## 🧭 Una nota sobre OpenRouter

Verás mucho esto:

```python
openai = OpenAI(base_url="https://openrouter.ai/api/v1", api_key=openrouter_api_key)
```

Es la **librería de OpenAI apuntando a OpenRouter**. ¿Por qué? Porque OpenRouter habla el mismo idioma que OpenAI pero te deja usar GPT, Claude, Gemini, DeepSeek, Llama... con **una sola clave y una sola API**. Aprenderás un patrón que se repite TODO el curso: *misma librería, distinta `base_url`*.

Empecemos. → [Módulo 00: Introducción](00_introduccion.md)

---

<sub>📚 Basado en la Semana 1 de *"Master AI Agentic Engineering"* de Edward Donner, reescrito y ampliado en formato guiado por Claude.</sub>
