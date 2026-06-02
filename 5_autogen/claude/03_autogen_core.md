# Módulo 03 · AutoGen Core: agentes que se pasan mensajes

[← AgentChat avanzado](02_agentchat_avanzado.md) · [Índice](README.md) · [Siguiente: Distribuido + el mundo →](04_distribuido_y_el_mundo.md)

---

## 🎯 Objetivo

Bajar a la capa de bajo nivel. Aquí no construyes "un agente" sino la **infraestructura de comunicación** entre agentes: el modelo de actores. Verás `RoutedAgent`, `@message_handler`, el `AgentId` y el **runtime** que entrega los mensajes.

> 📓 **Corresponde a:** `5_autogen/3_lab3_autogen_core.ipynb` (Week 5 · Day 3).

---

## 🧠 La idea: separar la lógica del agente de cómo viajan los mensajes

El principio fundamental de Core:

> 🧠 **Core desacopla la lógica del agente de la entrega de mensajes.** El framework te da la *infraestructura de comunicación* (el **runtime**) y el ciclo de vida de los agentes; cada agente solo se ocupa de **su** trabajo cuando le llega un mensaje.

Es el **modelo de actores**: muchas entidades independientes ("actores") que no se llaman entre sí directamente, sino que **se mandan mensajes** a través de un cartero central. ¿Por qué molestarse? Porque ese desacople es lo que permite, mañana, repartir los agentes por **varias máquinas** sin cambiar su lógica (Módulo 4).

---

## 💻 Pieza 1 · El mensaje

Tú defines la forma del mensaje. Lo más simple, un `dataclass`:

```python
from dataclasses import dataclass

@dataclass
class Message:
    content: str
```

> 💡 A diferencia de AgentChat (que trae `TextMessage`), en Core **el mensaje es tuyo**: defines la estructura que quieras. Core es agnóstico.

---

## 💻 Pieza 2 · El agente (`RoutedAgent`)

Un agente de Core hereda de `RoutedAgent` y marca con `@message_handler` el método que reacciona a mensajes:

```python
from autogen_core import RoutedAgent, MessageContext, message_handler

class SimpleAgent(RoutedAgent):
    def __init__(self) -> None:
        super().__init__("Simple")

    @message_handler
    async def on_my_message(self, message: Message, ctx: MessageContext) -> Message:
        return Message(content=f"This is {self.id.type}-{self.id.key}. You said '{message.content}' and I disagree.")
```

- **`@message_handler`** marca el método que recibe mensajes. Cuando le llegue un `Message`, se ejecuta y devuelve otro `Message`.
- **`self.id`** es el `AgentId` del agente, con dos partes:
  - `self.id.type` → qué **clase** de agente es (p. ej. "simple_agent").
  - `self.id.key` → su **identificador único** (p. ej. "default").

> 💡 El `AgentId` (type + key) es como "puesto + número de empleado". Permite tener varias instancias del mismo tipo de agente, cada una con su `key`. El cartero (runtime) usa el `AgentId` para saber a quién entregar.

---

## 💻 Pieza 3 · El runtime (el cartero)

El runtime entrega los mensajes y gestiona el ciclo de vida. El standalone, local:

```python
from autogen_core import SingleThreadedAgentRuntime, AgentId

runtime = SingleThreadedAgentRuntime()
await SimpleAgent.register(runtime, "simple_agent", lambda: SimpleAgent())   # registra el tipo

runtime.start()                                            # arranca el cartero

agent_id = AgentId("simple_agent", "default")
response = await runtime.send_message(Message("Well hi there!"), agent_id)
print(">>>", response.content)

await runtime.stop()                                       # para y cierra
await runtime.close()
```

El ciclo de vida, paso a paso:
1. **`register`**: le dices al runtime "existe un tipo de agente llamado `simple_agent`, créalo con esta función". Aún no hay instancia; das la *receta*.
2. **`start()`**: el cartero empieza a procesar mensajes en segundo plano.
3. **`send_message(msg, agent_id)`**: mandas un mensaje a un agente concreto (por su `AgentId`). El runtime lo crea si hace falta, le entrega el mensaje, y te devuelve la respuesta.
4. **`stop()` / `close()`**: paras y limpias.

---

## 💻 Combinar Core con AgentChat (el patrón "delegate")

Aquí está el truco más útil: un agente de Core que, por dentro, **usa un `AssistantAgent` de AgentChat** para pensar. Lo mejor de las dos capas:

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.messages import TextMessage
from autogen_ext.models.openai import OpenAIChatCompletionClient

class MyLLMAgent(RoutedAgent):
    def __init__(self) -> None:
        super().__init__("LLMAgent")
        model_client = OpenAIChatCompletionClient(model="gpt-4o-mini")
        self._delegate = AssistantAgent("LLMAgent", model_client=model_client)   # 👈 el "cerebro"

    @message_handler
    async def handle_my_message_type(self, message: Message, ctx: MessageContext) -> Message:
        text_message = TextMessage(content=message.content, source="user")
        response = await self._delegate.on_messages([text_message], ctx.cancellation_token)
        return Message(content=response.chat_message.content)
```

Léelo así: el `RoutedAgent` se ocupa de la **mensajería** (recibir/responder en el runtime), y delega el **razonamiento** a un `AssistantAgent` guardado en `self._delegate`. Core pone las tuberías; AgentChat pone la inteligencia.

### Tres agentes conversando

Registrando varios tipos, puedes orquestar una conversación pasando la respuesta de uno como mensaje al siguiente:

```python
runtime = SingleThreadedAgentRuntime()
await SimpleAgent.register(runtime, "simple_agent", lambda: SimpleAgent())
await MyLLMAgent.register(runtime, "LLMAgent", lambda: MyLLMAgent())
runtime.start()

response = await runtime.send_message(Message("Hi there!"), AgentId("LLMAgent", "default"))
response = await runtime.send_message(Message(response.content), AgentId("simple_agent", "default"))
response = await runtime.send_message(Message(response.content), AgentId("LLMAgent", "default"))
```

El mensaje rebota: LLMAgent → simple_agent → LLMAgent. Tú orquestas el "ping-pong" enviando mensajes. (En el Módulo 4 los agentes se mandarán mensajes **entre ellos** sin que tú medies cada paso.)

---

## ⚠️ Errores comunes

- **Olvidar `runtime.start()`.** Sin arrancar el cartero, los mensajes no se procesan.
- **No parar el runtime.** `stop()` + `close()` al terminar, o quedan recursos colgando.
- **Confundir `register` con crear el agente.** `register` da la *receta* (el `lambda`); la instancia la crea el runtime cuando hace falta.
- **`AgentId` equivocado.** El `type` debe coincidir con el nombre que usaste en `register`; el `key` suele ser `"default"`.
- **Esperar que Core sea "fácil" como AgentChat.** Es a propósito de más bajo nivel: ganas control y escalado a cambio de más código.

---

## 🧪 Pruébalo tú

1. **Crea un tercer tipo de agente** (otra personalidad) y mételo en el ping-pong de mensajes.
2. **Cambia el `_delegate`** de un agente a Ollama y comprueba que la mensajería no cambia.
3. **Imprime `self.id.type` y `self.id.key`** dentro de los handlers para ver quién recibe cada mensaje.

---

## 📌 Para llevar

- Core **desacopla la lógica del agente de la entrega de mensajes**: tú escribes agentes, el **runtime** (cartero) entrega los mensajes. Es el modelo de actores.
- Un agente de Core: hereda de **`RoutedAgent`** y marca con **`@message_handler`** el método que reacciona a mensajes. Su **`AgentId`** = `type` + `key`.
- Ciclo del runtime: **`register`** (receta) → **`start()`** → **`send_message`** → **`stop()`/`close()`**.
- Patrón **delegate**: un `RoutedAgent` hace la mensajería y delega el razonamiento a un `AssistantAgent` de AgentChat (`self._delegate`).
- Este desacople es lo que permite, en el Módulo 4, **distribuir** los agentes por la red.

---

[← AgentChat avanzado](02_agentchat_avanzado.md) · [Índice](README.md) · [Siguiente: Distribuido + el mundo →](04_distribuido_y_el_mundo.md)
