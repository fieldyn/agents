# Módulo 00 · Introducción: ¿Qué es realmente un "agente"?

[← Volver al índice](README.md) · [Siguiente: Tu primer agente →](01_tu_primer_agente.md)

---

## 🎯 Objetivo

Que al terminar este módulo tengas un **modelo mental claro** de lo que es un agente, antes de escribir una sola línea de código. Si entiendes esto, todo lo demás del curso encaja solo.

---

## 🧠 La idea: de "chatbot" a "agente"

Imagina dos empleados.

El **primero** es un asistente al que le haces una pregunta y te responde. Le preguntas, contesta, fin. Si necesitas algo más, le vuelves a preguntar. Es brillante, pero pasivo. **Eso es un LLM normal** (como llamar a la API de chat directamente).

El **segundo** empleado es distinto. Le das un *objetivo* — "consígueme el mejor email de ventas y envíalo" — y se pone en marcha solo: escribe borradores, los compara, elige, da formato, y lo manda. Por el camino **decide qué hacer a continuación** y **usa herramientas** (el email, el navegador, una calculadora) sin que tú le digas paso a paso. **Eso es un agente.**

> 💡 **La definición que usaremos:** un agente es un LLM al que le damos (1) **instrucciones**, (2) acceso a **herramientas**, y (3) un **bucle** que le deja actuar varias veces hasta cumplir el objetivo.

---

## 🔁 El "bucle agéntico" (el corazón de todo)

Casi todo lo que verás en el curso es alguna variación de este bucle. Memorízalo como una cancioncilla:

```
        ┌─────────────────────────────────────────┐
        │                                         │
        ▼                                         │
   ┌─────────┐    ¿necesito        ┌──────────┐   │
   │  LLM    │──  una tool? ──sí──▶│  ejecuta │───┘
   │ piensa  │                     │  la tool │
   └─────────┘                     └──────────┘
        │
        │ no, ya tengo la respuesta
        ▼
   ┌─────────┐
   │ resultado│
   │  final   │
   └─────────┘
```

1. El LLM recibe el objetivo y **piensa**.
2. Decide: *¿puedo responder ya, o necesito una herramienta?*
3. Si necesita una herramienta, la **llama**, recibe el resultado, y **vuelve a pensar** con esa información nueva.
4. Repite hasta que tiene la respuesta final.

El SDK de OpenAI implementa este bucle por ti. Tú solo describes **el agente** y **las herramientas**; el `Runner` se encarga de dar vueltas al bucle.

---

## 🧩 Las 4 piezas que vas a combinar todo el curso

Todo lo que construyamos —desde un chiste hasta un investigador autónomo— sale de combinar estas cuatro piezas:

| Pieza | Qué es | Analogía |
|---|---|---|
| **Agent** | Un LLM + instrucciones + herramientas | El empleado con su descripción de puesto |
| **Tool** | Una función que el agente puede llamar | Las herramientas en su mesa de trabajo |
| **Handoff** | Pasar el control a *otro* agente | Derivar el caso a otro departamento |
| **Runner** | El motor que ejecuta el bucle | El jefe que dice "empieza" y recoge el resultado |

> 📐 **Tools vs. Handoffs** (esto confunde a todo el mundo, lo veremos a fondo en el Módulo 2):
> - Con una **tool**, el agente llama a algo, recibe el resultado, y **sigue mandando él**. El control *vuelve*.
> - Con un **handoff**, el agente **cede el mando** a otro agente. El control *pasa* y no vuelve.

---

## 🤔 ¿Por qué el OpenAI Agents SDK y no "a mano"?

Podrías construir el bucle agéntico tú mismo: llamar a la API, parsear si pidió una tool, ejecutarla, volver a llamar... Mucha gente lo hace. Pero es repetitivo y propenso a errores (sobre todo el famoso "boilerplate JSON" para describir herramientas).

El SDK te da, gratis:

- ✅ El **bucle** ya implementado y probado.
- ✅ Conversión **automática** de funciones Python a herramientas (sin JSON a mano).
- ✅ **Trazas** (`traces`) para ver, paso a paso, qué pensó y qué hizo tu agente.
- ✅ **Handoffs**, **guardrails** y **structured outputs** como ciudadanos de primera clase.
- ✅ Compatibilidad con **otros modelos** (Gemini, DeepSeek, Llama...) usando la misma API.

Y lo mejor: la API es minúscula. En el próximo módulo tendrás un agente funcionando con **tres líneas**.

---

## 📌 Para llevar

- Un **agente** = LLM + instrucciones + herramientas + un **bucle** que le deja actuar varias veces.
- El **bucle agéntico** (pensar → ¿tool? → ejecutar → repensar → responder) es el patrón que se repite en TODO el curso.
- Las 4 piezas básicas son **Agent, Tool, Handoff y Runner**.
- **Tool** = el control vuelve al agente. **Handoff** = el control pasa a otro agente.
- El SDK existe para que no tengas que reimplementar el bucle ni el boilerplate.

---

[← Volver al índice](README.md) · [Siguiente: Tu primer agente →](01_tu_primer_agente.md)
