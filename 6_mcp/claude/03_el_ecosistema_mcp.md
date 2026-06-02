# Módulo 03 · El ecosistema MCP: los tres tipos de servidor

[← Tu propio servidor y cliente](02_tu_propio_servidor_y_cliente.md) · [Índice](README.md) · [Siguiente: El trading floor →](04_el_trading_floor.md)

---

## 🎯 Objetivo

Conocer la **variedad** de servidores MCP que hay, organizados en tres tipos según dónde corren y a qué acceden. Y reunir las piezas (memoria, búsqueda, datos de bolsa) que alimentarán el capstone.

> 📓 **Corresponde a:** `6_mcp/3_lab3.ipynb` (Week 6 · Day 3).

---

## 🧠 La idea: tres tipos de servidor MCP

No todos los servidores son iguales. Distinguir tres tipos te ayuda a razonar sobre coste, privacidad y dependencias:

```
   TIPO 1                  TIPO 2                   TIPO 3
   Local-local            Local-web                Remoto
   ┌──────────┐           ┌──────────┐             ☁️ ┌──────────┐
   │ servidor │           │ servidor │──internet──▶│   servidor│
   │ + datos  │           │  (llama a │             │  (alojado │
   │ en tu PC │           │ una API) │             │  por otro)│
   └──────────┘           └──────────┘             └──────────┘
```

| Tipo | Corre... | Accede a... | Ejemplo |
|---|---|---|---|
| **1 · Local-local** | En tu máquina | Datos locales | Memoria (grafo de conocimiento) |
| **2 · Local-web** | En tu máquina | Una API por internet | Brave Search, Polygon.io |
| **3 · Remoto** | En otra máquina | Lo suyo | Servidores "hosted/managed" |

> 💡 El **tipo 3 (remoto)** es, hoy, el más difícil de encontrar (pocos servidores MCP están alojados públicamente). La mayoría que usarás son tipo 1 y 2: corren localmente, y algunos llaman a APIs externas.

---

## 💻 Tipo 1 · Memoria (grafo de conocimiento, todo local)

Un servidor que le da al agente **memoria persistente** estructurada como un grafo de conocimiento, guardada en un SQLite local:

```python
memory_params = {
    "command": "npx",
    "args": ["-y", "mcp-memory-libsql"],
    "env": {"LIBSQL_URL": "file:./memory/ed.db"},     # 👈 base de datos local por agente
}
```

Todo ocurre en tu máquina: el servidor y sus datos. Cero dependencia de internet. En el capstone, **cada trader tendrá su propia memoria** (`./memory/<nombre>.db`) para recordar lo aprendido entre sesiones.

---

## 💻 Tipo 2 · Búsqueda y datos de bolsa (local que llama a la web)

### Brave Search

```python
brave_params = {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-brave-search"],
    "env": {"BRAVE_API_KEY": os.getenv("BRAVE_API_KEY")},   # 👈 clave gratis de brave
}
```

El servidor corre local, pero por dentro llama a la API de Brave por internet. Por eso necesita una **clave** (gratis). El agente gana búsqueda web.

### Polygon.io: datos financieros

Para el trading necesitamos precios de acciones. **Polygon.io** es un proveedor de datos de bolsa con plan gratis y... ¡servidor MCP propio! En `mcp_params.py` ves cómo el curso elige entre el servidor oficial de Polygon o uno casero según tu plan:

```python
from market import is_paid_polygon, is_realtime_polygon

if is_paid_polygon or is_realtime_polygon:
    market_mcp = {
        "command": "uvx",
        "args": ["--from", "git+https://github.com/polygon-io/mcp_polygon@v0.1.0", "mcp_polygon"],
        "env": {"POLYGON_API_KEY": polygon_api_key},
    }
else:
    market_mcp = {"command": "uv", "args": ["run", "market_server.py"]}   # 👈 tu servidor casero
```

Y el servidor casero (`market_server.py`) no es más que lo que aprendiste en el Módulo 2: un `FastMCP` que envuelve una función de precios (con caché de cierres de día en `market.py`):

```python
from mcp.server.fastmcp import FastMCP
from market import get_share_price

mcp = FastMCP("market_server")

@mcp.tool()
async def lookup_share_price(symbol: str) -> float:
    """This tool provides the current price of the given stock symbol."""
    return get_share_price(symbol)

if __name__ == "__main__":
    mcp.run(transport='stdio')
```

> 💡 **Lección de ingeniería:** el código que usa el mercado **no sabe** si detrás hay el servidor oficial de Polygon o el casero. Ambos hablan MCP y exponen una tool de precios. Cambiar de proveedor es cambiar una entrada en `mcp_params.py`. Eso es desacoplar por el protocolo.

---

## 🗂️ `mcp_params.py`: el "panel de control" de servidores

Este fichero centraliza **qué servidores usa cada agente** del capstone. Es el plano del Módulo 4:

```python
# Los servidores del TRADER: cuentas, notificaciones y mercado
trader_mcp_server_params = [
    {"command": "uv", "args": ["run", "accounts_server.py"]},   # tu servidor (Mód. 2)
    {"command": "uv", "args": ["run", "push_server.py"]},       # notificaciones
    market_mcp,                                                  # mercado (Polygon o casero)
]

# Los servidores del RESEARCHER: fetch, búsqueda y memoria
def researcher_mcp_server_params(name: str):
    return [
        {"command": "uvx", "args": ["mcp-server-fetch"]},                                    # web
        {"command": "npx", "args": ["-y", "@modelcontextprotocol/server-brave-search"], "env": brave_env},  # búsqueda
        {"command": "npx", "args": ["-y", "mcp-memory-libsql"], "env": {"LIBSQL_URL": f"file:./memory/{name}.db"}},  # memoria por agente
    ]
```

Léelo como un organigrama: **el trader** opera (cuentas + mercado + avisos); **el researcher** investiga (web + búsqueda + memoria). Cada uno con su juego de servidores. Esto es lo que orquestamos en el capstone.

> 💡 Nota el `push_server.py`: es otro servidor MCP casero (FastMCP) que envuelve la notificación Pushover que arrastras desde la Semana 1. Tu vieja función `push`, ahora servida por MCP.

---

## ⚠️ Errores comunes

- **Faltan claves (`BRAVE_API_KEY`, `POLYGON_API_KEY`).** Ambas tienen plan gratis; sin ellas, esos servidores no arrancan.
- **`npx`/`uvx` no encontrados.** Instala Node (`setup/SETUP-node.md`); `uvx` viene con `uv`.
- **Demasiadas tools abruman al modelo.** El servidor de pago de Polygon expone *muchísimas* tools; un modelo pequeño puede agobiarse. Usa el casero o un modelo más capaz.
- **Memoria que no persiste.** Revisa la ruta `LIBSQL_URL` y que exista la carpeta `memory/`.
- **Windows.** Todo esto va por WSL.

---

## 🧪 Pruébalo tú

1. **Enchufa el servidor de memoria** a un agente: dile algo en una sesión, reinicia, y comprueba si lo recuerda en otra.
2. **Compara Brave vs fetch:** uno busca, el otro descarga una URL concreta. ¿Cuándo usa el agente cada uno?
3. **Mira `market.py`:** ¿cómo cachea los precios de cierre? Entender el dato real evita sorpresas en el capstone.
4. **Explora un marketplace** (mcp.so) y clasifica 5 servidores en los tres tipos.

---

## 📌 Para llevar

- Tres tipos de servidor MCP: **local-local** (todo en tu PC, p. ej. memoria), **local-web** (local pero llama a una API, p. ej. Brave/Polygon) y **remoto** (alojado por otro, aún escaso).
- El código que consume un servicio **no sabe** qué servidor hay detrás: cambiar de proveedor = cambiar una receta en `mcp_params.py`. Desacople por protocolo.
- `mcp_params.py` centraliza qué servidores usa cada agente: el **trader** (cuentas, mercado, push) y el **researcher** (web, búsqueda, memoria).
- Servidores caseros (`market_server.py`, `push_server.py`) reusan tu trabajo previo envuelto en `FastMCP`.

---

[← Tu propio servidor y cliente](02_tu_propio_servidor_y_cliente.md) · [Índice](README.md) · [Siguiente: El trading floor →](04_el_trading_floor.md)
