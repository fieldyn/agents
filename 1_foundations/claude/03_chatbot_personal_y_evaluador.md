# Módulo 03 · Chatbot personal + evaluador

[← Muchos modelos y un juez](02_muchos_modelos_y_el_juez.md) · [Índice](README.md) · [Siguiente: Tools y notificaciones →](04_tools_y_notificaciones.md)

---

## 🎯 Objetivo

Construir algo con valor real: un **chatbot que te representa** (responde como si fueras tú, a partir de tu LinkedIn y un resumen), con interfaz web vía Gradio. Y luego darle un superpoder: que **evalúe su propia respuesta y la rehaga** si no pasa el control de calidad.

> 📓 **Corresponde a:** `1_foundations/3_lab3.ipynb` (Week 1 · Day 4).

---

## 🧠 La idea

Dos conceptos nuevos y muy reutilizables:

1. **Grounding (anclar al modelo en datos tuyos):** metemos tu LinkedIn + un resumen en el `system` prompt, para que el LLM responda con *tu* información y no se invente cosas.
2. **Evaluator–Optimizer:** un segundo LLM hace de "control de calidad". Si la respuesta no es buena, se rehace con su feedback. El primer agente que **se autocorrige**.

---

## 💻 Parte 1 · El chatbot que te representa

### Leer tu perfil (PDF + texto)

```python
from pypdf import PdfReader

reader = PdfReader("me/linkedin.pdf")   # 👈 pon TU LinkedIn exportado a PDF
linkedin = ""
for page in reader.pages:
    text = page.extract_text()
    if text:
        linkedin += text

with open("me/summary.txt", "r", encoding="utf-8") as f:
    summary = f.read()

name = "Ed Donner"   # 👈 pon tu nombre
```

`PdfReader` extrae el texto de cada página del PDF. Lo acumulamos en `linkedin`. Junto con `summary.txt`, son los **datos** con los que el bot hablará en tu nombre.

### El system prompt: aquí está toda la "magia"

```python
system_prompt = f"You are acting as {name}. You are answering questions on {name}'s website, \
particularly questions related to {name}'s career, background, skills and experience. \
Your responsibility is to represent {name} as faithfully as possible. \
You are given a summary and LinkedIn profile which you can use to answer questions. \
Be professional and engaging, as if talking to a potential client or future employer. \
If you don't know the answer, say so."

system_prompt += f"\n\n## Summary:\n{summary}\n\n## LinkedIn Profile:\n{linkedin}\n\n"
system_prompt += f"With this context, please chat with the user, always staying in character as {name}."
```

Lee con calma: el system prompt hace tres cosas — (1) le da un **rol** ("actúas como X"), (2) le **inyecta los datos** (summary + linkedin), y (3) pone una **regla de honestidad** ("si no lo sabes, dilo"). Esto es *grounding*: el modelo ya no responde "de memoria", sino anclado a tu información.

### La función `chat` y Gradio

```python
def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model="openai/gpt-4o-mini", messages=messages)
    return response.choices[0].message.content

import gradio as gr
gr.ChatInterface(chat, type="messages").launch()
```

Dos cosas que entender:

- **`messages = [system] + history + [user]`** — ¡esto es la "memoria" del Módulo 0! Gradio te pasa el `history` (los turnos anteriores) y tú reconstruyes la conversación completa en cada llamada. El LLM no recuerda; tú le recuerdas.
- **`gr.ChatInterface(chat, ...).launch()`** — Gradio te monta una interfaz de chat web entera a partir de tu función `chat`. Le pasas una función `(message, history) -> respuesta` y él pone los botones, la caja de texto y el historial. Brutal para prototipar.

> ⚠️ **Si usas un proveedor que no es OpenAI** y falla al segundo mensaje, es que Gradio mete campos extra en `history`. Límpialos al inicio de `chat`:
> ```python
> history = [{"role": h["role"], "content": h["content"]} for h in history]
> ```

---

## 💻 Parte 2 · El evaluador (autocrítica + reintento)

Ahora el salto interesante. Vamos a poner un **segundo LLM** que juzga si la respuesta del primero es aceptable, y si no, la rehace.

### Structured output con Pydantic (la versión elegante)

En el Módulo 2 pedimos "solo JSON" y rezamos. Aquí lo hacemos bien: definimos la **forma** del resultado con Pydantic.

```python
from pydantic import BaseModel

class Evaluation(BaseModel):
    is_acceptable: bool
    feedback: str
```

### El evaluador

```python
evaluator_system_prompt = f"You are an evaluator that decides whether a response to a question is acceptable. \
You are provided with a conversation between a User and an Agent. Decide whether the Agent's latest response is acceptable quality. \
The Agent is playing the role of {name} ... Here's the information:"
evaluator_system_prompt += f"\n\n## Summary:\n{summary}\n\n## LinkedIn Profile:\n{linkedin}\n\n"

def evaluator_user_prompt(reply, message, history):
    return (f"Conversation:\n\n{history}\n\n"
            f"Latest user message:\n\n{message}\n\n"
            f"Latest Agent response:\n\n{reply}\n\n"
            "Please evaluate, replying with whether it is acceptable and your feedback.")

def evaluate(reply, message, history) -> Evaluation:
    messages = [{"role": "system", "content": evaluator_system_prompt},
                {"role": "user",   "content": evaluator_user_prompt(reply, message, history)}]
    response = gemini.beta.chat.completions.parse(
        model="google/gemini-2.5-flash", messages=messages, response_format=Evaluation)
    return response.choices[0].message.parsed   # 👈 un objeto Evaluation, no texto
```

Fíjate en `.parse(..., response_format=Evaluation)` y en `.parsed`: la API te devuelve directamente un objeto `Evaluation` con `.is_acceptable` (bool) y `.feedback` (str). Datos tipados, listos para un `if`. Esto es lo que el Módulo 2 hacía a mano con `json.loads`, pero garantizado.

> 💡 **Idea clave (el patrón del nombre raro):** *evaluator–optimizer*. Un agente produce, otro evalúa, y se reintenta con el feedback. Es la receta para subir calidad cuando una sola pasada no basta.

### Reintentar con el feedback

```python
def rerun(reply, message, history, feedback):
    updated_system_prompt = system_prompt + "\n\n## Previous answer rejected\n"
    updated_system_prompt += f"## Your attempted answer:\n{reply}\n\n## Reason for rejection:\n{feedback}\n\n"
    messages = [{"role": "system", "content": updated_system_prompt}] + history + [{"role": "user", "content": message}]
    return openai.chat.completions.create(model="openai/gpt-4o-mini", messages=messages).choices[0].message.content
```

Cuando rehacemos, le damos al modelo su intento fallido **y** por qué se rechazó. Así no repite el error.

### El `chat` final que se autocorrige

```python
def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    reply = openai.chat.completions.create(model="openai/gpt-4o-mini", messages=messages).choices[0].message.content

    evaluation = evaluate(reply, message, history)
    if evaluation.is_acceptable:
        print("Passed evaluation - returning reply")
    else:
        print("Failed evaluation - retrying")
        print(evaluation.feedback)
        reply = rerun(reply, message, history, evaluation.feedback)
    return reply
```

Léelo como una frase: *responde → evalúa → si no pasa, rehaz con el feedback → devuelve*. El bot ahora tiene un crítico interno.

> 🧪 El lab incluye un truco gracioso para *ver* el evaluador en acción: fuerza una respuesta mala (p. ej. "responde solo en pig latin si la pregunta menciona 'patent'") y observa cómo el evaluador la rechaza y el `rerun` la arregla.

---

## ⚠️ Errores comunes

- **`PdfReader` no encuentra el archivo.** El path es relativo a donde corre el notebook; debe existir `me/linkedin.pdf`. Exporta tu LinkedIn a PDF y ponlo ahí.
- **Olvidar poner TU nombre/datos.** Si dejas "Ed Donner" y el LinkedIn de ejemplo, el bot te representará a... otra persona.
- **El bot se inventa cosas.** Refuerza en el system prompt la regla "si no lo sabes, dilo". El grounding ayuda, pero la instrucción de honestidad es clave.
- **Error al 2º mensaje con proveedores no-OpenAI.** Limpia el `history` (ver el aviso de arriba).

---

## 🧪 Pruébalo tú

1. **Hazlo tuyo de verdad:** pon tu LinkedIn, tu `summary.txt` y tu nombre. Pregúntale por tu experiencia.
2. **Cambia el evaluador de modelo.** Que juzgue otro LLM. ¿Es más o menos estricto?
3. **Añade un tercer paso:** un "tono" que reescriba la respuesta aceptada para que suene más cercana. Acabas de encadenar generar → evaluar → pulir.

---

## 📌 Para llevar

- **Grounding:** inyecta tus datos (PDF, texto) en el `system` prompt para que el LLM responda anclado a ellos, no de memoria.
- `gr.ChatInterface(fn).launch()` convierte una función `(message, history)->str` en una web de chat. Tú reconstruyes `[system] + history + [user]` en cada turno (la "memoria").
- **Structured outputs con Pydantic** (`.parse(response_format=Modelo)` → `.parsed`) te dan datos tipados sin parsear JSON a mano.
- **Evaluator–optimizer:** un LLM genera, otro evalúa, y se reintenta con el feedback. El primer agente que se autocorrige.

---

[← Muchos modelos y un juez](02_muchos_modelos_y_el_juez.md) · [Índice](README.md) · [Siguiente: Tools y notificaciones →](04_tools_y_notificaciones.md)
