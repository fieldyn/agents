# Módulo 05 · Patrones y chuleta (cheat sheet)

[← Deep Research](04_deep_research.md) · [Volver al índice](README.md)

---

## 🎯 Para qué sirve este módulo

Esto no es lección nueva: es tu **página de referencia**. Cuando termines el curso y un mes después quieras montar un agente sin releerlo todo, vienes aquí. Glosario, patrones, y código mínimo de cada cosa, todo de un vistazo.

---

## 📖 Glosario rápido

| Término | En una frase |
|---|---|
| **Agent** | LLM + instrucciones + (opcional) tools/handoffs. El "empleado". |
| **Runner** | El motor que ejecuta el bucle agéntico. `Runner.run(agent, prompt)`. |
| **Bucle agéntico** | pensar → ¿tool? → ejecutar → repensar → responder. |
| **Tool** | Función que el agente puede llamar; el control **vuelve** al agente. |
| **`@function_tool`** | Decorador que convierte una función Python en tool (sin JSON a mano). |
| **`.as_tool()`** | Convierte un **agente** en tool de otro agente. |
| **Handoff** | El agente **cede** el control a otro agente; el control **pasa**, no vuelve. |
| **`handoff_description`** | Cómo se "anuncia" un agente ante quienes pueden delegarle. |
| **Hosted tool** | Tool alojada por OpenAI: `WebSearchTool`, `FileSearchTool`, `ComputerTool`. |
| **Structured Output** | Salida con forma fija definida con Pydantic (`output_type=...`). |
| **Guardrail** | Control que intercepta entrada/salida y puede **abortar** (`tripwire_triggered`). |
| **Trace** | Registro inspeccionable de todo lo que hizo el agente. `with trace("..."):`. |
| **Streaming** | Recibir la salida token a token con `Runner.run_streamed`. |
| **Workflow** | El **orden lo decide tu código**. Predecible. |
| **Agente (autónomo)** | El **orden lo decide el modelo** con sus tools/handoffs. Flexible. |

---

## 🧩 Los patrones agénticos del curso

| Patrón | Qué es | Lo viste en |
|---|---|---|
| **Prompt chaining** | Encadenar pasos en orden fijo | Deep Research (Mód. 4) |
| **Paralelización** | Lanzar tareas independientes a la vez (`asyncio.gather`) | Mód. 2 y 4 |
| **LLM as a judge** | Un agente evalúa/elige la salida de otros | `sales_picker` (Mód. 2) |
| **Agente como herramienta** | Un agente expuesto como tool | `.as_tool()` (Mód. 2) |
| **Orquestador / sub-agentes** | Un jefe dirige especialistas | Sales Manager (Mód. 2) |
| **Handoff / delegación** | Ceder el control a otro agente | Sales Manager → Email Manager (Mód. 2) |
| **Guardrails** | Barreras de seguridad de entrada/salida | Mód. 3 |
| **Descomposición** | Partir una tarea grande en agentes especializados | Deep Research (Mód. 4) |

---

## 💻 La chuleta de código

### Crear y ejecutar un agente

```python
from dotenv import load_dotenv
from agents import Agent, Runner, trace

load_dotenv(override=True)

agent = Agent(name="MiAgente", instructions="Eres ...", model="gpt-4o-mini")

with trace("mi tarea"):
    result = await Runner.run(agent, "haz esto")
    print(result.final_output)
```

### Streaming (token a token)

```python
from openai.types.responses import ResponseTextDeltaEvent

result = Runner.run_streamed(agent, input="...")
async for event in result.stream_events():
    if event.type == "raw_response_event" and isinstance(event.data, ResponseTextDeltaEvent):
        print(event.data.delta, end="", flush=True)
```

### Paralelizar varios agentes

```python
import asyncio

results = await asyncio.gather(
    Runner.run(agent_a, msg),
    Runner.run(agent_b, msg),
    Runner.run(agent_c, msg),
)
outputs = [r.final_output for r in results]
```

### Una función como tool

```python
from agents import function_tool

@function_tool
def mi_tool(param: str) -> dict:
    """Descripción clara: el modelo la lee para decidir cuándo usarla."""
    ...
    return {"status": "ok"}
```

### Un agente como tool de otro

```python
tool = mi_agente.as_tool(tool_name="mi_agente", tool_description="qué hace")

jefe = Agent(name="Jefe", instructions="...", tools=[tool], model="gpt-4o-mini")
```

### Handoff (ceder el control)

```python
especialista = Agent(
    name="Especialista",
    instructions="...",
    handoff_description="qué hace, para que otros decidan delegarle",
    model="gpt-4o-mini",
)

jefe = Agent(
    name="Jefe",
    instructions="...",
    tools=[...],
    handoffs=[especialista],   # ceder control aquí
    model="gpt-4o-mini",
)
```

### Otro modelo (Gemini / DeepSeek / Groq…)

```python
from openai import AsyncOpenAI
from agents import OpenAIChatCompletionsModel

client = AsyncOpenAI(base_url="https://api.deepseek.com/v1", api_key=deepseek_api_key)
model  = OpenAIChatCompletionsModel(model="deepseek-chat", openai_client=client)

agent = Agent(name="...", instructions="...", model=model)   # igual que siempre
```

### Structured Output

```python
from pydantic import BaseModel, Field

class Salida(BaseModel):
    ok: bool
    motivo: str = Field(description="por qué")

agent = Agent(name="...", instructions="...", output_type=Salida, model="gpt-4o-mini")

result = await Runner.run(agent, "...")
result.final_output.ok        # 👈 booleano de verdad, no texto
```

### Guardrail de entrada

```python
from agents import input_guardrail, GuardrailFunctionOutput

@input_guardrail
async def mi_guardrail(ctx, agent, message):
    result = await Runner.run(detector_agent, message, context=ctx.context)
    return GuardrailFunctionOutput(
        output_info={"info": result.final_output},
        tripwire_triggered=result.final_output.ok,   # True = abortar
    )

agente_protegido = Agent(..., input_guardrails=[mi_guardrail])
```

### Hosted tool: búsqueda web

```python
from agents import WebSearchTool
from agents.model_settings import ModelSettings

search_agent = Agent(
    name="Buscador",
    instructions="...",
    tools=[WebSearchTool(search_context_size="low")],   # "low" = más barato
    model_settings=ModelSettings(tool_choice="required"),  # obliga a buscar
    model="gpt-4o-mini",
)
```

---

## 🧠 Cómo decidir: workflow o agente autónomo

```
   ¿Necesitas que el orden de pasos sea SIEMPRE el mismo y fácil de depurar?
        │
        ├── Sí ──▶ WORKFLOW: orquesta tú en código (funciones async, gather)
        │           ✅ predecible  ✅ fácil de depurar  ❌ rígido
        │
        └── No, quiero que se adapte ──▶ AGENTE AUTÓNOMO: dale tools/handoffs y un objetivo
                    ✅ flexible  ✅ maneja casos raros  ❌ menos predecible
```

Regla práctica: **empieza con workflow** (entiendes y controlas todo) y dale autonomía solo donde la rigidez te estorbe.

---

## 🐛 Tabla de "no me funciona"

| Síntoma | Causa probable | Arreglo |
|---|---|---|
| `ImportError` de `agents` | Kernel equivocado | Selecciona `.venv (Python 3.12)` |
| `API Key not set` | `.env` no cargado | `.env` en la raíz + `load_dotenv(override=True)` |
| `SyntaxError` en `await` | Estás en un `.py`, no en notebook | Usa `asyncio.run(...)` |
| El email no llega | Sender no verificado / Spam | Verifica sender en SendGrid; mira Spam |
| `SSL: CERTIFICATE_VERIFY_FAILED` | Certificados | `uv pip install --upgrade certifi` + `os.environ['SSL_CERT_FILE']=certifi.where()` |
| `final_output` no es texto | Pusiste `output_type=...` | Accede a sus campos (`.campo`) |
| Gemini falla | Falta clave | `GOOGLE_API_KEY` **y** `GEMINI_API_KEY` al mismo valor |
| Factura inesperada | `WebSearchTool` cuesta | Baja `HOW_MANY_SEARCHES`, usa `search_context_size="low"` |
| No entiendo qué hizo el agente | No miraste la traza | `with trace(...)` + platform.openai.com/traces |

---

## 🚀 Hacia dónde seguir

- 📚 **Docs oficiales del SDK** (claras y cortas): https://openai.github.io/openai-agents-python/
- 🔭 **Traces**: https://platform.openai.com/traces — vuélvelo un hábito.
- 🧱 El proyecto **`2_openai/deep_research/`** lleva el Módulo 4 a una app completa con interfaz. Ábrelo cuando quieras ver "esto en grande".
- 🗂️ **`community_contributions/`** está lleno de variantes de estudiantes (incluida la alternativa de email con Resend). Buena mina de ideas.
- ➡️ En el curso, las siguientes semanas exploran **otros frameworks** (CrewAI, LangGraph, AutoGen, MCP). Verás que, una vez entiendes los patrones de aquí, todos te suenan.

---

## ✅ Checklist de "ya lo domino"

Marca mentalmente cada uno. Si dudas, vuelve al módulo entre paréntesis:

- [ ] Sé crear y ejecutar un agente, y leer su `final_output` (Mód. 1)
- [ ] Entiendo el bucle agéntico (Mód. 0)
- [ ] Sé usar `trace` y leer una traza (Mód. 1)
- [ ] Sé paralelizar con `asyncio.gather` (Mód. 2)
- [ ] Sé crear tools con `@function_tool` y convertir agentes con `.as_tool()` (Mód. 2)
- [ ] Distingo **tool** (vuelve) de **handoff** (pasa) (Mód. 2)
- [ ] Sé cambiar de modelo con endpoints compatibles (Mód. 3)
- [ ] Sé forzar formato con structured outputs Pydantic (Mód. 3)
- [ ] Sé poner un guardrail que aborte (Mód. 3)
- [ ] Sé descomponer una tarea en agentes especializados (Mód. 4)
- [ ] Sé decidir entre workflow y agente autónomo (Mód. 4/5)

Si marcaste todo: enhorabuena, dominas el OpenAI Agents SDK. 🎉

---

[← Deep Research](04_deep_research.md) · [Volver al índice](README.md)
