# Módulo 04 · Distribuido + el mundo de agentes que se crean a sí mismos

[← AutoGen Core](03_autogen_core.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)

---

## 🎯 Objetivo

El gran final de AutoGen, en dos partes. Primero, hacer que el runtime sea **distribuido** (los agentes se comunican por red, vía gRPC). Segundo, el capstone más alucinante del curso: un **"mundo"** donde un agente **Creator escribe el código de otros agentes**, los registra, y los 20 conversan entre sí generando ideas de negocio.

> 📓 **Corresponde a:** `5_autogen/4_lab4_autogen_distributed.ipynb` + `world.py`, `creator.py`, `agent.py`, `messages.py` (Week 5 · Day 4).
> ⚠️ Abre un puerto de red (`localhost:50051`). El capstone se ejecuta con `uv run world.py`, no en notebook.

---

## 🧠 La idea: del cartero local al cartero en red

En el Módulo 3, el runtime (`SingleThreadedAgentRuntime`) entregaba mensajes **dentro de un proceso**. Ahora cambiamos ese cartero por uno que entrega **por red**, usando **gRPC**:

```
                ┌──────── HOST gRPC (localhost:50051) ────────┐
                │            el "cartero central"             │
                └───────┬───────────────┬───────────────┬─────┘
                        │               │               │
                   WORKER 1         WORKER 2         WORKER 3
                  (agentes)        (agentes)        (agentes)
              (pueden estar en máquinas distintas)
```

Lo precioso: **la lógica de tus agentes (Módulo 3) no cambia**. Solo cambias el runtime. Ese era todo el sentido del desacople de Core.

---

## 💻 Parte 1 · Un runtime distribuido

```python
from autogen_ext.runtimes.grpc import GrpcWorkerAgentRuntimeHost, GrpcWorkerAgentRuntime

# El HOST: el cartero central que coordina (puede vivir en su propia máquina)
host = GrpcWorkerAgentRuntimeHost(address="localhost:50051")
host.start()

# Un WORKER: se conecta al host y aloja agentes
worker = GrpcWorkerAgentRuntime(host_address="localhost:50051")
await worker.start()
```

- **`GrpcWorkerAgentRuntimeHost`**: el cartero central. Escucha en `localhost:50051`.
- **`GrpcWorkerAgentRuntime`**: un trabajador que se conecta al host y aloja agentes. Puedes tener varios, en distintas máquinas, todos hablando con el mismo host.

El resto (registrar agentes, `send_message`, `AgentId`) es **idéntico** al Módulo 3. Solo cambió quién hace de cartero.

### El ejemplo del lab: Player1, Player2 y un Judge

Tres `RoutedAgent` (con el patrón delegate del Módulo 3): dos investigan los pros y los contras de usar AutoGen (con la tool de búsqueda), y un **Judge** los coordina mandándoles mensajes y sintetizando un veredicto:

```python
class Judge(RoutedAgent):
    @message_handler
    async def handle_my_message_type(self, message: Message, ctx: MessageContext) -> Message:
        # el Judge manda mensajes a los otros DOS agentes...
        response1 = await self.send_message(Message(content=instruction1), AgentId("player1", "default"))
        response2 = await self.send_message(Message(content=instruction2), AgentId("player2", "default"))
        # ...y combina sus respuestas en un juicio
        ...
```

Fíjate: el Judge usa **`self.send_message(...)`** para hablar con otros agentes **desde dentro**. Ya no eres tú quien orquesta cada paso (como en el Módulo 3): los agentes se coordinan **entre ellos**. Eso es un sistema multi-agente de verdad.

---

## 💻 Parte 2 · El capstone: un mundo de agentes auto-generados

Esto es lo más ambicioso del curso. La idea: un agente **Creator** que, dado un **template** de agente, **escribe el código Python de un agente nuevo** con personalidad distinta, lo guarda como `.py`, lo registra en el runtime, y lo pone a generar ideas. Repetido 20 veces.

### El template (`agent.py`)

Un agente "emprendedor creativo" con una personalidad concreta y, lo más interesante, la capacidad de **rebotar su idea a otro agente** para mejorarla:

```python
class Agent(RoutedAgent):
    system_message = """
    You are a creative entrepreneur. Your task is to come up with a new business idea using Agentic AI...
    Your personal interests are in these sectors: Healthcare, Education.
    You are drawn to ideas that involve disruption...
    """
    CHANCES_THAT_I_BOUNCE_IDEA_OFF_ANOTHER = 0.5

    @message_handler
    async def handle_message(self, message: messages.Message, ctx: MessageContext) -> messages.Message:
        response = await self._delegate.on_messages([TextMessage(content=message.content, source="user")], ctx.cancellation_token)
        idea = response.chat_message.content
        if random.random() < self.CHANCES_THAT_I_BOUNCE_IDEA_OFF_ANOTHER:
            recipient = messages.find_recipient()         # 👈 elige otro agente al azar
            message = f"Here is my business idea. Please refine it and make it better. {idea}"
            response = await self.send_message(messages.Message(content=message), recipient)
            idea = response.content                        # 👈 la idea, mejorada por otro
        return messages.Message(content=idea)
```

El `find_recipient()` (en `messages.py`) busca otros ficheros `agent*.py` y elige uno al azar. Así, con un 50% de probabilidad, cada agente **pide a un compañero que mejore su idea**. Emergencia: colaboración no planificada entre agentes que ni existían al arrancar.

### El Creator (`creator.py`)

Un agente cuyo trabajo es **escribir agentes**:

```python
class Creator(RoutedAgent):
    system_message = """
    You are an Agent that is able to create new AI Agents.
    You receive a template... create a new Agent with a unique system message...
    The class must be named Agent and inherit from RoutedAgent...
    Respond only with the python code, no markdown.
    """

    @message_handler
    async def handle_my_message_type(self, message: messages.Message, ctx: MessageContext) -> messages.Message:
        filename = message.content                                  # p. ej. "agent3.py"
        agent_name = filename.split(".")[0]
        response = await self._delegate.on_messages([TextMessage(content=self.get_user_prompt(), source="user")], ctx.cancellation_token)
        with open(filename, "w") as f:
            f.write(response.chat_message.content)                  # 👈 ¡escribe el .py!
        module = importlib.import_module(agent_name)                # 👈 lo importa
        await module.Agent.register(self.runtime, agent_name, lambda: module.Agent(agent_name))  # 👈 lo registra
        result = await self.send_message(messages.Message(content="Give me an idea"), AgentId(agent_name, "default"))
        return messages.Message(content=result.content)
```

Para en serio a digerir esto: el Creator **genera código Python con un LLM, lo escribe a disco, lo importa con `importlib`, lo registra como agente vivo en el runtime, y le pide una idea**. Un agente que crea y da vida a otros agentes. Metaprogramación agéntica.

### El mundo (`world.py`)

Orquesta los 20, en paralelo, sobre el runtime distribuido:

```python
HOW_MANY_AGENTS = 20

async def main():
    host = GrpcWorkerAgentRuntimeHost(address="localhost:50051")
    host.start()
    worker = GrpcWorkerAgentRuntime(host_address="localhost:50051")
    await worker.start()

    await Creator.register(worker, "Creator", lambda: Creator("Creator"))
    creator_id = AgentId("Creator", "default")

    # pide al Creator que cree y ponga a trabajar 20 agentes, en paralelo
    coroutines = [create_and_message(worker, creator_id, i) for i in range(1, HOW_MANY_AGENTS+1)]
    await asyncio.gather(*coroutines)
    # cada idea final se guarda en idea{i}.md
```

```sh
cd 5_autogen
uv run world.py     # tarda; genera agent1.py..agent20.py e idea1.md..idea20.md
```

Al terminar tienes 20 ficheros `agent*.py` (todos distintos, escritos por IA) y 20 `idea*.md` con ideas de negocio, muchas **refinadas colaborativamente** entre agentes. Un pequeño ecosistema que se construye y conversa solo.

> 💡 **Por qué este capstone cierra el curso:** combina todo —AgentChat (el cerebro), Core (la mensajería), runtime distribuido (la escala), tools, y generación de código— en un sistema que **se expande a sí mismo**. Es una probada de hacia dónde van los sistemas multi-agente.

---

## ⚠️ Errores comunes

- **Puerto `50051` ocupado.** Si quedó un host de una ejecución anterior, libéralo o reinicia. Un solo host a la vez.
- **No es para notebook.** El capstone usa procesos y red: `uv run world.py`.
- **Coste/tiempo.** 20 agentes que generan código y conversan = muchas llamadas. Baja `HOW_MANY_AGENTS` para probar.
- **Código generado inválido.** A veces el Creator produce un `.py` que no importa bien. Es el riesgo de generar código; el system prompt insiste "respond only with python code, no markdown".
- **Windows.** El runtime distribuido y MCP van mejor en WSL.

---

## 🧪 Pruébalo tú

1. **Empieza pequeño:** `HOW_MANY_AGENTS = 3`. Lee los `agent*.py` generados: ¿qué personalidades inventó el Creator?
2. **Cambia el template** (`agent.py`): otra personalidad base, otros sectores. ¿Cómo cambian las ideas?
3. **Sube la probabilidad** `CHANCES_THAT_I_BOUNCE_IDEA_OFF_ANOTHER` a 0.9 y observa más colaboración entre agentes.
4. **Lee los `idea*.md`** y localiza ideas que fueron claramente refinadas por más de un agente.

---

## 📌 Para llevar

- Un **runtime distribuido** (gRPC) cambia el "cartero local" por uno **en red**: un **host** central + **workers** que alojan agentes (posiblemente en varias máquinas). La lógica de los agentes **no cambia** — ese era el sentido del desacople de Core.
- Dentro de un handler, **`self.send_message(...)`** deja que los agentes se coordinen **entre ellos**, sin que tú medies cada paso.
- El capstone: un **Creator** genera código de agentes con un LLM, lo escribe, lo importa (`importlib`) y lo registra vivo en el runtime. Agentes que crean agentes.
- `world.py` lanza 20 agentes en paralelo que generan y **refinan ideas colaborativamente** — un ecosistema multi-agente auto-expandible.
- Se ejecuta con `uv run world.py` (abre `localhost:50051`), no en notebook.

---

[← AutoGen Core](03_autogen_core.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)
