# Módulo 00 · Introducción: ¿Qué es un LLM y cómo se le habla?

[← Volver al índice](README.md) · [Siguiente: Tu primera llamada →](01_primera_llamada.md)

---

## 🎯 Objetivo

Que entiendas el modelo mental mínimo —**mensajes, roles y la naturaleza "sin memoria" del LLM**— antes de escribir código. Con esto, todo lo de la semana encaja.

---

## 🧠 La idea: un LLM es una función de texto a texto

Quítale el misterio. En el fondo, un LLM es una función:

```
   texto de entrada  ──▶  [ LLM ]  ──▶  texto de salida
```

Le das una conversación y te devuelve la siguiente respuesta probable. Ya está. No "recuerda", no "navega", no "ejecuta" nada por sí mismo. Todo lo que parezca lo contrario (memoria, herramientas, agentes) lo construimos **nosotros** alrededor de esa función. Esta semana vas a construir justo eso, a mano.

---

## 💬 Mensajes y roles: el formato universal

No le hablas al LLM con un string suelto, sino con una **lista de mensajes**, cada uno con un **rol**:

```python
messages = [
    {"role": "system", "content": "Eres un asistente conciso."},
    {"role": "user",   "content": "¿Cuánto es 2+2?"},
]
```

| Rol | Quién habla | Para qué |
|---|---|---|
| `system` | Tú, entre bastidores | Define **quién es** el asistente y las reglas. El usuario no lo ve. |
| `user` | La persona | La pregunta o instrucción actual. |
| `assistant` | El modelo | Sus respuestas anteriores (así "recuerda" la conversación). |

> 💡 **Idea clave:** el LLM no tiene memoria. La "memoria" de un chat es que **tú le reenvías toda la lista de mensajes** cada vez. Cuando en el Módulo 3 veas `messages = [system] + history + [user]`, eso es exactamente esto: reconstruir la conversación entera en cada turno.

---

## 🔁 El patrón que se repite toda la semana

Casi todo lo que harás sigue esta forma:

```python
from openai import OpenAI
openai = OpenAI()

messages = [{"role": "user", "content": "..."}]
response = openai.chat.completions.create(model="...", messages=messages)
print(response.choices[0].message.content)
```

Tres pasos: (1) montas la lista de mensajes, (2) llamas a `create`, (3) lees `response.choices[0].message.content`. Memorízalo; lo verás cien veces.

---

## 🧩 Lo que vas a construir esta semana (y por qué importa)

| Módulo | Construyes | El "salto" conceptual |
|---|---|---|
| 1 | Encadenar llamadas | La salida de un LLM es la entrada del siguiente |
| 2 | Varios modelos + un juez | Un LLM puede **evaluar** a otros LLMs |
| 3 | Chatbot + autoevaluador | Un LLM puede **criticar y rehacer** su propia respuesta |
| 4 | Tools + notificaciones | El LLM puede **pedir que ejecutes funciones** → esto ya es un agente |

Fíjate en el hilo: cada módulo le da al LLM una capacidad nueva, y todas las construyes tú con Python normal. Para el Módulo 4 tendrás un agente de verdad **sin ningún framework**.

---

## 📌 Para llevar

- Un LLM es una **función texto→texto** sin memoria ni capacidades propias; todo lo demás lo añades tú alrededor.
- Le hablas con una **lista de mensajes** con roles `system` / `user` / `assistant`.
- La "memoria" de un chat = reenviarle el historial completo cada turno.
- El patrón base: montar `messages` → `create(...)` → leer `choices[0].message.content`.

---

[← Volver al índice](README.md) · [Siguiente: Tu primera llamada →](01_primera_llamada.md)
