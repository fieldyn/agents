# Módulo 00 · Introducción: pensar en grafos de estado

[← Volver al índice](README.md) · [Siguiente: Los 5 pasos →](01_los_cinco_pasos.md)

---

## 🎯 Objetivo

Instalarte el modelo mental de LangGraph —**estado, nodos, aristas**— antes de programar. Si "ves" el grafo, el código se vuelve obvio.

---

## 🧠 La idea: un agente es un diagrama de flujo con un estado que viaja

Olvida por un momento los LLMs. Imagina un **diagrama de flujo**: cajas (pasos) unidas por flechas. Por ese diagrama viaja un **portafolios** (el estado) que cada caja puede leer y modificar. Algunas flechas tienen una condición ("si X, ve aquí; si no, allá").

Eso **es** LangGraph:

```
   START ──▶ [ nodo A ] ──▶ [ nodo B ] ──▶ END
                              │   ▲
                       ¿condición?│  (las aristas pueden formar ciclos)
                              ▼   │
                          [ nodo C ]
```

| Pieza | Qué es | Analogía |
|---|---|---|
| **State** | Un objeto con todos los datos que viajan por el grafo | El portafolios que pasa de mano en mano |
| **Node** | Una función Python que recibe el estado y devuelve cambios | Cada empleado que añade algo al portafolios |
| **Edge** | Una conexión entre nodos | La flecha que dice "ahora ve a..." |
| **Conditional edge** | Una arista que elige destino según el estado | "Si el informe está aprobado, archívalo; si no, devuélvelo" |
| **Graph** | Todo ensamblado y "compilado", listo para ejecutar | El diagrama de flujo completo |

> 💡 **Lo que hace especial a LangGraph: los CICLOS.** Las aristas condicionales pueden volver a un nodo anterior. Así modelas "trabaja → evalúa → si no está bien, vuelve a trabajar". Otros frameworks asumen flujo hacia delante; LangGraph abraza los bucles. Por eso es ideal para agentes que iteran.

---

## 🔑 El concepto que más confunde: el *reducer*

Cuando un nodo devuelve cambios al estado, ¿cómo se **combinan** con lo que ya había? ¿Se sobrescribe? ¿Se añade? Eso lo decide un **reducer**: una función que dice cómo fundir el valor viejo con el nuevo.

El caso más común son los **mensajes** de una conversación: no quieres reemplazar el historial, quieres **añadir** el mensaje nuevo. Para eso LangGraph trae el reducer `add_messages`. Lo declaras así:

```python
from typing import Annotated
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]   # 👈 "para messages, usa add_messages al combinar"
```

Ese `Annotated[list, add_messages]` se lee: *"`messages` es una lista, y cuando un nodo aporte mensajes nuevos, fúndelos con `add_messages`" (es decir, **acumúlalos**)*. Sin reducer, el campo se sobrescribiría. Lo veremos en acción en el Módulo 1; por ahora quédate con la intuición.

---

## 🆚 LangGraph vs. lo que ya conoces

| | Semana 1-2 | CrewAI (Sem. 3) | **LangGraph (Sem. 4)** |
|---|---|---|---|
| Cómo defines el flujo | Bucle a mano / SDK | YAML + proceso | **Grafo explícito** |
| Control del flujo | Total pero manual | Poco (lo decide el framework) | **Total y visual** |
| Ciclos / reintentos | A mano | Difícil | **Naturales** |
| Estado compartido | Variables sueltas | Implícito | **Un objeto `State` central** |

> 💡 Si la Semana 2 te dio "agentes fáciles" y la 3 "equipos por configuración", la 4 te da **control fino del flujo**. Es la opción cuando necesitas que el agente siga una lógica precisa con bucles y bifurcaciones.

---

## 📌 Para llevar

- LangGraph modela un agente como un **grafo**: un **State** (datos) viaja por **nodos** (funciones) unidos por **aristas** (flujo).
- Las **aristas condicionales** eligen el camino según el estado y permiten **ciclos** — la clave para iterar (trabajar → evaluar → reintentar).
- El **reducer** decide cómo se combinan los cambios al estado; `add_messages` acumula mensajes en vez de sobrescribir.
- LangGraph **no necesita LLMs**: el grafo es mecánica Python; los LLMs van *dentro* de los nodos.
- Su fortaleza frente a otros frameworks es el **control explícito del flujo**.

---

[← Volver al índice](README.md) · [Siguiente: Los 5 pasos →](01_los_cinco_pasos.md)
