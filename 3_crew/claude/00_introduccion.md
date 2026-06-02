# Módulo 00 · Introducción: la anatomía de una crew

[← Volver al índice](README.md) · [Siguiente: Tu primera crew →](01_tu_primera_crew_debate.md)

---

## 🎯 Objetivo

Entender las **cuatro piezas** de cualquier proyecto CrewAI antes de tocar uno. Una vez que las "veas", los cinco proyectos del curso son variaciones del mismo molde.

---

## 🧠 La idea: describir, no programar

En CrewAI no escribes un bucle. Escribes **quién** trabaja y **qué** hay que hacer, y dejas que el framework orqueste. La gran ventaja: los *qué* y *quién* viven en archivos **YAML** legibles, separados del código Python. Un perfil de no-programador puede ajustar los prompts sin tocar lógica.

```
  config/agents.yaml   →  los empleados (rol, objetivo, backstory, llm)
  config/tasks.yaml    →  los encargos (descripción, salida esperada, quién, orden)
  crew.py              →  el pegamento: conecta agentes + tareas en una "crew"
  main.py              →  el arranque: define los inputs y hace kickoff
```

---

## 🧩 Pieza 1 · Agents (`agents.yaml`)

Cada agente se define con **tres campos narrativos** y un modelo:

```yaml
debater:
  role: >
    A compelling debater
  goal: >
    Present a clear argument either in favor of or against the motion: {motion}
  backstory: >
    You're an experienced debater with a knack for giving concise but convincing arguments.
  llm: openai/gpt-4o-mini
```

| Campo | Para qué |
|---|---|
| `role` | Quién es (el "puesto"). |
| `goal` | Qué intenta conseguir. |
| `backstory` | Su "historia": da personalidad y contexto. CrewAI lo usa para construir el system prompt. |
| `llm` | Qué modelo usa (¡cada agente puede usar uno distinto!). |

> 💡 **Los `{motion}` son variables.** CrewAI las rellena ("interpola") en tiempo de ejecución con los `inputs` que pasas en `main.py`. Así el mismo agente sirve para cualquier moción.

---

## 🧩 Pieza 2 · Tasks (`tasks.yaml`)

Una tarea es un **encargo concreto** asignado a un agente:

```yaml
propose:
  description: >
    You are proposing the motion: {motion}. Come up with a clear argument in favor.
  expected_output: >
    Your clear argument in favor of the motion, in a concise manner.
  agent: debater
  output_file: output/propose.md
```

| Campo | Para qué |
|---|---|
| `description` | Qué hacer. |
| `expected_output` | Cómo debe ser el resultado (guía mucho la calidad). |
| `agent` | Qué agente la ejecuta. |
| `context` | (Opcional) Otras tareas cuyo resultado se le pasa como entrada → **encadenar**. |
| `output_file` | (Opcional) Dónde guardar el resultado. |

> 💡 **`agent` vs `role`:** en la tarea, `agent: debater` referencia la **clave** del agente en `agents.yaml`. Es como decir "este encargo es para Juan".

---

## 🧩 Pieza 3 · La Crew (`crew.py`)

Aquí Python conecta todo, usando **decoradores** que CrewAI provee:

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

    @task
    def propose(self) -> Task:
        return Task(config=self.tasks_config['propose'])

    @crew
    def crew(self) -> Crew:
        return Crew(agents=self.agents, tasks=self.tasks, process=Process.sequential, verbose=True)
```

Las claves:
- `@CrewBase` marca la clase como una crew y carga los YAML.
- `@agent` / `@task` registran cada agente/tarea. Fíjate: `Agent(config=self.agents_config['debater'])` **lee la configuración del YAML**.
- `@crew` ensambla todo. `self.agents` y `self.tasks` se rellenan **solos** a partir de los decoradores — no los listas a mano.
- `process=` decide la coordinación (lo vemos abajo).

---

## 🧩 Pieza 4 · El arranque (`main.py`)

Define los **inputs** (los valores de las `{variables}`) y lanza la crew con `kickoff`:

```python
from debate.crew import Debate

def run():
    inputs = {'motion': 'There needs to be strict laws to regulate LLMs'}
    result = Debate().crew().kickoff(inputs=inputs)
    print(result.raw)
```

`kickoff(inputs=...)` arranca todo el equipo. `result.raw` es el texto del resultado final.

---

## 🔀 El concepto clave: el `Process`

Cómo se coordinan los agentes. Hay dos modos, y entender la diferencia es media asignatura:

```
   SEQUENTIAL (en fila)              HIERARCHICAL (con jefe)

   Tarea1 → Tarea2 → Tarea3              ┌── Manager ──┐
   (orden fijo que defines tú)           │  delega y    │
                                         │  coordina    │
                                    Agente1  Agente2  Agente3
                                    (el manager decide quién y cuándo)
```

| Proceso | Quién decide el orden | Cuándo usarlo |
|---|---|---|
| `Process.sequential` | **Tú**, con el orden de las tareas | Pipelines claros y predecibles (Módulos 1-2) |
| `Process.hierarchical` | **Un agente "manager"** que delega | Cuando quieres que el equipo se auto-organice (Módulo 3) |

> 💡 ¿Te suena? Es la misma tensión **workflow vs. agente autónomo** de la Semana 2. `sequential` = workflow (tú mandas). `hierarchical` = un manager-agente decide. CrewAI te da las dos con cambiar una palabra.

---

## 📌 Para llevar

- Un proyecto CrewAI = **`agents.yaml`** (quién) + **`tasks.yaml`** (qué) + **`crew.py`** (pegamento) + **`main.py`** (arranque).
- Un agente se describe con **role / goal / backstory / llm**; cada uno puede usar un modelo distinto.
- Una tarea tiene **description / expected_output / agent**, y opcionalmente **context** (encadenar) y **output_file**.
- Los decoradores `@CrewBase`, `@agent`, `@task`, `@crew` conectan los YAML con Python; `self.agents`/`self.tasks` se rellenan solos.
- El **`Process`** decide la coordinación: `sequential` (tú mandas) o `hierarchical` (un manager delega).

---

[← Volver al índice](README.md) · [Siguiente: Tu primera crew →](01_tu_primera_crew_debate.md)
