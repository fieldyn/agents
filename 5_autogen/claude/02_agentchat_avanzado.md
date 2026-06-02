# Módulo 02 · AgentChat avanzado: equipos, multimodal y MCP

[← AgentChat básico](01_agentchat_basico.md) · [Índice](README.md) · [Siguiente: AutoGen Core →](03_autogen_core.md)

---

## 🎯 Objetivo

Cuatro capacidades nuevas de AgentChat: hablar con **imágenes** (multimodal), **structured outputs**, reusar **tools de LangChain**, y juntar agentes en **equipos** (teams). Y un adelanto del protocolo **MCP** (que será el plato fuerte de la Semana 6).

> 📓 **Corresponde a:** `5_autogen/2_lab2_autogen_agentchat.ipynb` (Week 5 · Day 2).

---

## 💻 Parte 1 · Conversación multimodal (texto + imagen)

AutoGen puede mandar imágenes al modelo, no solo texto:

```python
from io import BytesIO
import requests
from PIL import Image
from autogen_core import Image as AGImage
from autogen_agentchat.messages import MultiModalMessage

url = "https://edwarddonner.com/.../una-imagen.jpeg"
pil_image = Image.open(BytesIO(requests.get(url).content))
img = AGImage(pil_image)

multi_modal_message = MultiModalMessage(content=["Describe the content of this image in detail", img], source="User")
```

`MultiModalMessage` lleva una **lista** que mezcla texto e imágenes (`AGImage`). Se la pasas a un agente normal (con un modelo que entienda imágenes, como `gpt-4o-mini`) y la describe. Útil para análisis de documentos, capturas, fotos de productos...

---

## 💻 Parte 2 · Structured Outputs

Igual de fácil que en las otras semanas: defines la forma con Pydantic y se la das al agente como `output_content_type`:

```python
from pydantic import BaseModel, Field
from typing import Literal

class ImageDescription(BaseModel):
    scene: str = Field(description="Briefly, the overall scene of the image")
    message: str = Field(description="The point that the image is trying to convey")
    style: str = Field(description="The artistic style of the image")
    orientation: Literal["portrait", "landscape", "square"] = Field(description="The orientation")

describer = AssistantAgent(
    name="description_agent",
    model_client=model_client,
    system_message="You are good at describing images in detail",
    output_content_type=ImageDescription,    # 👈 la salida será este objeto
)

response = await describer.on_messages([multi_modal_message], cancellation_token=CancellationToken())
reply = response.chat_message.content        # ¡un ImageDescription, no texto!
print(reply.scene, reply.orientation)        # acceso tipado
```

Fíjate en `Literal["portrait", "landscape", "square"]`: además de tipar, **restringe** los valores posibles a esos tres. El modelo está obligado a elegir uno. Structured output con validación de opciones.

---

## 💻 Parte 3 · Reusar tools de LangChain

No tienes que reescribir tus tools de la Semana 4. AutoGen las **adapta** con `LangChainToolAdapter`:

```python
from autogen_ext.tools.langchain import LangChainToolAdapter
from langchain_community.utilities import GoogleSerperAPIWrapper
from langchain_community.agent_toolkits import FileManagementToolkit
from langchain.agents import Tool

# Tool de LangChain → tool de AutoGen
serper = GoogleSerperAPIWrapper()
langchain_serper = Tool(name="internet_search", func=serper.run, description="search the internet")
autogen_tools = [LangChainToolAdapter(langchain_serper)]

# Y un toolkit entero (gestión de ficheros)
for tool in FileManagementToolkit(root_dir="sandbox").get_tools():
    autogen_tools.append(LangChainToolAdapter(tool))

agent = AssistantAgent(name="searcher", model_client=model_client, tools=autogen_tools, reflect_on_tool_use=True)
```

> 💡 **Lección de ecosistema:** los frameworks no viven aislados. `LangChainToolAdapter` te deja traer todo el catálogo de tools de LangChain a AutoGen. Aprende a reusar entre frameworks; te ahorra reinventar la rueda.

---

## 💻 Parte 4 · Equipos (Teams)

Aquí está lo más jugoso del módulo. Un **team** hace que varios agentes conversen entre sí hasta cumplir una condición de fin. El más simple es el **round-robin** (por turnos):

```python
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import TextMentionTermination

primary_agent = AssistantAgent("primary", model_client=model_client, tools=[autogen_serper],
    system_message="You research flight deals. Incorporate any feedback you receive.")

evaluation_agent = AssistantAgent("evaluator", model_client=model_client,
    system_message="Provide constructive feedback. Respond with 'APPROVE' when your feedback is addressed.")

text_termination = TextMentionTermination("APPROVE")     # 👈 termina cuando alguien dice "APPROVE"

team = RoundRobinGroupChat([primary_agent, evaluation_agent],
                           termination_condition=text_termination,
                           max_turns=20)                 # 👈 red de seguridad

result = await team.run(task="Find a one-way non-stop flight from JFK to LHR in June 2025.")
for message in result.messages:
    print(f"{message.source}:\n{message.content}\n")
```

Lee la dinámica: el `primary` propone, el `evaluator` critica, el `primary` mejora con ese feedback, el `evaluator` revisa... y **cuando el evaluador dice "APPROVE", el team para**. Es el patrón **evaluator–optimizer** (Semana 1) pero implementado como una **conversación entre dos agentes por turnos**.

Dos piezas de control imprescindibles:
- **`TextMentionTermination("APPROVE")`**: la condición de parada — cuando aparece esa palabra, fin.
- **`max_turns=20`**: una red de seguridad. Sin ella, si nunca se aprueba, el equipo giraría sin fin. **Pon siempre un tope.**

> 💡 `RoundRobinGroupChat` reparte el turno en orden fijo. AutoGen tiene otros tipos de team (p. ej. selección por un modelo), pero el round-robin es el más claro para empezar.

---

## 💻 Parte 5 · Adelanto de MCP

Un anticipo de la Semana 6. **MCP (Model Context Protocol)**, de Anthropic, es un estándar para que los agentes usen herramientas que viven en **servidores** externos. AutoGen las consume casi como las de LangChain:

```python
from autogen_ext.tools.mcp import StdioServerParams, mcp_server_tools

fetch_mcp_server = StdioServerParams(command="uvx", args=["mcp-server-fetch"], read_timeout_seconds=30)
fetcher = await mcp_server_tools(fetch_mcp_server)     # tools servidas por el servidor MCP

agent = AssistantAgent(name="fetcher", model_client=model_client, tools=fetcher, reflect_on_tool_use=True)
result = await agent.run(task="Review edwarddonner.com and summarize what you learn. Reply in Markdown.")
```

`StdioServerParams(command="uvx", args=["mcp-server-fetch"])` arranca un servidor MCP (aquí, uno que descarga páginas web) y `mcp_server_tools(...)` te da sus herramientas. Por ahora quédate con la idea: **las tools pueden venir de servidores externos y estandarizados**. Lo desarrollamos en la Semana 6.

> ⚠️ **Windows:** los servidores MCP no corren nativamente; usa WSL (`setup/SETUP-WSL.md`).

---

## ⚠️ Errores comunes

- **Team sin `max_turns`.** Si la condición de fin no se cumple, bucle infinito. Pon siempre un tope.
- **`TextMentionTermination` que no salta.** La palabra debe aparecer **literal** en un mensaje. Si el evaluador dice "approved" en vez de "APPROVE", no termina. Sé exacto.
- **Modelo sin visión + multimodal.** `MultiModalMessage` necesita un modelo que entienda imágenes.
- **MCP en Windows.** Usa WSL.
- **Esperar texto con `output_content_type`.** La respuesta es un objeto Pydantic; accede a sus campos.

---

## 🧪 Pruébalo tú

1. **Pásale una captura tuya** y un `output_content_type` con campos a tu gusto (¿qué hay?, ¿qué colores?). Mira la salida estructurada.
2. **Cambia la condición de fin** del team a `max_turns=4` y observa cómo cambia el resultado al cortar antes.
3. **Añade un tercer agente** al `RoundRobinGroupChat` (un "moderador") y observa el turno rotar entre los tres.

---

## 📌 Para llevar

- **`MultiModalMessage`** mezcla texto e imágenes (`AGImage`) — conversaciones multimodales.
- **`output_content_type=Modelo`** fuerza structured output; `Literal[...]` restringe los valores posibles.
- **`LangChainToolAdapter`** reutiliza tools de LangChain en AutoGen — los ecosistemas se cruzan.
- Un **team** (`RoundRobinGroupChat`) hace que los agentes conversen por turnos hasta una **condición de fin** (`TextMentionTermination`). **Pon siempre `max_turns`.**
- **MCP** (adelanto): tools servidas por servidores externos estandarizados. Protagonista de la Semana 6.

---

[← AgentChat básico](01_agentchat_basico.md) · [Índice](README.md) · [Siguiente: AutoGen Core →](03_autogen_core.md)
