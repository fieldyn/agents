# Módulo 04 · Tools y notificaciones (tu primer agente de verdad)

[← Chatbot personal + evaluador](03_chatbot_personal_y_evaluador.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)

---

## 🎯 Objetivo

El gran salto de la semana: darle **herramientas** al LLM. Hasta ahora solo hablaba; ahora podrá **pedir que ejecutes funciones** (guardar un contacto, registrar una pregunta sin respuesta) y avisarte al móvil. Construirás el **bucle de tool use a mano** — sin frameworks — y verás que un "agente" no es magia.

> 📓 **Corresponde a:** `1_foundations/4_lab4.ipynb` + `1_foundations/app.py` (Week 1 · Day 5).

---

## 🧠 La idea: el LLM no ejecuta nada, lo *pide*

Esto es lo más importante del módulo, así que léelo dos veces:

> Cuando un LLM "usa una herramienta", **no ejecuta código**. Lo que hace es **devolverte un mensaje que dice "por favor, llama a la función `X` con estos argumentos"**. Tú ejecutas la función en tu Python, le devuelves el resultado, y el LLM continúa.

```
   LLM: "no sé esto, pero pídeme que llame a record_unknown_question('...')"
              │
              ▼
   TÚ ejecutas record_unknown_question('...') en Python  →  resultado
              │
              ▼
   LLM recibe el resultado y sigue la conversación
```

El "agente" es ese **bucle**: llamar al LLM → ¿pidió una tool? → ejecutarla → devolver el resultado → repetir. Lo vas a escribir tú mismo.

---

## 💻 Parte 1 · Notificaciones con Pushover

Una herramienta útil de verdad: que el bot te mande un **push al móvil**. Crea cuenta gratis en https://pushover.net, saca tus claves y añádelas al `.env`:

```
PUSHOVER_USER=u...
PUSHOVER_TOKEN=a...
```

```python
import requests, os
pushover_url = "https://api.pushover.net/1/messages.json"

def push(message):
    print(f"Push: {message}")
    payload = {"user": os.getenv("PUSHOVER_USER"), "token": os.getenv("PUSHOVER_TOKEN"), "message": message}
    requests.post(pushover_url, data=payload)

push("HEY!!")   # debería sonarte el móvil
```

Es una simple petición HTTP POST. Nada de IA todavía: solo una función Python que manda una notificación.

---

## 💻 Parte 2 · Definir las herramientas

### Las funciones Python

```python
def record_user_details(email, name="Name not provided", notes="not provided"):
    push(f"Recording interest from {name} with email {email} and notes {notes}")
    return {"recorded": "ok"}

def record_unknown_question(question):
    push(f"Recording {question} asked that I couldn't answer")
    return {"recorded": "ok"}
```

Dos herramientas: una guarda el contacto de un interesado, la otra registra preguntas que el bot no supo responder (oro para mejorar tu perfil).

### Describir las tools al LLM (el "boilerplate JSON")

El LLM necesita saber qué herramientas tiene y cómo se usan. Se lo decimos con un **esquema JSON**:

```python
record_user_details_json = {
    "name": "record_user_details",
    "description": "Use this tool to record that a user is interested in being in touch and provided an email address",
    "parameters": {
        "type": "object",
        "properties": {
            "email": {"type": "string", "description": "The email address of this user"},
            "name":  {"type": "string", "description": "The user's name, if they provided it"},
            "notes": {"type": "string", "description": "Any additional context worth recording"},
        },
        "required": ["email"],
        "additionalProperties": False,
    },
}
# ...y otro igual para record_unknown_question

tools = [{"type": "function", "function": record_user_details_json},
         {"type": "function", "function": record_unknown_question_json}]
```

> 💡 **Graba esto:** este JSON tedioso —nombre, descripción, tipos de cada parámetro— es EXACTAMENTE lo que en el Módulo 2 del curso `2_openai` desaparece con el decorador `@function_tool`. Aquí lo escribimos a mano para que, cuando un framework te lo ahorre, sepas qué te está ahorrando y por qué la **descripción** importa tanto: el LLM la lee para decidir cuándo usar la tool.

---

## 💻 Parte 3 · Ejecutar las tools que el LLM pide

Cuando el LLM pide una o más tools, recibimos una lista de `tool_calls`. Hay que ejecutarlas. La primera versión, explícita, con **el gran `if`**:

```python
def handle_tool_calls(tool_calls):
    results = []
    for tool_call in tool_calls:
        tool_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)
        print(f"Tool called: {tool_name}", flush=True)

        # EL GRAN IF
        if tool_name == "record_user_details":
            result = record_user_details(**arguments)
        elif tool_name == "record_unknown_question":
            result = record_unknown_question(**arguments)

        results.append({"role": "tool", "content": json.dumps(result), "tool_call_id": tool_call.id})
    return results
```

Para cada tool pedida: leemos su nombre, parseamos sus argumentos (vienen como string JSON), llamamos a la función, y devolvemos el resultado con `role: "tool"` y el `tool_call_id` (para que el LLM sepa a qué petición corresponde).

Y una versión **más elegante** que se libra del `if` usando `globals()`:

```python
def handle_tool_calls(tool_calls):
    results = []
    for tool_call in tool_calls:
        tool_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)
        print(f"Tool called: {tool_name}", flush=True)
        tool = globals().get(tool_name)          # 👈 busca la función por su nombre
        result = tool(**arguments) if tool else {}
        results.append({"role": "tool", "content": json.dumps(result), "tool_call_id": tool_call.id})
    return results
```

`globals().get("record_user_details")` te devuelve la función con ese nombre. Así, añadir una tool nueva no te obliga a tocar un `if`. Truco bonito para tener a mano.

---

## 💻 Parte 4 · El bucle agéntico (¡a mano!)

Aquí se junta todo. Este `while` **es** el agente:

```python
def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    done = False
    while not done:
        # 1. Llamamos al LLM, pasándole las tools disponibles
        response = openai.chat.completions.create(model="gpt-4o-mini", messages=messages, tools=tools)

        # 2. ¿El LLM pidió usar una tool?
        if response.choices[0].finish_reason == "tool_calls":
            message = response.choices[0].message
            tool_calls = message.tool_calls
            results = handle_tool_calls(tool_calls)   # 3. la ejecutamos
            messages.append(message)                  # 4. añadimos la petición...
            messages.extend(results)                  # ...y su resultado al historial
            # 5. el while vuelve a llamar al LLM, ahora con el resultado en mano
        else:
            done = True   # el LLM ya tiene su respuesta final
    return response.choices[0].message.content
```

Recórrelo despacio, porque esto es el corazón de TODO el curso:

1. Llamas al LLM con `tools=tools`.
2. Miras `finish_reason`: si es `"tool_calls"`, el LLM **no terminó**: quiere que ejecutes algo.
3. Ejecutas las tools con `handle_tool_calls`.
4. Metes en `messages` tanto la petición del LLM como el resultado de la tool.
5. El `while` repite: el LLM vuelve a pensar, ahora con el resultado. Cuando ya no pide tools (`finish_reason != "tool_calls"`), salimos y devolvemos la respuesta.

> 💡 **Este es el "bucle agéntico" del que hablan todos los frameworks.** OpenAI Agents SDK, CrewAI, LangGraph... todos implementan esta misma idea por debajo. Tú la acabas de escribir en 12 líneas. A partir de ahora, cuando un framework te dé un "agente con tools", sabrás exactamente qué bucle está corriendo por dentro.

---

## 💻 Parte 5 · De notebook a app desplegable (`app.py`)

[app.py](../app.py) empaqueta todo lo anterior en una clase `Me` y lanza la interfaz Gradio. La estructura clave:

```python
class Me:
    def __init__(self):
        self.openai = OpenAI()
        self.name = "Ed Donner"
        # ...carga linkedin.pdf y summary.txt...

    def handle_tool_call(self, tool_calls): ...   # como en la Parte 3
    def system_prompt(self): ...                  # rol + datos + reglas
    def chat(self, message, history): ...         # el bucle agéntico de la Parte 4

if __name__ == "__main__":
    me = Me()
    gr.ChatInterface(me.chat, type="messages").launch()
```

Lo ejecutas desde la carpeta del módulo con:

```sh
uv run app.py
```

Y tienes tu "Career Conversation" funcionando: un bot que te representa, registra interesados (te avisa al móvil) y anota lo que no supo contestar. Un agente pequeño pero **completo y útil**, sin un solo framework.

---

## ⚠️ Errores comunes

- **El push no llega.** Revisa `PUSHOVER_USER`/`PUSHOVER_TOKEN` en `.env` y que instalaste la app de Pushover en el móvil.
- **`KeyError`/argumentos raros al ejecutar la tool.** El LLM mandó argumentos que no esperabas. Los `description` de tu esquema JSON guían al LLM: cuanto más claros, mejores los argumentos.
- **Bucle infinito.** Si el LLM pide tools sin parar, suele ser un system prompt confuso o tools mal descritas. Añade prints y revisa qué pide en cada vuelta.
- **`finish_reason` nunca es `tool_calls`.** Olvidaste pasar `tools=tools` a `create(...)`, o el modelo no admite tools.

---

## 🧪 Pruébalo tú

1. **Añade una tool nueva** (p. ej. `record_meeting_request(date)`). Con la versión `globals()` ni tocas el bucle. Pruébala pidiéndole al bot una reunión.
2. **Combina con el evaluador del Módulo 3:** evalúa la respuesta final del bucle antes de devolverla.
3. **Despliega `app.py`** con tus datos y compártelo. Pregúntale cosas que no sepa y mira llegar los pushes con `record_unknown_question`.

---

## 📌 Para llevar

- Un LLM **no ejecuta** herramientas: te **pide** que las ejecutes tú y le devuelvas el resultado.
- Las tools se describen con un **esquema JSON** (nombre, descripción, parámetros). La **descripción** es lo que el LLM lee para decidir usarlas — el mismo JSON que `@function_tool` genera solo en otros frameworks.
- El **bucle agéntico** = llamar al LLM con `tools` → si `finish_reason == "tool_calls"`, ejecutar y reinyectar el resultado → repetir hasta que no pida más. **Eso es un agente.**
- `globals().get(nombre)` evita el gran `if` para despachar tools.
- `app.py` + `uv run app.py` convierte el notebook en una app Gradio real y desplegable.

---

[← Chatbot personal + evaluador](03_chatbot_personal_y_evaluador.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)
