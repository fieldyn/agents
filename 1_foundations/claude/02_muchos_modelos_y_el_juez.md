# Módulo 02 · Muchos modelos y un juez

[← Tu primera llamada](01_primera_llamada.md) · [Índice](README.md) · [Siguiente: Chatbot personal + evaluador →](03_chatbot_personal_y_evaluador.md)

---

## 🎯 Objetivo

Hacer la misma pregunta a **varios modelos** y luego pedirle a **otro LLM que los juzgue** y los ordene de mejor a peor. Por el camino te vuelves cómodo con APIs y descubres el patrón *LLM as a judge*.

> 📓 **Corresponde a:** `1_foundations/2_lab2.ipynb` (Week 1 · Day 3).

---

## 🧠 La idea

Ningún modelo es el mejor en todo. Una técnica potentísima en producción es **preguntar a varios y combinar/elegir** — sube la calidad y la fiabilidad. Hoy montamos una "competición" de modelos con un jurado automático.

---

## 💻 El código, explicado

### Paso 1 · Generar una pregunta difícil

Primero usamos un LLM para que **invente** la pregunta del examen (encadenar llamadas, Módulo 1):

```python
request = "Please come up with a challenging, nuanced question that I can ask a number of LLMs to evaluate their intelligence. Answer only with the question, no explanation."
messages = [{"role": "user", "content": request}]

openai = OpenAI(base_url=OPENROUTER_BASE_URL, api_key=openrouter_api_key)
question = openai.chat.completions.create(model="openai/gpt-5-mini", messages=messages).choices[0].message.content
print(question)
```

### Paso 2 · Preguntar a muchos modelos

Preparamos dos listas para ir acumulando quién respondió qué:

```python
competitors = []
answers = []
messages = [{"role": "user", "content": question}]
```

Y ahora preguntamos a modelo tras modelo. Fíjate en que el **código es idéntico**, solo cambia `model_name`:

```python
model_name = "anthropic/claude-sonnet-4.5"
answer = openai.chat.completions.create(model=model_name, messages=messages).choices[0].message.content
display(Markdown(answer))
competitors.append(model_name)
answers.append(answer)
```

Repites ese bloque para `openai/gpt-5-nano`, `google/gemini-2.5-flash`, `deepseek/deepseek-chat`, `openai/gpt-oss-120b`... Todos con la **misma** llamada gracias a OpenRouter.

> 💡 **Modelos lentos ≠ rotos.** Algunos modelos "que piensan" (razonadores) tardan 1-2 minutos. No están colgados. Si tienes prisa, usa variantes `mini`/`nano`/`flash`.

### Paso 3 (opcional) · Un modelo local con Ollama

Puedes meter en la competición un modelo que corre **en tu propia máquina, gratis**, con [Ollama](https://ollama.com):

```python
# en terminal o con ! en el notebook:  ollama pull llama3.2
ollama = OpenAI(base_url='http://localhost:11434/v1', api_key='ollama')
answer = ollama.chat.completions.create(model="llama3.2", messages=messages).choices[0].message.content
```

¡Otra vez el mismo patrón! Ollama también expone un endpoint compatible con OpenAI; solo cambia la `base_url` a `localhost:11434`.

> ⚠️ **No descargues `llama3.3`** en un portátil: es gigante y te come la máquina. Quédate con `llama3.2` o `llama3.2:1b`.

### Paso 4 · Juntar las respuestas

Construimos un único texto con todas las respuestas numeradas:

```python
together = ""
for index, answer in enumerate(answers):
    together += f"# Response from competitor {index+1}\n\n{answer}\n\n"
```

`enumerate` nos da el índice y el valor a la vez — útil para numerar a los competidores.

### Paso 5 · El juez (LLM as a judge)

Aquí está el patrón estrella. Le pedimos a un LLM que evalúe las respuestas y devuelva **solo JSON** con el ranking:

```python
judge = f"""You are judging a competition between {len(competitors)} competitors.
Each model has been given this question: {question}

Evaluate each response for clarity and strength of argument, and rank them best to worst.
Respond with JSON, and only JSON, in this format:
{{"results": ["best competitor number", "second best", ...]}}

Here are the responses: {together}

Respond with only the JSON, no markdown."""

judge_messages = [{"role": "user", "content": judge}]
results = openai.chat.completions.create(model="openai/gpt-5-mini", messages=judge_messages).choices[0].message.content
```

Y convertimos ese JSON en algo legible:

```python
import json
results_dict = json.loads(results)
ranks = results_dict["results"]
for index, result in enumerate(ranks):
    competitor = competitors[int(result)-1]
    print(f"Rank {index+1}: {competitor}")
```

> 💡 **Por qué pedimos "solo JSON":** así podemos `json.loads(...)` la respuesta y tratarla como datos, no como prosa. Es la versión "a mano" de los *structured outputs* — un puente entre el texto del LLM y tu código. En el Módulo 3 verás la forma elegante con Pydantic.

---

## ⚠️ Errores comunes

- **`json.loads` falla.** El juez metió ```` ```json ```` o texto extra. Por eso el prompt insiste "only JSON, no markdown". Si pasa, recórtalo o usa structured outputs (Módulo 3).
- **Ollama: "connection refused".** No tienes Ollama corriendo. Instálalo, y en terminal `ollama serve`; comprueba http://localhost:11434.
- **Todo va lentísimo.** Probablemente un modelo razonador. Cambia a variantes rápidas.
- **El ranking apunta al competidor equivocado.** Ojo al `int(result)-1`: los competidores se numeran desde 1 en el prompt pero la lista de Python desde 0.

---

## 🧪 Pruébalo tú

1. **Añade un modelo más** a la competición (otro de OpenRouter o uno local de Ollama).
2. **Cambia el criterio** del juez: en vez de "claridad y argumentación", pídele "concisión" o "creatividad". ¿Cambia el ranking?
3. **Mejóralo con un patrón nuevo:** haz que tres jueces distintos voten y combina sus rankings (esto se acerca a *ensemble*). ¿Qué pasa si no se ponen de acuerdo?

---

## 📌 Para llevar

- Preguntar a **varios modelos** y combinar/elegir sube la calidad — patrón habitual en producción.
- El mismo código sirve para todos: solo cambia `model_name` (gracias a OpenRouter) o la `base_url` (para Ollama local).
- **LLM as a judge**: un LLM evalúa las salidas de otros. Pídele **solo JSON** para poder procesarlo con `json.loads`.
- Ollama corre modelos locales gratis con endpoint compatible con OpenAI; no descargues modelos gigantes en un portátil.

---

[← Tu primera llamada](01_primera_llamada.md) · [Índice](README.md) · [Siguiente: Chatbot personal + evaluador →](03_chatbot_personal_y_evaluador.md)
