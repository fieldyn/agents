# Módulo 01 · Usar servidores MCP existentes

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: Tu propio servidor y cliente →](02_tu_propio_servidor_y_cliente.md)

---

## 🎯 Objetivo

Lo más adictivo de MCP: enchufar **servidores ya hechos** a un agente y ver cómo, de repente, sabe navegar la web y manejar ficheros. Sin escribir ni una tool. Volvemos al OpenAI Agents SDK de la Semana 2.

> 📓 **Corresponde a:** `6_mcp/1_lab1.ipynb` (Week 6 · Day 1).

---

## 🧠 La idea

En tres pasos: (1) creas un cliente, (2) que **lanza** un servidor, (3) recoges las tools que ese servidor ofrece y se las das al agente. El SDK lo hace casi todo.

---

## 💻 Lanzar un servidor y ver sus tools

```python
from agents.mcp import MCPServerStdio

fetch_params = {"command": "uvx", "args": ["mcp-server-fetch"]}

async with MCPServerStdio(params=fetch_params, client_session_timeout_seconds=60) as server:
    fetch_tools = await server.list_tools()

print(fetch_tools)
```

Desglose:
- **`MCPServerStdio(params=...)`** crea un cliente que arranca el servidor descrito en `params` (aquí, `uvx mcp-server-fetch`, un servidor que descarga páginas web).
- **`async with ... as server:`** gestiona el ciclo de vida: arranca el subproceso al entrar, lo cierra al salir. Imprescindible: así no dejas procesos colgados.
- **`await server.list_tools()`** pregunta al servidor "¿qué sabes hacer?" y devuelve sus tools.

Cada "servidor" es solo una **receta para arrancar un programa**. Mira lo fácil que es sumar más, en distintos lenguajes:

```python
# Un navegador completo (JavaScript, vía npx)
playwright_params = {"command": "npx", "args": ["@playwright/mcp@latest"]}

# Acceso a ficheros, limitado a una carpeta "sandbox" (JavaScript)
import os
sandbox_path = os.path.abspath(os.path.join(os.getcwd(), "sandbox"))
files_params = {"command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", sandbox_path]}
```

> 💡 **Fíjate en el `sandbox_path`:** al servidor de ficheros le pasas **una sola carpeta** como argumento. Así el agente solo puede tocar ahí dentro. Es una barrera de seguridad: nunca le des a un agente acceso a todo tu disco.

---

## 💻 Darle los servidores a un agente

Aquí está la magia. Le pasas los servidores al `Agent` con `mcp_servers=[...]` y el SDK **expone automáticamente todas sus tools** al agente:

```python
from agents import Agent, Runner, trace

instructions = """
You browse the internet to accomplish your instructions.
You are highly capable at browsing independently, including accepting cookies and clicking
'not now' as appropriate. If one website isn't fruitful, try another. Be persistent.
When you need to write files, you do that inside the sandbox folder only.
"""

async with MCPServerStdio(params=files_params, client_session_timeout_seconds=60) as mcp_server_files:
    async with MCPServerStdio(params=playwright_params, client_session_timeout_seconds=60) as mcp_server_browser:
        agent = Agent(
            name="investigator",
            instructions=instructions,
            model="gpt-4.1-mini",
            mcp_servers=[mcp_server_files, mcp_server_browser],   # 👈 dos servidores a la vez
        )
        with trace("investigate"):
            result = await Runner.run(agent, "Find a great recipe for Banoffee Pie, then summarize it in markdown to banoffee.md")
            print(result.final_output)
```

Para y admira lo que acaba de pasar: con **cero tools escritas por ti**, el agente **navega la web de verdad**, busca una receta, y la **guarda en un fichero**. Toda esa capacidad vino de enchufar dos servidores. Eso es el poder de MCP.

> 💡 **Los `async with` anidados** mantienen vivos ambos servidores durante la ejecución del agente. Cuando el bloque termina, ambos se cierran limpiamente. En el capstone (Módulo 4) verás cómo gestionar **muchos** servidores con `AsyncExitStack` para no anidar tantos `with`.

---

## 🛒 Los marketplaces de MCP

¿De dónde salen estos servidores? De un ecosistema en plena explosión. Catálogos donde buscar servidores listos:

- **https://mcp.so**
- **https://glama.ai/mcp**
- **https://smithery.ai**

Antes de programar una integración, busca ahí: probablemente ya exista un servidor para lo que necesitas (GitHub, Slack, Notion, Postgres...).

---

## ⚠️ Errores comunes

- **La celda se "cuelga".** Casi siempre es Node/npx mal instalado o un servidor que tarda en arrancar. Revisa el final de `setup/SETUP-node.md` (troubleshooting). Sube `client_session_timeout_seconds`.
- **Windows.** Los servidores MCP no corren nativos; usa WSL.
- **Olvidar el `async with`.** Si no envuelves el servidor, no se arranca/cierra bien y quedan procesos colgados.
- **Error de "missing positional arguments" tras actualizar el SDK.** Si actualizaste OpenAI Agents SDK, puede haber un cambio: usa `await server.session.list_tools()` y `fetch_tools.tools`. (El repo está fijado a una versión que no lo necesita.)
- **El agente intenta escribir fuera del sandbox.** El servidor de ficheros solo permite la carpeta que le pasaste; refuérzalo también en las instrucciones.

---

## 🧪 Pruébalo tú

1. **Cambia la tarea:** pídele investigar otra cosa y guardarla en un `.md`. Mira el navegador trabajar y abre el fichero del sandbox.
2. **Quita el servidor de navegador** y deja solo el de ficheros. ¿Qué ya no puede hacer? Así ves qué capacidad aporta cada servidor.
3. **Busca un servidor nuevo** en mcp.so (p. ej. uno de tiempo/fecha) y enchúfalo a un agente.
4. **Mira la traza** en platform.openai.com/traces: verás las llamadas a las tools de cada servidor.

---

## 📌 Para llevar

- `MCPServerStdio(params={command, args})` crea un cliente que **lanza un servidor** como subproceso. Envuélvelo en `async with`.
- `await server.list_tools()` lista lo que el servidor ofrece.
- Pasas servidores a un agente con **`Agent(..., mcp_servers=[...])`**, y el SDK expone sus tools automáticamente — **sin escribir tools**.
- Limita el alcance (p. ej. el servidor de ficheros a una carpeta **sandbox**) por seguridad.
- Hay **marketplaces** (mcp.so, glama.ai, smithery.ai) llenos de servidores listos para enchufar.

---

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: Tu propio servidor y cliente →](02_tu_propio_servidor_y_cliente.md)
