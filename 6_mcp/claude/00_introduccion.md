# Módulo 00 · Introducción: ¿qué es MCP y por qué importa tanto?

[← Volver al índice](README.md) · [Siguiente: Usar servidores MCP →](01_usar_servidores_mcp.md)

---

## 🎯 Objetivo

Entender el modelo mental de MCP —**cliente, servidor, tools y resources**— antes de programar. Es un concepto sencillo con consecuencias enormes.

---

## 🧠 La idea: el conector universal para herramientas de IA

Recuerda cómo dabas herramientas a un agente hasta ahora: escribías la función, la describías (a mano en la Semana 1, con decoradores después), y la metías en *ese* agente, en *ese* framework. Si querías la misma herramienta en otro proyecto, copiar y pegar.

MCP rompe eso. Define un **protocolo estándar**: las herramientas viven en **servidores** independientes, y cualquier **cliente** (tu agente) que hable MCP puede usarlas. Como un puerto USB-C: el mismo conector sirve para el móvil, el portátil y la pantalla.

```
   ┌─────────────┐         MCP          ┌──────────────────┐
   │   CLIENTE   │ ◀──────(protocolo)──▶ │     SERVIDOR     │
   │ (tu agente) │   "¿qué tools tienes?"│  (expone tools   │
   │             │   "llama a buy_shares"│   y resources)   │
   └─────────────┘                       └──────────────────┘
```

> 💡 **Por qué es un terremoto:** una vez que alguien publica un servidor MCP (para GitHub, Slack, una base de datos, la bolsa...), **cualquier agente del mundo** puede usarlo sin escribir pegamento. Hay *marketplaces* llenos de servidores listos. Tu agente se vuelve tan capaz como herramientas le enchufes.

---

## 🧩 Los tres papeles

| Papel | Qué es | En este curso |
|---|---|---|
| **Host** | La app que coordina (donde vive el agente) | Tu notebook / `app.py` |
| **Cliente (Client)** | El que se conecta a un servidor y usa sus capacidades | El agente (vía el SDK) |
| **Servidor (Server)** | El que expone tools y resources | `accounts_server.py`, `mcp-server-fetch`, etc. |

Un host puede tener varios clientes, cada uno conectado a un servidor distinto. Tu agente del capstone hablará con **varios servidores a la vez** (cuentas, mercado, notificaciones...).

---

## 🧩 Las dos cosas que sirve un servidor MCP

Esto es clave y suele pasar desapercibido. Un servidor MCP ofrece **dos** tipos de cosas:

| | **Tools** | **Resources** |
|---|---|---|
| Qué es | Funciones que el agente **ejecuta** | Datos que el agente **lee** |
| Analogía | Verbos ("compra acciones") | Sustantivos ("el informe de la cuenta") |
| Ejemplo | `buy_shares(name, symbol, qty)` | `accounts://.../Ed` → el reporte de Ed |
| Cómo se identifica | Por nombre | Por una **URI** (como una dirección web) |

> 💡 **Tools = acciones, Resources = información.** El agente *llama* a una tool para *hacer* algo, y *lee* un resource para *saber* algo. (MCP también define "prompts", plantillas reutilizables, pero en este curso nos centramos en tools y resources.)

---

## 🔌 Cómo se conecta: el transporte "stdio"

¿Cómo se hablan cliente y servidor? En este curso, casi siempre por **stdio** (entrada/salida estándar): el cliente **lanza el servidor como un subproceso** y se comunican por sus tuberías. Por eso verás cosas como:

```python
{"command": "uvx", "args": ["mcp-server-fetch"]}        # arranca un servidor Python
{"command": "npx", "args": ["@playwright/mcp@latest"]}  # arranca uno de JavaScript
{"command": "uv",  "args": ["run", "accounts_server.py"]}# arranca el tuyo
```

Cada uno es una **receta para arrancar un servidor**. El cliente lo lanza, le pregunta "¿qué tools tienes?", y se las da al agente. (Existe también transporte por red/HTTP para servidores remotos; lo veremos en el Módulo 3.)

---

## 🪜 Cómo encaja con todo el curso

Has recorrido seis semanas. MCP es el broche porque es **ortogonal** a los frameworks: no compite con ellos, los **potencia**.

```
   Semana 1: el bucle a mano         ─┐
   Semana 2: OpenAI Agents SDK        │  Todos pueden CONSUMIR
   Semana 3: CrewAI                   │  herramientas vía MCP.
   Semana 4: LangGraph                │  MCP es la "capa de tools"
   Semana 5: AutoGen                  │  estándar para todos.
   Semana 6: MCP  ◀── el conector ───┘
```

> 💡 Ya viste un adelanto en la Semana 5 (AutoGen consumía tools MCP). Aquí lo dominas: no solo **usar** servidores, sino **crear los tuyos**.

---

## 📌 Para llevar

- **MCP** es un protocolo estándar para conectar agentes (**clientes**) con herramientas (**servidores**) — el "USB-C de la IA".
- Tres papeles: **host** (la app), **cliente** (el agente que se conecta) y **servidor** (el que expone capacidades).
- Un servidor sirve **tools** (acciones que el agente ejecuta) y **resources** (datos que el agente lee, direccionados por **URI**).
- El transporte habitual es **stdio**: el cliente lanza el servidor como subproceso. Cada servidor es una "receta" `{command, args}`.
- MCP no compite con los frameworks: es la **capa de herramientas estándar** que todos pueden usar.

---

[← Volver al índice](README.md) · [Siguiente: Usar servidores MCP →](01_usar_servidores_mcp.md)
