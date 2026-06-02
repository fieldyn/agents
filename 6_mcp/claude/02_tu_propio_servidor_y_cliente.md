# Módulo 02 · Tu propio servidor y cliente MCP

[← Usar servidores MCP](01_usar_servidores_mcp.md) · [Índice](README.md) · [Siguiente: El ecosistema MCP →](03_el_ecosistema_mcp.md)

---

## 🎯 Objetivo

Pasar de consumidor a productor: **crear tu propio servidor MCP** (que expone una cuenta de trading) y tu propio **cliente** para usarlo. Usar servidores es facilísimo; crearlos lleva algo más de trabajo, pero es sorprendentemente poco.

> 📓 **Corresponde a:** `6_mcp/2_lab2.ipynb`, `accounts.py`, `accounts_server.py`, `accounts_client.py` (Week 6 · Day 2).

---

## 🧠 El dominio: una cuenta de trading

Antes del MCP, hay una clase Python normal, `accounts.py`, con la lógica de una cuenta: depositar, comprar/vender acciones, calcular el valor de la cartera, etc.

```python
from accounts import Account

account = Account.get("Ed")
account.buy_shares("AMZN", 3, "Because this bookstore website looks promising")
account.report()
account.list_transactions()
```

> 🔭 **Guiño al curso:** este `accounts.py` es **el mismo dominio** que generó el equipo de software de CrewAI en la Semana 3 (`3_crew/engineering_team`). Una pieza creada por agentes, que ahora envolvemos en MCP. Todo conecta.

Nuestro trabajo en este módulo: **exponer esa lógica como un servidor MCP**, para que cualquier agente pueda gestionar cuentas.

---

## 💻 Parte 1 · Crear el servidor con FastMCP

`FastMCP` hace que montar un servidor sea casi tan fácil como decorar funciones. Mira `accounts_server.py`:

```python
from mcp.server.fastmcp import FastMCP
from accounts import Account

mcp = FastMCP("accounts_server")        # 👈 crea el servidor

@mcp.tool()
async def get_balance(name: str) -> float:
    """Get the cash balance of the given account name.

    Args:
        name: The name of the account holder
    """
    return Account.get(name).balance

@mcp.tool()
async def buy_shares(name: str, symbol: str, quantity: int, rationale: str) -> float:
    """Buy shares of a stock.

    Args:
        name: The name of the account holder
        symbol: The symbol of the stock
        quantity: The quantity of shares to buy
        rationale: The rationale for the purchase and fit with the account's strategy
    """
    return Account.get(name).buy_shares(symbol, quantity, rationale)

# ...sell_shares, get_holdings, change_strategy igual...

if __name__ == "__main__":
    mcp.run(transport='stdio')          # 👈 arranca el servidor por stdio
```

Lo importante:
- **`@mcp.tool()`** convierte una función en una tool MCP. El **docstring y los type hints** se vuelven el esquema que verá el agente (igual que en toda la formación: la descripción importa).
- **`mcp.run(transport='stdio')`** arranca el servidor escuchando por entrada/salida estándar.

> 💡 Fíjate en el parámetro `rationale` de `buy_shares`: obliga al agente a **justificar** cada operación. Diseñar bien la *interfaz* de tus tools guía el comportamiento del agente — aquí, a que piense antes de operar.

### Resources: datos de solo lectura

Además de tools (acciones), el servidor expone **resources** (datos), direccionados por URI:

```python
@mcp.resource("accounts://accounts_server/{name}")
async def read_account_resource(name: str) -> str:
    return Account.get(name.lower()).report()

@mcp.resource("accounts://strategy/{name}")
async def read_strategy_resource(name: str) -> str:
    return Account.get(name.lower()).get_strategy()
```

- **`@mcp.resource("accounts://.../{name}")`** declara un recurso con una **URI** que lleva un parámetro (`{name}`). El agente "lee" `accounts://accounts_server/Ed` y recibe el informe de Ed.
- **Tool vs resource, otra vez:** `buy_shares` es una **acción** (cambia algo); `read_account_resource` es **información** (solo lee). El capstone usará los resources para que cada trader consulte su propia cuenta y estrategia.

### Usarlo desde un agente (lo del Módulo 1)

```python
params = {"command": "uv", "args": ["run", "accounts_server.py"]}
async with MCPServerStdio(params=params, client_session_timeout_seconds=30) as mcp_server:
    agent = Agent(name="account_manager", instructions="You manage a client's account.",
                  model="gpt-4.1-mini", mcp_servers=[mcp_server])
    result = await Runner.run(agent, "My name is Ed. What's my balance and holdings?")
```

¡Tu servidor, enchufado igual que los de catálogo! Acabas de cerrar el ciclo: **lo creaste y lo consumiste**.

---

## 💻 Parte 2 · Construir tu propio cliente MCP

Usar `MCPServerStdio` del SDK es lo cómodo. Pero para entender qué pasa por debajo, el lab construye un **cliente a mano** (`accounts_client.py`) con la librería MCP base:

```python
import mcp
from mcp.client.stdio import stdio_client
from mcp import StdioServerParameters

params = StdioServerParameters(command="uv", args=["run", "accounts_server.py"], env=None)

async def list_accounts_tools():
    async with stdio_client(params) as streams:
        async with mcp.ClientSession(*streams) as session:
            await session.initialize()
            tools_result = await session.list_tools()
            return tools_result.tools

async def call_accounts_tool(tool_name, tool_args):
    async with stdio_client(params) as streams:
        async with mcp.ClientSession(*streams) as session:
            await session.initialize()
            return await session.call_tool(tool_name, tool_args)

async def read_accounts_resource(name):
    async with stdio_client(params) as streams:
        async with mcp.ClientSession(*streams) as session:
            await session.initialize()
            result = await session.read_resource(f"accounts://accounts_server/{name}")
            return result.contents[0].text
```

El patrón de un cliente MCP, en bruto:
1. **`stdio_client(params)`** arranca el servidor y abre los canales de comunicación.
2. **`ClientSession(*streams)`** + **`session.initialize()`** establece la sesión MCP (el "apretón de manos").
3. Luego puedes **`list_tools()`**, **`call_tool(nombre, args)`** o **`read_resource(uri)`**.

Esto es lo que `MCPServerStdio` te hacía gratis. Verlo a mano te quita el misterio.

### Convertir tools MCP en tools de OpenAI

Y el puente final: tomar las tools del servidor MCP y envolverlas como `FunctionTool` del Agents SDK, para usarlas **sin** `mcp_servers=`:

```python
from agents import FunctionTool
import json

async def get_accounts_tools_openai():
    openai_tools = []
    for tool in await list_accounts_tools():
        schema = {**tool.inputSchema, "additionalProperties": False}
        openai_tool = FunctionTool(
            name=tool.name,
            description=tool.description,
            params_json_schema=schema,
            on_invoke_tool=lambda ctx, args, toolname=tool.name: call_accounts_tool(toolname, json.loads(args)),
        )
        openai_tools.append(openai_tool)
    return openai_tools
```

Cada tool MCP se traduce a una `FunctionTool` cuyo `on_invoke_tool` llama, por debajo, a `call_accounts_tool`. Es el "adaptador" entre el mundo MCP y el mundo del SDK — conceptualmente, lo mismo que el `LangChainToolAdapter` de la Semana 5.

---

## ⚠️ Errores comunes

- **El servidor no arranca.** Pruébalo solo: `uv run accounts_server.py`. Si peta ahí, el problema es del servidor, no del cliente.
- **URI de resource mal escrita.** Debe coincidir **exactamente** con el patrón del `@mcp.resource(...)`, incluidos los `{parámetros}`.
- **Olvidar `session.initialize()`.** Sin el apretón de manos, las llamadas fallan.
- **Docstrings/type hints flojos en `@mcp.tool()`.** El agente no sabrá usar bien la tool; son su única documentación.
- **Confundir tool y resource.** Acción que cambia estado → tool. Lectura de datos → resource.

---

## 🧪 Pruébalo tú (el ejercicio del lab)

1. **Tu primer servidor:** crea un `@mcp.tool()` que devuelva la fecha de hoy. Exponlo y pide a un agente que te diga qué día es.
2. **Difícil (opcional):** escribe un cliente MCP y usa tu tool con una **llamada nativa de OpenAI** (sin el Agents SDK), convirtiéndola como en `get_accounts_tools_openai`.
3. **Añade un resource** que exponga, por ejemplo, una lista de tareas, y haz que un agente la lea por su URI.

---

## 📌 Para llevar

- **`FastMCP`** + **`@mcp.tool()`** convierte funciones en un servidor MCP; `mcp.run(transport='stdio')` lo arranca. Docstring + type hints = el esquema.
- **`@mcp.resource("uri://{param}")`** expone datos de solo lectura, leídos por URI. **Tool = acción; resource = información.**
- Un **cliente MCP a mano**: `stdio_client` → `ClientSession` → `initialize()` → `list_tools` / `call_tool` / `read_resource`. Es lo que `MCPServerStdio` automatiza.
- Puedes **adaptar** tools MCP a `FunctionTool` del SDK (o a llamadas nativas) — un puente entre mundos.
- Diseñar bien la interfaz de las tools (p. ej. pedir un `rationale`) guía el comportamiento del agente.

---

[← Usar servidores MCP](01_usar_servidores_mcp.md) · [Índice](README.md) · [Siguiente: El ecosistema MCP →](03_el_ecosistema_mcp.md)
