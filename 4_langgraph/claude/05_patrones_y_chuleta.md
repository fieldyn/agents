# Módulo 05 · Patrones y chuleta (cheat sheet)

[← El Sidekick](04_el_sidekick.md) · [Volver al índice](README.md)

---

## 🎯 Para qué sirve

Tu página de referencia de LangGraph. Glosario, la receta de 5 pasos, y el código mínimo de cada cosa.

---

## 📖 Glosario rápido

| Término | En una frase |
|---|---|
| **State** | El objeto con los datos que viajan por el grafo (TypedDict o BaseModel). |
| **Reducer** | Cómo se combinan los cambios al estado; `add_messages` acumula mensajes. |
| **Node** | Una función `State -> cambios`. Puede usar (o no) un LLM. |
| **Edge** | Conexión entre nodos. `START` y `END` son los extremos. |
| **Conditional edge** | Arista que elige destino según el estado (permite ciclos). |
| **`StateGraph`** | El constructor del grafo. |
| **`compile()`** | Cierra el grafo y lo deja ejecutable. |
| **`bind_tools`** | Ata herramientas a un LLM (genera el esquema). |
| **`ToolNode`** | Nodo prefabricado que ejecuta las tools pedidas. |
| **`tools_condition`** | Routing prefabricado: ¿ir a tools o terminar? |
| **Checkpointer** | Persiste el estado (`MemorySaver` RAM / `SqliteSaver` disco). |
| **`thread_id`** | Identifica una conversación para la memoria. |
| **`with_structured_output`** | Fuerza la salida del LLM a un modelo Pydantic. |
| **LangSmith** | Trazas/observabilidad de LangGraph (langsmith.com). |

---

## 🧩 Los patrones de esta semana

| Patrón | Dónde |
|---|---|
| **Grafo lineal** | Los 5 pasos (Mód. 1) |
| **Bucle agéntico (LLM↔tools)** | chatbot + ToolNode + tools_condition (Mód. 2) |
| **Memoria por conversación** | checkpointer + thread_id (Mód. 3) |
| **Navegación web autónoma** | Playwright tools (Mód. 3) |
| **Worker/Evaluator (autocrítica + reintento)** | Sidekick (Mód. 4) |
| **Routing a medida** | worker_router, route_based_on_evaluation (Mód. 4) |

---

## 💻 La chuleta de código

### La receta de 5 pasos

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

# 1. State
class State(TypedDict):
    messages: Annotated[list, add_messages]

# 2. Builder
graph_builder = StateGraph(State)

# 3. Nodos
def mi_nodo(state: State):
    return {"messages": [...]}
graph_builder.add_node("mi_nodo", mi_nodo)

# 4. Aristas
graph_builder.add_edge(START, "mi_nodo")
graph_builder.add_edge("mi_nodo", END)

# 5. Compilar
graph = graph_builder.compile()
```

### Dibujar el grafo

```python
from IPython.display import Image, display
display(Image(graph.get_graph().draw_mermaid_png()))
```

### Nodo con LLM

```python
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini")

def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}
```

### Tools + bucle condicional

```python
from langgraph.prebuilt import ToolNode, tools_condition

llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools=tools))
graph_builder.add_conditional_edges("chatbot", tools_condition, "tools")
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
```

### Crear una tool

```python
from langchain.agents import Tool
mi_tool = Tool(name="nombre", func=mi_funcion, description="cuándo usarla")
```

### Memoria

```python
# RAM
from langgraph.checkpoint.memory import MemorySaver
graph = graph_builder.compile(checkpointer=MemorySaver())

# Disco
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver
conn = sqlite3.connect("memory.db", check_same_thread=False)
graph = graph_builder.compile(checkpointer=SqliteSaver(conn))

# Invocar con thread_id
config = {"configurable": {"thread_id": "usuario-1"}}
graph.invoke({"messages": [...]}, config=config)
```

### Structured output + routing a medida

```python
from pydantic import BaseModel, Field

class Decision(BaseModel):
    done: bool = Field(description="...")

llm_struct = llm.with_structured_output(Decision)

def router(state: State) -> str:
    return "END" if state["done"] else "worker"

graph_builder.add_conditional_edges("evaluator", router, {"worker": "worker", "END": END})
```

### Async

```python
result = await graph.ainvoke(state, config=config)   # en vez de graph.invoke
```

---

## 🐛 Tabla de "no me funciona"

| Síntoma | Causa probable | Arreglo |
|---|---|---|
| El chat "olvida" en una misma sesión | Falta reducer `add_messages` | `Annotated[list, add_messages]` |
| `KeyError`/`AttributeError` en el estado | TypedDict vs BaseModel | TypedDict: `state["x"]`; BaseModel: `state.x` |
| El LLM nunca usa tools | Falta `bind_tools` | `llm.bind_tools(tools)` |
| Tras la tool no continúa | Falta `add_edge("tools","chatbot")` | Cierra el ciclo |
| La memoria no recuerda | Falta `config`/`thread_id` | Pasa `config={"configurable": {"thread_id": ...}}` |
| Grafo no termina nunca | Routing sin salida a END | Añade condición a END (p. ej. `user_input_needed`) |
| Playwright `NotImplementedError` (Win) | Notebook + Windows | Mueve a módulo `.py` |
| El evaluador no decide | Falta `with_structured_output` | Fuerza salida Pydantic |
| Búsqueda falla | Falta `SERPER_API_KEY` | Añádela al `.env` |

---

## ✅ Checklist de "ya lo domino"

- [ ] Conozco la receta de 5 pasos (Mód. 1)
- [ ] Entiendo State, nodos, aristas y el reducer `add_messages` (Mód. 0-1)
- [ ] Sé que un nodo es una función y que el grafo no necesita LLMs (Mód. 1)
- [ ] Sé dibujar el grafo con `draw_mermaid_png` (Mód. 1)
- [ ] Sé montar el bucle LLM↔tools con `ToolNode` + `tools_condition` (Mód. 2)
- [ ] Sé dar memoria con checkpointer + `thread_id` (Mód. 3)
- [ ] Sé usar tools async y Playwright (Mód. 3)
- [ ] Sé el patrón worker/evaluator y el routing a medida (Mód. 4)
- [ ] Sé forzar structured output para decidir el flujo (Mód. 4)
- [ ] Sé desplegar con Gradio Blocks + `gr.State` (Mód. 4)

Si marcaste todo: dominas LangGraph. 🎉

---

[← El Sidekick](04_el_sidekick.md) · [Volver al índice](README.md)
