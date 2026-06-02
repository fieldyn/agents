# Módulo 01 · Los 5 pasos para construir un grafo

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: Tools y aristas condicionales →](02_tools_y_aristas_condicionales.md)

---

## 🎯 Objetivo

Aprender la **receta de 5 pasos** que sirve para construir CUALQUIER grafo en LangGraph. La vas a repetir tanto que se te quedará grabada. Y verás la prueba de que LangGraph es Python puro: el primer grafo no usa ningún LLM.

> 📓 **Corresponde a:** `4_langgraph/1_lab1.ipynb` (Week 4 · Day 2).

---

## 🧠 La receta de 5 pasos

Memorízala como un mantra. Todo grafo se construye así:

```
   1. Define el State        (qué datos viajan)
   2. Crea el StateGraph     (el constructor del grafo)
   3. Añade Nodos            (funciones que transforman el estado)
   4. Añade Aristas          (cómo se conectan)
   5. Compila                (lo cierras y queda listo)
```

Vamos paso a paso con el ejemplo más tonto posible (a propósito).

---

## 💻 Ejemplo 1 · Un grafo SIN LLM

### Paso 1 · Define el State

```python
from typing import Annotated
from langgraph.graph.message import add_messages
from pydantic import BaseModel

class State(BaseModel):
    messages: Annotated[list, add_messages]
```

`State` es el objeto que viaja por el grafo. Aquí solo lleva `messages`, con el reducer `add_messages` (Módulo 0): los mensajes se **acumulan**, no se pisan. Puedes usar un `BaseModel` de Pydantic o un `TypedDict` (más sobre esto en el Módulo 2).

### Paso 2 · Crea el constructor del grafo

```python
from langgraph.graph import StateGraph, START, END

graph_builder = StateGraph(State)
```

`StateGraph(State)` crea el "lienzo" donde añadiremos nodos y aristas. Le pasas tu clase `State` para que sepa qué datos maneja.

### Paso 3 · Añade un nodo

Un nodo es **cualquier función** que recibe el estado y devuelve uno nuevo:

```python
import random
nouns = ["Cabbages", "Unicorns", "Toasters", "Penguins", "Bananas"]
adjectives = ["outrageous", "smelly", "pedantic", "existential", "moody"]

def our_first_node(old_state: State) -> State:
    reply = f"{random.choice(nouns)} are {random.choice(adjectives)}"
    messages = [{"role": "assistant", "content": reply}]
    return State(messages=messages)

graph_builder.add_node("first_node", our_first_node)
```

Fíjate: **ni rastro de IA.** El nodo solo arma una frase absurda al azar. `add_node("first_node", our_first_node)` lo registra con un nombre. Ese nombre lo usaremos para conectarlo.

### Paso 4 · Añade las aristas

```python
graph_builder.add_edge(START, "first_node")
graph_builder.add_edge("first_node", END)
```

`START` y `END` son nodos especiales (el principio y el final). Esto dice: empieza → ve a `first_node` → termina. Un grafo lineal de un solo paso.

### Paso 5 · Compila

```python
graph = graph_builder.compile()

from IPython.display import Image, display
display(Image(graph.get_graph().draw_mermaid_png()))   # ¡dibújalo!
```

`compile()` cierra el grafo y lo deja listo para ejecutar. Y `draw_mermaid_png()` te lo **dibuja**: una de las gracias de LangGraph es que puedes *ver* tu grafo. Úsalo siempre para comprobar que conectaste lo que creías.

### Ejecutarlo

```python
import gradio as gr

def chat(user_input: str, history):
    state = State(messages=[{"role": "user", "content": user_input}])
    result = graph.invoke(state)
    return result["messages"][-1].content

gr.ChatInterface(chat, type="messages").launch()
```

`graph.invoke(state)` ejecuta el grafo con un estado inicial y devuelve el estado final. Escribas lo que escribas, contesta "Penguins are existential" o similar. Tonto, sí — pero **acabas de construir y ejecutar un grafo completo**.

> 💡 **¿Por qué empezar con algo tan bobo?** Para grabarte que **el grafo es mecánica pura**. Los nodos son funciones normales. Cuando metamos un LLM, será solo "una función que además llama a un modelo". Cero magia.

---

## 💻 Ejemplo 2 · El mismo grafo, ahora CON un LLM

Repetimos los 5 pasos, pero el nodo ahora llama a un modelo. Mira lo poco que cambia:

```python
from langchain_openai import ChatOpenAI

# Paso 1: State (igual)
class State(BaseModel):
    messages: Annotated[list, add_messages]

# Paso 2: builder
graph_builder = StateGraph(State)

# Paso 3: nodo... que ahora invoca un LLM
llm = ChatOpenAI(model="gpt-4o-mini")

def chatbot_node(old_state: State) -> State:
    response = llm.invoke(old_state.messages)
    return State(messages=[response])

graph_builder.add_node("chatbot", chatbot_node)

# Paso 4: aristas (igual)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)

# Paso 5: compilar (igual)
graph = graph_builder.compile()
```

La **única** diferencia real es el cuerpo del nodo: `llm.invoke(old_state.messages)` llama al modelo con el historial y devuelve su respuesta. Como `messages` usa el reducer `add_messages`, la respuesta se **añade** al estado automáticamente. Y ya tienes un chatbot funcional.

> 💡 `ChatOpenAI` es la clase de **LangChain** para hablar con modelos (LangGraph se apoya en LangChain). `llm.invoke(messages)` es el equivalente al `chat.completions.create` que usabas en la Semana 1.

---

## ⚠️ Errores comunes

- **Olvidar el reducer `add_messages`.** Sin `Annotated[list, add_messages]`, cada nodo *sobrescribe* los mensajes y pierdes el historial.
- **Nombres de nodo que no coinciden.** El string de `add_edge("chatbot", END)` debe ser exactamente el de `add_node("chatbot", ...)`.
- **Olvidar `compile()`.** El grafo no se puede `invoke` hasta compilarlo.
- **No dibujar el grafo.** `draw_mermaid_png()` te ahorra horas: ves al instante si una arista quedó suelta.
- **Acceder al estado mal.** Tras `invoke`, el resultado es como un dict: `result["messages"][-1].content`.

---

## 🧪 Pruébalo tú

1. **Añade un segundo nodo** entre `START` y el chatbot que meta un mensaje de sistema en el estado. Conecta START → setup → chatbot → END.
2. **Dibuja siempre.** Tras cada cambio, ejecuta `draw_mermaid_png()` y comprueba que el diagrama es lo que esperabas.
3. **Rompe algo a propósito:** quita el reducer `add_messages` y observa cómo el chat "olvida" lo anterior. Así interiorizas para qué sirve.

---

## 📌 Para llevar

- Todo grafo se construye con **5 pasos**: State → StateGraph → nodos → aristas → `compile()`.
- Un **nodo es una función** `State -> State` (o que devuelve los campos a actualizar). Puede o no usar un LLM.
- `START` y `END` son los extremos; las aristas (`add_edge`) conectan los nodos por su **nombre**.
- El **reducer** `add_messages` hace que los mensajes se **acumulen** en el estado.
- `graph.invoke(state)` ejecuta; `draw_mermaid_png()` **dibuja** el grafo (úsalo siempre).

---

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: Tools y aristas condicionales →](02_tools_y_aristas_condicionales.md)
