# Módulo 01 · AgentChat básico

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: AgentChat avanzado →](02_agentchat_avanzado.md)

---

## 🎯 Objetivo

Crear tu primer agente de AutoGen AgentChat y darle una **tool** (consultar precios de vuelos en una base de datos SQLite). Te resultará familiar tras las semanas anteriores; lo que cambia es la sintaxis.

> 📓 **Corresponde a:** `5_autogen/1_lab1_autogen_agentchat.ipynb` (Week 5 · Day 1).

---

## 💻 Los tres conceptos, en código

### 1 · El model client

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
model_client = OpenAIChatCompletionClient(model="gpt-4o-mini")
```

El cliente del modelo. ¿Quieres un modelo local? Cambias el cliente, no el resto:

```python
from autogen_ext.models.ollama import OllamaChatCompletionClient
ollama_client = OllamaChatCompletionClient(model="llama3.2")
```

> 💡 Mismo patrón de toda la formación: el agente no sabe qué LLM hay detrás. Cambiar de proveedor = cambiar el `model_client`.

### 2 · El mensaje

```python
from autogen_agentchat.messages import TextMessage
message = TextMessage(content="I'd like to go to London", source="user")
```

Un mensaje en AutoGen lleva `content` y `source` (quién lo dice). Más adelante veremos `MultiModalMessage` para imágenes.

### 3 · El agente

```python
from autogen_agentchat.agents import AssistantAgent

agent = AssistantAgent(
    name="airline_agent",
    model_client=model_client,
    system_message="You are a helpful assistant for an airline. You give short, humorous answers.",
    model_client_stream=True,
)
```

`AssistantAgent` = nombre + cliente del modelo + system message. `model_client_stream=True` hace que la respuesta llegue en streaming (token a token).

### Ejecutarlo: `on_messages`

```python
from autogen_core import CancellationToken

response = await agent.on_messages([message], cancellation_token=CancellationToken())
print(response.chat_message.content)
```

- `on_messages([message], ...)` es el equivalente a `Runner.run` / `graph.invoke`: ejecuta el agente con una lista de mensajes. Es `async` → `await`.
- El `CancellationToken()` es un mecanismo para poder **cancelar** una ejecución larga. De momento solo lo pasas; existe por si quieres abortar.
- La respuesta está en `response.chat_message.content`.

---

## 💻 Darle una tool: precios de vuelos

### La "base de datos" (SQLite)

```python
import sqlite3, os

if os.path.exists("tickets.db"):
    os.remove("tickets.db")

conn = sqlite3.connect("tickets.db")
conn.execute("CREATE TABLE cities (city_name TEXT PRIMARY KEY, round_trip_price REAL)")
conn.commit(); conn.close()

def save_city_price(city_name, round_trip_price):
    conn = sqlite3.connect("tickets.db")
    conn.execute("REPLACE INTO cities (city_name, round_trip_price) VALUES (?, ?)",
                 (city_name.lower(), round_trip_price))
    conn.commit(); conn.close()

for city, price in [("London",299),("Paris",399),("Rome",499),("Madrid",550)]:
    save_city_price(city, price)
```

### La función-tool

```python
def get_city_price(city_name: str) -> float | None:
    """ Get the roundtrip ticket price to travel to the city """
    conn = sqlite3.connect("tickets.db")
    c = conn.execute("SELECT round_trip_price FROM cities WHERE city_name = ?", (city_name.lower(),))
    result = c.fetchone()
    conn.close()
    return result[0] if result else None
```

Nada nuevo en la función: Python normal con docstring y type hints (que el LLM usará para entenderla). **Lo bonito de AutoGen: la tool es solo una función Python; se la pasas directamente al agente, sin envoltorios.**

### El agente con la tool

```python
smart_agent = AssistantAgent(
    name="smart_airline_agent",
    model_client=model_client,
    system_message="You are a helpful airline assistant. You give short, humorous answers, including the price of a roundtrip ticket.",
    model_client_stream=True,
    tools=[get_city_price],        # 👈 ¡la función, tal cual!
    reflect_on_tool_use=True,      # 👈 importante (abajo)
)

response = await smart_agent.on_messages([message], cancellation_token=CancellationToken())
for inner in response.inner_messages:     # los pasos intermedios (llamadas a tools)
    print(inner.content)
print(response.chat_message.content)      # la respuesta final
```

Dos cosas que entender:

- **`tools=[get_city_price]`**: le pasas la función directamente. AutoGen genera el esquema solo (a partir del nombre, docstring y type hints). Cero boilerplate JSON.
- **`reflect_on_tool_use=True`**: tras ejecutar la tool, el agente **vuelve a pensar con el resultado** para redactar una respuesta natural (en vez de soltarte el dato crudo). Sin esto, el agente devolvería el número pelado; con esto, lo integra en su respuesta humorística.

> 💡 **`inner_messages` vs `chat_message`:** `chat_message` es la respuesta final; `inner_messages` son los **pasos intermedios** (qué tool llamó, qué devolvió). Es tu ventana al "pensamiento" del agente — el equivalente a una traza en miniatura.

---

## ⚠️ Errores comunes

- **Olvidar `await`.** `on_messages` es asíncrono. En notebook, `await` directo; en script, `asyncio.run(...)`.
- **`reflect_on_tool_use=False` y respuestas raras.** Si quieres que el agente comente el resultado de la tool, ponlo en `True`.
- **La tool sin docstring/type hints.** AutoGen los usa para construir el esquema; sin ellos, el LLM no sabrá usar bien la función.
- **`tickets.db` bloqueada.** SQLite no lleva bien accesos concurrentes; cierra siempre la conexión (`conn.close()`).

---

## 🧪 Pruébalo tú

1. **Añade ciudades** a la base de datos y pregunta por ellas. Pregunta por una que no exista: ¿cómo responde el agente al recibir `None`?
2. **Quita `reflect_on_tool_use`** y compara la respuesta. Verás por qué importa.
3. **Cambia a Ollama** (`OllamaChatCompletionClient`) y comprueba que el resto del código no cambia.
4. **Añade una segunda tool** (p. ej. `get_weather(city)`) y observa al agente combinar ambas.

---

## 📌 Para llevar

- AgentChat = **model client** + **`TextMessage`** + **`AssistantAgent`**, ejecutado con `await agent.on_messages([...], cancellation_token=...)`.
- Cambiar de modelo = cambiar el **model client** (`OpenAIChatCompletionClient`, `OllamaChatCompletionClient`...).
- Las **tools son funciones Python** que pasas directamente en `tools=[...]`; AutoGen genera el esquema desde docstring + type hints.
- **`reflect_on_tool_use=True`** hace que el agente repiense con el resultado de la tool para dar una respuesta natural.
- `response.chat_message.content` = respuesta final; `response.inner_messages` = pasos intermedios.

---

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: AgentChat avanzado →](02_agentchat_avanzado.md)
