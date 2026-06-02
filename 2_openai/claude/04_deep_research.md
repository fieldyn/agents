# Módulo 04 · Deep Research: tu investigador autónomo

[← Modelos, Outputs y Guardrails](03_modelos_structured_outputs_guardrails.md) · [Volver al índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)

---

## 🎯 Objetivo

Juntar **todo** lo aprendido para construir un agente que de verdad impresiona: un **Deep Research**. Le das una pregunta y él, solo:

1. **Planifica** qué búsquedas hacer.
2. **Busca** en la web (varias búsquedas en paralelo).
3. **Escribe** un informe largo y coherente.
4. Te lo **envía** por email.

Es uno de los casos de uso agénticos más valiosos que existen, aplicable a cualquier sector. Y, atención: lo vas a montar con piezas que **ya conoces**.

> 📓 **Corresponde a:** `2_openai/4_lab4.ipynb` (Deep Research). El proyecto completo y con interfaz está en `2_openai/deep_research/`.

---

## 🧠 La idea: descomponer un problema grande

Un humano experto, ante "investiga X", no hace una sola búsqueda gigante. Hace algo así:

```
   pregunta
      │
      ▼
  ┌─────────┐   "¿qué necesito saber?"   → planifica N búsquedas
  │ plannear │
  └────┬─────┘
       ▼
  ┌─────────┐   ejecuta las N búsquedas (¡en paralelo!)
  │ buscar  │
  └────┬─────┘
       ▼
  ┌─────────┐   sintetiza todo en un informe coherente
  │ escribir │
  └────┬─────┘
       ▼
  ┌─────────┐   entrega (email)
  │ enviar  │
  └─────────┘
```

Cada caja será **un agente especializado**. La clave del diseño agéntico: **descomponer** una tarea compleja en sub-tareas, cada una con su agente experto. Más fácil de construir, de depurar y de mejorar por partes.

---

## 🌐 Las hosted tools (herramientas que aloja OpenAI)

En el Módulo 2 hicimos tools con `@function_tool` (funciones nuestras). OpenAI también ofrece **herramientas alojadas**, listas para usar:

| Hosted tool | Para qué |
|---|---|
| `WebSearchTool` | Buscar en la web |
| `FileSearchTool` | Recuperar info de tus Vector Stores de OpenAI |
| `ComputerTool` | Automatizar uso del ordenador (capturas, clicks) |

Hoy usamos `WebSearchTool`.

> 💰 **Aviso de coste (importante).** `WebSearchTool` de OpenAI cuesta dinero — en el curso, ~2,5 céntimos por llamada, y a veces más si hace varias búsquedas por llamada. En los próximos labs del curso se usan buscadores gratuitos/baratos. Si el coste te preocupa, **puedes saltarte la ejecución** y quedarte con la lectura, o reducir el número de búsquedas. Precios: https://platform.openai.com/docs/pricing#web-search

---

## 💻 El código, agente por agente

### Agente 1 · El buscador (`search_agent`)

Recibe un término, busca en la web y **resume**. Fíjate en dos detalles de diseño muy buenos en sus instrucciones.

```python
from agents import Agent, WebSearchTool
from agents.model_settings import ModelSettings

INSTRUCTIONS = "You are a research assistant. Given a search term, you search the web for that term and \
produce a concise summary of the results. The summary must 2-3 paragraphs and less than 300 \
words. Capture the main points. Write succintly, no need to have complete sentences or good \
grammar. This will be consumed by someone synthesizing a report, so it's vital you capture the \
essence and ignore any fluff. Do not include any additional commentary other than the summary itself."

search_agent = Agent(
    name="Search agent",
    instructions=INSTRUCTIONS,
    tools=[WebSearchTool(search_context_size="low")],
    model="gpt-4o-mini",
    model_settings=ModelSettings(tool_choice="required"),
)
```

Dos decisiones de diseño que vale la pena copiar:

- **`WebSearchTool(search_context_size="low")`** — "low" = menos contexto recuperado = **más barato**. Un buen default mientras aprendes.
- **`ModelSettings(tool_choice="required")`** — fuerza al agente a **usar la herramienta sí o sí**. No le dejamos "responder de memoria"; lo obligamos a buscar de verdad. Esto es control fino del comportamiento.

> 💡 Las instrucciones piden resúmenes **telegráficos** ("sin necesidad de frases completas ni buena gramática"). ¿Por qué? Porque otro agente los va a consumir, no un humano. Optimizas el texto para su lector real: ahorras tokens y vas al grano.

### Agente 2 · El planificador (`planner_agent`) — con structured output

Aquí vuelve, a lo grande, el structured output del Módulo 3. El planificador no devuelve texto: devuelve una **lista estructurada de búsquedas**.

```python
from pydantic import BaseModel, Field

HOW_MANY_SEARCHES = 3

INSTRUCTIONS = f"You are a helpful research assistant. Given a query, come up with a set of web searches \
to perform to best answer the query. Output {HOW_MANY_SEARCHES} terms to query for."

class WebSearchItem(BaseModel):
    reason: str = Field(description="Your reasoning for why this search is important to the query.")
    query: str  = Field(description="The search term to use for the web search.")

class WebSearchPlan(BaseModel):
    searches: list[WebSearchItem] = Field(description="A list of web searches to perform to best answer the query.")

planner_agent = Agent(
    name="PlannerAgent",
    instructions=INSTRUCTIONS,
    model="gpt-4o-mini",
    output_type=WebSearchPlan,    # 👈 devuelve un WebSearchPlan, no texto
)
```

Mira el poder de combinar piezas: `output_type=WebSearchPlan` garantiza que `result.final_output.searches` sea una **lista de objetos** con `.query` y `.reason`. Podemos iterarla en código sin parsear nada. El `Field(description=...)` además le explica al modelo qué poner en cada campo. Limpio y robusto.

### Agente 3 · El escritor (`writer_agent`) — también estructurado

Recibe la pregunta original y los resúmenes, y produce un informe largo. Su salida también es estructurada, con tres campos pensados para ser útiles:

```python
INSTRUCTIONS = (
    "You are a senior researcher tasked with writing a cohesive report for a research query. "
    "You will be provided with the original query, and some initial research done by a research assistant.\n"
    "You should first come up with an outline for the report that describes the structure and "
    "flow of the report. Then, generate the report and return that as your final output.\n"
    "The final output should be in markdown format, and it should be lengthy and detailed. Aim "
    "for 5-10 pages of content, at least 1000 words."
)

class ReportData(BaseModel):
    short_summary: str            = Field(description="A short 2-3 sentence summary of the findings.")
    markdown_report: str          = Field(description="The final report")
    follow_up_questions: list[str] = Field(description="Suggested topics to research further")

writer_agent = Agent(
    name="WriterAgent",
    instructions=INSTRUCTIONS,
    model="gpt-4o-mini",
    output_type=ReportData,
)
```

> 💡 Las instrucciones le piden **primero un esquema y luego el informe**. Es el "piensa antes de escribir" aplicado a un agente: estructurar antes de redactar mejora mucho la coherencia de textos largos. Un truco de prompting que deberías robar.

### Agente 4 · El emisor (`email_agent`)

Igual que en el Módulo 2: una `@function_tool` para enviar, y un agente que la usa para mandar el informe en HTML.

```python
@function_tool
def send_email(subject: str, html_body: str) -> Dict[str, str]:
    """ Send out an email with the given subject and HTML body """
    sg = sendgrid.SendGridAPIClient(api_key=os.environ.get('SENDGRID_API_KEY'))
    from_email = Email("tu_email_verificado@ejemplo.com")  # 👈 cámbialo
    to_email   = To("tu_email@ejemplo.com")                # 👈 cámbialo
    content = Content("text/html", html_body)
    mail = Mail(from_email, to_email, subject, content).get()
    sg.client.mail.send.post(request_body=mail)
    return "success"

INSTRUCTIONS = """You are able to send a nicely formatted HTML email based on a detailed report.
You will be provided with a detailed report. You should use your tool to send one email, providing the
report converted into clean, well presented HTML with an appropriate subject line."""

email_agent = Agent(name="Email agent", instructions=INSTRUCTIONS, tools=[send_email], model="gpt-4o-mini")
```

---

## 🔗 La orquestación: funciones que pegan los agentes

Aquí viene una decisión de diseño interesante: en vez de un único "agente jefe" con handoffs (como en el Módulo 2), el lab orquesta **desde código Python**, con funciones `async` que envuelven cada agente. Es el enfoque **workflow** del Módulo 2, y es perfectamente válido — a veces quieres tú el control del flujo.

```python
async def plan_searches(query: str):
    """ Usa el planner_agent para decidir qué búsquedas hacer """
    print("Planning searches...")
    result = await Runner.run(planner_agent, f"Query: {query}")
    print(f"Will perform {len(result.final_output.searches)} searches")
    return result.final_output      # un WebSearchPlan

async def perform_searches(search_plan: WebSearchPlan):
    """ Lanza todas las búsquedas EN PARALELO """
    print("Searching...")
    tasks = [asyncio.create_task(search(item)) for item in search_plan.searches]
    results = await asyncio.gather(*tasks)
    print("Finished searching")
    return results

async def search(item: WebSearchItem):
    """ Ejecuta UNA búsqueda con el search_agent """
    input = f"Search term: {item.query}\nReason for searching: {item.reason}"
    result = await Runner.run(search_agent, input)
    return result.final_output
```

Fíjate en `perform_searches`: crea una tarea por cada búsqueda del plan y las lanza con `asyncio.gather`. **El paralelismo del Módulo 2, otra vez.** Si el plan tiene 3 búsquedas, se hacen las 3 a la vez. Reconocer estos patrones repetidos es exactamente el objetivo del curso.

```python
async def write_report(query: str, search_results: list[str]):
    """ El writer_agent sintetiza el informe """
    print("Thinking about report...")
    input = f"Original query: {query}\nSummarized search results: {search_results}"
    result = await Runner.run(writer_agent, input)
    print("Finished writing report")
    return result.final_output      # un ReportData

async def send_email(report: ReportData):
    """ El email_agent envía el informe """
    print("Writing email...")
    result = await Runner.run(email_agent, report.markdown_report)
    print("Email sent")
    return report
```

### ¡Showtime! El pipeline completo

```python
query = "Latest AI Agent frameworks in 2025"

with trace("Research trace"):
    print("Starting research...")
    search_plan    = await plan_searches(query)        # 1. planifica
    search_results = await perform_searches(search_plan)  # 2. busca (en paralelo)
    report         = await write_report(query, search_results)  # 3. escribe
    await send_email(report)                            # 4. envía
    print("Hooray!")
```

Léelo en voz alta: planifica → busca → escribe → envía. Cuatro agentes especializados, encadenados por ti, con búsquedas en paralelo en medio, y **una sola traza** que lo envuelve todo para que puedas inspeccionar el viaje completo en platform.openai.com/traces.

> 💡 **Workflow vs. agente, otra vez.** Aquí *tú* decides el orden (`plan → search → write → email`). Podrías hacerlo "más agéntico" dándole los cuatro como tools/handoffs a un orquestador y dejándole decidir. Ninguno es mejor en absoluto: el workflow es **predecible y fácil de depurar**; el agente autónomo es **flexible**. Elegir entre ambos es una de las decisiones de diseño más importantes que tomarás.

---

## ⚠️ Errores comunes

- **Sustos en la factura.** `WebSearchTool` cuesta por llamada. Baja `HOW_MANY_SEARCHES` a 1-2 mientras pruebas, usa `search_context_size="low"`, y no lo dejes en bucle.
- **Pasar el objeto Pydantic donde se espera texto** (o al revés). El planner devuelve un `WebSearchPlan` (accede a `.searches`); el writer devuelve un `ReportData` (accede a `.markdown_report`). Ten claro qué devuelve cada paso.
- **Dos funciones llamadas `send_email`.** En el lab hay una `@function_tool send_email` (la tool de bajo nivel) y una `async def send_email(report)` (el paso del pipeline). La segunda redefine el nombre en el notebook. No te líes: son capas distintas. Si te molesta, renómbralas (p. ej. `send_report_email`).
- **El email no llega.** Mismas comprobaciones del Módulo 2: sender verificado, carpeta de Spam, errores SSL/certifi.
- **No envolver en `trace`.** Sin la traza, depurar un pipeline de 4 agentes es a ciegas. Envuélvelo siempre.

---

## 🧪 Pruébalo tú

1. **Cambia la pregunta** por algo de tu interés real. Mira el plan que genera: ¿las búsquedas tienen sentido? Lee los `reason`.
2. **Sube las búsquedas** a `HOW_MANY_SEARCHES = 5` (ojo al coste) y observa cómo `perform_searches` las paraleliza en la traza.
3. **Sin email.** Cambia el último paso por `display(Markdown(report.markdown_report))` para verlo en el notebook en vez de enviarlo. Cero coste de SendGrid.
4. **Usa los `follow_up_questions`.** El `ReportData` los incluye. Encadena: coge una pregunta de seguimiento y vuelve a lanzar el pipeline con ella. Acabas de hacer investigación recursiva 🤯
5. **Reto:** convierte el pipeline en un agente autónomo. Da los cuatro pasos como tools/handoffs a un "Research Manager" con un buen prompt y deja que él decida el flujo. Compara robustez vs. el workflow.

---

## 🎓 ¿Qué acabas de lograr?

Para en serio un momento. Has construido un **agente de Deep Research** con uno de los frameworks más modernos que existen. Y lo importante: no aprendiste nada *nuevo* en este módulo. Reusaste todo:

- **Agentes especializados** (Módulo 1)
- **Tools y paralelismo** (Módulo 2)
- **Structured outputs** (Módulo 3)
- **Trazas** para verlo todo (desde el Módulo 1)

Eso es lo que significa "dominar" un framework: que un proyecto que impresiona se construya combinando piezas que ya entiendes. Si esto te ha hecho clic, el curso ha cumplido su objetivo.

---

## 📌 Para llevar

- **Descomponer** una tarea grande en agentes especializados (planificar / buscar / escribir / enviar) es el núcleo del diseño agéntico.
- `WebSearchTool` y otras **hosted tools** vienen listas — pero `WebSearchTool` **cuesta dinero**; contrólalo.
- `tool_choice="required"` **obliga** a usar la herramienta; `search_context_size="low"` abarata.
- Los **structured outputs** (`WebSearchPlan`, `ReportData`) son la columna vertebral: permiten que la salida de un agente alimente el siguiente paso sin parsear texto.
- Puedes orquestar con **código** (workflow, predecible) o con **un agente** (autónomo, flexible). Elegir es una decisión de diseño, no un accidente.
- Todo el proyecto reutiliza piezas de los módulos anteriores: eso es dominar el framework.

---

[← Modelos, Outputs y Guardrails](03_modelos_structured_outputs_guardrails.md) · [Volver al índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)
