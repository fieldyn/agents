# Módulo 04 · El Sidekick: un co-trabajador con autocrítica

[← Memoria y async](03_memoria_y_async.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)

---

## 🎯 Objetivo

El capstone de LangGraph. Construyes **Sidekick**, un asistente personal que recibe una tarea **y un criterio de éxito**, trabaja con muchas herramientas (web, ficheros, Python, Wikipedia), y un **evaluador** decide si cumplió. Si no, **lo manda a reintentar**. Junta TODO: grafo con ciclos, tools, structured output, routing y una UI real.

> 📓 **Corresponde a:** `4_langgraph/4_lab4.ipynb`, `sidekick.py`, `sidekick_tools.py`, `app.py` (Week 4 · Day 5).

---

## 🧠 La idea: worker + evaluator

Dos roles dentro de un mismo grafo:
- **Worker:** hace el trabajo, usando tools, hasta que cree haber terminado (o tiene una pregunta).
- **Evaluator:** juzga el resultado del worker contra el **criterio de éxito** que diste. Si no se cumple, devuelve feedback y el worker **vuelve a intentarlo**.

```
   START ──▶ [ worker ] ──(¿pidió tool?)──▶ [ tools ] ──┐
                  │                                       │
                  │ (terminó)                             │ (vuelve)
                  ▼                                       │
            [ evaluator ] ◀─────────────────────────────-┘
                  │
        ¿criterio cumplido / hace falta el usuario?
            │ sí → END          │ no → vuelve a [ worker ]
```

Es el patrón **evaluator–optimizer** de la Semana 1, pero ahora como un grafo con dos ciclos (worker↔tools y evaluator→worker). Aquí LangGraph brilla: este flujo con bucles sería un lío con otros frameworks.

---

## 💻 El State: más rico que un chat

```python
from typing import Annotated, List, Any, Optional
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[List[Any], add_messages]
    success_criteria: str
    feedback_on_work: Optional[str]
    success_criteria_met: bool
    user_input_needed: bool
```

El estado ya no es solo `messages`. Lleva el **criterio de éxito**, el **feedback** del evaluador, y dos banderas de control (¿cumplido?, ¿hace falta el usuario?). **El estado es la memoria de trabajo del grafo:** cada nodo lee y escribe estos campos para coordinarse.

---

## 💻 El output estructurado del evaluador

El evaluador debe devolver una decisión, no prosa. Pydantic + structured output:

```python
from pydantic import BaseModel, Field

class EvaluatorOutput(BaseModel):
    feedback: str = Field(description="Feedback on the assistant's response")
    success_criteria_met: bool = Field(description="Whether the success criteria have been met")
    user_input_needed: bool = Field(description="True if more input is needed from the user, or the assistant is stuck")
```

```python
evaluator_llm = ChatOpenAI(model="gpt-4o-mini")
evaluator_llm_with_output = evaluator_llm.with_structured_output(EvaluatorOutput)
```

`with_structured_output(EvaluatorOutput)` obliga al LLM a rellenar esa estructura. Así el grafo puede leer `success_criteria_met` (un bool) para **decidir el routing**. Structured output = el puente entre el juicio del LLM y la lógica del grafo.

---

## 💻 Los nodos

### Worker

```python
def worker(self, state: State) -> Dict[str, Any]:
    system_message = f"""You are a helpful assistant that can use tools to complete tasks.
    You keep working until either you have a question for the user, or the success criteria is met.
    ...
    This is the success criteria:
    {state["success_criteria"]}
    Reply either with a question for the user, or with your final response."""

    if state.get("feedback_on_work"):
        system_message += f"""
    Previously your reply was rejected because the success criteria was not met.
    Here is the feedback: {state["feedback_on_work"]}
    Please continue, ensuring you meet the success criteria."""

    # ...inserta/actualiza el system message en messages...
    response = self.worker_llm_with_tools.invoke(messages)
    return {"messages": [response]}
```

Lo elegante: si hay `feedback_on_work` en el estado (porque el evaluador lo rechazó antes), el worker lo **incorpora a su system prompt** y reintenta mejor. El estado transporta el feedback de una vuelta a la siguiente.

### Evaluator

```python
def evaluator(self, state: State) -> State:
    last_response = state["messages"][-1].content
    # ...arma user_message con la conversación, el criterio y la última respuesta...
    eval_result = self.evaluator_llm_with_output.invoke(evaluator_messages)
    return {
        "messages": [{"role": "assistant", "content": f"Evaluator Feedback: {eval_result.feedback}"}],
        "feedback_on_work": eval_result.feedback,
        "success_criteria_met": eval_result.success_criteria_met,
        "user_input_needed": eval_result.user_input_needed,
    }
```

El evaluador escribe en el estado las tres cosas que el routing necesita: feedback, ¿cumplido?, ¿hace falta el usuario?

---

## 💻 El routing (las dos decisiones del grafo)

Aquí está la lógica de control, en dos funciones de routing **a medida** (más allá del `tools_condition` del Módulo 2):

```python
def worker_router(self, state: State) -> str:
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"        # el worker pidió una tool
    else:
        return "evaluator"    # el worker cree haber terminado → a evaluar

def route_based_on_evaluation(self, state: State) -> str:
    if state["success_criteria_met"] or state["user_input_needed"]:
        return "END"          # listo, o necesitamos al humano
    else:
        return "worker"       # no cumple → reintentar
```

---

## 💻 Montar el grafo

```python
graph_builder = StateGraph(State)
graph_builder.add_node("worker", self.worker)
graph_builder.add_node("tools", ToolNode(tools=self.tools))
graph_builder.add_node("evaluator", self.evaluator)

graph_builder.add_conditional_edges("worker", self.worker_router, {"tools": "tools", "evaluator": "evaluator"})
graph_builder.add_edge("tools", "worker")
graph_builder.add_conditional_edges("evaluator", self.route_based_on_evaluation, {"worker": "worker", "END": END})
graph_builder.add_edge(START, "worker")

self.graph = graph_builder.compile(checkpointer=self.memory)
```

Lee las conexiones y reconoce los **dos ciclos**: worker↔tools (usar herramientas) y evaluator→worker (reintentar). Compilado con `checkpointer` (Módulo 3) para tener memoria. Este grafo es la síntesis de toda la semana.

---

## 💻 El arsenal de tools (`sidekick_tools.py`)

El worker es potente porque tiene muchas manos:

```python
async def playwright_tools():     # navegar la web (Módulo 3)
async def other_tools():
    # push notification + gestión de ficheros (sandbox) + búsqueda (Serper)
    # + PythonREPLTool (ejecutar Python) + Wikipedia
```

| Tool | Qué le permite |
|---|---|
| Playwright | Navegar y leer páginas web |
| FileManagementToolkit | Leer/escribir ficheros (en `sandbox/`) |
| Serper | Buscar en Google |
| PythonREPLTool | Ejecutar código Python |
| Wikipedia | Consultar Wikipedia |
| push | Avisarte al móvil |

Con esto, el Sidekick puede de verdad investigar, calcular, guardar resultados y avisarte.

---

## 💻 La app (`app.py`): UI con criterio de éxito

`app.py` usa **Gradio Blocks** (más flexible que `ChatInterface`) para una UI con dos cajas: el mensaje y el **criterio de éxito**.

```python
with gr.Blocks(title="Sidekick", theme=gr.themes.Default(primary_hue="emerald")) as ui:
    sidekick = gr.State(delete_callback=free_resources)   # guarda la instancia del Sidekick
    chatbot = gr.Chatbot(label="Sidekick", type="messages")
    message = gr.Textbox(placeholder="Your request to the Sidekick")
    success_criteria = gr.Textbox(placeholder="What are your success criteria?")
    # botones Reset y Go!
    ui.load(setup, [], [sidekick])                        # crea el Sidekick al cargar
    go_button.click(process_message, [sidekick, message, success_criteria, chatbot], [chatbot, sidekick])
```

Detalles que valen oro:
- **`gr.State(delete_callback=free_resources)`** guarda la instancia viva del Sidekick por usuario y llama a `free_resources` (que cierra el navegador Playwright) al limpiar. Gestión de recursos correcta.
- **El criterio de éxito es un campo de la UI:** el usuario dice no solo *qué* quiere, sino *cuándo está bien hecho*. Eso alimenta al evaluador.

Ejecútalo:

```sh
cd 4_langgraph
uv run app.py
```

Escribe una tarea ("encuentra los 3 mejores X y guárdalos en un fichero") y un criterio de éxito ("una lista con 3 items y sus precios"). Mira al worker trabajar (¡el navegador se abre solo!), al evaluador juzgar, y reintentar si hace falta.

---

## ⚠️ Errores comunes

- **El grafo no termina nunca.** Revisa `route_based_on_evaluation`: si el criterio nunca se marca como cumplido, gira sin fin. El campo `user_input_needed` es la válvula de escape.
- **Olvidar `with_structured_output`.** Sin él, el evaluador devuelve texto y el routing no puede leer `success_criteria_met`.
- **No cerrar el navegador.** Sin `free_resources`/`cleanup`, los procesos de Playwright quedan colgados. Por eso el `delete_callback`.
- **Criterio de éxito vago.** Si no defines bien "cuándo está hecho", el evaluador no puede juzgar. Sé concreto.
- **Estado mal leído.** Es `TypedDict`: `state["success_criteria"]`, no `state.success_criteria`.

---

## 🧪 Pruébalo tú

1. **Dale una tarea con criterio claro** ("resume esta web en 5 bullets" / criterio: "exactamente 5 bullets, cada uno < 20 palabras"). Observa si el evaluador rechaza una respuesta que no cumple.
2. **Pon un criterio imposible** y observa cómo `user_input_needed` evita el bucle infinito.
3. **Añade una tool** a `other_tools` (p. ej. una de hora/fecha) y compruébala en una tarea.
4. **Dibuja el grafo del Sidekick** (`draw_mermaid_png`) y localiza los dos ciclos.

---

## 📌 Para llevar

- El **patrón worker/evaluator**: un nodo trabaja (con tools), otro evalúa contra un **criterio de éxito**, y reintenta si no se cumple. El bucle es natural en LangGraph.
- El **State** transporta la memoria de trabajo: criterio, feedback, banderas de control. Los nodos se coordinan leyéndolo/escribiéndolo.
- **`with_structured_output(Modelo)`** hace que el evaluador devuelva datos (`success_criteria_met`) que el **routing** puede leer.
- Las funciones de **routing a medida** (`worker_router`, `route_based_on_evaluation`) dirigen el flujo según el estado.
- `app.py` con **Gradio Blocks** + `gr.State` da una UI real con criterio de éxito y limpieza de recursos (cerrar Playwright).

---

[← Memoria y async](03_memoria_y_async.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)
