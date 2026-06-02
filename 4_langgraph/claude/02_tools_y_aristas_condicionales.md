# Módulo 02 · Tools y aristas condicionales (el bucle agéntico, como grafo)

[← Los 5 pasos](01_los_cinco_pasos.md) · [Índice](README.md) · [Siguiente: Memoria y async →](03_memoria_y_async.md)

---

## 🎯 Objetivo

Dar herramientas a tu grafo y, con una **arista condicional**, construir el bucle agéntico (LLM ↔ tools) que ya montaste a mano en la Semana 1 — pero ahora como un grafo elegante y dibujable.

> 📓 **Corresponde a:** `4_langgraph/2_lab2.ipynb` (Week 4 · Day 3).

---

## 🧠 La idea

¿Recuerdas el bucle de la Semana 1? *Llama al LLM → ¿pidió una tool? → ejecútala → vuelve al LLM → repite*. En LangGraph eso es, literalmente, un grafo con un **ciclo**:

```
   START ──▶ [ chatbot ] ──(¿pidió tool?)──▶ [ tools ] ──┐
                  │                                       │
                  │ (no pidió tool)                       │
                  ▼                                       │
                 END          ◀───────────────────────────┘
                              (tras ejecutar la tool, vuelve al chatbot)
```

La arista que decide "¿voy a `tools` o termino?" es una **arista condicional**. LangGraph trae piezas hechas para esto, así que casi no escribes lógica.

---

## 💻 Las herramientas

LangChain te da tools de la comunidad y un envoltorio para las tuyas.

### Una tool de catálogo (búsqueda web)

```python
from langchain_community.utilities import GoogleSerperAPIWrapper
from langchain.agents import Tool

serper = GoogleSerperAPIWrapper()      # requiere SERPER_API_KEY
tool_search = Tool(
    name="search",
    func=serper.run,
    description="Useful for when you need more information from an online search",
)
```

### Una tool propia (push, ya conocida)

```python
import os, requests

def push(text: str):
    """Send a push notification to the user"""
    requests.post("https://api.pushover.net/1/messages.json",
                  data={"token": os.getenv("PUSHOVER_TOKEN"), "user": os.getenv("PUSHOVER_USER"), "message": text})

tool_push = Tool(name="send_push_notification", func=push,
                 description="useful for when you want to send a push notification")

tools = [tool_search, tool_push]
```

`Tool(name, func, description)` envuelve cualquier función. Igual que en toda la formación, la **description** es lo que el LLM lee para decidir cuándo usarla.

---

## 💻 El grafo con tools (los 5 pasos otra vez)

### State con TypedDict

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]
```

> 💡 **TypedDict vs. BaseModel:** en el Módulo 1 usamos `BaseModel`; aquí `TypedDict`. Ambos valen como State. `TypedDict` es un simple diccionario tipado (más ligero) y es lo más común en LangGraph. Detalle: con `TypedDict` accedes con `state["messages"]`; con `BaseModel`, `state.messages`.

### Atar las tools al LLM

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")
llm_with_tools = llm.bind_tools(tools)    # 👈 el LLM ahora "sabe" qué tools tiene
```

`bind_tools(tools)` es el equivalente a pasar el JSON de herramientas en la Semana 1 — pero automático. LangChain genera el esquema por ti.

### Los nodos: chatbot + tools

```python
from langgraph.prebuilt import ToolNode, tools_condition

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder = StateGraph(State)
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools=tools))    # 👈 nodo hecho que ejecuta tools
```

- El nodo `chatbot` llama al LLM (que puede pedir tools).
- **`ToolNode(tools=tools)`** es un nodo **prefabricado** que ejecuta las tools que el LLM pidió y mete los resultados en el estado. Es el `handle_tool_calls` de la Semana 1, regalado.

### La arista condicional (aquí está la magia)

```python
graph_builder.add_conditional_edges("chatbot", tools_condition, "tools")
graph_builder.add_edge("tools", "chatbot")   # tras ejecutar tools, vuelve al chatbot
graph_builder.add_edge(START, "chatbot")
```

Desglose:
- **`add_conditional_edges("chatbot", tools_condition, "tools")`**: después del `chatbot`, ejecuta la función `tools_condition`. Si el LLM pidió una tool, va a `"tools"`; si no, va a `END`. **`tools_condition` es una función prefabricada** que mira exactamente eso (el `finish_reason == "tool_calls"` que comprobabas a mano).
- **`add_edge("tools", "chatbot")`**: tras ejecutar la tool, **vuelve** al chatbot. Aquí está el **ciclo**.

```python
graph = graph_builder.compile()
display(Image(graph.get_graph().draw_mermaid_png()))
```

Dibújalo y verás el bucle chatbot↔tools con tus propios ojos. **Eso que en la Semana 1 era un `while` con `if finish_reason == "tool_calls"`, aquí es un grafo de 2 nodos y una arista condicional.** Mismo concepto, presentación distinta.

---

## 🔭 LangSmith: las trazas de LangGraph

Antes de ejecutar, regístrate en https://langsmith.com y pon `LANGSMITH_API_KEY` en tu `.env`. LangSmith registra cada paso del grafo (qué nodo, qué entró, qué salió, qué tool se llamó). Es el equivalente a las trazas de OpenAI de la Semana 2 — y con grafos que iteran, es oro para depurar.

---

## ⚠️ Errores comunes

- **Olvidar `bind_tools`.** Sin él, el LLM no sabe que tiene herramientas y nunca las pide.
- **No cerrar el ciclo.** Falta `add_edge("tools", "chatbot")` → tras la tool el grafo no sabe a dónde ir.
- **Mezclar acceso a State.** Con `TypedDict` es `state["messages"]`; no `state.messages`.
- **`SERPER_API_KEY` ausente.** La búsqueda falla. Regístrate gratis en serper.dev.
- **Esperar que `tools_condition` valga para todo.** Funciona para el caso estándar "¿pidió tool?"; para rutas a medida escribirás tu propia función de routing (Módulo 4).

---

## 🧪 Pruébalo tú

1. **Pídele algo que requiera buscar** ("¿qué pasó hoy en...") y mira en la consola/LangSmith cómo entra al nodo `tools`.
2. **Pídele que te mande un push** y comprueba que usa `send_push_notification`.
3. **Añade una tool más** (p. ej. una calculadora) y obsérvala aparecer en el grafo dibujado.
4. **Compara mentalmente** este grafo con tu bucle `while` de la Semana 1. ¿Ves que hacen lo mismo?

---

## 📌 Para llevar

- El **bucle agéntico** (LLM ↔ tools) es un grafo con un **ciclo**: `chatbot` → (condicional) → `tools` → `chatbot`.
- `llm.bind_tools(tools)` ata las herramientas al modelo (genera el esquema solo).
- **`ToolNode`** ejecuta las tools pedidas; **`tools_condition`** decide si ir a `tools` o terminar. Ambas vienen hechas en `langgraph.prebuilt`.
- `add_conditional_edges(origen, función, destino)` es cómo se bifurca el flujo según el estado.
- **LangSmith** (langsmith.com) te muestra cada paso del grafo — actívalo para depurar.

---

[← Los 5 pasos](01_los_cinco_pasos.md) · [Índice](README.md) · [Siguiente: Memoria y async →](03_memoria_y_async.md)
