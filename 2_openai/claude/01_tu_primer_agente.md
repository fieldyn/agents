# Módulo 01 · Tu primer agente

[← Introducción](00_introduccion.md) · [Volver al índice](README.md) · [Siguiente: Workflows, Tools y Handoffs →](02_workflows_tools_handoffs.md)

---

## 🎯 Objetivo

Crear, ejecutar y **observar** tu primer agente. Vas a ver con tus propios ojos lo ligero que es el SDK: un agente completo en tres líneas, y una traza para espiar lo que hace por dentro.

> 📓 **Corresponde a:** `2_openai/1_lab1.ipynb` (Week 2 · Day 1).

---

## 🧠 La idea

En el módulo anterior dijimos que un agente es *LLM + instrucciones + herramientas + bucle*. Hoy vamos a hacer la versión más sencilla posible: un agente **sin herramientas**, solo instrucciones. Un contador de chistes.

¿Por qué empezar tan simple? Porque quiero que veas que la "magia" del SDK no está en la complejidad, sino en lo poco que tienes que escribir. Una vez que el esqueleto te resulte natural, le iremos colgando piezas.

---

## 💻 El código, explicado

### Paso 1 · Los imports

```python
from dotenv import load_dotenv
from agents import Agent, Runner, trace
```

- `load_dotenv` lee tu archivo `.env` y mete las claves (como `OPENAI_API_KEY`) en el entorno. Sin esto, el SDK no sabe con qué credenciales hablar con OpenAI.
- `Agent`, `Runner`, `trace` son **las tres piezas** de hoy. Fíjate en lo corto que es: el paquete se llama, literalmente, `agents`.

### Paso 2 · Cargar las claves

```python
load_dotenv(override=True)
```

`override=True` es importante y conviene entenderlo: significa *"si ya había una variable de entorno con ese nombre, písala con la del `.env`"*. Así te aseguras de usar siempre la clave de tu archivo y no una vieja que tu shell tuviera cargada de otra sesión. Es un pequeño seguro contra el clásico "pero si yo cambié la clave...".

### Paso 3 · Crear el agente

```python
agent = Agent(
    name="Jokester",
    instructions="You are a joke teller",
    model="gpt-4o-mini",
)
```

Aquí está tu primer agente. Tres parámetros, tres conceptos:

| Parámetro | Para qué sirve |
|---|---|
| `name` | Una etiqueta para identificarlo. **No es decorativa:** aparecerá en las trazas y, cuando tengas varios agentes, te salvará la vida para saber quién hizo qué. |
| `instructions` | El **system prompt**. Define quién es y cómo se comporta. "Eres un contador de chistes." |
| `model` | Qué LLM usa por debajo. `gpt-4o-mini` es barato y rápido, perfecto para aprender. |

> 💡 **Idea clave:** crear el agente **no llama a OpenAI todavía**. Solo estás *describiendo* al empleado. No trabaja hasta que el `Runner` le dice "empieza".

### Paso 4 · Ejecutarlo (y observarlo)

```python
with trace("Telling a joke"):
    result = await Runner.run(agent, "Tell a joke about Autonomous AI Agents")
    print(result.final_output)
```

Desglosemos esto, porque hay tres cosas nuevas:

**1. `Runner.run(agent, "...")`** — Aquí *sí* arranca el bucle agéntico. Le pasas el agente y el mensaje del usuario (el *prompt*). El `Runner` ejecuta el bucle del Módulo 00 y devuelve un objeto `result`.

**2. `await`** — `Runner.run` es **asíncrono**. En un notebook puedes usar `await` directamente en una celda. Más adelante (Módulo 2) veremos por qué la asincronía es una ventaja brutal: nos dejará lanzar varios agentes **a la vez**.

**3. `result.final_output`** — El `result` trae mucha información (todos los pasos intermedios, las tools usadas...), pero lo que normalmente quieres es `final_output`: la respuesta final en texto.

**4. `with trace("Telling a joke"):`** — Esto envuelve la ejecución en una **traza** con nombre. No cambia el resultado, pero registra cada paso para que puedas inspeccionarlo después.

Resultado de ejemplo:

```
Why did the autonomous AI agent bring a ladder to work?
Because it heard the job had high expectations!
```

---

## 🔍 Las trazas: tu superpoder de depuración

Después de ejecutar, ve a:

👉 **https://platform.openai.com/traces**

Ahí verás tu traza "Telling a joke" con el detalle de lo que pasó: el prompt, la respuesta, el tiempo, el coste. Con un solo agente esto parece poco. Pero cuando tengas **cinco agentes pasándose el trabajo entre sí**, la traza será la diferencia entre entender tu sistema y rezar para que funcione.

> 💡 **Cógele cariño a las trazas desde el día 1.** Es el hábito que separa a quien "hace funcionar" agentes de quien de verdad los *entiende*. Cada vez que ejecutes algo en este curso, ábrela y mírala.

---

## ⚠️ Errores comunes

- **"OpenAI API Key not set" / errores de autenticación.** Tu `.env` no se cargó o la clave no está. Revisa que el `.env` esté en la **raíz del repo** y que ejecutaste `load_dotenv(override=True)`.
- **`SyntaxError` con `await`.** Estás intentando usar `await` fuera de una celda de notebook (por ejemplo en un script `.py` normal). En un script tendrías que envolverlo en `asyncio.run(...)`. En el notebook, `await` directo funciona.
- **El kernel equivocado.** Si los imports de `agents` fallan, casi siempre es que el notebook no está usando el kernel `.venv (Python 3.12)`. Cámbialo arriba a la derecha.
- **No ver nada en las trazas.** Asegúrate de estar logueado en la misma cuenta de OpenAI cuya clave usas, y date unos segundos: a veces tardan en aparecer.

---

## 🧪 Pruébalo tú

1. **Cambia la personalidad.** Pon `instructions="Eres un contador de chistes muy malo que siempre se ríe de sus propios chistes"`. Vuelve a ejecutar. ¿Notas el cambio? Acabas de ver el poder del system prompt.
2. **Cambia el tema.** Pide un chiste sobre otra cosa cambiando el prompt en `Runner.run`. Distingue bien las dos palancas: las **instructions** definen *quién es*, el **prompt** define *qué le pides ahora*.
3. **Ponle otro nombre a la traza** y ejecuta dos veces con nombres distintos. Mira cómo aparecen separadas en el panel de trazas.

---

## 📌 Para llevar

- Un agente se crea con `Agent(name, instructions, model)` — y crearlo **no llama** al modelo todavía.
- `Runner.run(agent, prompt)` es lo que arranca el **bucle agéntico**; es `async`, así que usas `await`.
- La respuesta en texto está en `result.final_output`.
- `with trace("nombre"):` registra la ejecución para inspeccionarla en **platform.openai.com/traces**. Úsalo siempre.
- `instructions` = quién es el agente (fijo). `prompt` = qué le pides ahora (variable).

---

[← Introducción](00_introduccion.md) · [Volver al índice](README.md) · [Siguiente: Workflows, Tools y Handoffs →](02_workflows_tools_handoffs.md)
