# Módulo 05 · Patrones y chuleta (cheat sheet)

[← Tools y notificaciones](04_tools_y_notificaciones.md) · [Volver al índice](README.md)

---

## 🎯 Para qué sirve

Tu página de referencia de la Semana 1. Glosario, los patrones agénticos que aprendiste **sin framework**, y el código mínimo de cada cosa.

---

## 📖 Glosario rápido

| Término | En una frase |
|---|---|
| **LLM** | Función texto→texto, sin memoria propia. |
| **Mensajes / roles** | `system` (reglas), `user` (pregunta), `assistant` (respuestas previas). |
| **`base_url`** | A qué proveedor apunta la librería OpenAI (OpenRouter, Ollama, Gemini...). |
| **OpenRouter** | Un proxy: una clave y una API para muchos modelos. |
| **Prompt chaining** | Encadenar llamadas: la salida de una es la entrada de la siguiente. |
| **LLM as a judge** | Un LLM evalúa/ordena las salidas de otros. |
| **Grounding** | Inyectar tus datos en el prompt para anclar las respuestas. |
| **Structured output** | Forzar la salida a una forma fija (JSON / Pydantic). |
| **Evaluator–optimizer** | Un LLM genera, otro evalúa, se reintenta con feedback. |
| **Tool / function calling** | El LLM pide ejecutar una función; tú la ejecutas y le devuelves el resultado. |
| **Bucle agéntico** | Llamar al LLM → ¿pidió tool? → ejecutar → reinyectar → repetir. |
| **Gradio** | Librería para montar una UI de chat con una función. |

---

## 🧩 Los patrones de esta semana

| Patrón | Dónde apareció |
|---|---|
| **Prompt chaining** | Área→dolor→solución (Mód. 1) |
| **Paralelización / multi-modelo** | Preguntar a varios modelos (Mód. 2) |
| **LLM as a judge** | El jurado que rankea (Mód. 2) |
| **Grounding** | LinkedIn + summary en el system prompt (Mód. 3) |
| **Evaluator–optimizer** | Evaluar y rehacer la respuesta (Mód. 3) |
| **Tool use / bucle agéntico** | record_user_details, el `while` (Mód. 4) |

> 💡 Lo potente: **todos estos patrones los montaste con Python normal.** Los frameworks de las próximas semanas no inventan magia nueva; te dan estos mismos patrones más cómodos.

---

## 💻 La chuleta de código

### Llamada básica

```python
from openai import OpenAI
openai = OpenAI(base_url="https://openrouter.ai/api/v1", api_key=openrouter_api_key)

messages = [{"role": "system", "content": "..."}, {"role": "user", "content": "..."}]
resp = openai.chat.completions.create(model="openai/gpt-4o-mini", messages=messages)
print(resp.choices[0].message.content)
```

### Cambiar de proveedor (solo cambia base_url)

```python
ollama = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")   # local, gratis
gemini = OpenAI(base_url="https://generativelanguage.googleapis.com/v1beta/openai/", api_key=google_key)
```

### Structured output con Pydantic

```python
from pydantic import BaseModel

class Evaluation(BaseModel):
    is_acceptable: bool
    feedback: str

resp = openai.beta.chat.completions.parse(model="...", messages=msgs, response_format=Evaluation)
resultado = resp.choices[0].message.parsed   # objeto Evaluation
```

### Chat con memoria + Gradio

```python
def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    return openai.chat.completions.create(model="...", messages=messages).choices[0].message.content

import gradio as gr
gr.ChatInterface(chat, type="messages").launch()
```

### Tool use: el bucle agéntico completo

```python
tools = [{"type": "function", "function": {
    "name": "mi_tool",
    "description": "qué hace y cuándo usarla",
    "parameters": {"type": "object",
                   "properties": {"x": {"type": "string", "description": "..."}},
                   "required": ["x"], "additionalProperties": False}}}]

def handle_tool_calls(tool_calls):
    results = []
    for tc in tool_calls:
        fn = globals().get(tc.function.name)
        args = json.loads(tc.function.arguments)
        result = fn(**args) if fn else {}
        results.append({"role": "tool", "content": json.dumps(result), "tool_call_id": tc.id})
    return results

def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    while True:
        resp = openai.chat.completions.create(model="gpt-4o-mini", messages=messages, tools=tools)
        if resp.choices[0].finish_reason == "tool_calls":
            msg = resp.choices[0].message
            messages.append(msg)
            messages.extend(handle_tool_calls(msg.tool_calls))
        else:
            return resp.choices[0].message.content
```

### Notificación al móvil (Pushover)

```python
import requests, os
requests.post("https://api.pushover.net/1/messages.json",
              data={"user": os.getenv("PUSHOVER_USER"), "token": os.getenv("PUSHOVER_TOKEN"), "message": "hola"})
```

---

## 🐛 Tabla de "no me funciona"

| Síntoma | Causa probable | Arreglo |
|---|---|---|
| `load_dotenv()` → `False` | `.env` no guardado / mal ubicado | `.env` en la raíz, guardado, llamado exactamente `.env` |
| `ImportError` de `openai` | Kernel equivocado | Selecciona `.venv (Python 3.12)` |
| `NameError` | Celda no ejecutada | Ejecuta las celdas en orden de arriba abajo |
| Modelo 404 en OpenRouter | Falta prefijo de proveedor | `openai/...`, `anthropic/...`, etc. |
| Error al 2º mensaje (no-OpenAI) | Gradio mete campos extra en history | `history = [{"role": h["role"], "content": h["content"]} for h in history]` |
| Ollama "connection refused" | Ollama no corre | Instálalo + `ollama serve` + comprueba :11434 |
| `json.loads` falla en el juez | El LLM metió markdown | Insiste "only JSON"; o usa Pydantic |
| El push no llega | Claves Pushover / app no instalada | Revisa `.env` e instala Pushover en el móvil |
| `finish_reason` nunca es tool_calls | Olvidaste `tools=tools` | Pásalas a `create(...)` |

---

## 🚀 Hacia dónde seguir

- Ya tienes el **bucle agéntico a mano**. La siguiente unidad, [`2_openai/claude`](../../2_openai/claude/README.md), te enseña a hacer **exactamente lo mismo** con el OpenAI Agents SDK — y verás cuánto código te ahorra.
- Luego: CrewAI (`3_crew`), LangGraph (`4_langgraph`), AutoGen (`5_autogen`) y MCP (`6_mcp`). Cada uno es otra forma de empaquetar estos mismos patrones.
- `community_contributions/` está lleno de variantes de estudiantes.

---

## ✅ Checklist de "ya lo domino"

- [ ] Sé montar `messages` con roles y leer `choices[0].message.content` (Mód. 0-1)
- [ ] Entiendo que el LLM no tiene memoria; yo le reenvío el historial (Mód. 0)
- [ ] Sé usar `base_url` para cambiar de proveedor / OpenRouter / Ollama (Mód. 1-2)
- [ ] Sé encadenar llamadas (prompt chaining) (Mód. 1)
- [ ] Sé montar un "LLM as a judge" con salida JSON (Mód. 2)
- [ ] Sé hacer grounding metiendo mis datos en el system prompt (Mód. 3)
- [ ] Sé forzar structured output con Pydantic (Mód. 3)
- [ ] Sé el patrón evaluator–optimizer (evaluar + rehacer) (Mód. 3)
- [ ] **Sé escribir el bucle de tool use a mano** (Mód. 4)
- [ ] Sé montar una UI con Gradio y desplegar `app.py` (Mód. 3-4)

Si marcaste todo: entiendes los agentes **desde los cimientos**. Ahora los frameworks serán fáciles. 🎉

---

[← Tools y notificaciones](04_tools_y_notificaciones.md) · [Volver al índice](README.md)
