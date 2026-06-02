# 👥 Curso Guiado: CrewAI — equipos de agentes por configuración

> Reescritura de la **Semana 3** del curso, en formato guiado para **leerse y entenderse sin video**.
> Yo (Claude) hago de profesor: el *qué*, el *cómo* y el *por qué*.

---

## 👋 Qué cambia aquí (y por qué te va a gustar)

En las semanas 1 y 2 escribías el bucle agéntico tú (a mano o con el OpenAI SDK). **CrewAI da un giro de filosofía**: en vez de programar la orquestación, **la describes en archivos de configuración**. Defines un equipo (*crew*) de agentes, cada uno con su rol, y una lista de tareas. CrewAI se encarga del resto.

> 💡 **La metáfora central:** CrewAI es como **montar un equipo de trabajo**. Cada agente es un empleado con su *rol*, su *objetivo* y su *historia* (backstory). Las *tareas* son el encargo. El *proceso* es cómo se coordinan (en fila, o con un jefe que delega). Tú haces de director de RRHH; CrewAI hace de oficina.

---

## 🗺️ El mapa del curso

Vamos a recorrer **5 proyectos reales**, de menos a más complejidad, y cada uno introduce una capacidad nueva:

```
  debate          financial_       stock_picker        engineering_team
  (2 agentes,     researcher       (jerárquico,        + coder
   secuencial,    (+ tools web,    + memoria, tool      (ejecutan
   multi-LLM)     + contexto)      propia, structured)  código real)
      │                │                  │                   │
   Módulo 1        Módulo 2           Módulo 3            Módulo 4
```

| # | Módulo | Proyecto(s) | Concepto estrella |
|---|--------|-------------|-------------------|
| [00](00_introduccion.md) | **Introducción** | — | Anatomía de una crew |
| [01](01_tu_primera_crew_debate.md) | **Tu primera crew** | `debate` | Agents · Tasks · Process sequential |
| [02](02_tools_y_contexto.md) | **Tools y contexto** | `financial_researcher` | Tools (web) · `context` |
| [03](03_jerarquia_memoria_tools.md) | **Jerarquía, memoria y tools propias** | `stock_picker` | Process hierarchical · memoria · `output_pydantic` |
| [04](04_equipos_de_software.md) | **Equipos que escriben software** | `engineering_team`, `coder` | Ejecución de código (Docker) |
| [05](05_patrones_y_chuleta.md) | **Patrones y chuleta** | — | Glosario + cheat sheet |

---

## ✅ Requisitos previos (¡CrewAI es especial!)

CrewAI **no** se ejecuta como el resto del repo. Lee esto con atención:

1. **El CLI de CrewAI es una herramienta `uv` aparte** (no está en el venv raíz):
   ```sh
   uv tool install crewai --python 3.12
   ```
2. **Cada proyecto tiene su propio entorno.** Para correr uno:
   ```sh
   cd 3_crew/<proyecto>      # p. ej. cd 3_crew/debate
   crewai install            # crea el venv del proyecto e instala deps
   crewai run                # ejecuta la crew
   ```
   > ⚠️ **No** uses `uv run` desde la raíz para los proyectos de CrewAI: tienen su propio `pyproject.toml` y `uv.lock`.
3. **Claves en el `.env` de la raíz** (los proyectos lo cargan):
   ```
   OPENAI_API_KEY=sk-...
   ANTHROPIC_API_KEY=...      # debate y engineering_team usan Claude
   SERPER_API_KEY=...         # Módulos 2-3: búsqueda web (serper.dev, gratis)
   PUSHOVER_USER=... PUSHOVER_TOKEN=...   # Módulo 3: notificación
   ```
4. **Docker Desktop** instalado y corriendo → solo para el Módulo 4 (ejecución de código en sandbox).

> 🔍 Cada proyecto se generó con `crewai create`. Si quieres crear el tuyo desde cero: `crewai create crew mi_proyecto`.

Empecemos. → [Módulo 00: Introducción](00_introduccion.md)

---

<sub>📚 Basado en la Semana 3 de *"Master AI Agentic Engineering"* de Edward Donner, reescrito y ampliado en formato guiado por Claude.</sub>
