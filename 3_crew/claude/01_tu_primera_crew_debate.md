# Módulo 01 · Tu primera crew: un debate

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: Tools y contexto →](02_tools_y_contexto.md)

---

## 🎯 Objetivo

Construir y ejecutar tu primera crew completa: un **debate**. Un mismo agente argumenta a favor y en contra de una moción, y un **juez** (con otro modelo) decide quién ganó. Verás el proceso `sequential` y el truco de usar **modelos distintos por agente**.

> 📁 **Corresponde a:** `3_crew/debate/`.

---

## 🧠 La idea

Un debate tiene tres pasos naturales: proponer, oponerse, juzgar. Es el ejemplo perfecto para un proceso **secuencial**: las tareas se ejecutan en orden, una tras otra. Y nos sirve para ver algo muy CrewAI: **dar a cada agente el modelo que mejor le va** (un modelo barato para argumentar, uno potente para juzgar).

---

## 💻 Los agentes (`config/agents.yaml`)

Solo dos agentes:

```yaml
debater:
  role: >
    A compelling debater
  goal: >
    Present a clear argument either in favor of or against the motion. The motion is: {motion}
  backstory: >
    You're an experienced debater with a knack for giving concise but convincing arguments.
    The motion is: {motion}
  llm: openai/gpt-4o-mini

judge:
  role: >
    Decide the winner of the debate based on the arguments presented
  goal: >
    Given arguments for and against this motion: {motion}, decide which side is more convincing,
    based purely on the arguments presented.
  backstory: >
    You are a fair judge with a reputation for weighing up arguments without factoring in
    your own views, and making a decision based purely on the merits of the argument.
  llm: anthropic/claude-sonnet-4-6
```

Fíjate en el detalle de oro: **`debater` usa `openai/gpt-4o-mini`** (barato, suficiente para argumentar) y **`judge` usa `anthropic/claude-sonnet-4-6`** (más potente, para una decisión matizada). En CrewAI, mezclar proveedores es tan fácil como cambiar una línea. Eliges la herramienta adecuada para cada trabajo.

> 💡 El `{motion}` aparece en varios sitios. CrewAI lo rellenará con el input que demos en `main.py`. El mismo equipo debate cualquier tema.

---

## 💻 Las tareas (`config/tasks.yaml`)

Tres encargos. Mira cómo `propose` y `oppose` van al mismo agente (`debater`), y `decide` al `judge`:

```yaml
propose:
  description: >
    You are proposing the motion: {motion}. Come up with a clear argument in favor. Be very convincing.
  expected_output: >
    Your clear argument in favor of the motion, in a concise manner.
  agent: debater
  output_file: output/propose.md

oppose:
  description: >
    You are in opposition to the motion: {motion}. Come up with a clear argument against. Be very convincing.
  expected_output: >
    Your clear argument against the motion, in a concise manner.
  agent: debater
  output_file: output/oppose.md

decide:
  description: >
    Review the arguments presented by the debaters and decide which side is more convincing.
  expected_output: >
    Your decision on which side is more convincing, and why.
  agent: judge
  output_file: output/decide.md
```

> 💡 **Un mismo agente, varias tareas.** El `debater` hace dos encargos opuestos (proponer y oponerse). Un agente no es de un solo uso: es un "rol" reutilizable. Y cada tarea guarda su resultado en un `output_file` distinto.

> 🤔 **¿Cómo ve el juez los argumentos?** En un proceso secuencial, el resultado de las tareas anteriores forma parte del contexto del trabajo. El juez ejecuta su tarea después de `propose` y `oppose`, así que tiene los argumentos disponibles. (En el Módulo 2 veremos cómo encadenar contexto de forma **explícita** con `context:`.)

---

## 💻 La crew (`crew.py`)

El pegamento. Registra los dos agentes, las tres tareas, y las ensambla en modo secuencial:

```python
from crewai import Agent, Crew, Process, Task
from crewai.project import CrewBase, agent, crew, task

@CrewBase
class Debate():
    agents_config = 'config/agents.yaml'
    tasks_config  = 'config/tasks.yaml'

    @agent
    def debater(self) -> Agent:
        return Agent(config=self.agents_config['debater'], verbose=True)

    @agent
    def judge(self) -> Agent:
        return Agent(config=self.agents_config['judge'], verbose=True)

    @task
    def propose(self) -> Task:
        return Task(config=self.tasks_config['propose'])

    @task
    def oppose(self) -> Task:
        return Task(config=self.tasks_config['oppose'])

    @task
    def decide(self) -> Task:
        return Task(config=self.tasks_config['decide'])

    @crew
    def crew(self) -> Crew:
        return Crew(
            agents=self.agents,    # se rellena solo con los @agent
            tasks=self.tasks,      # se rellena solo con los @task
            process=Process.sequential,
            verbose=True,
        )
```

- `verbose=True` hace que CrewAI imprima por consola cada paso (qué agente trabaja, qué piensa, qué produce). **Déjalo activado mientras aprendes:** es tu ventana al proceso.
- `Process.sequential` → las tareas corren **en el orden** en que las definiste: propose → oppose → decide.

---

## 💻 El arranque (`main.py`)

```python
from debate.crew import Debate

def run():
    inputs = {'motion': 'There needs to be strict laws to regulate LLMs'}
    result = Debate().crew().kickoff(inputs=inputs)
    print(result.raw)
```

`inputs` rellena el `{motion}` de todos los YAML. `kickoff` lanza el equipo. `result.raw` es la decisión final del juez.

### Ejecutarlo

```sh
cd 3_crew/debate
crewai install      # primera vez: crea el venv del proyecto
crewai run
```

Verás en consola al `debater` proponiendo, luego oponiéndose, y al `judge` (con Claude) emitiendo el veredicto. Y en `output/` quedan los tres `.md`.

---

## ⚠️ Errores comunes

- **`crewai: command not found`.** No instalaste el CLI: `uv tool install crewai --python 3.12`.
- **Lanzarlo con `uv run` desde la raíz.** No: `cd` al proyecto y usa `crewai run`. Cada proyecto tiene su propio venv.
- **Falta `ANTHROPIC_API_KEY`.** El juez usa Claude; sin la clave, falla. Ponla en el `.env` de la raíz (o cambia el `llm` del juez a un modelo de OpenAI).
- **`{motion}` aparece literal en la salida.** Olvidaste pasar `inputs={'motion': ...}` en el `kickoff`.

---

## 🧪 Pruébalo tú

1. **Cambia la moción** en `main.py` por un tema que te interese. Vuelve a correr.
2. **Cambia el juez de modelo.** Pon `openai/gpt-4o` o un modelo local. ¿Decide distinto?
3. **Añade un cuarto paso:** una tarea `summarize` para otro agente que resuma el debate en 3 puntos. Tendrás que añadirla en `tasks.yaml`, un agente en `agents.yaml`, y registrarla en `crew.py`.

---

## 📌 Para llevar

- Una crew secuencial ejecuta las tareas **en orden**: `Process.sequential`.
- **Cada agente puede usar un modelo distinto** (`llm:` en el YAML) — barato para tareas simples, potente para las críticas.
- Un mismo agente puede ejecutar **varias tareas** (es un rol reutilizable).
- Se ejecuta con `crewai install` (una vez) + `crewai run` **dentro** de la carpeta del proyecto.
- `kickoff(inputs=...)` rellena las `{variables}` y lanza el equipo; `result.raw` es el resultado final.

---

[← Introducción](00_introduccion.md) · [Índice](README.md) · [Siguiente: Tools y contexto →](02_tools_y_contexto.md)
