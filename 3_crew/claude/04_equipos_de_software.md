# Módulo 04 · Equipos que escriben (y ejecutan) software

[← Jerarquía, memoria y tools propias](03_jerarquia_memoria_tools.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)

---

## 🎯 Objetivo

La demo que deja a todos con la boca abierta: una crew que **escribe código de verdad, lo ejecuta y lo prueba**, dentro de un sandbox seguro (Docker). Veremos dos proyectos: `coder` (un agente que programa) y `engineering_team` (un equipo de software completo: diseño → backend → frontend → tests).

> 📁 **Corresponde a:** `3_crew/coder/` y `3_crew/engineering_team/`.
> ⚠️ **Requiere Docker Desktop** instalado y corriendo.

---

## 🧠 La idea: agentes que ejecutan código

Hasta ahora los agentes producían texto. Ahora les damos un superpoder peligroso: **ejecutar código Python**. ¿Por qué es transformador? Porque cierra el bucle: el agente escribe código, lo **corre**, ve si falla, y lo arregla. Razonamiento + acción real.

¿Y la seguridad? No quieres que un LLM ejecute código arbitrario en tu máquina. Por eso CrewAI lo corre en un **contenedor Docker aislado** (`code_execution_mode="safe"`).

---

## 💻 Proyecto A · `coder`: un agente que programa solo

El más simple: **un** agente, **una** tarea, pero con ejecución de código activada.

```python
@agent
def coder(self) -> Agent:
    return Agent(
        config=self.agents_config['coder'],
        verbose=True,
        allow_code_execution=True,      # 👈 puede ejecutar código
        code_execution_mode="safe",     # 👈 dentro de Docker (aislado)
        max_execution_time=30,          # límite de segundos
        max_retry_limit=3,              # reintentos si falla
    )
```

Su `goal` describe el método de trabajo (planear → escribir → ejecutar → comprobar):

```yaml
coder:
  role: Python Developer
  goal: >
    You write python code to achieve this assignment: {assignment}
    First you plan how the code will work, then you write the code, then you run it and check the output.
  backstory: You're a seasoned python developer with a knack for writing clean, efficient code.
  llm: gpt-4o-mini
```

Y la tarea:

```yaml
coding_task:
  description: Write python code to achieve this: {assignment}
  expected_output: A text file that includes the code itself, along with the output of the code.
  agent: coder
  output_file: output/code_and_output.txt
```

El `main.py` le da un reto matemático (calcular π por una serie):

```python
assignment = 'Write a python program to calculate the first 10,000 terms of this series, \
    multiplying the total by 4: 1 - 1/3 + 1/5 - 1/7 + ...'
inputs = {'assignment': assignment}
result = Coder().crew().kickoff(inputs=inputs)
```

```sh
cd 3_crew/coder
crewai install
crewai run     # Docker debe estar corriendo
```

El agente **escribe el programa, lo ejecuta en Docker, lee el resultado** (≈ 3.14159...) y lo guarda. Un mini-ChatGPT-con-intérprete que montaste tú.

> 💡 Las cuatro opciones (`allow_code_execution`, `code_execution_mode`, `max_execution_time`, `max_retry_limit`) son tu **cinturón de seguridad**: aíslan, limitan el tiempo y permiten reintentar. Cópialas siempre que des ejecución de código.

---

## 💻 Proyecto B · `engineering_team`: un equipo de software completo

Aquí escalamos a **cuatro roles** que imitan un equipo real, en proceso secuencial:

```
  engineering_lead   →   backend_engineer   →   frontend_engineer   →   test_engineer
  (diseña el módulo)     (escribe el código,     (UI Gradio sobre        (escribe los
                          lo ejecuta en Docker)   el backend)             tests, los corre)
```

Los modelos están elegidos con criterio: el **lead** usa `gpt-4o` (diseño), y los **ingenieros** usan `claude-sonnet-4-5` (excelente escribiendo código).

```python
@agent
def engineering_lead(self) -> Agent:
    return Agent(config=self.agents_config['engineering_lead'], verbose=True)

@agent
def backend_engineer(self) -> Agent:
    return Agent(config=self.agents_config['backend_engineer'], verbose=True,
                 allow_code_execution=True, code_execution_mode="safe",
                 max_execution_time=500, max_retry_limit=3)
# frontend_engineer: sin ejecución (solo escribe la UI)
# test_engineer: con ejecución (corre los tests)
```

Fíjate en el detalle: **solo** los agentes que necesitan ejecutar (backend y test) llevan `allow_code_execution=True`. El lead (diseña) y el frontend (escribe UI) no. Das el permiso peligroso únicamente a quien lo necesita.

### Tareas encadenadas con `context` y salidas a ficheros

Cada tarea alimenta a la siguiente con `context`, y cada una escribe un fichero distinto:

```yaml
design_task:
  agent: engineering_lead
  output_file: output/{module_name}_design.md

code_task:
  agent: backend_engineer
  context: [design_task]              # 👈 codifica a partir del diseño
  output_file: output/{module_name}

frontend_task:
  agent: frontend_engineer
  context: [code_task]                # 👈 UI a partir del código
  output_file: output/app.py

test_task:
  agent: test_engineer
  context: [code_task]                # 👈 tests a partir del código
  output_file: output/test_{module_name}
```

Es el `context` del Módulo 2, pero formando un **pipeline de software**: diseño → código → (UI + tests). Las `{module_name}` se interpolan desde los inputs.

> 💡 **Truco de prompting que verás en las tareas:** *"Output ONLY the raw Python code without any markdown formatting, code block delimiters, or backticks."* ¿Por qué? Porque la salida se guarda **directamente como un `.py` ejecutable**. Si el LLM metiera ```` ```python ````, el fichero no correría. Cuando una salida va a ser un archivo de código, hay que pedir código crudo.

### El encargo

`main.py` pide un sistema de gestión de cuentas para una plataforma de trading simulado:

```python
requirements = """A simple account management system for a trading simulation platform.
Allow users to create an account, deposit/withdraw funds, record buy/sell of shares,
calculate portfolio value and profit/loss, report holdings and transactions...
Prevent withdrawing into negative balance, buying more than they can afford, or selling
shares they don't have. Has access to get_share_price(symbol)..."""
inputs = {'requirements': requirements, 'module_name': 'accounts.py', 'class_name': 'Account'}
```

```sh
cd 3_crew/engineering_team
crewai install
crewai run     # Docker corriendo; tarda, son 4 agentes
```

Al terminar, en `output/` tienes `accounts.py` (backend), `app.py` (UI Gradio), `test_accounts.py` (tests) y el diseño en markdown. **Un equipo de software entero, autónomo.** (Las carpetas `example_output_*` del proyecto muestran resultados de ejemplo con distintos modelos.)

> 🔭 ¿Te suena el `accounts.py` de gestión de cuentas? Es el **mismo dominio** que el capstone de MCP en la Semana 6 (`6_mcp`). Buen ejemplo de cómo una pieza generada por agentes alimenta otro proyecto.

---

## ⚠️ Errores comunes

- **Docker no está corriendo.** `allow_code_execution` + `safe` necesita Docker Desktop activo. Si no, falla al ejecutar.
- **El `.py` generado no corre por los backticks.** Asegúrate de que la tarea pide "raw code, no markdown".
- **Timeout.** Tareas pesadas pueden superar `max_execution_time`. Súbelo (en `engineering_team` ya es 500s).
- **Dar ejecución a agentes que no la necesitan.** Concede `allow_code_execution=True` solo a quien ejecuta; es un permiso sensible.
- **Coste/tiempo.** 4 agentes con modelos potentes tardan y cuestan. Normal.

---

## 🧪 Pruébalo tú

1. **Cambia el `assignment` de `coder`** por otro reto (ordenar una lista, un pequeño juego). Mira cómo planifica, ejecuta y corrige.
2. **Cambia los `requirements` de `engineering_team`** por una app distinta (una agenda de contactos, una to-do list). Observa el equipo construirla entera.
3. **Añade un quinto rol** `reviewer` que revise el código del backend antes de los tests, con `context: [code_task]`.

---

## 📌 Para llevar

- `allow_code_execution=True` + `code_execution_mode="safe"` da a un agente la capacidad de **ejecutar código en Docker aislado**. Acompáñalo de `max_execution_time` y `max_retry_limit`.
- Concede ejecución **solo** a los agentes que la necesitan (backend, tests), no a todos.
- Un **equipo de software** se modela con roles encadenados por `context` (diseño → código → UI/tests), cada uno escribiendo su `output_file`.
- Si una salida va a ser un `.py`, pide **código crudo sin markdown** en `expected_output`.
- Elige el modelo por rol: potente para diseñar/coordinar, fuerte en código para programar.

---

[← Jerarquía, memoria y tools propias](03_jerarquia_memoria_tools.md) · [Índice](README.md) · [Siguiente: Patrones y chuleta →](05_patrones_y_chuleta.md)
