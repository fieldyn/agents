# Módulo 03 · Modelos, Structured Outputs y Guardrails

[← Workflows, Tools y Handoffs](02_workflows_tools_handoffs.md) · [Volver al índice](README.md) · [Siguiente: Deep Research →](04_deep_research.md)

---

## 🎯 Objetivo

Tres mejoras que convierten un prototipo en algo de calidad de producción:

1. **Modelos distintos** — no estás casado con OpenAI: Gemini, DeepSeek, Llama... todos con la misma API.
2. **Structured Outputs** — obligar al agente a devolver datos con una **forma fija** (no texto libre), para poder programar encima.
3. **Guardrails** — barreras de seguridad que **bloquean** entradas o salidas indebidas antes de que causen daño.

> 📓 **Corresponde a:** `2_openai/3_lab3.ipynb` (Week 2 · Day 3).

---

## 🧠 La idea

Hasta ahora todo era OpenAI, texto libre y "que el modelo se porte bien". En la vida real querrás:

- **Elegir modelo** por coste, velocidad o privacidad. (Un modelo open-source corriendo barato puede bastar para tareas simples.)
- **Datos con forma**, no párrafos. Si un agente decide "¿hay un nombre de persona aquí? sí/no", quieres un booleano, no un ensayo.
- **Garantías**, no buena fe. Si tu agente no debe procesar datos personales, necesitas algo que lo **impida**, no que se lo pida por favor.

Vamos una por una, sobre el mismo ejemplo de ventas del módulo anterior.

---

## Parte 1 · Usar cualquier modelo (endpoints compatibles con OpenAI)

El truco que hace esto fácil: muchos proveedores ofrecen un endpoint **"compatible con OpenAI"**. Es decir, hablan el mismo idioma que la API de OpenAI, solo cambia la URL base y la clave. El SDK lo aprovecha.

### Las claves (todas opcionales menos OpenAI)

```python
openai_api_key   = os.getenv('OPENAI_API_KEY')
google_api_key   = os.getenv('GOOGLE_API_KEY')
deepseek_api_key = os.getenv('DEEPSEEK_API_KEY')
groq_api_key     = os.getenv('GROQ_API_KEY')
```

El lab imprime los primeros caracteres de cada una para confirmar que se cargaron (sin enseñar la clave entera). Buena costumbre: **verifica que tus credenciales existen antes de usarlas**, te ahorra depurar a ciegas.

> 💡 Si no tienes alguna de estas claves, no pasa nada: salta esa parte. Todo el módulo se puede seguir solo con OpenAI; estos proveedores son para que veas **que se puede**.

### Conectar los modelos

El patrón es siempre el mismo: un `AsyncOpenAI` apuntando a la **URL base** del proveedor, envuelto en un `OpenAIChatCompletionsModel`.

```python
from openai import AsyncOpenAI
from agents import OpenAIChatCompletionsModel

GEMINI_BASE_URL   = "https://generativelanguage.googleapis.com/v1beta/openai/"
DEEPSEEK_BASE_URL = "https://api.deepseek.com/v1"
GROQ_BASE_URL     = "https://api.groq.com/openai/v1"

deepseek_client = AsyncOpenAI(base_url=DEEPSEEK_BASE_URL, api_key=deepseek_api_key)
gemini_client   = AsyncOpenAI(base_url=GEMINI_BASE_URL,   api_key=google_api_key)
groq_client     = AsyncOpenAI(base_url=GROQ_BASE_URL,     api_key=groq_api_key)

deepseek_model  = OpenAIChatCompletionsModel(model="deepseek-chat",            openai_client=deepseek_client)
gemini_model    = OpenAIChatCompletionsModel(model="gemini-2.0-flash",         openai_client=gemini_client)
llama3_3_model  = OpenAIChatCompletionsModel(model="llama-3.3-70b-versatile",  openai_client=groq_client)
```

Lee la receta, porque la vas a reutilizar siempre:

1. **Cliente** (`AsyncOpenAI`): el "teléfono" hacia el proveedor. Solo cambian `base_url` y `api_key`.
2. **Modelo** (`OpenAIChatCompletionsModel`): envuelve el cliente y dice qué modelo concreto usar.

Y ahora lo bonito: ese `model` se lo pasas a un `Agent` **igual** que antes pasabas `"gpt-4o-mini"`:

```python
sales_agent1 = Agent(name="DeepSeek Sales Agent", instructions=instructions1, model=deepseek_model)
sales_agent2 = Agent(name="Gemini Sales Agent",   instructions=instructions2, model=gemini_model)
sales_agent3 = Agent(name="Llama3.3 Sales Agent", instructions=instructions3, model=llama3_3_model)
```

> 💡 **Idea clave:** el `Agent` no sabe ni le importa qué proveedor hay detrás. Puedes mezclar un equipo con un agente de OpenAI, otro de Google y otro open-source, y orquestarlos juntos exactamente igual que en el Módulo 2. Esa portabilidad es oro.

> ⚠️ **Detalle del curso:** Gemini suele necesitar `GOOGLE_API_KEY` **y** `GEMINI_API_KEY` puestas al mismo valor. Si Gemini te falla, revisa eso primero.

A partir de aquí, el resto del sistema de ventas (tools, handoffs, manager) es **idéntico** al Módulo 2. Solo hemos cambiado el motor de cada agente.

---

## Parte 2 · Structured Outputs: datos con forma, no párrafos

### El problema

Imagina un agente cuyo trabajo es decidir: *"¿el usuario está metiendo el nombre de una persona en esta petición?"*. Si te responde *"Pues mira, sí, parece que aparece el nombre Alice..."*, **no puedes programar con eso**. Necesitas un `True`/`False` y, opcionalmente, el nombre. Datos, no prosa.

### La solución: Pydantic + `output_type`

Defines la **forma** del resultado con una clase Pydantic, y se la pasas al agente como `output_type`. El SDK obliga al modelo a rellenar exactamente esa estructura.

```python
from pydantic import BaseModel

class NameCheckOutput(BaseModel):
    is_name_in_message: bool
    name: str

guardrail_agent = Agent(
    name="Name check",
    instructions="Check if the user is including someone's personal name in what they want you to do.",
    output_type=NameCheckOutput,     # 👈 fuerza la estructura
    model="gpt-4o-mini",
)
```

Ahora, cuando ejecutes este agente, `result.final_output` **no será texto**: será un objeto `NameCheckOutput` con sus campos tipados. Podrás hacer `result.final_output.is_name_in_message` y obtener un booleano de verdad.

> 💡 **Por qué importa tanto:** los structured outputs son el **puente entre el mundo difuso del LLM y tu código rígido**. En cuanto un agente tiene que *alimentar* a otro paso de tu programa (un `if`, una base de datos, una API), querrás structured outputs. Lo verás otra vez, y a lo grande, en el Módulo 4.

---

## Parte 3 · Guardrails: barreras de seguridad

### La idea

Un guardrail es un control que se ejecuta **alrededor** de tu agente para vigilar lo que entra (`input_guardrail`) o lo que sale (`output_guardrail`). Si detecta algo prohibido, **dispara un "tripwire"** (un cable trampa) y **aborta** la ejecución antes de que pase nada malo.

Ejemplo del lab: no queremos que nuestro SDR procese peticiones que incluyan **nombres de personas** (pongamos que por política de privacidad). Vamos a poner un guardrail de entrada que lo bloquee.

### Paso 1 · El agente detector (ya lo tenemos)

Es el `guardrail_agent` de arriba: usa structured output para devolver `is_name_in_message: bool`. ¿Ves cómo encajan las piezas? El guardrail se apoya en el structured output.

### Paso 2 · La función guardrail

```python
from agents import input_guardrail, GuardrailFunctionOutput

@input_guardrail
async def guardrail_against_name(ctx, agent, message):
    result = await Runner.run(guardrail_agent, message, context=ctx.context)
    is_name_in_message = result.final_output.is_name_in_message
    return GuardrailFunctionOutput(
        output_info={"found_name": result.final_output},
        tripwire_triggered=is_name_in_message,    # 👈 si hay nombre, salta el cable trampa
    )
```

Desglose:

- `@input_guardrail` marca esta función como un guardrail **de entrada** (se ejecuta sobre el mensaje del usuario, antes que el agente principal).
- Dentro, ejecuta el `guardrail_agent` para analizar el mensaje y leer su `is_name_in_message`.
- Devuelve un `GuardrailFunctionOutput` con:
  - `output_info`: información útil para depurar/trazar.
  - `tripwire_triggered`: el booleano clave. Si es `True`, el SDK **detiene** la ejecución del agente protegido y lanza una excepción de guardrail.

### Paso 3 · Enganchar el guardrail al agente

```python
careful_sales_manager = Agent(
    name="Sales Manager",
    instructions=sales_manager_instructions,
    tools=tools,
    handoffs=[emailer_agent],
    model="gpt-4o-mini",
    input_guardrails=[guardrail_against_name],   # 👈 aquí se conecta
)
```

Probemos las dos caras:

```python
# ❌ Lleva un nombre ("Alice") → el guardrail SALTA y aborta
message = "Send out a cold sales email addressed to Dear CEO from Alice"
with trace("Protected Automated SDR"):
    result = await Runner.run(careful_sales_manager, message)
```

```python
# ✅ Sin nombre de persona → pasa sin problema
message = "Send out a cold sales email addressed to Dear CEO from Head of Business Development"
with trace("Protected Automated SDR"):
    result = await Runner.run(careful_sales_manager, message)
```

En el primer caso, el SDK lanza una excepción de tipo "guardrail tripwire triggered": el agente **ni siquiera empieza** a trabajar. En el segundo, todo fluye normal. Misión cumplida: pusiste una barrera real, no una súplica en el prompt.

> 💡 **Prompt ≠ guardrail.** Pedirle al modelo "por favor no proceses nombres" es una sugerencia que puede ignorar. Un guardrail es **código** que lo intercepta. Para seguridad de verdad, quieres lo segundo.

```
   usuario ──▶ [ input_guardrail ] ──┬── tripwire? sí ──▶ 🛑 abortado
                                      │
                                      └── no ──▶ agente protegido ──▶ resultado
```

---

## ⚠️ Errores comunes

- **Gemini falla.** Suele ser por la clave: necesita `GOOGLE_API_KEY` y `GEMINI_API_KEY` al mismo valor. Y revisa el nombre exacto del modelo.
- **`base_url` mal copiada.** Un `/v1` de más o de menos y el proveedor responde 404. Copia las URLs tal cual.
- **Esperar texto de un agente con `output_type`.** Si pusiste `output_type=...`, `final_output` es un **objeto**, no un string. Accede a sus campos (`.is_name_in_message`), no lo trates como texto.
- **Olvidar que el guardrail aborta.** Cuando salta el tripwire, se lanza una **excepción**. Si quieres manejarla con elegancia (mostrar un mensaje al usuario), envuélvelo en `try/except`.
- **Creer que el prompt protege.** Repито porque es importante: las instrucciones no son seguridad. Los guardrails sí.

---

## 🧪 Pruébalo tú

1. **Cambia de motor.** Coge el sistema de ventas del Módulo 2 y haz que **un** vendedor use Gemini o DeepSeek. Compara los emails: ¿notas diferencias de estilo entre modelos?
2. **Otro structured output.** Pídele al `sales_picker` que, además del email elegido, devuelva una estructura con `motivo: str` y `puntuacion: int`. Tendrás que crear su clase Pydantic y `output_type`.
3. **Un guardrail de salida.** Crea un `@output_guardrail` que dispare si el email generado contiene una palabra prohibida (p. ej. "garantizado"). Engánchalo con `output_guardrails=[...]`.
4. **Más reglas de entrada.** Amplía `guardrail_against_name` para bloquear también números de tarjeta o emails personales.

---

## 📌 Para llevar

- Cualquier proveedor con **endpoint compatible con OpenAI** se usa con `AsyncOpenAI(base_url=..., api_key=...)` + `OpenAIChatCompletionsModel(...)`, y el `Agent` ni se entera del cambio.
- **Structured Outputs** (`output_type=ClasePydantic`) convierten la salida difusa del LLM en **datos tipados** que tu código puede usar. Son el puente LLM↔código.
- **Guardrails** (`@input_guardrail` / `@output_guardrail`) **interceptan y abortan** entradas/salidas indebidas. `tripwire_triggered=True` corta la ejecución.
- Un guardrail suele apoyarse en un structured output (un detector que devuelve un booleano).
- **El prompt pide; el guardrail obliga.** Para seguridad real, usa guardrails.

---

[← Workflows, Tools y Handoffs](02_workflows_tools_handoffs.md) · [Volver al índice](README.md) · [Siguiente: Deep Research →](04_deep_research.md)
