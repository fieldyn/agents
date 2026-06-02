# Módulo 03 · Jerarquía, memoria y tools propias: el stock picker

[← Tools y contexto](02_tools_y_contexto.md) · [Índice](README.md) · [Siguiente: Equipos que escriben software →](04_equipos_de_software.md)

---

## 🎯 Objetivo

El proyecto más completo de CrewAI. Reúne cuatro capacidades nuevas a la vez: un proceso **jerárquico** con un **manager** que delega, **structured outputs** con Pydantic, una **tool propia** (notificación push), y **memoria** persistente. Construyes un equipo que detecta empresas en tendencia, las investiga y elige la mejor para invertir.

> 📁 **Corresponde a:** `3_crew/stock_picker/`.

---

## 🧠 La idea

Tres especialistas y un jefe:
1. **trending_company_finder** — busca en noticias empresas que están dando que hablar.
2. **financial_researcher** — investiga a fondo cada una.
3. **stock_picker** — elige la mejor y te avisa al móvil.
4. **manager** — no hace trabajo directo: **coordina y delega** en los otros.

Es el salto de "fila india" (secuencial) a "equipo con jefe" (jerárquico).

---

## 💻 Capacidad 1 · Structured outputs con `output_pydantic`

En la Semana 1 forzábamos formato con Pydantic a mano. CrewAI lo integra: defines la forma y se la pasas a la tarea con `output_pydantic`.

```python
from pydantic import BaseModel, Field
from typing import List

class TrendingCompany(BaseModel):
    name: str   = Field(description="Company name")
    ticker: str = Field(description="Stock ticker symbol")
    reason: str = Field(description="Reason this company is trending in the news")

class TrendingCompanyList(BaseModel):
    companies: List[TrendingCompany] = Field(description="List of companies trending in the news")
```

```python
@task
def find_trending_companies(self) -> Task:
    return Task(
        config=self.tasks_config['find_trending_companies'],
        output_pydantic=TrendingCompanyList,    # 👈 la salida será este objeto, no texto libre
    )
```

> 💡 **Por qué importa en una crew:** la siguiente tarea recibe esto por `context`. Si el resultado es un objeto estructurado (lista de empresas con `name`/`ticker`/`reason`), el siguiente agente trabaja con datos limpios en vez de tener que "entender" un párrafo. Structured output = engranaje fiable entre tareas.

---

## 💻 Capacidad 2 · Una tool propia (`BaseTool`)

En el Módulo 2 usamos una tool de catálogo. Ahora hacemos **la nuestra**: una notificación push. Se crea heredando de `BaseTool`:

```python
from crewai.tools import BaseTool
from typing import Type
from pydantic import BaseModel, Field
import os, requests

class PushNotification(BaseModel):
    """A message to be sent to the user"""
    message: str = Field(..., description="The message to be sent to the user.")

class PushNotificationTool(BaseTool):
    name: str = "Send a Push Notification"
    description: str = "This tool is used to send a push notification to the user."
    args_schema: Type[BaseModel] = PushNotification     # 👈 esquema de argumentos

    def _run(self, message: str) -> str:
        payload = {"user": os.getenv("PUSHOVER_USER"),
                   "token": os.getenv("PUSHOVER_TOKEN"),
                   "message": message}
        requests.post("https://api.pushover.net/1/messages.json", data=payload)
        return '{"notification": "ok"}'
```

Anatomía de una tool de CrewAI:
- `name` y `description`: lo que el LLM lee para decidir cuándo usarla.
- `args_schema`: un modelo Pydantic que define los argumentos (aquí, `message`).
- `_run(...)`: el código que se ejecuta. Aquí, el mismo POST a Pushover de la Semana 1.

Y se la damos al agente que debe avisarte:

```python
@agent
def stock_picker(self) -> Agent:
    return Agent(config=self.agents_config['stock_picker'],
                 tools=[PushNotificationTool()], memory=True)
```

---

## 💻 Capacidad 3 · El proceso jerárquico (un manager que delega)

Aquí está el cambio gordo. En vez de un orden fijo, definimos un **manager** con permiso para delegar, y ponemos el proceso en `hierarchical`:

```python
@crew
def crew(self) -> Crew:
    manager = Agent(
        config=self.agents_config['manager'],
        allow_delegation=True,        # 👈 puede repartir trabajo a los demás
    )
    return Crew(
        agents=self.agents,           # los 3 especialistas (NO el manager)
        tasks=self.tasks,
        process=Process.hierarchical, # 👈 el manager coordina
        manager_agent=manager,        # 👈 quién manda
        verbose=True,
        memory=True,
        # ...config de memoria (abajo)...
    )
```

Diferencias con el secuencial:
- El `manager` se crea aparte y se pasa como `manager_agent`. **No** va en la lista `agents`.
- `allow_delegation=True` le da el poder de asignar tareas a los especialistas.
- `Process.hierarchical`: ahora **el manager decide** quién hace qué y cuándo, en lugar de un orden que fijas tú.

```
                ┌──────── manager (gpt-4o) ────────┐
                │   "necesito empresas trending" →  │ delega
        ┌───────┴───────┐         ┌─────────────────┴──┐
   trending_finder    financial_researcher        stock_picker
   (busca noticias)   (investiga c/u)        (elige + push 📱)
```

> 💡 **Secuencial vs. jerárquico, la decisión:** el secuencial es predecible y fácil de depurar; el jerárquico es flexible y se auto-organiza, pero menos predecible (y suele gastar más, porque el manager razona mucho). Es la misma elección **workflow vs. agente** de toda la formación. Fíjate además en que el manager usa `gpt-4o` (potente) mientras los obreros usan `gpt-4o-mini`: el que coordina necesita más cabeza.

---

## 💻 Capacidad 4 · Memoria

La crew activa **memoria** para recordar entre ejecuciones (p. ej. "no vuelvas a elegir la misma empresa"):

```python
from crewai.memory import LongTermMemory, ShortTermMemory, EntityMemory
from crewai.memory.storage.rag_storage import RAGStorage
from crewai.memory.storage.ltm_sqlite_storage import LTMSQLiteStorage

# dentro de Crew(...):
memory=True,
long_term_memory  = LongTermMemory(storage=LTMSQLiteStorage(db_path="./memory/long_term_memory_storage.db")),
short_term_memory = ShortTermMemory(storage=RAGStorage(embedder_config={"provider":"openai","config":{"model":"text-embedding-3-small"}}, type="short_term", path="./memory/")),
entity_memory     = EntityMemory(storage=RAGStorage(embedder_config={"provider":"openai","config":{"model":"text-embedding-3-small"}}, type="short_term", path="./memory/")),
```

Tres tipos de memoria, cada uno con su papel:

| Tipo | Qué recuerda | Cómo se guarda |
|---|---|---|
| **Short-term** | El contexto de la ejecución actual | RAG (búsqueda semántica con embeddings) |
| **Long-term** | Conocimiento entre sesiones (persiste) | SQLite en disco |
| **Entity** | Datos sobre "entidades" clave (empresas, personas) | RAG |

> 💡 No necesitas memorizar la config exacta (es copy-paste de la doc). Quédate con la **idea**: CrewAI puede dar a tu equipo memoria de corto plazo, de largo plazo y de entidades, usando embeddings + SQLite. Por eso el `goal` puede decir "no elijas la misma empresa dos veces" y funcionar entre ejecuciones.

> ⚠️ La carpeta `./memory/*.db` está en `.gitignore`: es estado en disco, no código.

---

## 💻 El arranque

```python
from datetime import datetime
from stock_picker.crew import StockPicker

def run():
    inputs = {'sector': 'Technology', 'current_date': str(datetime.now())}
    result = StockPicker().crew().kickoff(inputs=inputs)
    print(result.raw)
```

```sh
cd 3_crew/stock_picker
crewai install
crewai run    # necesita OPENAI, SERPER y PUSHOVER en el .env
```

---

## ⚠️ Errores comunes

- **El manager en la lista `agents`.** No: el `manager_agent` va aparte. Si lo metes en `agents`, se lía.
- **`allow_delegation=False` en el manager.** Sin delegación, el proceso jerárquico no reparte trabajo.
- **No llega el push.** Revisa `PUSHOVER_USER`/`TOKEN` y que la app esté instalada (igual que en la Semana 1).
- **Errores de memoria/embeddings.** La memoria usa `text-embedding-3-small` de OpenAI: necesitas `OPENAI_API_KEY` aunque tus agentes usen otro modelo.
- **Coste alto.** El jerárquico hace muchas llamadas (el manager razona mucho). Empieza con un sector y pocas empresas.

---

## 🧪 Pruébalo tú

1. **Cambia el sector** a "Healthcare" o "Energy". Corre dos veces y comprueba que la memoria evita repetir empresa.
2. **Crea otra tool propia** (p. ej. una que guarde la decisión en un fichero CSV) heredando de `BaseTool`, y dásela al `stock_picker`.
3. **Convierte el proyecto a secuencial** (quita el manager, pon `Process.sequential`). Compara: ¿más rápido? ¿menos flexible?

---

## 📌 Para llevar

- **`output_pydantic=Modelo`** fuerza la salida de una tarea a un objeto estructurado → engranaje fiable entre tareas vía `context`.
- Una **tool propia** se crea heredando de `BaseTool`: `name`, `description`, `args_schema` (Pydantic) y `_run(...)`.
- **`Process.hierarchical` + `manager_agent` (con `allow_delegation=True`)** = un jefe-agente que delega, en vez de un orden fijo. El manager va aparte de `agents`.
- CrewAI ofrece **memoria** short-term (RAG), long-term (SQLite) y entity. Necesita embeddings de OpenAI.
- Jerárquico = flexible pero más caro/impredecible; secuencial = predecible y barato. Elige según el caso.

---

[← Tools y contexto](02_tools_y_contexto.md) · [Índice](README.md) · [Siguiente: Equipos que escriben software →](04_equipos_de_software.md)
