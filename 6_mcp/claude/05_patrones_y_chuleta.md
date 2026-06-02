# Módulo 05 · Patrones y chuleta (cheat sheet)

[← El trading floor](04_el_trading_floor.md) · [Volver al índice](README.md)

---

## 🎯 Para qué sirve

Tu página de referencia de MCP — y el cierre de todo el curso. Glosario, código mínimo, y un mapa de las seis semanas.

---

## 📖 Glosario rápido

| Término | En una frase |
|---|---|
| **MCP** | Model Context Protocol: estándar para conectar agentes y herramientas. |
| **Host** | La app donde vive el agente. |
| **Cliente (Client)** | Lo que se conecta a un servidor y usa sus capacidades. |
| **Servidor (Server)** | Expone tools y resources. |
| **Tool** | Función que el agente **ejecuta** (acción). |
| **Resource** | Datos que el agente **lee**, por URI (información). |
| **`FastMCP`** | Forma fácil de crear un servidor (`@mcp.tool`, `@mcp.resource`). |
| **`MCPServerStdio`** | Cliente del SDK que lanza un servidor por stdio. |
| **stdio** | Transporte: el cliente lanza el servidor como subproceso. |
| **`mcp_servers=[...]`** | Pasar servidores a un `Agent` del SDK. |
| **`AsyncExitStack`** | Arrancar/cerrar muchos servidores de forma limpia. |
| **`as_tool()`** | Convertir un agente en herramienta de otro. |

---

## 🧩 Los patrones de esta semana

| Patrón | Dónde |
|---|---|
| **Consumir servidores MCP** | `mcp_servers=[...]` en el agente (Mód. 1) |
| **Crear un servidor MCP** | FastMCP + `@mcp.tool`/`@mcp.resource` (Mód. 2) |
| **Crear un cliente MCP** | stdio_client + ClientSession (Mód. 2) |
| **Tres tipos de servidor** | local-local / local-web / remoto (Mód. 3) |
| **Agente-como-herramienta** | researcher dentro del trader (Mód. 4) |
| **Muchos recursos a la vez** | AsyncExitStack (Mód. 4) |
| **Proceso de larga duración** | scheduler `while True` (Mód. 4) |

---

## 💻 La chuleta de código

### Usar un servidor en un agente

```python
from agents import Agent, Runner
from agents.mcp import MCPServerStdio

params = {"command": "uvx", "args": ["mcp-server-fetch"]}
async with MCPServerStdio(params=params, client_session_timeout_seconds=60) as server:
    agent = Agent(name="a", instructions="...", model="gpt-4.1-mini", mcp_servers=[server])
    result = await Runner.run(agent, "...")
```

### Crear un servidor (FastMCP)

```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("mi_server")

@mcp.tool()
async def mi_accion(x: str) -> str:
    """Qué hace y cuándo usarla.
    Args:
        x: ...
    """
    return "..."

@mcp.resource("miapp://datos/{id}")
async def mi_dato(id: str) -> str:
    return f"datos de {id}"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

### Crear un cliente (a mano)

```python
import mcp
from mcp.client.stdio import stdio_client
from mcp import StdioServerParameters

params = StdioServerParameters(command="uv", args=["run", "mi_server.py"], env=None)

async with stdio_client(params) as streams:
    async with mcp.ClientSession(*streams) as session:
        await session.initialize()
        tools = await session.list_tools()
        result = await session.call_tool("mi_accion", {"x": "hola"})
        data = await session.read_resource("miapp://datos/42")
```

### Muchos servidores con AsyncExitStack

```python
from contextlib import AsyncExitStack
async with AsyncExitStack() as stack:
    servers = [await stack.enter_async_context(MCPServerStdio(p, client_session_timeout_seconds=120))
               for p in lista_de_params]
    agent = Agent(name="a", instructions="...", model="...", mcp_servers=servers)
    await Runner.run(agent, "...")
```

### Recetas de servidor habituales

```python
{"command": "uvx", "args": ["mcp-server-fetch"]}                                   # descargar webs
{"command": "npx", "args": ["@playwright/mcp@latest"]}                             # navegador
{"command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", path]}# ficheros
{"command": "npx", "args": ["-y", "@modelcontextprotocol/server-brave-search"], "env": {...}}  # búsqueda
{"command": "uv",  "args": ["run", "mi_server.py"]}                                # tu servidor
```

---

## 🐛 Tabla de "no me funciona"

| Síntoma | Causa probable | Arreglo |
|---|---|---|
| Servidores MCP fallan (Windows) | No soportado nativo | Usa WSL (`setup/SETUP-WSL.md`) |
| La celda se cuelga | Node/npx o arranque lento | `setup/SETUP-node.md`; sube timeout |
| El servidor no arranca | Bug en el servidor | Pruébalo solo: `uv run mi_server.py` |
| `missing positional arguments` | Cambio de versión del SDK | `server.session.list_tools()` + `.tools` |
| Resource no encontrado | URI no coincide con el patrón | Igual al `@mcp.resource(...)`, con `{params}` |
| Brave/Polygon no van | Falta API key | Añade `BRAVE_API_KEY` / `POLYGON_API_KEY` (free) |
| Procesos colgados | Falta `async with` / ExitStack | Envuelve los servidores |
| Capstone no arranca | Lanzado en notebook | `uv run trading_floor.py` + `uv run app.py` |

---

## 🗺️ El curso entero, de un vistazo

```
   Sem 1  Fundamentos    →  el bucle de tools A MANO; el "por qué"
   Sem 2  OpenAI SDK     →  agentes ligeros, tools, handoffs, deep research
   Sem 3  CrewAI         →  equipos por configuración (YAML), jerarquía, memoria
   Sem 4  LangGraph      →  grafos de estado, ciclos, worker/evaluator
   Sem 5  AutoGen        →  AgentChat + Core (actores), distribuido
   Sem 6  MCP            →  el conector universal; el trading floor (capstone)
```

**El hilo conductor:** todos enseñan el mismo agente —LLM + tools + un bucle— con distinta ropa. Una vez lo "ves" (Semana 1), cada framework es una comodidad sobre lo mismo. Y MCP es la capa de herramientas que los une a todos.

> 📚 ¿Quieres repasar otra semana en este mismo formato? Cada unidad tiene su carpeta `claude/`:
> [1_foundations](../../1_foundations/claude/README.md) · [2_openai](../../2_openai/claude/README.md) · [3_crew](../../3_crew/claude/README.md) · [4_langgraph](../../4_langgraph/claude/README.md) · [5_autogen](../../5_autogen/claude/README.md)

---

## ✅ Checklist de "ya lo domino"

- [ ] Entiendo MCP: cliente, servidor, tools vs resources (Mód. 0)
- [ ] Sé enchufar servidores a un agente con `mcp_servers=[...]` (Mód. 1)
- [ ] Sé crear un servidor con FastMCP (`@mcp.tool`, `@mcp.resource`) (Mód. 2)
- [ ] Sé construir un cliente MCP a mano (Mód. 2)
- [ ] Conozco los tres tipos de servidor (Mód. 3)
- [ ] Sé componer agentes (researcher como tool del trader) (Mód. 4)
- [ ] Sé gestionar muchos servidores con AsyncExitStack (Mód. 4)
- [ ] Entiendo el scheduler de larga duración y el dashboard (Mód. 4)
- [ ] Veo cómo el capstone reúne las seis semanas (Mód. 4)

Si marcaste todo: **has completado el curso de Ingeniería Agéntica.** 🎓🎉

---

[← El trading floor](04_el_trading_floor.md) · [Volver al índice](README.md)
