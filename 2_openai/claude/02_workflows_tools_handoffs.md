# Módulo 02 · Workflows, Tools y Handoffs

[← Tu primer agente](01_tu_primer_agente.md) · [Volver al índice](README.md) · [Siguiente: Modelos, Outputs y Guardrails →](03_modelos_structured_outputs_guardrails.md)

---

## 🎯 Objetivo

Aquí pasas de "un agente que responde" a **un sistema de agentes que colaboran**. Vamos a construir, paso a paso, un equipo automático de ventas (un *SDR* — Sales Development Representative) que:

1. Genera varios borradores de email en **paralelo**.
2. Elige el mejor.
3. Le da formato y **lo envía**.

Por el camino aprenderás las tres herramientas de colaboración del SDK: **paralelismo**, **tools** (incluido el truco de convertir un agente en tool) y **handoffs**.

> 📓 **Corresponde a:** `2_openai/2_lab2.ipynb` (Week 2 · Day 2).

---

## 🧠 La idea: workflow vs. agente

Antes de programar, una distinción que te va a servir toda la vida (es de Anthropic, y aparece en el ejercicio del lab):

- **Workflow**: *tú* decides el orden de los pasos. "Primero genera, luego elige, luego envía." El código manda; el LLM solo rellena cada hueco.
- **Agente (autónomo)**: le das el objetivo y *él* decide qué pasos dar y en qué orden, usando sus tools.

Vamos a construir **los dos** en este módulo, y verás que la diferencia es a veces **una sola línea**. Empezamos con un workflow (control nuestro) y terminamos con un agente que se autogestiona.

---

## 📧 Sobre el envío de emails (SendGrid) — léelo, es opcional

Para "enviar de verdad" usaremos **SendGrid** (un servicio de envío de emails, gratis para empezar). **Pero es totalmente opcional.** Si no quieres pelearte con ello, puedes:

- Saltarte el envío y simplemente **imprimir** el email por pantalla.
- O usar la alternativa "Resend" que hay en `community_contributions/`.

Si quieres usar SendGrid:
1. Crea cuenta en https://sendgrid.com
2. **Settings → API Keys → Create API Key**, copia la clave.
3. Añade a tu `.env`: `SENDGRID_API_KEY=xxxx`
4. **Settings → Sender Authentication → Verify a Single Sender**: verifica tu propio email para poder enviar desde él.

> ⚠️ En el código verás emails como `info@innovatiox.com` o `ed@edwarddonner.com`. **Cámbialos por el tuyo verificado**, o no enviará nada.

Un test rápido de que funciona:

```python
import sendgrid, os
from sendgrid.helpers.mail import Mail, Email, To, Content

def send_test_email():
    sg = sendgrid.SendGridAPIClient(api_key=os.environ.get('SENDGRID_API_KEY'))
    from_email = Email("tu_email_verificado@ejemplo.com")  # 👈 cámbialo
    to_email = To("tu_email_verificado@ejemplo.com")       # 👈 cámbialo
    content = Content("text/plain", "This is an important test email")
    mail = Mail(from_email, to_email, "Test email", content).get()
    response = sg.client.mail.send.post(request_body=mail)
    print(response.status_code)   # 202 = ¡bien!

send_test_email()
```

Si imprime `202`, todo en orden. Si no llega, **mira la carpeta de Spam** (le pasa a mucha gente).

---

## Parte 1 · Workflow: generar, comparar y elegir

### Tres vendedores con tres personalidades

La gracia de los agentes es que el *mismo modelo* se comporta distinto según sus instrucciones. Creamos tres vendedores: uno serio, uno gracioso y uno conciso.

```python
instructions1 = "You are a sales agent ... You write professional, serious cold emails."
instructions2 = "You are a humorous, engaging sales agent ... You write witty, engaging cold emails ..."
instructions3 = "You are a busy sales agent ... You write concise, to the point cold emails."

sales_agent1 = Agent(name="Professional Sales Agent", instructions=instructions1, model="gpt-4o-mini")
sales_agent2 = Agent(name="Engaging Sales Agent",     instructions=instructions2, model="gpt-4o-mini")
sales_agent3 = Agent(name="Busy Sales Agent",         instructions=instructions3, model="gpt-4o-mini")
```

> 💡 Tres agentes, tres "personalidades", el mismo modelo barato. Las **instructions** son tu palanca de diseño más potente y más barata.

### Streaming: ver el texto según se genera

Antes de lanzar los tres, un truco simpático. En vez de esperar a que el agente termine, puedes ver el texto **aparecer token a token**, como en ChatGPT:

```python
result = Runner.run_streamed(sales_agent1, input="Write a cold sales email")
async for event in result.stream_events():
    if event.type == "raw_response_event" and isinstance(event.data, ResponseTextDeltaEvent):
        print(event.data.delta, end="", flush=True)
```

- `Runner.run_streamed(...)` en vez de `Runner.run(...)` te devuelve un flujo de **eventos**.
- Recorres los eventos con `async for`. Hay muchos tipos; aquí filtramos solo los `ResponseTextDeltaEvent`, que son los **trocitos de texto** (`delta`) según se generan.
- `end="", flush=True` hacen que se imprima seguido y al instante, sin saltos de línea ni buffer.

No es esencial para la lógica, pero mejora mucho la sensación de "está vivo". Útil para interfaces.

### Paralelismo: los tres a la vez ⚡

Aquí es donde la asincronía paga. En vez de ejecutar los tres vendedores uno tras otro (lento), los lanzamos **simultáneamente** con `asyncio.gather`:

```python
import asyncio

message = "Write a cold sales email"

with trace("Parallel cold emails"):
    results = await asyncio.gather(
        Runner.run(sales_agent1, message),
        Runner.run(sales_agent2, message),
        Runner.run(sales_agent3, message),
    )

outputs = [result.final_output for result in results]
```

- `asyncio.gather(...)` lanza las tres corutinas **a la vez** y espera a que terminen todas.
- Como cada `Runner.run` pasa la mayor parte del tiempo *esperando* a OpenAI, ejecutarlas en paralelo es casi tan rápido como ejecutar una sola. Tres emails por el precio (en tiempo) de uno.
- `outputs` es la lista de los tres textos finales.

> 📌 Acuérdate de esto: cuando tengas N tareas independientes que esperan a una API, `asyncio.gather` es tu mejor amigo.

### Elegir el mejor: un agente "juez"

Tenemos tres borradores. ¿Cuál enviamos? Que decida... otro agente.

```python
sales_picker = Agent(
    name="sales_picker",
    instructions="You pick the best cold sales email from the given options. \
Imagine you are a customer and pick the one you are most likely to respond to. \
Do not give an explanation; reply with the selected email only.",
    model="gpt-4o-mini",
)
```

Y lo usamos así: concatenamos los tres emails en un solo texto y se lo pasamos al juez.

```python
with trace("Selection from sales people"):
    results = await asyncio.gather(
        Runner.run(sales_agent1, message),
        Runner.run(sales_agent2, message),
        Runner.run(sales_agent3, message),
    )
    outputs = [result.final_output for result in results]

    emails = "Cold sales emails:\n\n" + "\n\nEmail:\n\n".join(outputs)
    best = await Runner.run(sales_picker, emails)

    print(f"Best sales email:\n{best.final_output}")
```

Acabas de construir tu primer patrón compuesto: **generar en paralelo + seleccionar**. Esto, en la jerga, es un patrón *"LLM as a judge"*. Y todo sigue siendo un **workflow**: el orden lo decidimos nosotros en código.

---

## Parte 2 · Tools: darle manos al agente

Hasta ahora los agentes solo hablan. Para que **hagan** cosas (enviar un email, consultar una base de datos, llamar a una API) necesitan **herramientas**.

### El decorador `@function_tool` (adiós al boilerplate)

¿Recuerdas el horror del JSON a mano para describir una función al modelo? Tipo, nombre, descripción, parámetros, tipos... El SDK lo genera por ti. Solo decoras tu función:

```python
@function_tool
def send_email(body: str):
    """ Send out an email with the given body to all sales prospects """
    sg = sendgrid.SendGridAPIClient(api_key=os.environ.get('SENDGRID_API_KEY'))
    from_email = Email("tu_email_verificado@ejemplo.com")  # 👈 cámbialo
    to_email = To("tu_email_verificado@ejemplo.com")       # 👈 cámbialo
    content = Content("text/plain", body)
    mail = Mail(from_email, to_email, "Sales email", content).get()
    sg.client.mail.send.post(request_body=mail)
    return {"status": "success"}
```

¿Qué hace el decorador por debajo? Mira lo que pasa si imprimes `send_email`:

```python
FunctionTool(
    name='send_email',
    description='Send out an email with the given body to all sales prospects',
    params_json_schema={'properties': {'body': {'title': 'Body', 'type': 'string'}}, ...},
    ...
)
```

¡Ahí está el JSON que **no tuviste que escribir**! El SDK lo dedujo solo. Fíjate de dónde sacó cada cosa:

| El SDK lo tomó de... | Para generar... |
|---|---|
| El **nombre** de la función (`send_email`) | `name` |
| El **docstring** (`""" ... """`) | `description` — ¡por eso el docstring importa, el modelo lo lee! |
| Los **type hints** (`body: str`) | El esquema de parámetros |

> ⚠️ **Lección práctica:** en una `@function_tool`, el **docstring y los type hints no son documentación, son la interfaz**. El LLM decide cuándo y cómo llamar a tu función leyéndolos. Un docstring vago = un agente que la usa mal.

### El truco estrella: convertir un agente en tool

Aquí viene algo precioso. Un **agente entero** puede convertirse en una herramienta para *otro* agente:

```python
description = "Write a cold sales email"

tool1 = sales_agent1.as_tool(tool_name="sales_agent1", tool_description=description)
tool2 = sales_agent2.as_tool(tool_name="sales_agent2", tool_description=description)
tool3 = sales_agent3.as_tool(tool_name="sales_agent3", tool_description=description)
```

`.as_tool(...)` envuelve un agente como si fuera una función. Ahora un agente "jefe" puede *llamar* a los tres vendedores como si fueran tools. Esto es la base de la **jerarquía de agentes**: jefes que orquestan a subordinados.

Juntamos todo en una lista de herramientas:

```python
tools = [tool1, tool2, tool3, send_email]
```

### El Sales Manager: nuestro primer agente autónomo

Ahora, en lugar de orquestar nosotros en código, le damos el objetivo a **un único agente** con acceso a todas las tools y dejamos que *él* decida:

```python
instructions = """
You are a Sales Manager at ComplAI. Your goal is to find the single best cold sales email using the sales_agent tools.

Follow these steps carefully:
1. Generate Drafts: Use all three sales_agent tools to generate three different email drafts. Do not proceed until all three drafts are ready.
2. Evaluate and Select: Review the drafts and choose the single best email using your judgment of which one is most effective.
3. Use the send_email tool to send the best email (and only the best email) to the user.

Crucial Rules:
- You must use the sales agent tools to generate the drafts — do not write them yourself.
- You must send ONE email using the send_email tool — never more than one.
"""

sales_manager = Agent(name="Sales Manager", instructions=instructions, tools=tools, model="gpt-4o-mini")

message = "Send a cold sales email addressed to 'Dear CEO'"

with trace("Sales manager"):
    result = await Runner.run(sales_manager, message)
```

🎉 **Esto ya es un agente, no un workflow.** Nosotros no encadenamos los pasos: le dimos un objetivo y unas herramientas, y él decide llamarlas (los tres vendedores, comparar, y `send_email`). El bucle agéntico del Módulo 00 en acción real.

> 💡 **Sobre las instrucciones:** fíjate en lo explícitas que son ("usa las tres tools", "envía solo UNO"). Con agentes autónomos, las instrucciones son **gobernanza**: cuanto más claro el procedimiento y las reglas, más fiable el agente. Vale la pena iterarlas.

> ⚠️ **¿No te llegó el email?** (1) Revisa Spam. (2) Haz `print(result)` y busca errores. (3) Si ves `SSL: CERTIFICATE_VERIFY_FAILED`, ejecuta en terminal `uv pip install --upgrade certifi` y luego, en Python:
> ```python
> import certifi, os
> os.environ['SSL_CERT_FILE'] = certifi.where()
> ```
> (4) Mira la traza en OpenAI y el panel de SendGrid para pistas.

---

## Parte 3 · Handoffs: pasar el mando

Llegamos al concepto que más confunde y que conviene tener clarísimo. Repasemos la frase del Módulo 00:

> - **Tool:** el agente llama, recibe el resultado y **sigue mandando él**. El control *vuelve*.
> - **Handoff:** el agente **cede el mando** a otro agente. El control *pasa* y no vuelve.

```
   TOOL                              HANDOFF
   ┌──────────┐                      ┌──────────┐
   │ Agente A │                      │ Agente A │
   └────┬─────┘                      └────┬─────┘
        │ llama tool                      │ handoff
        ▼                                 ▼
   ┌──────────┐                      ┌──────────┐
   │  tool /  │                      │ Agente B │  ◀── ahora manda B
   │ agente B │                      │ (termina │
   └────┬─────┘                      │  él)     │
        │ devuelve                   └──────────┘
        ▼
   ┌──────────┐
   │ Agente A │  ◀── sigue mandando A
   └──────────┘
```

### Montemos el "Email Manager" como destino de un handoff

Primero, dos sub-agentes especialistas (uno escribe el asunto, otro convierte a HTML), convertidos en tools:

```python
subject_writer = Agent(name="Email subject writer", instructions=subject_instructions, model="gpt-4o-mini")
subject_tool = subject_writer.as_tool(tool_name="subject_writer", tool_description="Write a subject for a cold sales email")

html_converter = Agent(name="HTML email body converter", instructions=html_instructions, model="gpt-4o-mini")
html_tool = html_converter.as_tool(tool_name="html_converter", tool_description="Convert a text email body to an HTML email body")
```

Una tool de función para enviar el email ya formateado en HTML:

```python
@function_tool
def send_html_email(subject: str, html_body: str) -> Dict[str, str]:
    """ Send out an email with the given subject and HTML body to all sales prospects """
    sg = sendgrid.SendGridAPIClient(api_key=os.environ.get('SENDGRID_API_KEY'))
    from_email = Email("tu_email_verificado@ejemplo.com")  # 👈 cámbialo
    to_email = To("tu_email_verificado@ejemplo.com")       # 👈 cámbialo
    content = Content("text/html", html_body)
    mail = Mail(from_email, to_email, subject, content).get()
    sg.client.mail.send.post(request_body=mail)
    return {"status": "success"}
```

Y ahora el **Email Manager**, un agente que tiene esas tres tools y un trabajo muy concreto:

```python
instructions = "You are an email formatter and sender. You receive the body of an email to be sent. \
You first use the subject_writer tool to write a subject for the email, then use the html_converter tool to convert the body to HTML. \
Finally, you use the send_html_email tool to send the email with the subject and HTML body."

emailer_agent = Agent(
    name="Email Manager",
    instructions=instructions,
    tools=[subject_tool, html_tool, send_html_email],
    model="gpt-4o-mini",
    handoff_description="Convert an email to HTML and send it",   # 👈 clave para handoffs
)
```

> 💡 **`handoff_description`** es la descripción que ve *el agente que va a delegar*, para decidir si le pasa el control a este. Es al handoff lo que el `tool_description` es a una tool: cómo se "anuncia" este agente ante los demás.

### El Sales Manager con handoff

Ahora el Sales Manager tiene **tools** (los tres vendedores) **y un handoff** (el Email Manager):

```python
tools = [tool1, tool2, tool3]
handoffs = [emailer_agent]

sales_manager_instructions = """
You are a Sales Manager at ComplAI. Your goal is to find the single best cold sales email using the sales_agent tools.

Follow these steps carefully:
1. Generate Drafts: Use all three sales_agent tools to generate three different email drafts. Do not proceed until all three drafts are ready.
2. Evaluate and Select: Review the drafts and choose the single best email using your judgment of which one is most effective.
   You can use the tools multiple times if you're not satisfied with the results from the first try.
3. Handoff for Sending: Pass ONLY the winning email draft to the 'Email Manager' agent. The Email Manager will take care of formatting and sending.

Crucial Rules:
- You must use the sales agent tools to generate the drafts — do not write them yourself.
- You must hand off exactly ONE email to the Email Manager — never more than one.
"""

sales_manager = Agent(
    name="Sales Manager",
    instructions=sales_manager_instructions,
    tools=tools,
    handoffs=handoffs,           # 👈 la novedad
    model="gpt-4o-mini",
)

message = "Send out a cold sales email addressed to Dear CEO from Alice"

with trace("Automated SDR"):
    result = await Runner.run(sales_manager, message)
```

Lee el flujo completo y saboréalo:

```
Sales Manager
  ├─ tool: sales_agent1  ─┐
  ├─ tool: sales_agent2   ├─ genera 3 borradores
  ├─ tool: sales_agent3  ─┘
  ├─ (elige el mejor)
  └─ HANDOFF ▶ Email Manager   ◀── cede el control
                 ├─ tool: subject_writer
                 ├─ tool: html_converter
                 └─ tool: send_html_email  ▶ ✉️ enviado
```

El Sales Manager **usa** a los vendedores (control vuelve) pero **cede** al Email Manager (control pasa). Esa es exactamente la diferencia entre tool y handoff, ahora vista en un sistema real.

---

## 🧩 Los patrones de diseño que acabas de usar

Sin darte cuenta, en este módulo has aplicado varios **patrones agénticos** clásicos. Ponerles nombre te ayuda a reconocerlos en el futuro:

| Patrón | Dónde apareció |
|---|---|
| **Paralelización** | Lanzar los 3 vendedores con `asyncio.gather` |
| **LLM as a judge** | El `sales_picker` eligiendo el mejor email |
| **Agente como herramienta** | `.as_tool()` sobre los vendedores |
| **Orquestador / sub-agentes** | El Sales Manager dirigiendo el equipo |
| **Handoff / delegación** | Sales Manager → Email Manager |

Y la pregunta del millón del ejercicio: **¿cuál es la línea que convirtió esto de "workflow" en "agente"?** Respuesta: el momento en que dejamos de encadenar pasos en código y le dimos las `tools` (y luego `handoffs`) a un agente con un objetivo, dejándole decidir. Ahí el control pasó de nuestro código al razonamiento del modelo.

---

## ⚠️ Errores comunes

- **El docstring vacío o vago en una `@function_tool`.** El modelo no sabrá cuándo usar tu tool. Escríbelo como si se lo explicaras a un becario listo pero sin contexto.
- **Olvidar cambiar los emails de SendGrid.** `from_email` debe ser tu **sender verificado**, o no envía.
- **Confundir `tools=` con `handoffs=`.** Si quieres que el control vuelva, es tool. Si quieres ceder el control, es handoff. Releer el diagrama de arriba.
- **Instrucciones flojas en un agente autónomo.** Si no le dices "usa las TRES tools" y "envía solo UNO", puede saltarse pasos o duplicar. Sé explícito con el procedimiento y las reglas.
- **No mirar la traza.** Con 5+ agentes, la traza en platform.openai.com/traces es la única forma sana de ver qué llamó a qué.

---

## 🧪 Pruébalo tú

1. **Sin email.** Sustituye el cuerpo de `send_email` por un simple `print(body)` y quita SendGrid. Todo el sistema sigue funcionando: enviar es solo "una tool más".
2. **Añade un cuarto vendedor** con otra personalidad (p. ej. "muy técnico y orientado a datos"). Conviértelo en tool y dáselo al manager. ¿Lo usa?
3. **Identifica los patrones** en tu propia traza: localiza visualmente dónde están las 3 llamadas en paralelo y dónde el handoff.
4. **Reto (difícil):** investiga cómo hacer que SendGrid llame a un *webhook* cuando alguien responde al email, y que el SDR continúe la conversación. Esto ya es vibe-coding del bueno 😄

---

## 📌 Para llevar

- `asyncio.gather` ejecuta varios agentes **en paralelo** — ideal cuando esperan a una API.
- `@function_tool` convierte una función Python en herramienta **sin JSON a mano**; el SDK usa el **nombre, el docstring y los type hints**.
- `.as_tool()` convierte un **agente entero** en herramienta de otro agente → jerarquías de agentes.
- **Tool** = el control vuelve. **Handoff** = el control pasa. `handoff_description` anuncia un agente ante los que pueden delegarle.
- Pasaste de **workflow** (orden en tu código) a **agente** (orden decidido por el modelo) casi sin cambiar nada: solo dándole tools/handoffs y un objetivo.

---

[← Tu primer agente](01_tu_primer_agente.md) · [Volver al índice](README.md) · [Siguiente: Modelos, Outputs y Guardrails →](03_modelos_structured_outputs_guardrails.md)
