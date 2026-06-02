# 🔌 Curso Guiado: MCP — el estándar que conecta agentes y herramientas

> Reescritura de la **Semana 6 (el gran final)** del curso, en formato guiado para **leerse y entenderse sin video**.
> Yo (Claude) hago de profesor: el *qué*, el *cómo* y el *por qué*.

---

## 👋 Bienvenido a la semana épica

Esta es la culminación. Aquí aprendes **MCP (Model Context Protocol)**, el estándar de Anthropic para conectar agentes con herramientas, y lo usas para construir el capstone del curso: un **parqué de bolsa autónomo** con cuatro traders-agente que investigan, deciden y operan solos, cada uno con su estrategia, sobre una pila de servidores MCP.

> 🧠 **La gran idea de MCP en una frase:** es el **"USB-C de las herramientas de IA"**. Antes, conectar cada herramienta a cada agente era un cable a medida. MCP es el conector universal: cualquier agente (cliente) puede enchufarse a cualquier herramienta (servidor) que hable MCP, sin código pegamento.

Y fíjate: volvemos al **OpenAI Agents SDK** de la Semana 2. Has cerrado el círculo. 💙

---

## 🗺️ El mapa del curso

```
   usar servidores    crear TU servidor    el ecosistema     EL TRADING FLOOR
   MCP existentes     y TU cliente MCP     MCP (3 tipos)     (4 traders, researcher,
   (fetch, browser,   (FastMCP, @tool,     memoria, búsqueda  scheduler, dashboard)
    filesystem)        @resource)           datos de bolsa)
       │                  │                    │                   │
    Módulo 1           Módulo 2             Módulo 3            Módulo 4
```

| # | Módulo | Aprenderás a... | Concepto estrella |
|---|--------|-----------------|-------------------|
| [00](00_introduccion.md) | **Introducción** | Entender qué es y por qué MCP | Cliente · Servidor · Tools/Resources |
| [01](01_usar_servidores_mcp.md) | **Usar servidores MCP** | Enchufar servidores a un agente | `MCPServerStdio` |
| [02](02_tu_propio_servidor_y_cliente.md) | **Tu propio servidor y cliente** | Crear y consumir MCP | `FastMCP` · `@mcp.tool` · `@mcp.resource` |
| [03](03_el_ecosistema_mcp.md) | **El ecosistema MCP** | Los 3 tipos de servidor y marketplaces | memoria · búsqueda · datos de bolsa |
| [04](04_el_trading_floor.md) | **El trading floor** | El capstone que junta todo | researcher-as-tool · scheduler · dashboard |
| [05](05_patrones_y_chuleta.md) | **Patrones y chuleta** | Tenerlo a mano | Glosario + cheat sheet |

---

## ✅ Requisitos previos

1. **Entorno** (desde la raíz): `uv sync`
2. **`.env`** en la raíz:
   ```
   OPENAI_API_KEY=sk-...
   POLYGON_API_KEY=...     # Módulos 3-4: datos de bolsa (polygon.io, plan free)
   BRAVE_API_KEY=...       # Módulos 3-4: búsqueda (brave, free)
   PUSHOVER_USER=... PUSHOVER_TOKEN=...   # notificaciones
   # opcionales para "muchos modelos" en el capstone:
   DEEPSEEK_API_KEY=... GOOGLE_API_KEY=... GROK_API_KEY=... OPENROUTER_API_KEY=...
   ```
3. **Node + `npx`** (servidores MCP en JavaScript) y **`uvx`** (servidores en Python). Guía: `setup/SETUP-node.md`.
4. **Kernel** `.venv (Python 3.12)`.

> ⚠️ **Usuarios de Windows:** los servidores MCP **no funcionan nativamente en Windows** (problema conocido). Usa **WSL** (Linux sobre Windows): guía en `setup/SETUP-WSL.md`. Mac y Linux van directos. Esto es **imprescindible** esta semana.

> 🔭 Volvemos a las **trazas de OpenAI** (platform.openai.com/traces): con tantos servidores y agentes anidados, son tu salvavidas.

Empecemos. → [Módulo 00: Introducción](00_introduccion.md)

---

<sub>📚 Basado en la Semana 6 de *"Master AI Agentic Engineering"* de Edward Donner, reescrito y ampliado en formato guiado por Claude.</sub>
