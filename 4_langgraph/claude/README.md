# 🕸️ Curso Guiado: LangGraph — agentes como grafos de estado

> Reescritura de la **Semana 4** del curso, en formato guiado para **leerse y entenderse sin video**.
> Yo (Claude) hago de profesor: el *qué*, el *cómo* y el *por qué*.

---

## 👋 Otro cambio de filosofía

Cada framework de este curso tiene una "gran idea". La de LangGraph es:

> 🧠 **Modela tu agente como un GRAFO.** Hay un objeto de **estado** que fluye por **nodos** (funciones Python) conectados por **aristas** (edges). Algunas aristas son **condicionales**: el grafo decide por dónde ir según el estado.

¿Por qué un grafo? Porque te da **control explícito del flujo**, incluyendo **ciclos** (volver a un nodo anterior). Eso es justo lo que necesitas para "intenta → evalúa → si no, reintenta". CrewAI te ocultaba el flujo; LangGraph te lo pone en las manos, dibujable como un diagrama.

> 💡 Sorpresa del primer lab: **LangGraph no necesita un LLM**. El grafo es pura mecánica de Python; los LLMs son solo lo que pones *dentro* de los nodos. Entender esto te quita todo el misterio.

---

## 🗺️ El mapa del curso

```
   los 5 pasos       tools + aristas      memoria +        EL SIDEKICK
   (state, nodes,    condicionales        async + web      (worker/evaluator,
    edges, compile)  (bucle agéntico)     (persistencia)    co-trabajador real)
       │                  │                   │                  │
    Módulo 1           Módulo 2            Módulo 3           Módulo 4
```

| # | Módulo | Aprenderás a... | Concepto estrella |
|---|--------|-----------------|-------------------|
| [00](00_introduccion.md) | **Introducción** | Pensar en grafos de estado | State · Node · Edge |
| [01](01_los_cinco_pasos.md) | **Los 5 pasos** | Construir cualquier grafo | `StateGraph` · reducer · `compile` |
| [02](02_tools_y_aristas_condicionales.md) | **Tools y aristas condicionales** | Hacer el bucle agéntico como grafo | `bind_tools` · `ToolNode` · `tools_condition` |
| [03](03_memoria_y_async.md) | **Memoria y async** | Dar persistencia y navegar la web | `checkpointer` · `thread_id` · Playwright |
| [04](04_el_sidekick.md) | **El Sidekick** | Un co-trabajador con autocrítica | Patrón worker/evaluator |
| [05](05_patrones_y_chuleta.md) | **Patrones y chuleta** | Tenerlo todo a mano | Glosario + cheat sheet |

---

## ✅ Requisitos previos

1. **Entorno** (desde la raíz): `uv sync`
2. **`.env`** en la raíz:
   ```
   OPENAI_API_KEY=sk-...
   SERPER_API_KEY=...                  # Módulos 2-4: búsqueda web (serper.dev)
   PUSHOVER_USER=... PUSHOVER_TOKEN=...# Módulos 2-4: notificaciones
   LANGSMITH_API_KEY=...               # opcional: trazas (langsmith.com)
   ```
3. **Node.js + Playwright** para los Módulos 3-4 (el agente navega webs de verdad). Guía: `setup/SETUP-node.md`. Tras instalar Playwright, normalmente hace falta `playwright install`.
4. **Kernel** `.venv (Python 3.12)` en los notebooks.

> 🔭 **LangSmith** es el equivalente a las "trazas" de OpenAI, pero para LangChain/LangGraph: en https://langsmith.com ves cada paso del grafo. Opcional pero muy recomendable.

Empecemos. → [Módulo 00: Introducción](00_introduccion.md)

---

<sub>📚 Basado en la Semana 4 de *"Master AI Agentic Engineering"* de Edward Donner, reescrito y ampliado en formato guiado por Claude.</sub>
