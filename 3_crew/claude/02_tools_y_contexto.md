# Módulo 02 · Tools y contexto: un investigador financiero

[← Tu primera crew](01_tu_primera_crew_debate.md) · [Índice](README.md) · [Siguiente: Jerarquía, memoria y tools propias →](03_jerarquia_memoria_tools.md)

---

## 🎯 Objetivo

Dar a tus agentes **herramientas** (buscar en la web) y encadenar tareas pasando el resultado de una como **contexto** de la siguiente. Construyes un equipo que investiga una empresa y redacta un informe.

> 📁 **Corresponde a:** `3_crew/financial_researcher/`.

---

## 🧠 La idea

Dos agentes con una división de trabajo clásica:
1. Un **researcher** que **busca en internet** datos de la empresa.
2. Un **analyst** que toma esa investigación y escribe un informe pulido.

Lo nuevo: el researcher necesita una **tool** (web), y el analyst necesita **recibir** lo que encontró el researcher. Eso es `context`.

---

## 💻 Los agentes y la tool de búsqueda

En el YAML los agentes son normales (role/goal/backstory). La novedad está en `crew.py`, donde le **enchufamos una tool** al researcher:

```python
from crewai_tools import SerperDevTool

@agent
def researcher(self) -> Agent:
    return Agent(
        config=self.agents_config['researcher'],
        verbose=True,
        tools=[SerperDevTool()],     # 👈 búsqueda web
    )

@agent
def analyst(self) -> Agent:
    return Agent(config=self.agents_config['analyst'], verbose=True)   # sin tools: solo escribe
```

- **`SerperDevTool`** es una herramienta lista de `crewai_tools` que busca en Google vía [serper.dev](https://serper.dev) (clave gratis). Necesita `SERPER_API_KEY` en tu `.env`.
- El `analyst` **no** lleva tools: su trabajo es razonar y redactar sobre lo que ya se investigó. No todo agente necesita herramientas.

> 💡 **`crewai_tools` trae muchas herramientas hechas** (búsqueda, scraping, ficheros, bases de datos...). Antes de programar una tool, mira si ya existe. (En el Módulo 3 sí construiremos una a medida.)

---

## 💻 El contexto: encadenar tareas explícitamente

Aquí está el concepto nuevo. La tarea de análisis declara que **depende** de la de investigación:

```yaml
research_task:
  description: >
    Conduct thorough research on company {company}. Focus on: current status, historical
    performance, challenges and opportunities, recent news, future outlook.
  expected_output: >
    A comprehensive research document with well-organized sections covering all requested aspects.
  agent: researcher

analysis_task:
  description: >
    Analyze the research findings and create a comprehensive report on {company}.
    Begin with an executive summary, include key info, analyze trends, offer a market outlook.
  expected_output: >
    A polished, professional report with executive summary, main sections, and conclusion.
  agent: analyst
  context:
    - research_task          # 👈 recibe el resultado de research_task como entrada
  output_file: output/report.md
```

El `context: [research_task]` es la clave: le dice a CrewAI *"cuando ejecutes `analysis_task`, pásale como contexto lo que produjo `research_task`"*. Así el analyst escribe sobre datos reales, no inventados.

```
   research_task  ──(su resultado)──▶  analysis_task  ──▶  output/report.md
   (busca en web)      [context]        (redacta informe)
```

> 💡 **`context` vs. orden secuencial:** en el Módulo 1, el juez "veía" los argumentos solo porque iba después. Aquí lo hacemos **explícito y robusto**: `context` declara exactamente qué resultado alimenta a qué tarea. Es la forma fiable de encadenar — el equivalente al *prompt chaining* de la Semana 1, pero declarativo.

---

## 💻 La crew y el arranque

```python
@crew
def crew(self) -> Crew:
    return Crew(agents=self.agents, tasks=self.tasks, process=Process.sequential, verbose=True)
```

```python
# main.py
import os
from financial_researcher.crew import ResearchCrew

os.makedirs('output', exist_ok=True)   # asegura que exista la carpeta de salida

def run():
    inputs = {'company': 'Apple'}
    result = ResearchCrew().crew().kickoff(inputs=inputs)
    print(result.raw)
```

Sigue siendo `Process.sequential`: research_task → analysis_task. La diferencia con el Módulo 1 es que ahora hay **tools** y **context** explícito.

### Ejecutarlo

```sh
cd 3_crew/financial_researcher
crewai install
crewai run
```

El researcher buscará en la web datos de Apple (lo verás en la consola por `verbose`), el analyst escribirá el informe, y quedará en `output/report.md`.

---

## ⚠️ Errores comunes

- **Falta `SERPER_API_KEY`.** Sin ella, `SerperDevTool` no busca. Regístrate gratis en serper.dev y añádela al `.env`.
- **El informe se inventa datos.** Si el researcher no usó la tool, el analyst no tiene datos reales. Revisa la consola: ¿aparece la búsqueda? Refuerza en la descripción "search online".
- **Olvidar `context`.** Sin `context: [research_task]`, el analyst no recibe la investigación de forma fiable.
- **No existe `output/`.** Por eso `main.py` hace `os.makedirs('output', exist_ok=True)`.

---

## 🧪 Pruébalo tú

1. **Cambia la empresa** a una que conozcas. Lee el `output/report.md`: ¿los datos son actuales? (gracias a la búsqueda web).
2. **Añade una tercera tarea** `fact_check` con otro agente que verifique el informe, usando `context: [analysis_task]`.
3. **Dale una tool al analyst** también (p. ej. otra de `crewai_tools`) y observa si la usa.

---

## 📌 Para llevar

- Se dan tools a un agente con `tools=[...]` en `crew.py`. `crewai_tools` trae muchas hechas (p. ej. `SerperDevTool` para web, requiere `SERPER_API_KEY`).
- No todo agente necesita tools: unos investigan, otros solo razonan/redactan.
- **`context: [otra_task]`** pasa el resultado de una tarea como entrada de otra — la forma declarativa y fiable de **encadenar** trabajo.
- Asegura la carpeta de salida con `os.makedirs('output', exist_ok=True)`.

---

[← Tu primera crew](01_tu_primera_crew_debate.md) · [Índice](README.md) · [Siguiente: Jerarquía, memoria y tools propias →](03_jerarquia_memoria_tools.md)
