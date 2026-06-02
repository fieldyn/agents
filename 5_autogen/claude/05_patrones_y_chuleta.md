# Módulo 05 · Patrones y chuleta (cheat sheet)

[← Distribuido + el mundo](04_distribuido_y_el_mundo.md) · [Volver al índice](README.md)

---

## 🎯 Para qué sirve

Tu página de referencia de AutoGen. Glosario de las dos capas y el código mínimo de cada cosa.

---

## 📖 Glosario rápido

| Término | En una frase |
|---|---|
| **AgentChat** | La capa alta: agentes y equipos, fácil. |
| **Core** | La capa baja: mensajería entre agentes (modelo actor). |
| **Model client** | El cliente del LLM (`OpenAIChatCompletionClient`, `OllamaChatCompletionClient`). |
| **`AssistantAgent`** | Agente de AgentChat (modelo + system message + tools). |
| **`on_messages`** | Ejecuta un AssistantAgent. |
| **`reflect_on_tool_use`** | El agente repiensa con el resultado de la tool. |
| **`MultiModalMessage`** | Mensaje con texto + imágenes. |
| **`output_content_type`** | Structured output (Pydantic). |
| **`LangChainToolAdapter`** | Reutiliza tools de LangChain. |
| **Team** | Agentes que conversan (`RoundRobinGroupChat`). |
| **Termination** | Condición de fin de un team (`TextMentionTermination`). |
| **`RoutedAgent`** | Agente de Core; reacciona a mensajes. |
| **`@message_handler`** | Método que recibe/responde mensajes en Core. |
| **`AgentId`** | Identidad de un agente: `type` + `key`. |
| **Runtime** | El "cartero": `SingleThreadedAgentRuntime` (local) o gRPC (distribuido). |
| **delegate** | Patrón: un `RoutedAgent` delega el razonamiento en un `AssistantAgent`. |

---

## 🧩 Los patrones de esta semana

| Patrón | Dónde |
|---|---|
| **Agente + tools** | AgentChat básico (Mód. 1) |
| **Multimodal** | Imágenes (Mód. 2) |
| **Structured outputs** | `output_content_type` (Mód. 2) |
| **Reuso entre frameworks** | `LangChainToolAdapter` (Mód. 2) |
| **Evaluator–optimizer (como team)** | RoundRobinGroupChat primary+evaluator (Mód. 2) |
| **Modelo de actores** | RoutedAgent + runtime (Mód. 3) |
| **Delegate (Core + AgentChat)** | `self._delegate` (Mód. 3) |
| **Distribución** | runtime gRPC (Mód. 4) |
| **Agentes que crean agentes** | Creator + world.py (Mód. 4) |

---

## 💻 La chuleta de código

### AgentChat: agente con tools

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.messages import TextMessage
from autogen_core import CancellationToken

model_client = OpenAIChatCompletionClient(model="gpt-4o-mini")

def mi_tool(x: str) -> str:
    """Qué hace y cuándo usarla"""
    return "..."

agent = AssistantAgent(name="a", model_client=model_client,
                       system_message="...", tools=[mi_tool], reflect_on_tool_use=True)

msg = TextMessage(content="...", source="user")
resp = await agent.on_messages([msg], cancellation_token=CancellationToken())
print(resp.chat_message.content)
```

### Structured output

```python
from pydantic import BaseModel, Field
from typing import Literal

class Salida(BaseModel):
    campo: str = Field(description="...")
    tipo: Literal["a", "b"] = Field(description="...")

agent = AssistantAgent(name="a", model_client=model_client, output_content_type=Salida)
resp = await agent.on_messages([msg], cancellation_token=CancellationToken())
resp.chat_message.content.campo     # objeto tipado
```

### Team (round-robin con fin)

```python
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import TextMentionTermination

team = RoundRobinGroupChat([agent_a, agent_b],
                           termination_condition=TextMentionTermination("APPROVE"),
                           max_turns=20)
result = await team.run(task="...")
for m in result.messages:
    print(f"{m.source}: {m.content}")
```

### Reusar una tool de LangChain

```python
from autogen_ext.tools.langchain import LangChainToolAdapter
from langchain.agents import Tool
autogen_tool = LangChainToolAdapter(Tool(name="search", func=fn, description="..."))
```

### Tools MCP

```python
from autogen_ext.tools.mcp import StdioServerParams, mcp_server_tools
params = StdioServerParams(command="uvx", args=["mcp-server-fetch"], read_timeout_seconds=30)
tools = await mcp_server_tools(params)
```

### Core: agente + runtime local

```python
from dataclasses import dataclass
from autogen_core import RoutedAgent, MessageContext, message_handler, SingleThreadedAgentRuntime, AgentId

@dataclass
class Message:
    content: str

class MiAgente(RoutedAgent):
    def __init__(self) -> None:
        super().__init__("MiTipo")
    @message_handler
    async def handle(self, message: Message, ctx: MessageContext) -> Message:
        return Message(content=f"recibí: {message.content}")

runtime = SingleThreadedAgentRuntime()
await MiAgente.register(runtime, "mi_agente", lambda: MiAgente())
runtime.start()
resp = await runtime.send_message(Message("hola"), AgentId("mi_agente", "default"))
await runtime.stop(); await runtime.close()
```

### Core: patrón delegate (Core + AgentChat)

```python
class LLMAgent(RoutedAgent):
    def __init__(self) -> None:
        super().__init__("LLM")
        self._delegate = AssistantAgent("LLM", model_client=OpenAIChatCompletionClient(model="gpt-4o-mini"))
    @message_handler
    async def handle(self, message: Message, ctx: MessageContext) -> Message:
        r = await self._delegate.on_messages([TextMessage(content=message.content, source="user")], ctx.cancellation_token)
        return Message(content=r.chat_message.content)
```

### Core: runtime distribuido (gRPC)

```python
from autogen_ext.runtimes.grpc import GrpcWorkerAgentRuntimeHost, GrpcWorkerAgentRuntime

host = GrpcWorkerAgentRuntimeHost(address="localhost:50051"); host.start()
worker = GrpcWorkerAgentRuntime(host_address="localhost:50051"); await worker.start()
# register / send_message igual que en local
```

---

## 🐛 Tabla de "no me funciona"

| Síntoma | Causa probable | Arreglo |
|---|---|---|
| Mensajes no se procesan | Falta `runtime.start()` | Arranca el runtime |
| Recursos colgados | No paraste el runtime | `stop()` + `close()` |
| Team en bucle infinito | Sin `max_turns` o termination que no salta | Pon `max_turns`; palabra exacta |
| El agente suelta dato crudo | `reflect_on_tool_use=False` | Ponlo en `True` |
| Puerto 50051 ocupado | Host previo vivo | Libera el puerto / reinicia |
| MCP falla (Windows) | No soportado nativo | Usa WSL (`setup/SETUP-WSL.md`) |
| `output_content_type` da texto | Mal acceso | Es objeto: `resp.chat_message.content.campo` |
| Multimodal falla | Modelo sin visión | Usa un modelo con visión |

---

## 🧠 ¿AgentChat o Core?

```
   ¿Quieres construir rápido un agente o un equipo pequeño?
        │
        ├── Sí ──▶ AutoGen AgentChat   (alto nivel, familiar)
        │
        └── Necesito muchos agentes, mensajería a medida o varias máquinas
                    ──▶ AutoGen Core   (modelo actor, runtime, distribuible)
```

---

## ✅ Checklist de "ya lo domino"

- [ ] Entiendo las dos capas: AgentChat vs Core (Mód. 0)
- [ ] Sé crear un AssistantAgent con tools y `on_messages` (Mód. 1)
- [ ] Sé multimodal, structured outputs y reusar tools de LangChain (Mód. 2)
- [ ] Sé montar un team con condición de fin y `max_turns` (Mód. 2)
- [ ] Entiendo MCP a alto nivel (Mód. 2)
- [ ] Sé el modelo de actores: RoutedAgent + message_handler + runtime (Mód. 3)
- [ ] Sé el patrón delegate (Core usa AgentChat) (Mód. 3)
- [ ] Entiendo el runtime distribuido por gRPC (Mód. 4)
- [ ] Entiendo el capstone: agentes que crean agentes (Mód. 4)

Si marcaste todo: dominas AutoGen. 🎉

---

[← Distribuido + el mundo](04_distribuido_y_el_mundo.md) · [Volver al índice](README.md)
