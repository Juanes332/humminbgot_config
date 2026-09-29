# Condor + Hummingbot API en entorno local, con Hyperliquid

**Guía técnica de instalación, configuración y primeras estrategias**

| | |
|---|---|
| Alcance | Instalación **local** (una sola máquina) de Condor y Hummingbot API, integración con Hyperliquid (perpetuos) y estrategias básicas de prueba |
| Fuente | Documentación oficial: `condor.hummingbot.org`, `hummingbot.org` y el código de `hummingbot/hummingbot` (connector de Hyperliquid) |
| Fecha de revisión | 2026-09-29 |
| Fuera de alcance | Despliegue en VPS o servidores remotos, Tailscale y acceso desde otras redes |

> **Convención.** Los bloques marcados como **[Nota del autor]** son aportes propios que la documentación oficial no recoge, como la adaptación a Fedora o los tamaños de prueba. Todo lo demás reproduce lo que dice la documentación.

---

## Índice

1. [Arquitectura](#1-arquitectura)
2. [Requisitos](#2-requisitos)
3. [Proveedor LLM (instalarlo antes que Condor)](#3-proveedor-llm-instalarlo-antes-que-condor)
4. [Instalación](#4-instalación)
5. [Verificación y operación diaria](#5-verificación-y-operación-diaria)
6. [Configuración de Condor](#6-configuración-de-condor)
7. [Integración con Hyperliquid](#7-integración-con-hyperliquid)
8. [El harness de Condor: Trading Agents](#8-el-harness-de-condor-trading-agents)
9. [Executors](#9-executors)
10. [Bots y controllers](#10-bots-y-controllers)
11. [Routines](#11-routines)
12. [Estrategias básicas de prueba en Hyperliquid](#12-estrategias-básicas-de-prueba-en-hyperliquid)
13. [Seguridad en local](#13-seguridad-en-local)
14. [Troubleshooting](#14-troubleshooting)
15. [Referencias](#15-referencias)

---

## 1. Arquitectura

Condor sigue una **arquitectura de dos servidores** que separa el razonamiento del agente de la ejecución de órdenes. En esta guía ambos corren en la misma máquina.

```
┌──────────────────────── máquina local ─────────────────────────┐
│                                                                 │
│  Condor  (capa agéntica, probabilística)                        │
│   ├─ Trading Agents (agent.md + tick loop + LLM vía ACP)        │
│   ├─ Routines (Python determinista)                             │
│   ├─ MCP servers: hummingbot-mcp · condor-mcp                   │
│   ├─ Web dashboard  → 127.0.0.1:8088                            │
│   └─ Bot de Telegram (opcional: alertas y aprobaciones)         │
│            │                                                    │
│            ▼  HTTP + basic auth                                 │
│  Hummingbot API (capa de ejecución, determinista, Docker)       │
│   ├─ API REST     → 127.0.0.1:8000  (Swagger en /docs)          │
│   ├─ PostgreSQL   → 127.0.0.1:5432                              │
│   ├─ EMQX (MQTT)  → 127.0.0.1:1883  (dashboard 18083)           │
│   ├─ Executors · Bots (contenedores Hummingbot)                 │
│   └─ Gateway (DEX, opcional)                                    │
│            │                                                    │
└────────────┼────────────────────────────────────────────────────┘
             ▼
     Exchanges: hyperliquid_perpetual, ...
```

| Componente | Responsabilidad |
|---|---|
| **Condor** | *Harness* open source para desplegar y gestionar flotas de agentes de trading con IA. Contiene los agentes, las rutinas, el dashboard y la interfaz de Telegram. |
| **Hummingbot API** | Datos de mercado, credenciales, executors, bots, portafolio y backtesting. Es la única pieza que habla con los exchanges. |
| **Executors** | Operaciones de trading autocontenidas con ciclo de vida y P&L estandarizados. Los agentes **solo** actúan a través de ellos. |
| **Bots** | Contenedores Docker de Hummingbot para estrategias de larga duración (controllers V2 o scripts). |
| **Gateway** | Middleware para DEX (Jupiter, Orca, Raydium, Meteora, Uniswap). Hyperliquid **no** lo necesita, porque su connector es nativo. |

---

## 2. Requisitos

### 2.1 Hardware (documentación de Condor)

| Componente | Mínimo | Recomendado |
|---|---|---|
| SO | Linux x64/ARM (Ubuntu 20.04+), macOS | Ubuntu 22.04 LTS |
| RAM | 4 GB | 8 GB |
| Disco | 10 GB SSD | 20 GB SSD |
| CPU | 2 vCPU | 4 vCPU |

En Windows solo se admite WSL2 con Ubuntu.

### 2.2 Software

- **Docker** instalado y **en ejecución** antes de lanzar el instalador, porque la Hummingbot API corre en contenedores.
- `git`, `curl`, `make`.
- Dependencias que `make doctor` comprueba: `uv`, `tmux`, `node`, `npm` y `tsc`.

> **[Nota del autor] Fedora.** La documentación no cubre Fedora, pero el stack funciona sin cambios. Preparación mínima:
> ```bash
> sudo dnf -y install dnf-plugins-core git make curl tmux nodejs npm
> sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
> sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
> sudo systemctl enable --now docker
> sudo usermod -aG docker "$USER"   # cerrar sesión y volver a entrar
> sudo npm install -g typescript     # provee tsc
> curl -LsSf https://astral.sh/uv/install.sh | sh
> docker run --rm hello-world        # sanity check
> ```

### 2.3 Cuentas necesarias

| Recurso | Obligatorio | Para qué |
|---|---|---|
| Wallet con fondos en Hyperliquid (USDC) | Sí | Operar perpetuos |
| Proveedor LLM (Claude Code, Codex, Gemini, OpenRouter u Ollama) | Sí, si se usarán agentes | Motor de decisión de los Trading Agents |
| Bot de Telegram (@BotFather) y user ID (@userinfobot) | Opcional | Modo *Local + Telegram alerts* |

---

## 3. Proveedor LLM (instalarlo antes que Condor)

La documentación indica que el proveedor LLM se instala y autentica **antes** que Condor. Si se instala después, hay que reiniciar Condor.

Condor admite dos mecanismos de integración:

- **ACP (Agent Client Protocol):** integración nativa con Claude Code, Gemini, GitHub Copilot y Codex.
- **PydanticAI:** acceso directo por API a OpenRouter y a modelos locales (Ollama, LM Studio).

| Proveedor | `agent_key` | Instalación | Autenticación |
|---|---|---|---|
| **Claude Code** (proveedor por defecto y el más integrado) | `claude-code` | `curl -fsSL https://claude.ai/install.sh \| sh` | `claude auth login` |
| Codex | `codex` | Codex CLI + `npx @agentclientprotocol/codex-acp` | Login con ChatGPT o API key |
| Gemini | `gemini` | `npm install -g @google/gemini-cli` | `gemini` → Login with Google, o `GEMINI_API_KEY` |
| GitHub Copilot | `copilot` | `npm install -g @github/copilot-cli` | `gh auth login && copilot auth login` |
| OpenRouter | `openrouter` | — | `OPENROUTER_API_KEY` en `.env` |
| Ollama (local) | `ollama` | `curl -fsSL https://ollama.com/install.sh \| sh` | — |
| LM Studio (local) | `lmstudio` | lmstudio.ai | Servidor local en el puerto 1234 |

Verificación con Claude Code:

```bash
claude --version
claude            # debe arrancar sin errores
```

Recomendaciones de la documentación:

- **Producción:** Claude Code o Codex.
- **Desarrollo:** Ollama o LM Studio, sin coste de API.
- Evitar modelos por debajo de 8B parámetros para estrategias complejas.

Los modelos locales reciben menos herramientas según su tamaño:

| Tamaño del modelo | Herramientas disponibles |
|---|---|
| ≤8B | Esenciales |
| 9–32B | Moderadas |
| >32B y proveedores cloud | Completas |

---

## 4. Instalación

### 4.1 Modo de uso

La documentación define tres rutas. En local aplican dos:

| | **Local** | **Local + Telegram alerts** (recomendado) |
|---|---|---|
| Interfaz | Dashboard en `http://localhost:8088` | Comandos de Telegram, más `/web` para el dashboard |
| Login | **Ninguno**: quien alcance el puerto tiene control total | Autenticación vía Telegram; `/web` genera un enlace temporal |
| Alertas y aprobaciones | Solo con el dashboard abierto | Se envían al móvil |
| Sigue operando si se cierra el portátil | No | No |

El modo queda registrado como `CONDOR_MODE` (`telegram` o `local`). Se puede cambiar después con `make setup`.

> **[Nota del autor]** Se recomienda *Local + Telegram alerts*. Una petición de aprobación de un agente llega al móvil aunque el usuario no esté frente a la máquina. En modo *Local* esa petición espera hasta que alguien abra el dashboard.

Si se elige Telegram, antes de instalar hay que preparar dos valores:

1. **Token del bot:** en @BotFather, enviar `/newbot` y copiar el token (formato `123456789:ABC...`).
2. **User ID:** en @userinfobot, enviar `/start` y copiar el ID numérico.

### 4.2 Instalación rápida (método oficial)

Desde un directorio vacío:

```bash
mkdir -p ~/hummingbot-stack && cd ~/hummingbot-stack
curl -fsSL https://raw.githubusercontent.com/hummingbot/deploy/main/setup.sh | bash
```

El asistente pregunta, en este orden:

| # | Pregunta | Respuesta para entorno local |
|---|---|---|
| 1 | Modo de uso | `Telegram` o `Local` |
| 2 | Bot token y user ID | Los obtenidos en 4.1 (solo en modo Telegram) |
| 3 | ¿Usar Tailscale? | **`n`** (no aplica en local) |
| 4 | IP pública u hostname | **Enter** (Condor corre en la propia máquina) |
| 5 | Modelo de IA | Elegir el proveedor instalado en §3. El asistente instala el bridge CLI (unos 250 MB, 1–3 min) y avisa antes de descargar. Se puede omitir y ejecutar `make pick-model` más tarde |
| 6 | ¿Configurar y lanzar Hummingbot API con Docker? | **Sí** (Condor y la API en la misma máquina). Si ya hay una API en el puerto 8000, el asistente ofrece reutilizarla |
| 7 | Credenciales de la API | **usuario admin**, **password** y **config password**. Son obligatorias y no tienen valor por defecto. Se escriben en `hummingbot-api/.env` y en `condor/config.yml` para que coincidan |

Al terminar, el instalador ejecuta `make doctor` automáticamente. Sus *warnings* no bloquean la instalación, así que conviene leer el informe completo.

Estructura resultante:

```
~/hummingbot-stack/
├── condor/            # harness, agentes, rutinas, dashboard
└── hummingbot-api/    # API + Postgres + EMQX (docker compose)
```

### 4.3 Instalación manual (alternativa)

Solo se usa si el script falla o si se va a desarrollar sobre el código fuente.

```bash
# Hummingbot API
git clone https://github.com/hummingbot/hummingbot-api.git
cd hummingbot-api
make setup
docker compose pull
make deploy
cd ..

# Condor
git clone https://github.com/hummingbot/condor.git
cd condor
make install     # setup interactivo + dependencias + CLIs de IA
make run
```

### 4.4 Actualización

La actualización está disponible desde tres sitios:

- **Telegram:** `/update` (solo admin).
- **Dashboard:** **Settings → Updates**.
- **Script**, desde el directorio que contiene `condor/`:
  ```bash
  curl -fsSL https://raw.githubusercontent.com/hummingbot/deploy/main/setup.sh | bash -s -- --upgrade
  ```

La actualización se detiene justo antes del relanzamiento, a propósito. Para aplicarla hay que ejecutar `make restart`. Si se reinicia el contenedor de la API, **se detienen sus executors**.

---

## 5. Verificación y operación diaria

### 5.1 Verificación

```bash
cd ~/hummingbot-stack/condor && make doctor
cd ~/hummingbot-stack/hummingbot-api && make doctor
curl -u USUARIO:PASSWORD http://localhost:8000/     # nombre y versión de la API
```

`make doctor` es de solo lectura. Comprueba dependencias, `.env` y `config.yml`, el modelo de IA, el binding del dashboard y la conectividad y autenticación con cada servidor de `config.yml`. Solo sale con código distinto de 0 ante fallos reales.

- **Modo Telegram:** enviar `/start` al bot. El admin debe recibir el mensaje *"Condor is online and ready."*
- **Modo Local:** abrir `http://localhost:8088`. No hay pantalla de login.

### 5.2 Puntos de acceso

| Servicio | URL |
|---|---|
| Hummingbot API | `http://localhost:8000` |
| Swagger | `http://localhost:8000/docs` |
| Dashboard (modo Local) | `http://localhost:8088` |
| Dashboard (modo Telegram) | Enlace temporal generado con `/web` |
| EMQX dashboard | `http://localhost:18083` |

### 5.3 Comandos

Condor (desde `condor/`; corre en una sesión tmux llamada `condor`):

```bash
make run        # arranca
make run-fg     # primer plano, para depurar errores de arranque
make logs       # adjunta a tmux (salir: Ctrl+B, luego D)
make status
make restart
make stop
make doctor
make pick-model # cambiar o verificar el modelo LLM
make setup      # re-configurar (p. ej. cambiar de modo)
```

Hummingbot API (desde `hummingbot-api/`):

```bash
docker compose ps
docker compose logs -f
make deploy
make doctor
make emqx-audit
```

---

## 6. Configuración de Condor

Condor se configura en dos archivos:

- **`.env`:** modo, tokens, dashboard, privacidad y retenciones. Tras editarlo hay que ejecutar `make restart`.
- **`config.yml`:** servidores, usuarios, permisos y valores por defecto de cada chat. Lo escriben `make setup` y `/servers`; no se edita a mano.

### 6.1 Variables relevantes en `.env`

| Variable | Default | Uso |
|---|---|---|
| `CONDOR_MODE` | `telegram` | `telegram` o `local` |
| `TELEGRAM_TOKEN` | — | Obligatorio salvo con `CONDOR_MODE=local` |
| `ADMIN_USER_ID` | — | ID del admin. Todo el estado queda asociado a este ID |
| `WEB_PORT` | `8088` | Puerto del dashboard |
| `WEB_HOST` | `0.0.0.0` (telegram) / `127.0.0.1` (local) | Interfaz del dashboard. **En local conviene fijarlo en `127.0.0.1`** |
| `CONDOR_DEFAULT_AGENT` | lo escribe `make pick-model` | Modelo con el que razona Condor |
| `OPENROUTER_API_KEY` | — | Si se usa OpenRouter |
| `CONDOR_TELEMETRY` | unset | `off` desactiva la telemetría |
| `CONDOR_SHARING` | unset | `off` impide compartir transcripciones |
| `UPDATE_CHECK_INTERVAL` | `3600` | Segundos entre comprobaciones de actualización; `0` las desactiva |

Los ajustes propios de la API (`API_BIND`, `DB_BIND`, `GATEWAY_PASSPHRASE`) viven en `hummingbot-api/.env`.

### 6.2 Servidor de API

En modo Telegram se configura con `/servers`; en modo Local, en **Settings → Servers**. Valores:

- Host: `localhost`
- Puerto: `8000`
- Usuario y password: los mismos que en `hummingbot-api/.env`

---

## 7. Integración con Hyperliquid

### 7.1 Connectors disponibles

| Connector | Mercado |
|---|---|
| `hyperliquid` | Spot |
| `hyperliquid_perpetual` | Perpetuos (el que Condor lista en `/keys`) |
| `hyperliquid_perpetual_testnet` | Perpetuos en testnet (`app.hyperliquid-testnet.xyz`) |

Particularidades documentadas:

- **Tipos de orden:** `LIMIT`, `LIMIT_MAKER` y `MARKET`.
- **Modo de posición:** los perpetuos solo admiten *one-way*, es decir, una posición por par.
- **Mercados HIP-3** (RWA, acciones, commodities): solo margen aislado, y el ticker lleva el prefijo del emisor (p. ej. `XYZ:CL-USD`).
- **Formato de par en perpetuos:** `BTC-USD`, `ETH-USD`, `SOL-USD`, etc.
- **Comisiones por defecto del connector:** maker 0 % y taker 0,025 %. Con el *builder code* de Hummingbot, el cliente estándar no añade comisión propia (0 bps).
- **Intervalos de velas:** 1m, 3m, 5m, 15m, 30m, 1h, 2h, 4h, 6h, 8h, 12h, 1d, 1w, 1M.

### 7.2 Crear la API wallet en Hyperliquid

Hyperliquid no emite API key y secret clásicos. Se autoriza una **API wallet** (agent wallet) que puede operar en nombre de la cuenta **sin permiso de retiro**.

1. Entrar en `https://app.hyperliquid.xyz/` y conectar la wallet principal.
2. Ir a **More → API**.
3. Crear la API wallet: asignarle un nombre y la dirección de la cuenta principal.
4. Fijar la validez. Se recomienda un máximo de 180 días.
5. **Copiar la Private Key** de la API wallet. Hace el papel de *secret* y solo se muestra una vez.
6. Autorizarla firmando con la wallet principal.
7. Anotar la **dirección de la cuenta principal** (menú superior derecho). Hummingbot la pide como *address*.

> **[Nota del autor]** Se debe usar el modo `api_wallet` y nunca la private key de la wallet principal (`arb_wallet`). La API wallet no puede retirar fondos y se revoca desde la misma página.

### 7.3 Campos que pide el connector

Según `HyperliquidPerpetualConfigMap` en el código de Hummingbot:

| Campo | Valor |
|---|---|
| `hyperliquid_perpetual_mode` | `api_wallet` (alternativa: `arb_wallet`) |
| `use_vault` | `no`, salvo que se opere un vault |
| `hyperliquid_perpetual_address` | Dirección de la **cuenta principal** (o del vault si `use_vault=yes`) |
| `hyperliquid_perpetual_secret_key` | **Private key de la API wallet** |

En testnet los campos llevan el prefijo `hyperliquid_perpetual_testnet_*`.

### 7.4 Alta de credenciales

**Opción A: Telegram**

1. Enviar `/keys`.
2. Elegir **Perpetual**.
3. Elegir **Hyperliquid**.
4. Rellenar los campos de 7.3.

**Opción B: Dashboard**

Ir a **Settings → Keys and Wallets → Add API Key** y seleccionar `hyperliquid_perpetual`.

**Opción C: Hummingbot API**

Útil para testnet o para cuentas separadas.

```bash
# (opcional) cuenta aislada para pruebas
curl -u USUARIO:PASSWORD -X POST "http://localhost:8000/accounts/add-account?account_name=hl_test"

# credenciales de testnet
curl -u USUARIO:PASSWORD -X POST \
  http://localhost:8000/accounts/add-credential/hl_test/hyperliquid_perpetual_testnet \
  -H "Content-Type: application/json" \
  -d '{
    "hyperliquid_perpetual_testnet_mode": "api_wallet",
    "use_vault": false,
    "hyperliquid_perpetual_testnet_address": "0xDIRECCION_CUENTA_PRINCIPAL",
    "hyperliquid_perpetual_testnet_secret_key": "0xPRIVATE_KEY_API_WALLET"
  }'

# verificación
curl -u USUARIO:PASSWORD http://localhost:8000/accounts/hl_test/credentials
```

> **[Nota del autor]** El cuerpo JSON se ha construido a partir del config map del connector. Antes de usarlo conviene contrastarlo con el endpoint `Get Connector Config Map` en Swagger (`/docs`). La documentación recomienda usar cuentas separadas para *paper vs. live* y así no mezclar credenciales.

### 7.5 Verificación

- `/portfolio` en Telegram o **Portfolio** en el dashboard.
- En lenguaje natural con el agente: *"What's my balance on Hyperliquid?"*

### 7.6 Ajustes previos en perpetuos

Se configuran desde `/trade` o desde la API:

```bash
curl -u USUARIO:PASSWORD -X POST \
  http://localhost:8000/trading/master_account/hyperliquid_perpetual/leverage \
  -H "Content-Type: application/json" \
  -d '{"trading_pair": "BTC-USD", "leverage": 2}'
```

---

## 8. El harness de Condor: Trading Agents

### 8.1 Concepto

Un **Trading Agent** es una entidad autónoma respaldada en archivos. Se ejecuta con un intervalo fijo (*tick*), razona sobre el mercado con un LLM y traduce sus decisiones en operaciones reales a través de executors de Hummingbot.

Modelo mental que da la documentación:

> Un agente es una *carpeta* en disco y un *tick loop* en memoria. La carpeta es su memoria a largo plazo; los providers le dan contexto inmediato; el LLM es su función de decisión; los executors son sus manos; y el risk engine es la muñeca que no puede forzar.

### 8.2 Dos capas y el bucle OODA

| Fase | Naturaleza | Implementación |
|---|---|---|
| **Observe** | Determinista | Datos vía Hummingbot API: velas, order book, posiciones, balances |
| **Orient** | Probabilística / routines | Indicadores y señales en routines de Python |
| **Decide** | Probabilística | El LLM aplica las reglas de `agent.md` |
| **Act** | Determinista | Crear o detener executors vía MCP → API |

La **capa de ejecución** (executors, posiciones y risk engine) es Python determinista. El resultado es un agente que *piensa* como un trader discrecional y *actúa* como uno sistemático, con trazabilidad completa de cada tick.

### 8.3 Principio clave: todo pasa por executors

Cada agente crea sus executors con `controller_id == agent_id`. Eso aporta:

1. **Aislamiento:** dos agentes sobre la misma cuenta no ven ni tocan los executors del otro.
2. **Portafolio virtual:** cada agente lleva sus propias posiciones, precio de breakeven y P&L realizado y no realizado.
3. **Position handover:** con `keep_position=true`, el inventario de un executor cerrado pasa al agente para el siguiente tick.

### 8.4 Ciclo de un tick (TickEngine)

En cada tick, cada N segundos, el TickEngine:

1. Ejecuta los *providers* y las routines.
2. Lee el journal.
3. Construye el prompt: `agent.md`, configs, datos y directivas.
4. Abre la sesión ACP con el LLM.
5. Captura las tool calls (MCP).
6. Escribe el snapshot.

### 8.5 Estructura en disco

```
~/condor/trading_agents/<agente>/
├── agent.md              # estrategia: frontmatter YAML + instrucciones Markdown
├── config.yml            # configuración de runtime (editable)
├── learnings.md          # memoria entre sesiones (máx. 20 insights)
├── routines/             # routines propias del agente
├── trading_sessions/
│   └── session_N/
│       ├── journal.md    # summary, decisions, ticks, executors
│       └── snapshots/    # prompt + respuesta + tool calls por tick (máx. 100)
└── dry_runs/
    └── experiment_N.md
```

| Archivo | Alcance | Función |
|---|---|---|
| `journal.md` | Una sesión | Memoria de corto plazo: log de ticks, decisiones y estado de executors |
| `learnings.md` | Todas las sesiones | Memoria de largo plazo: errores comunes, ajustes, observaciones de mercado |
| `snapshots/` | Un tick | Observabilidad total: permite depurar, entender, reproducir y auditar cada decisión |

### 8.6 `agent.md`: configs vs. limits

```yaml
---
name: <nombre>
tick_interval: 60           # segundos entre ticks
connectors:
  - hyperliquid_perpetual
configs:                    # el agente puede PROPONER cambios; el usuario los aprueba
  trading_pair: BTC-USD
limits:                     # SOLO el usuario los modifica; los aplica el Risk Engine
  max_position_size_quote: 100
  max_single_order_quote: 25
  max_daily_loss_quote: 10
  max_open_executors: 1
  max_drawdown_pct: 5
---
## Goal
...
## Rules
...
```

Límites que soporta el risk engine, con sus valores por defecto según el anuncio oficial de Condor:

| Límite | Default |
|---|---|
| `max_position_size_quote` | $500 |
| `max_daily_loss_quote` | $50 |
| `max_drawdown_pct` | 10 % |
| `max_open_executors` | 5 |
| `max_single_order_quote` | $100 |
| `max_cost_per_day_usd` | $5 (presupuesto de LLM) |
| `cooldown_after_loss_sec` | 300 |

Cuando se alcanza un límite, la sesión pasa a **Blocked** hasta que los límites se reinician.

### 8.7 Modos de ejecución

| Modo | Comportamiento |
|---|---|
| `dry_run` | Un tick **sin capacidad de trading**; el prompt indica que crear executors está bloqueado. Guarda un experimento |
| `run_once` | Un tick **con trading real** |
| `loop` | Modo estándar: un tick cada `frequency_sec` hasta que se detiene o se alcanza `max_ticks`. Crea una sesión con journal completo |

Flujo recomendado: `dry_run` → `run_once` → `loop`.

### 8.8 Agent Builder

Se abre con `/agent` → **Switch Mode** → **Agent Builder**, o desde la sección Agents del dashboard. Guía en fases:

1. **Strategy Design:** tipo de estrategia (grid, DCA, trend following, mean reversion), dirección, presupuesto, timeframe y tolerancia al riesgo.
2. **Market Data Routine:** crea una routine en Python que calcula indicadores de forma determinista, sin gastar tokens.
3. **Strategy Logic:** genera `agent.md`, que se refina iterando en conversación.
4. **Dry Run:** muestra qué haría el agente sin operar.
5. **Run Once:** un tick real.
6. **Loop:** despliegue con frecuencia y límites.

Buenas prácticas de la documentación:

- Empezar siempre con dry runs.
- Mover el cálculo de indicadores a routines.
- Estructurar `agent.md` en pasos explícitos: analizar → decidir → ejecutar.
- Revisar `learnings.md` con regularidad.

### 8.9 Gestión de sesiones

| Acción | Telegram | CLI / dashboard |
|---|---|---|
| Ver estado | `/agent` → agente → View Status | **Agents → [agente] → Sessions** |
| Nueva sesión | `/agent` → agente → New Session | `claude --agent ~/condor/trading_agents/<agente> --new-session` |
| Retomar sesión | `/agent` → agente → Resume | `claude --agent ~/condor/trading_agents/<agente>` |
| Inyectar directiva | `/agent` → agente → Inject Directive | `engine.inject_directive("...")` |
| Detener | `/agent` → agente → Stop Agent | — |

- **Directivas:** se incluyen en el prompt del siguiente tick bajo `USER DIRECTIVES` y después se borran. Ejemplo: *"Reduce exposure, news event in 10 min."*
- **Detener el agente:** las posiciones abiertas se conservan para gestionarlas a mano o reanudarlas.
- **Continuidad entre interfaces:** el estado vive en `~/condor` y la conexión con el LLM usa ACP. Una conversación iniciada en Telegram puede continuarse en Claude Code con el mismo historial y el mismo estado.

### 8.10 MCP: herramientas del agente

Condor levanta dos servidores MCP (JSON-RPC 2.0) automáticamente al lanzar un agente vía ACP.

**`hummingbot-mcp`: operaciones de trading contra la API**

| Tool | Función |
|---|---|
| `setup_connector()` / `configure_server()` | Credenciales y servidor |
| `get_portfolio_overview()` | Balances, posiciones, LP y órdenes |
| `get_market_data()` | Precios, velas, funding, order book |
| `set_account_position_mode_and_leverage()` | Ajustes de perpetuos |
| `search_history()` | Histórico de órdenes, posiciones y LP |
| `manage_executors()` | Crear, buscar, detener y ver logs. Tipos: `order_executor`, `position_executor`, `grid_executor`, `dca_executor`, `lp_executor` |
| `manage_bots()` / `manage_controllers()` | Bots y plantillas de controllers |
| `run_backtest()` / `manage_backtest_tasks()` | Backtesting |
| `explore_dex_pools()` / `explore_geckoterminal()` | Descubrimiento de mercado |

**`condor-mcp`: operaciones locales**

| Tool | Función |
|---|---|
| `manage_skill()` | Playbooks compartidos del asistente |
| `consult()` | Consulta bloqueante a un agente de dominio |
| `delegate()` | Tarea desatendida a un agente de dominio; notifica al terminar |
| `manage_routines()` | Descubrir, ejecutar, crear y editar routines |
| `manage_trading_agent()` | CRUD y ciclo de vida de agentes |
| `trading_agent_journal_read()` / `trading_agent_journal_write()` | Journal |
| `manage_memory()` / `manage_notes()` | Memoria por usuario y almacenamiento clave-valor |
| `send_notification()` | Mensajes por Telegram |
| `run_code()` | Python arbitrario dentro del proceso del bot (permiso `code_run`; solo admin por defecto) |

Las tools admiten **descubrimiento progresivo**: al llamarlas sin parámetros devuelven las opciones disponibles, y con parámetros parciales indican los siguientes pasos.

Configuración de `hummingbot-mcp`, mediante variables de entorno o `~/.hummingbot_mcp/server.yml`:

| Variable | Default |
|---|---|
| `HUMMINGBOT_API_URL` | `http://localhost:8000` |
| `HUMMINGBOT_USERNAME` | `admin` |
| `HUMMINGBOT_PASSWORD` | `admin` |

Estos valores deben coincidir con las credenciales reales de la API.

Arranque manual para pruebas:

```bash
python -m mcp_servers.hummingbot_api
python -m mcp_servers.condor
```

Para añadir tools propias se usa el decorador `@mcp.tool()` en `mcp_servers/hummingbot_api/tools/custom.py`.

---

## 9. Executors

| Executor | Descripción | Se basa en |
|---|---|---|
| Order | Una orden (`LIMIT`, `LIMIT_MAKER`, `MARKET`, `LIMIT_CHASER`) | — |
| **Position** | Orden + TP/SL/trailing/time limit (*triple barrier*) | Order |
| **Grid** | Varios niveles entre `start_price` y `end_price`, cada uno con su TP | Position |
| DCA | Varias entradas a distintos precios | Order |
| TWAP | Órdenes repartidas en el tiempo | Order |
| XEMM | Market making entre exchanges | Order |
| Arbitrage | Arbitraje entre mercados | Order |
| LP | Liquidez concentrada (CLMM, vía Gateway) | — |

**Tipos de cierre:** `TAKE_PROFIT`, `STOP_LOSS`, `TIME_LIMIT`, `TRAILING_STOP`, `EARLY_STOP`, `COMPLETED` y `FAILED`.

**Métricas estándar:** `net_pnl_quote`, `fees_paid_quote`, `volume_quote`, `close_type` y `duration_seconds`.

**Triple barrier (Position Executor):**

| Parámetro | Descripción |
|---|---|
| `take_profit` | Fracción de beneficio (0.02 = 2 %) |
| `stop_loss` | Fracción de pérdida |
| `time_limit` | Segundos |
| `trailing_stop` | `activation_price` y `trailing_delta` |
| `open_order_type` | Tipo de orden de entrada. Default: `LIMIT` |
| `take_profit_order_type` | Default: `MARKET` |
| `stop_loss_order_type` | Default: `MARKET` |
| `time_limit_order_type` | Default: `MARKET` |

**Parámetros principales del Grid Executor:**

| Grupo | Parámetros |
|---|---|
| Rango y dirección | `start_price`, `end_price`, `limit_price` (stop global), `side` |
| Tamaño | `total_amount_quote`, `min_spread_between_orders`, `min_order_amount_quote` |
| Ejecución | `max_open_orders`, `activation_bounds` |
| Riesgo | `triple_barrier_config`, `leverage`, `keep_position` |

**Operaciones con executors vía API:**

```bash
curl -u USUARIO:PASSWORD http://localhost:8000/executors                          # listar
curl -u USUARIO:PASSWORD "http://localhost:8000/executors?controller_id=mi-agente" # filtrar
curl -u USUARIO:PASSWORD -X DELETE http://localhost:8000/executors/{id}            # detener
```

---

## 10. Bots y controllers

| | Executors | Bots |
|---|---|---|
| Duración | Minutos u horas | Días o semanas |
| Alcance | Una operación | Estrategia completa |
| Uso típico | Operaciones puntuales | Market making o grid 24/7 |

Un **bot** es un contenedor Hummingbot que ejecuta uno o varios **controllers** (estrategias V2 que generan executors) o **scripts**. Los controllers integrados se agrupan por tipo:

| Tipo | Controllers |
|---|---|
| `market_making` | `pmm_simple`, `pmm_dynamic` |
| `directional_trading` | `bollinger_v1`, `macd_bb_v1`, `dman_v3` |
| `generic` | `grid_strike`, `arbitrage_controller`, `xemm_multiple_levels`, `pmm_mister` |

Flujo de trabajo:

1. Crear la config del controller en el **Editor** del dashboard, en Telegram o vía API.
2. Desplegar con `/bots` → **Create New Bot**, seleccionando las configs.
3. Deploy.

Antes del despliegue se puede backtestear con `run_backtest()` o con la sección Backtesting.

---

## 11. Routines

Las routines son scripts de Python deterministas que se ejecutan directamente, sin cargarse en el contexto del LLM. Según la documentación, llevaron el tiempo de sesión de unos 2 min a menos de 1 min y redujeron mucho el consumo de tokens.

Estructura mínima: una clase `Config` (Pydantic) y una función `async run()`.

```python
# routines/hello_world.py
from pydantic import BaseModel, Field
from telegram.ext import ContextTypes

class Config(BaseModel):
    """Simple hello world example routine."""   # el docstring se muestra como descripción en la UI
    name: str = Field(default="World", description="Name to greet")

async def run(config: Config, context: ContextTypes.DEFAULT_TYPE) -> str:
    return f"Hello, {config.name}!"
```

Tipos de routine:

| Tipo | Comportamiento | Ubicación |
|---|---|---|
| One-shot | Se ejecuta una vez y puede programarse | — |
| Continuous | `CONTINUOUS = True`, con bucle interno (monitorización y alertas) | — |
| Global | Disponible para todos los agentes | `condor/routines/` |
| De agente | Solo para un agente | `trading_agents/<agente>/routines/` |

Las routines se ejecutan con `/routines` o desde el dashboard. También pueden generar **reports** en HTML.

---

## 12. Estrategias básicas de prueba en Hyperliquid

> **[Nota del autor]** Plan incremental: cada nivel valida una capa del stack antes de subir la complejidad. Los importes son de prueba. Hyperliquid exige un nocional mínimo por orden (del orden de 10 USD), así que conviene comprobar las *trading rules* antes (`GET /connectors/hyperliquid_perpetual/trading-rules` en Swagger). Si se dispone de fondos en testnet, se recomienda empezar allí (`hyperliquid_perpetual_testnet`).

### Nivel 0: conectividad

1. `/portfolio` debe mostrar el saldo de Hyperliquid.
2. Consultar un precio: `GET /market-data/prices` en Swagger, o preguntar al agente *"price of BTC-USD on hyperliquid_perpetual"*.

### Nivel 1: orden manual (valida credenciales y trading)

- `/trade` → Hyperliquid → `BTC-USD` → BUY → LIMIT alejado del precio (no debe ejecutarse) → confirmar.
- Comprobar la orden en el order book y cancelarla.
- Repetir con MARKET por el mínimo y cerrar la posición.

### Nivel 2: Position Executor (triple barrier)

Una entrada con salidas automáticas, creada desde **Executors** en el dashboard o por API:

```bash
curl -u USUARIO:PASSWORD -X POST http://localhost:8000/executors/ \
  -H "Content-Type: application/json" \
  -d '{
    "executor_config": {
      "type": "position_executor",
      "connector_name": "hyperliquid_perpetual",
      "trading_pair": "BTC-USD",
      "side": "BUY",
      "amount": 0.0002,
      "leverage": 2,
      "triple_barrier_config": {
        "take_profit": 0.01,
        "stop_loss": 0.005,
        "time_limit": 1800
      }
    }
  }'
```

> **[Nota del autor]** El esquema exacto (p. ej. si `account_name` va en el cuerpo) debe validarse con `GET /executors/types/position_executor/config` antes de lanzar la petición. `amount` está en unidades del activo base; hay que ajustarlo para que el nocional supere el mínimo.

Qué observar:

- El executor pasa por `ACTIVE` y termina con `TAKE_PROFIT`, `STOP_LOSS` o `TIME_LIMIT`.
- Revisar `net_pnl_quote` y `fees_paid_quote`.

### Nivel 3: Grid Executor en rango

Conviene elegir un par líquido y lateral (`ETH-USD`, `SOL-USD`):

- `side`: `BUY`
- `start_price` / `end_price`: ±1–2 % alrededor del precio medio
- `limit_price`: por debajo de `start_price`
- `total_amount_quote`: 50–100
- `max_open_orders`: 3
- `leverage`: 1–2
- `triple_barrier_config.stop_loss`: 0.03

Se sigue en el dashboard (**Executors**), vigilando el estado de cada nivel (`OPEN_ORDER_PLACED` → `CLOSE_ORDER_PLACED` → `COMPLETE`).

### Nivel 4: bot con controller `pmm_simple`

Market making con spreads fijos durante un periodo largo. Config adaptada del ejemplo oficial (que usa `hyperliquid` spot con `HYPE-USDC`) a perpetuos:

```yaml
id: pmm_simple_hl_test
controller_name: pmm_simple
controller_type: market_making
total_amount_quote: '100'
connector_name: hyperliquid_perpetual
trading_pair: HYPE-USD
buy_spreads:
- 0.002
sell_spreads:
- 0.002
buy_amounts_pct:
- '1'
sell_amounts_pct:
- '1'
executor_refresh_time: 60
cooldown_time: 15
leverage: 1
position_mode: ONEWAY        # los perpetuos de Hyperliquid solo admiten one-way
stop_loss: '0.005'
take_profit: '0.002'
time_limit: 2700
```

Proceso:

1. Crear la config en el Editor.
2. Opcional: backtest.
3. `/bots` → Create New Bot → seleccionar la config → Deploy.
4. Seguimiento en **Bots**: performance, runs y logs.

### Nivel 5: Trading Agent (harness completo)

Se crea con Agent Builder (`/agent` → Switch Mode → Agent Builder) y una descripción como esta:

```
Quiero un grid scalper en hyperliquid_perpetual para SOL-USD.
Presupuesto total 100 USD, leverage 2, un solo grid a la vez.
Crea una routine que calcule mid price, volatilidad 1h y un spread recomendado.
Si el precio se aleja más de 1x el spread del centro del grid, reemplázalo.
Stop si el PnL no realizado cae por debajo de -10 USD.
```

`agent.md` esperado, siguiendo el ejemplo *Grid Scalper* de la documentación adaptado a Hyperliquid:

```yaml
---
name: Grid Scalper SOL-USD (HL)
description: Grid scalping en SOL-USD, hyperliquid_perpetual
agent_key: claude-code
default_config:
  connector_name: hyperliquid_perpetual
  trading_pair: SOL-USD
  total_amount_quote: 100
  leverage: 2
  frequency_sec: 60
  risk_limits:
    max_loss_per_tick_quote: 10
    max_total_drawdown_quote: 20
---
# Grid Scalper — SOL-USD

## Step 1 — Run Analysis
Llamar a la routine `spread_analyzer`: mid_price, volatility_1h,
recommended_spread_pct, grid_upper, grid_lower, signal.

## Step 2 — Decision Logic
- bullish_bias → grid LONG · bearish_bias → grid SHORT · neutral → LONG
- Saltar tick si recommended_spread_pct < 0.08
- Reemplazar grid si mid_price se movió > 1x el spread desde el centro

## CRITICAL: One Position Only
Hyperliquid solo permite una posición por par. Antes de desplegar,
comprobar executors activos; si existe uno, NO desplegar otro.

## Risk Rules
- Máx. 100 USD desplegados
- Un único grid executor activo
- Si el PnL no realizado < -10 USD, detener el grid inmediatamente
- Registrar cada decisión con su razonamiento en el journal
```

Secuencia de validación:

1. **Dry Run** (`/agent` → agente → Dry Run). Revisar el snapshot en **Agents → [agente] → Sessions**: datos recibidos, razonamiento y decisión.
2. Corregir `agent.md` si el razonamiento no es el esperado. Un error típico que recoge la documentación es crear grids por debajo del precio, que se cierran de inmediato por TP.
3. **Run Once:** un tick real. Verificar el executor creado.
4. **Loop** con 60 s de frecuencia. Supervisar el journal y `learnings.md`, y usar directivas si hace falta.
5. **Stop Agent** al terminar la prueba. Las posiciones se conservan, así que hay que cerrarlas a mano si procede.

---

## 13. Seguridad en local

- **Bindings:** todos los puertos de la API (8000, 5432, 1883, 18083) escuchan en `127.0.0.1` por defecto y no deben ampliarse (`API_BIND`, `DB_BIND`).
- **Dashboard en modo Local:** no tiene login y **no debe exponerse**, así que `WEB_HOST=0.0.0.0` queda descartado. En modo Telegram conviene fijar `WEB_HOST=127.0.0.1`, porque en local no hace falta acceder desde otra máquina.
- **Credenciales de la API:** deben ser robustas. Las de `hummingbot-mcp` tienen que coincidir con ellas y no quedarse en `admin/admin`.
- **Hyperliquid:** usar solo una **API wallet** (sin retiros) y rotarla antes de que caduque. La private key de la wallet principal nunca va en Hummingbot.
- **`run_code()`:** ejecuta Python arbitrario, así que no debe concederse a usuarios que no sean admin.
- **Límites:** son la última barrera, porque el agente no puede modificarlos. Deben configurarse con valores conservadores en cada `agent.md`.
- **Privacidad:** `CONDOR_TELEMETRY=off` y `CONDOR_SHARING=off` en `.env` si no se desea enviar telemetría ni compartir transcripciones.
- **Git:** no versionar `.env`, `config.yml` ni `hummingbot-api/.env`.

---

## 14. Troubleshooting

| Síntoma | Diagnóstico / solución |
|---|---|
| Fallo general | `make doctor` en `condor/` y en `hummingbot-api/` |
| Condor no arranca | `make run-fg` para ver el traceback; `make logs` |
| El bot de Telegram no responde | Revisar `TELEGRAM_TOKEN` y `ADMIN_USER_ID`; `make status` |
| Los comandos fallan | Comprobar la API: `docker compose ps` y `curl -u U:P http://localhost:8000/` |
| Error de autenticación con la API | Las credenciales de `/servers` o de Settings → Servers deben coincidir con `hummingbot-api/.env` |
| "Command not found" del LLM | `which claude` y `echo $PATH`; después `make stop && make run` |
| Se cambió el LLM y sigue usando el anterior | `Change LLM` solo afecta a sesiones nuevas: `/agent` → Stop → Start |
| Autenticación de Claude Code | `claude auth logout && claude auth login` |
| Ollama no responde | `curl http://localhost:11434/api/tags`; `ollama serve` |
| Hyperliquid rechaza la orden | Revisar el nocional mínimo, el formato del par (`BTC-USD`), el modo one-way y el margen aislado en HIP-3 |
| El agente abre posiciones duplicadas | Añadir la regla *One Position Only* en `agent.md` y fijar `max_open_executors: 1` |

---

## 15. Referencias

| Recurso | Enlace |
|---|---|
| Documentación Condor | https://condor.hummingbot.org |
| Índice para LLM | https://condor.hummingbot.org/llms.txt |
| Instalación | https://condor.hummingbot.org/getting-started/installing |
| Integración LLM | https://condor.hummingbot.org/llm-integration |
| Trading Agents | https://condor.hummingbot.org/trading-agents/overview |
| Executors | https://condor.hummingbot.org/executors/overview |
| Connector Hyperliquid | https://hummingbot.org/exchanges/hyperliquid/ |
| Instalación del cliente Hummingbot | https://hummingbot.org/client/installation/ |
| Anuncio de Condor | https://hummingbot.org/blog/introducing-condor-the-open-source-harness-for-trading-agents/ |
| Repositorios | https://github.com/hummingbot/condor · https://github.com/hummingbot/hummingbot-api · https://github.com/hummingbot/deploy |
| Soporte | Discord de Hummingbot, canal `#condor-feedback` |
