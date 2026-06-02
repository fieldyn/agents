# 🤝 Curso Guiado: AutoGen — de agentes simples a un mundo distribuido

> Reescritura de la **Semana 5** del curso, en formato guiado para **leerse y entenderse sin video**.
> Yo (Claude) hago de profesor: el *qué*, el *cómo* y el *por qué*.

---

## 👋 La gran idea de AutoGen: dos capas

AutoGen (de Microsoft) es distinto a los anteriores porque son, en realidad, **dos frameworks en uno**:

```
   ┌─────────────────────────────────────────────────────────┐
   │  AutoGen AgentChat   (capa ALTA, fácil)                  │
   │  Se parece a CrewAI y al OpenAI SDK: agentes, tools,     │
   │  equipos. Para construir rápido.                         │
   ├─────────────────────────────────────────────────────────┤
   │  AutoGen Core        (capa BAJA, potente)                │
   │  Un "runtime" de mensajería entre agentes (modelo actor).│
   │  Agnóstico, escalable, y... DISTRIBUIDO (varias máquinas)│
   └─────────────────────────────────────────────────────────┘
```

> 💡 **La intuición:** **AgentChat** es para escribir un agente o un equipo en minutos (como ya sabes hacer). **Core** baja un nivel: te da la *infraestructura de comunicación* para que muchos agentes se manden mensajes, incluso **repartidos por varias máquinas** vía red. Es la diferencia entre "montar un equipo" y "montar el sistema de correo de toda la empresa".

---

## 🗺️ El mapa del curso

```
   AgentChat        AgentChat          AutoGen Core      Distribuido +
   básico           avanzado           (el runtime)      el mundo de agentes
   (agente+tools)   (equipos,          (modelo actor,    (gRPC, un Creator
                     multimodal,        message passing)  que crea agentes)
                     structured)
       │                │                   │                  │
    Módulo 1         Módulo 2            Módulo 3           Módulo 4
```

| # | Módulo | Aprenderás a... | Concepto estrella |
|---|--------|-----------------|-------------------|
| [00](00_introduccion.md) | **Introducción** | Situar las dos capas | AgentChat vs Core |
| [01](01_agentchat_basico.md) | **AgentChat básico** | Crear un agente con tools | `AssistantAgent` · `on_messages` |
| [02](02_agentchat_avanzado.md) | **AgentChat avanzado** | Equipos, multimodal, structured | `RoundRobinGroupChat` · MCP |
| [03](03_autogen_core.md) | **AutoGen Core** | Agentes que se pasan mensajes | `RoutedAgent` · runtime |
| [04](04_distribuido_y_el_mundo.md) | **Distribuido + el mundo** | Un sistema multi-agente por red | gRPC · agentes que crean agentes |
| [05](05_patrones_y_chuleta.md) | **Patrones y chuleta** | Tenerlo a mano | Glosario + cheat sheet |

---

## ✅ Requisitos previos

1. **Entorno** (desde la raíz): `uv sync`
2. **`.env`** en la raíz:
   ```
   OPENAI_API_KEY=sk-...
   SERPER_API_KEY=...      # Módulos 2-4: búsqueda web
   ```
3. **Ollama** (opcional) para modelos locales — Módulos 1 y 3.
4. **Node + `uvx`** para las tools MCP del Módulo 2 (`mcp-server-fetch`).
5. **Kernel** `.venv (Python 3.12)`.

> ⚠️ **Usuarios de Windows:** los servidores MCP (Módulo 2) y partes de la Semana 6 **no funcionan nativamente en Windows** (problema conocido). La solución es usar **WSL** (Linux sobre Windows): guía en `setup/SETUP-WSL.md`. Mac y Linux van directos.

> ⚠️ **El Módulo 4 abre un puerto de red** (gRPC en `localhost:50051`) y se ejecuta con `uv run world.py`, no en notebook.

Empecemos. → [Módulo 00: Introducción](00_introduccion.md)

---

<sub>📚 Basado en la Semana 5 de *"Master AI Agentic Engineering"* de Edward Donner, reescrito y ampliado en formato guiado por Claude.</sub>
