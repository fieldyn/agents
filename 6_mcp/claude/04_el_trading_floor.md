# Módulo 04 · El trading floor: el capstone que junta todo

[← El ecosistema MCP](03_el_ecosistema_mcp.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)

---

## 🎯 Objetivo

El gran final del curso entero. Construyes un **parqué de bolsa autónomo**: cuatro traders-agente, cada uno con su personalidad y estrategia, que **investigan el mercado, deciden y operan solos**, en bucle, sobre la pila de servidores MCP de los módulos anteriores. Con un dashboard en vivo.

> 📓 **Corresponde a:** `6_mcp/4_lab4.ipynb`, `5_lab5.ipynb`, `traders.py`, `trading_floor.py`, `app.py` (Week 6 · Days 4-5).
> ⚠️ Se ejecuta con `uv run trading_floor.py` y `uv run app.py` (no en notebook). Necesita la pila de servidores MCP del Módulo 3.

---

## 🧠 La arquitectura

Cada trader es un agente que tiene **dos niveles de agentes anidados** y **dos juegos de servidores MCP**:

```
   ┌───────────────────── TRADER (Warren, George, Ray, Cathie) ────────────────────┐
   │  servidores MCP: accounts · push · market                                     │
   │                                                                               │
   │   herramienta ▶  ┌──────────── RESEARCHER (sub-agente como tool) ──────────┐  │
   │                  │  servidores MCP: fetch · brave-search · memory          │  │
   │                  │  investiga el mercado y devuelve un informe             │  │
   │                  └─────────────────────────────────────────────────────────┘  │
   └───────────────────────────────────────────────────────────────────────────────┘

   trading_floor.py:  ejecuta los 4 traders EN PARALELO, cada N minutos
   app.py:            dashboard Gradio con la cartera de cada trader en vivo
```

Reconoce los patrones de todo el curso: **agente-como-herramienta** (Semana 2), **paralelismo**, **structured/resources**, **múltiples modelos**, **scheduler**. Todo confluye aquí.

---

## 💻 Pieza 1 · El Researcher como herramienta

El researcher es un agente con sus propios servidores MCP (web, búsqueda, memoria). Pero al trader no se lo damos como "agente": se lo damos **como una tool** (el truco `.as_tool()` de la Semana 2):

```python
async def get_researcher(mcp_servers, model_name) -> Agent:
    return Agent(name="Researcher", instructions=researcher_instructions(),
                 model=get_model(model_name), mcp_servers=mcp_servers)

async def get_researcher_tool(mcp_servers, model_name) -> Tool:
    researcher = await get_researcher(mcp_servers, model_name)
    return researcher.as_tool(tool_name="Researcher", tool_description=research_tool())
```

> 💡 **Composición de agentes:** el trader no sabe *cómo* investiga el researcher (qué servidores usa); solo lo llama como "Researcher" y recibe un informe. Encapsulación, como en buen software. Un agente dentro de otro.

---

## 💻 Pieza 2 · El Trader

El trader junta su researcher-tool con sus propios servidores MCP (cuentas, mercado, push):

```python
class Trader:
    def __init__(self, name, lastname="Trader", model_name="gpt-4o-mini"):
        self.name = name; self.lastname = lastname; self.model_name = model_name
        self.do_trade = True

    async def create_agent(self, trader_mcp_servers, researcher_mcp_servers) -> Agent:
        tool = await get_researcher_tool(researcher_mcp_servers, self.model_name)
        self.agent = Agent(
            name=self.name,
            instructions=trader_instructions(self.name),
            model=get_model(self.model_name),
            tools=[tool],                       # 👈 el researcher, como herramienta
            mcp_servers=trader_mcp_servers,     # 👈 cuentas + mercado + push
        )
        return self.agent
```

Y lee su cuenta y estrategia **vía resources MCP** (Módulo 2) antes de operar:

```python
    async def get_account_report(self) -> str:
        account = await read_accounts_resource(self.name)   # resource: datos de la cuenta
        ...

    async def run_agent(self, trader_mcp_servers, researcher_mcp_servers):
        self.agent = await self.create_agent(trader_mcp_servers, researcher_mcp_servers)
        account = await self.get_account_report()
        strategy = await read_strategy_resource(self.name)   # resource: su estrategia
        message = trade_message(...) if self.do_trade else rebalance_message(...)
        await Runner.run(self.agent, message, max_turns=MAX_TURNS)
```

Fíjate en `self.do_trade`: alterna entre **operar** y **rebalancear** en cada ciclo (al final de `run` hace `self.do_trade = not self.do_trade`). El agente no hace siempre lo mismo.

---

## 💻 Pieza 3 · Gestionar muchos servidores con `AsyncExitStack`

En el Módulo 1 anidábamos `async with`. Con **seis** servidores (tres del trader, tres del researcher) eso sería ilegible. La solución elegante es `AsyncExitStack`:

```python
async def run_with_mcp_servers(self):
    async with AsyncExitStack() as stack:
        trader_mcp_servers = [
            await stack.enter_async_context(MCPServerStdio(params, client_session_timeout_seconds=120))
            for params in trader_mcp_server_params
        ]
        async with AsyncExitStack() as stack:
            researcher_mcp_servers = [
                await stack.enter_async_context(MCPServerStdio(params, client_session_timeout_seconds=120))
                for params in researcher_mcp_server_params(self.name)
            ]
            await self.run_agent(trader_mcp_servers, researcher_mcp_servers)
```

`AsyncExitStack` arranca **una lista** de servidores y garantiza que **todos** se cierren al final, sin anidar mil `with`. Patrón imprescindible cuando manejas N recursos dinámicos.

---

## 💻 Pieza 4 · Múltiples modelos (opcional)

Como en la Semana 1/3: cada trader puede usar un LLM distinto. `get_model` enruta al proveedor según el nombre, vía endpoints compatibles con OpenAI:

```python
def get_model(model_name: str):
    if "/" in model_name:       return OpenAIChatCompletionsModel(model=model_name, openai_client=openrouter_client)
    elif "deepseek" in model_name: return OpenAIChatCompletionsModel(model=model_name, openai_client=deepseek_client)
    elif "grok" in model_name:  return OpenAIChatCompletionsModel(model=model_name, openai_client=grok_client)
    elif "gemini" in model_name: return OpenAIChatCompletionsModel(model=model_name, openai_client=gemini_client)
    else: return model_name      # OpenAI por defecto
```

Con `USE_MANY_MODELS=true`, los cuatro traders corren con GPT, DeepSeek, Gemini y Grok — una competición de modelos invirtiendo. Si no, los cuatro usan `gpt-4o-mini`.

---

## 💻 Pieza 5 · El scheduler (`trading_floor.py`)

El corazón que mantiene vivo el parqué: ejecuta los cuatro traders **en paralelo**, cada N minutos, mientras el mercado esté abierto:

```python
names = ["Warren", "George", "Ray", "Cathie"]
lastnames = ["Patience", "Bold", "Systematic", "Crypto"]   # ¡sus personalidades!

async def run_every_n_minutes():
    add_trace_processor(LogTracer())
    traders = create_traders()
    while True:
        if RUN_EVEN_WHEN_MARKET_IS_CLOSED or is_market_open():
            await asyncio.gather(*[trader.run() for trader in traders])   # 👈 los 4 a la vez
        else:
            print("Market is closed, skipping run")
        await asyncio.sleep(RUN_EVERY_N_MINUTES * 60)
```

- **`asyncio.gather(*[trader.run() ...])`**: el paralelismo de la Semana 2, ahora moviendo cuatro traders completos a la vez.
- **`is_market_open()`**: no opera con la bolsa cerrada (configurable con `RUN_EVEN_WHEN_MARKET_IS_CLOSED`).
- **`while True` + `sleep`**: un bucle infinito que late cada `RUN_EVERY_N_MINUTES`. Esto es un proceso **de larga duración**, no un script de usar y tirar.

> 💡 Los nombres y apellidos (Warren *Patience*, Cathie *Crypto*) son guiños a estilos de inversión reales. Cada estrategia vive en su cuenta y la lee por resource. Personalidad = system prompt + estrategia, como en todo el curso.

---

## 💻 Pieza 6 · El dashboard (`app.py`)

`app.py` es una UI Gradio que muestra, para cada trader, su **valor de cartera en el tiempo** (gráfica Plotly), sus posiciones y su registro de operaciones — leyendo de la base de datos donde los traders apuntan todo.

```sh
# En una terminal: el motor que opera en bucle
uv run trading_floor.py

# En otra terminal: el panel para mirarlo en vivo
uv run app.py
```

Abre el dashboard y observa a los cuatro agentes investigar, comprar, vender y rebalancear **solos**, cada uno con su estilo, en tiempo real. Ese es el destino del viaje. 🎉

---

## 🏆 Lo que acabas de lograr (en seis semanas)

Para un momento. Mira todo lo que confluye en este capstone:

| De la Semana... | ...usaste aquí |
|---|---|
| **1 · Fundamentos** | El bucle de tools, múltiples modelos, push notifications |
| **2 · OpenAI SDK** | Agentes, `Runner`, trazas, **agente-como-herramienta** |
| **3 · CrewAI** | El `accounts.py` lo generó un equipo de software de agentes |
| **4 · LangGraph** | El patrón de agente que itera con un objetivo |
| **5 · AutoGen** | Tu primer contacto con MCP, sistemas multi-agente |
| **6 · MCP** | Servidores propios y de catálogo, resources, el ecosistema |

Has pasado de "llamar a un LLM con `2+2`" a **un sistema multi-agente autónomo, multi-modelo, con herramientas estandarizadas y un dashboard en vivo**. Eso es ingeniería agéntica de verdad.

---

## ⚠️ Errores comunes

- **No arranca el capstone en notebook.** Son procesos largos con red: `uv run trading_floor.py` + `uv run app.py`.
- **Faltan claves.** El trader necesita `POLYGON_API_KEY`, `BRAVE_API_KEY`, `PUSHOVER_*`, y `OPENAI_API_KEY`. Para muchos modelos, además DEEPSEEK/GOOGLE/GROK/OPENROUTER.
- **Servidores que tardan/cuelgan.** Seis servidores por trader es mucho; sube los timeouts (`client_session_timeout_seconds=120`) y revisa Node/uvx.
- **El mercado está cerrado y "no hace nada".** Es lo correcto. Para probar a cualquier hora: `RUN_EVEN_WHEN_MARKET_IS_CLOSED=true`.
- **Coste.** Cuatro agentes con sub-agentes investigadores, en bucle cada hora, suman. Sube `RUN_EVERY_N_MINUTES` y vigila el gasto. Las DBs de estado están en `.gitignore`.
- **Windows.** Imprescindible WSL.

---

## 🧪 Pruébalo tú

1. **Lánzalo con `RUN_EVEN_WHEN_MARKET_IS_CLOSED=true`** y `RUN_EVERY_N_MINUTES=5` para ver acción rápido. Mira el dashboard moverse.
2. **Cambia una estrategia:** edita la estrategia de un trader y observa cómo cambian sus decisiones en los siguientes ciclos.
3. **Activa `USE_MANY_MODELS=true`** y compara qué modelo "invierte" mejor con el tiempo.
4. **Sigue una traza** en platform.openai.com/traces: localiza al trader llamando al researcher, y al researcher usando sus servidores MCP.
5. **Añade un quinto trader** con tu propia personalidad y estrategia.

---

## 📌 Para llevar

- El capstone compone **agentes dentro de agentes**: cada trader usa un **researcher como herramienta** (`.as_tool()`), y cada uno tiene su propia pila de servidores MCP.
- Los traders leen su cuenta y estrategia por **resources MCP**, y operan con **tools MCP** (cuentas, mercado, push).
- **`AsyncExitStack`** arranca y cierra **muchos servidores** de forma limpia, sin anidar `with`.
- **`trading_floor.py`** es un proceso de **larga duración** (`while True` + `sleep`) que corre los traders en **paralelo** (`asyncio.gather`) cuando el mercado abre.
- El sistema es **multi-modelo** opcional, con un **dashboard Gradio** en vivo.
- Este proyecto reúne **las seis semanas** del curso: el destino del viaje.

---

[← El ecosistema MCP](03_el_ecosistema_mcp.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)
