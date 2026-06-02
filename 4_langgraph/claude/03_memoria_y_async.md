# Módulo 03 · Memoria y async (persistencia y navegar la web)

[← Tools y aristas condicionales](02_tools_y_aristas_condicionales.md) · [Índice](README.md) · [Siguiente: El Sidekick →](04_el_sidekick.md)

---

## 🎯 Objetivo

Dos capacidades que el Sidekick del próximo módulo necesita: **memoria persistente** (que el grafo recuerde entre llamadas) y **ejecución asíncrona** (para que el agente navegue webs reales con Playwright).

> 📓 **Corresponde a:** `4_langgraph/3_lab3.ipynb` (Week 4 · Day 4).

---

## 🧠 La idea

Hasta ahora cada `invoke` empezaba de cero. Para un agente útil quieres dos cosas más:
1. **Memoria:** que recuerde la conversación entre invocaciones, e incluso entre sesiones (guardada en disco).
2. **Async:** muchas tools (navegar la web) son asíncronas; necesitamos la versión `async` del grafo.

---

## 💻 Parte 1 · Memoria con *checkpointer*

LangGraph guarda el estado con un **checkpointer**: un objeto que persiste el estado tras cada paso. Lo conectas al compilar.

### Memoria en RAM (efímera)

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

### Memoria en disco (persiste entre sesiones)

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver

conn = sqlite3.connect("memory.db", check_same_thread=False)
sql_memory = SqliteSaver(conn)
graph = graph_builder.compile(checkpointer=sql_memory)
```

`SqliteSaver` guarda el estado en un fichero SQLite (`memory.db`). Cierras el notebook, lo reabres, y el grafo **recuerda**. (Esos `memory.db*` están en `.gitignore`: son estado, no código.)

### El `thread_id`: ¿memoria de quién?

Aquí está la clave de cómo funciona la memoria. Al invocar, pasas un **`thread_id`** que identifica la conversación:

```python
config = {"configurable": {"thread_id": "3"}}

def chat(user_input: str, history):
    result = graph.invoke({"messages": [{"role": "user", "content": user_input}]}, config=config)
    return result["messages"][-1].content
```

- El checkpointer guarda un estado **por `thread_id`**. Mismo `thread_id` = misma conversación que continúa. `thread_id` distinto = conversación nueva y separada.
- Por eso ya **no** tienes que reconstruir `[system] + history + [user]` a mano (como en la Semana 1): el checkpointer recuerda el historial por ti. Le pasas solo el mensaje nuevo.

> 💡 **Esta es la gran comodidad:** la "memoria que reenvías a mano" de la Semana 1 ahora la gestiona el checkpointer. Cambiar de usuario es cambiar el `thread_id`. Multiusuario gratis.

---

## 💻 Parte 2 · Async y navegar la web con Playwright

### ¿Por qué async?

Sync vs async, la chuleta:

| | Síncrono | Asíncrono |
|---|---|---|
| Invocar el grafo | `graph.invoke(state)` | `await graph.ainvoke(state)` |
| Ejecutar una tool | `tool.run(inputs)` | `await tool.arun(inputs)` |

Muchas herramientas potentes (un navegador real) son asíncronas por naturaleza. Por eso pasamos a la versión `a...` (ainvoke, arun).

### Playwright: un navegador de verdad

[Playwright](https://playwright.dev) controla un navegador Chromium real. LangChain trae un toolkit que lo expone como tools (navegar, extraer texto, hacer clic...):

```python
from langchain_community.agent_toolkits import PlayWrightBrowserToolkit
from langchain_community.tools.playwright.utils import create_async_playwright_browser

async_browser = create_async_playwright_browser(headless=False)   # headless=False = ves la ventana
toolkit = PlayWrightBrowserToolkit.from_browser(async_browser=async_browser)
tools = toolkit.get_tools()    # navigate_browser, extract_text, click_element, ...
```

Con `headless=False` **verás el navegador abrirse y moverse solo** mientras el agente trabaja. Espectacular la primera vez.

> ⚠️ **Requiere Node.js + Playwright instalados** (`setup/SETUP-node.md`, luego `playwright install`). Sin esto, falla.

### `nest_asyncio`: el parche para notebooks

```python
import nest_asyncio
nest_asyncio.apply()
```

Python solo permite un "event loop" de async a la vez, y los notebooks ya tienen uno corriendo. `nest_asyncio` parchea eso para poder anidar loops. Es un apaño habitual **solo en notebooks**; en los módulos `.py` del Sidekick no hará falta.

> ⚠️ **Usuarios de Windows:** Playwright en notebook puede dar `NotImplementedError`. El lab explica un parche (comentar un `else` en `kernelapp.py`), pero la solución limpia es **pasar a un módulo `.py`** — que es justo lo que hace el Sidekick en el Módulo 4. Si te atascas, salta directo allí.

---

## ⚠️ Errores comunes

- **Memoria que "no recuerda".** Olvidaste pasar el `config` con el `thread_id` en cada `invoke`, o usaste un `thread_id` distinto sin querer.
- **`SqliteSaver` y threads.** Necesita `check_same_thread=False` en la conexión SQLite.
- **`NotImplementedError` con Playwright en notebook (Windows).** Ver el aviso de arriba; lo más sano es mover a `.py`.
- **Olvidar `await`.** Con grafos/tools async, `graph.invoke` se queda corto: usa `await graph.ainvoke(...)`.
- **Playwright no instalado.** Instala Node + `playwright install`.

---

## 🧪 Pruébalo tú

1. **Memoria:** invoca dos veces con el mismo `thread_id` ("me llamo Ana" y luego "¿cómo me llamo?"). Luego cambia el `thread_id` y repite: el bot "nuevo" no te conoce.
2. **Persistencia:** usa `SqliteSaver`, cierra el kernel, reábrelo, recompila y pregunta algo de antes. Sigue recordando.
3. **Web:** con `headless=False`, pídele que busque algo y extraiga el texto de una página. Mira el navegador moverse solo.

---

## 📌 Para llevar

- La memoria se activa con un **checkpointer** al compilar: `MemorySaver` (RAM) o `SqliteSaver` (disco, persiste entre sesiones).
- El **`thread_id`** (en `config={"configurable": {"thread_id": ...}}`) identifica cada conversación; el checkpointer guarda el historial por ti — ya no lo reenvías a mano.
- Para tools/grafos asíncronos usa las versiones `a...`: `await graph.ainvoke(...)`, `await tool.arun(...)`.
- **Playwright** da tools de navegador real (requiere Node + `playwright install`); `headless=False` para verlo.
- `nest_asyncio.apply()` es un parche **solo para notebooks**; en módulos `.py` no hace falta.

---

[← Tools y aristas condicionales](02_tools_y_aristas_condicionales.md) · [Índice](README.md) · [Siguiente: El Sidekick →](04_el_sidekick.md)
