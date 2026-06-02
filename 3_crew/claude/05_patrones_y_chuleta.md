# Módulo 05 · Patrones y chuleta (cheat sheet)

[← Equipos que escriben software](04_equipos_de_software.md) · [Volver al índice](README.md)

---

## 🎯 Para qué sirve

Tu página de referencia de CrewAI. Glosario, estructura de un proyecto, y el código mínimo de cada capacidad.

---

## 📖 Glosario rápido

| Término | En una frase |
|---|---|
| **Crew** | El equipo: agentes + tareas + proceso. |
| **Agent** | Empleado con `role` / `goal` / `backstory` / `llm`. |
| **Task** | Encargo con `description` / `expected_output` / `agent`. |
| **Process** | Cómo se coordinan: `sequential` o `hierarchical`. |
| **Manager** | Agente que delega en `hierarchical` (`manager_agent` + `allow_delegation`). |
| **`context`** | Pasar el resultado de una tarea como entrada de otra (encadenar). |
| **`output_file`** | Guardar el resultado de una tarea en disco. |
| **`output_pydantic`** | Forzar la salida de una tarea a un objeto Pydantic. |
| **Tool** | Capacidad externa: de catálogo (`crewai_tools`) o propia (`BaseTool`). |
| **Memory** | short-term (RAG), long-term (SQLite), entity. |
| **Code execution** | `allow_code_execution` + Docker (`safe`). |
| **`{variable}`** | Hueco en el YAML que se rellena con `inputs` del `kickoff`. |
| **`kickoff(inputs=...)`** | Arranca la crew; `result.raw` es el resultado. |

---

## 📁 Anatomía de un proyecto

```
mi_proyecto/
├── pyproject.toml            # deps propias (venv separado del repo)
└── src/mi_proyecto/
    ├── config/
    │   ├── agents.yaml       # quién: role / goal / backstory / llm
    │   └── tasks.yaml        # qué: description / expected_output / agent / context
    ├── crew.py               # pegamento: @CrewBase, @agent, @task, @crew
    ├── main.py               # arranque: inputs + kickoff
    └── tools/                # tools propias (BaseTool)
```

Crear uno nuevo: `crewai create crew mi_proyecto`

---

## 💻 La chuleta de código

### Agente (YAML)

```yaml
mi_agente:
  role: >
    Qué es
  goal: >
    Qué intenta lograr, usando {variable}
  backstory: >
    Su historia y personalidad
  llm: openai/gpt-4o-mini      # o anthropic/claude-..., gpt-4o, etc.
```

### Tarea (YAML)

```yaml
mi_tarea:
  description: >
    Qué hacer con {variable}
  expected_output: >
    Cómo debe ser el resultado
  agent: mi_agente
  context: [tarea_previa]      # opcional: encadenar
  output_file: output/algo.md  # opcional: guardar
```

### Crew secuencial (`crew.py`)

```python
from crewai import Agent, Crew, Process, Task
from crewai.project import CrewBase, agent, crew, task

@CrewBase
class MiCrew():
    agents_config = 'config/agents.yaml'
    tasks_config  = 'config/tasks.yaml'

    @agent
    def mi_agente(self) -> Agent:
        return Agent(config=self.agents_config['mi_agente'], verbose=True)

    @task
    def mi_tarea(self) -> Task:
        return Task(config=self.tasks_config['mi_tarea'])

    @crew
    def crew(self) -> Crew:
        return Crew(agents=self.agents, tasks=self.tasks,
                    process=Process.sequential, verbose=True)
```

### Crew jerárquica (con manager)

```python
@crew
def crew(self) -> Crew:
    manager = Agent(config=self.agents_config['manager'], allow_delegation=True)
    return Crew(agents=self.agents, tasks=self.tasks,
                process=Process.hierarchical, manager_agent=manager, verbose=True)
```

### Tool de catálogo

```python
from crewai_tools import SerperDevTool
Agent(config=..., tools=[SerperDevTool()])   # requiere SERPER_API_KEY
```

### Tool propia

```python
from crewai.tools import BaseTool
from pydantic import BaseModel, Field
from typing import Type

class MisArgs(BaseModel):
    texto: str = Field(..., description="...")

class MiTool(BaseTool):
    name: str = "Nombre visible"
    description: str = "Qué hace y cuándo usarla"
    args_schema: Type[BaseModel] = MisArgs
    def _run(self, texto: str) -> str:
        ...
        return "ok"
```

### Structured output en una tarea

```python
class Salida(BaseModel):
    campo: str = Field(description="...")

Task(config=..., output_pydantic=Salida)
```

### Ejecución de código (Docker)

```python
Agent(config=..., allow_code_execution=True, code_execution_mode="safe",
      max_execution_time=30, max_retry_limit=3)
```

### Arranque (`main.py`)

```python
from mi_proyecto.crew import MiCrew
inputs = {'variable': 'valor'}
result = MiCrew().crew().kickoff(inputs=inputs)
print(result.raw)
```

---

## 🐛 Tabla de "no me funciona"

| Síntoma | Causa probable | Arreglo |
|---|---|---|
| `crewai: command not found` | CLI no instalado | `uv tool install crewai --python 3.12` |
| Falla con `uv run` desde la raíz | Proyecto tiene su propio venv | `cd` al proyecto + `crewai run` |
| `{variable}` aparece literal | Falta `inputs` en kickoff | Pasa `kickoff(inputs={...})` |
| `SerperDevTool` no busca | Falta `SERPER_API_KEY` | Regístrate en serper.dev, añádela al `.env` |
| El juez/Claude falla | Falta `ANTHROPIC_API_KEY` | Añádela o cambia el `llm` |
| Code execution falla | Docker parado | Arranca Docker Desktop |
| `.py` generado no corre | Salida con backticks markdown | Pide "raw code, no markdown" |
| Errores de memoria | Falta clave de embeddings | `OPENAI_API_KEY` (para `text-embedding-3-small`) |
| Jerárquico muy caro/lento | El manager razona mucho | Reduce alcance o usa secuencial |

---

## 🧠 Cómo decidir: secuencial o jerárquico

```
   ¿El orden de las tareas es claro y siempre el mismo?
        │
        ├── Sí ──▶ Process.sequential   ✅ predecible ✅ barato ✅ fácil de depurar
        │
        └── No, quiero que el equipo se auto-organice ──▶ Process.hierarchical
                    (manager_agent + allow_delegation)   ✅ flexible ❌ más caro/impredecible
```

---

## 🧩 Los patrones de esta semana

| Patrón | Dónde |
|---|---|
| **Roles especializados** | Todos los proyectos |
| **Pipeline secuencial** | debate, financial_researcher |
| **Encadenar por `context`** | financial_researcher, engineering_team |
| **Multi-modelo por rol** | debate, engineering_team |
| **Orquestador/manager (jerárquico)** | stock_picker |
| **Structured outputs** | stock_picker |
| **Tools (catálogo y propias)** | financial_researcher, stock_picker |
| **Memoria persistente** | stock_picker |
| **Ejecución de código** | coder, engineering_team |

---

## ✅ Checklist de "ya lo domino"

- [ ] Entiendo las 4 piezas: agents.yaml, tasks.yaml, crew.py, main.py (Mód. 0)
- [ ] Sé definir agentes (role/goal/backstory/llm) y tareas (Mód. 0-1)
- [ ] Sé correr una crew: `crewai install` + `crewai run` en el proyecto (Mód. 1)
- [ ] Sé dar un modelo distinto a cada agente (Mód. 1)
- [ ] Sé dar tools de catálogo y encadenar con `context` (Mód. 2)
- [ ] Sé el proceso jerárquico con manager y `allow_delegation` (Mód. 3)
- [ ] Sé forzar structured outputs con `output_pydantic` (Mód. 3)
- [ ] Sé crear una tool propia con `BaseTool` (Mód. 3)
- [ ] Entiendo los tipos de memoria (Mód. 3)
- [ ] Sé activar ejecución de código en Docker de forma segura (Mód. 4)

Si marcaste todo: dominas CrewAI. 🎉

---

[← Equipos que escriben software](04_equipos_de_software.md) · [Volver al índice](README.md)
