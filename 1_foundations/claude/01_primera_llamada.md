# Módulo 01 · Tu primera llamada (y cómo encadenarlas)

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: Muchos modelos y un juez →](02_muchos_modelos_y_el_juez.md)

---

## 🎯 Objetivo

Hacer tu primera llamada a un LLM y descubrir el primer "truco agéntico" de verdad: **encadenar** llamadas, donde la respuesta de una alimenta la siguiente.

> 📓 **Corresponde a:** `1_foundations/1_lab1.ipynb` (Week 1 · Day 1).

---

## 💻 El código, explicado

### Cargar las claves

```python
from dotenv import load_dotenv
load_dotenv(override=True)
```

`load_dotenv` lee tu `.env`. Si esto devuelve `False`, casi siempre es que **no guardaste** el `.env` tras añadir la clave, o no está en la raíz del repo. `override=True` = "usa siempre la clave del `.env`, aunque hubiera otra cargada".

### Crear el cliente (apuntando a OpenRouter)

```python
import os
from openai import OpenAI

OPENROUTER_BASE_URL = "https://openrouter.ai/api/v1"
openrouter_api_key = os.getenv("OPENROUTER_API_KEY")

openai = OpenAI(base_url=OPENROUTER_BASE_URL, api_key=openrouter_api_key)
```

Aquí está el patrón estrella de la semana: usamos la **librería `OpenAI`** pero apuntándola a **OpenRouter** con `base_url`. Resultado: una sola clave para acceder a GPT, Claude, Gemini, DeepSeek, Llama... Memoriza esta forma; volverá una y otra vez.

> 💡 Aunque uses Gemini o DeepSeek, sigues importando `from openai import OpenAI`. La librería es solo el "idioma"; `base_url` decide con quién hablas.

### La llamada

```python
messages = [{"role": "user", "content": "What is 2+2?"}]

response = openai.chat.completions.create(
    model="xiaomi/mimo-v2.5",
    messages=messages,
)
print(response.choices[0].message.content)
```

Tres piezas:
- `model=` — qué modelo concreto quieres (en OpenRouter, con prefijo de proveedor, p. ej. `openai/gpt-5-mini`).
- `messages=` — tu lista de mensajes (Módulo 0).
- `response.choices[0].message.content` — el texto de la respuesta. Ese `choices[0]` es porque la API puede devolver varias alternativas; normalmente quieres la primera.

¡Felicidades, ya hablaste con un LLM por código!

---

## 🔗 El primer patrón agéntico: encadenar llamadas

Aquí está la chispa. ¿Qué pasa si la **respuesta** de una llamada se convierte en la **pregunta** de la siguiente?

```python
# 1. Pide una pregunta difícil de IQ
question = "Please propose a hard, challenging question to assess someone's IQ. Respond only with the question."
messages = [{"role": "user", "content": question}]
response = openai.chat.completions.create(model="deepseek/deepseek-v4-flash", messages=messages)
question = response.choices[0].message.content   # 👈 la respuesta...

# 2. ...se la devolvemos al modelo como nueva pregunta
messages = [{"role": "user", "content": question}]
response = openai.chat.completions.create(model="openai/gpt-5-mini", messages=messages)
answer = response.choices[0].message.content
print(answer)
```

Mira lo que pasó: el LLM se inventó una pregunta y luego (otra llamada, incluso otro modelo) la respondió. **La salida de un paso fue la entrada del siguiente.** Esto, en esencia, es lo que hace un agente: encadenar pasos de razonamiento. Lo estás haciendo a mano, y eso es bueno: cuando un framework lo automatice, sabrás qué hay debajo.

### Mostrar bonito en el notebook

```python
from IPython.display import Markdown, display
display(Markdown(answer))
```

Si la respuesta trae markdown (listas, **negritas**...), `display(Markdown(...))` la renderiza con formato en vez de mostrar el texto crudo.

---

## 🧪 Pruébalo tú (el ejercicio "business idea")

El lab propone un mini-pipeline de 3 eslabones. Es oro porque es el germen de todo el curso:

```python
# 1. Elige un área de negocio con oportunidad para IA agéntica
messages = [{"role": "user", "content": "Pick a business area worth exploring for an Agentic AI opportunity."}]
business_idea = openai.chat.completions.create(model="deepseek/deepseek-v4-flash", messages=messages).choices[0].message.content

# 2. Encuentra un "pain-point" en ese sector
messages = [{"role": "user", "content": f"Please propose a pain-point in the {business_idea} industry."}]
pain_point = openai.chat.completions.create(model="deepseek/deepseek-v4-flash", messages=messages).choices[0].message.content

# 3. Propón una solución agéntica a ese pain-point
messages = [{"role": "user", "content": f"Please propose an Agentic AI solution to the {pain_point} pain-point."}]
solution = openai.chat.completions.create(model="deepseek/deepseek-v4-flash", messages=messages).choices[0].message.content
print(solution)
```

Área → dolor → solución. Tres llamadas encadenadas. Esto se llama **prompt chaining** y es uno de los patrones agénticos fundamentales (lo verás con nombre y apellidos en el [Módulo 05](05_patrones_y_chuleta.md)).

Otras variaciones para fijar:
1. Cambia los modelos de cada eslabón. ¿Cambia la calidad?
2. Añade un cuarto eslabón que critique la solución del tercero.
3. Pon `print` en cada paso para "ver pensar" al pipeline.

---

## ⚠️ Errores comunes

- **`load_dotenv()` devuelve `False`.** No guardaste el `.env`, o no está en la raíz, o no se llama exactamente `.env`.
- **`ImportError` al hacer `from openai import OpenAI`.** Kernel equivocado: selecciona `.venv (Python 3.12)`.
- **`NameError`.** Usaste una variable antes de definir su celda. En notebooks el **orden de ejecución** importa: ejecuta las celdas de arriba abajo.
- **Modelo desconocido (404).** En OpenRouter el `model` lleva prefijo de proveedor (`openai/...`, `anthropic/...`). Sin prefijo no lo encuentra.

---

## 📌 Para llevar

- Patrón base: `messages` → `openai.chat.completions.create(...)` → `choices[0].message.content`.
- `OpenAI(base_url=..., api_key=...)` te deja usar muchos proveedores con una sola librería. Con OpenRouter, una sola clave.
- **Encadenar** llamadas (la salida de una es la entrada de otra) es el patrón **prompt chaining**: el germen de todo agente.
- En notebooks, ejecuta las celdas en orden; los `NameError` casi siempre son por saltarse una.

---

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: Muchos modelos y un juez →](02_muchos_modelos_y_el_juez.md)
