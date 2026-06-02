# Módulo 00 · Introducción: las dos capas de AutoGen

[← Volver al índice](README.md) · [Siguiente: AgentChat básico →](01_agentchat_basico.md)

---

## 🎯 Objetivo

Entender **dónde estás parado** en cada momento de esta semana: cuándo usas la capa fácil (AgentChat) y cuándo bajas a la potente (Core). Si tienes claro el mapa, no te perderás.

---

## 🧠 La idea: una capa para construir, otra para escalar

### AutoGen AgentChat (la capa alta)

Es lo que ya sabes hacer, con otra sintaxis. Defines un `AssistantAgent` (modelo + system message + tools), le mandas mensajes, te responde. Puedes juntar varios en un **equipo** (team). Te resultará **muy familiar** tras CrewAI y el OpenAI SDK.

```
   tú ──mensaje──▶ [ AssistantAgent ] ──▶ respuesta
                    (modelo + tools)
```

### AutoGen Core (la capa baja)

Aquí cambia el chip. Core **no** es "un agente"; es una **infraestructura de mensajería**. Defines agentes que reaccionan a mensajes (`@message_handler`), y un **runtime** se encarga de entregar esos mensajes entre ellos. Es el patrón **"modelo de actores"**: muchas entidades independientes que se comunican por mensajes.

```
            ┌─────────── Runtime (cartero) ───────────┐
            │                                          │
   [ Agente A ] ──msg──▶ [ Agente B ] ──msg──▶ [ Agente C ]
   (cada uno reacciona a los mensajes que recibe)
```

> 💡 **La analogía clave:** AgentChat te da **empleados**. Core te da el **sistema de correo interno** de toda la empresa, con el que cualquier empleado (esté donde esté, incluso en otra oficina/máquina) puede mandar un mensaje a cualquier otro. Por eso Core puede ser **distribuido**.

---

## 🧩 Tres conceptos de AgentChat (Módulos 1-2)

| Concepto | Qué es |
|---|---|
| **Model client** | El cliente del LLM: `OpenAIChatCompletionClient(model=...)` (u Ollama, etc.). |
| **Message** | El mensaje: `TextMessage`, o `MultiModalMessage` (texto + imagen). |
| **Agent** | `AssistantAgent(name, model_client, system_message, tools=...)`. |

Y se ejecuta con `await agent.on_messages([message], cancellation_token=...)`.

---

## 🧩 Tres conceptos de Core (Módulos 3-4)

| Concepto | Qué es |
|---|---|
| **RoutedAgent** | La clase base de un agente en Core; reacciona a mensajes. |
| **`@message_handler`** | Decora el método que recibe y responde mensajes. |
| **Runtime** | El "cartero" que entrega mensajes. `SingleThreadedAgentRuntime` (local) o el gRPC (distribuido). |

Y cada agente tiene un **`AgentId`** con dos partes: `type` (qué clase de agente es) y `key` (su identificador único).

---

## 🪜 Cómo encaja con lo que ya sabes

| | OpenAI SDK / CrewAI | LangGraph | **AutoGen** |
|---|---|---|---|
| Foco | Agentes/equipos | Grafo de estado | **Mensajería entre agentes** |
| Nivel | Alto | Medio | **Alto (AgentChat) + bajo (Core)** |
| Punto fuerte | Rapidez | Control del flujo | **Escalado y distribución** |

> 💡 AgentChat compite con CrewAI/OpenAI SDK; **Core** se posiciona más como LangGraph: una infraestructura sobre la que construir. Lo único realmente único de AutoGen en este curso es que Core puede correr **distribuido** entre máquinas (Módulo 4).

---

## 📌 Para llevar

- AutoGen son **dos capas**: **AgentChat** (alto nivel, para construir rápido) y **Core** (bajo nivel, infraestructura de mensajería).
- **AgentChat**: `model client` + `Message` + `AssistantAgent`, ejecutado con `on_messages`. Familiar tras CrewAI/OpenAI SDK.
- **Core**: agentes (`RoutedAgent` + `@message_handler`) que se mandan mensajes a través de un **runtime** (el "cartero"). Es el modelo de actores.
- Cada agente de Core tiene un **`AgentId`** = `type` + `key`.
- Lo distintivo de AutoGen: Core puede ser **distribuido** (varias máquinas por red).

---

[← Volver al índice](README.md) · [Siguiente: AgentChat básico →](01_agentchat_basico.md)
