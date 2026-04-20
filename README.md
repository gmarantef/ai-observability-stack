# ai-observability-stack

Stack de observabilidad self-hosted para runtimes de LLMs locales y remotos — Prometheus + Grafana para métricas de hardware, OpenLIT para trazas semánticas.

## Visión general

Proyecto Docker Compose independiente que se conecta a cualquier runtime de modelos local e intercepta llamadas API de agentes corriendo en el host. Diseñado para funcionar junto a entornos de modelos separados sin acoplarse a ellos.

Dos pipelines independientes:

| Pipeline | Protocolo | Herramientas | Alcance |
|---|---|---|---|
| Hardware y runtime | Prometheus scrape | Prometheus + Grafana | Modelos locales |
| Trazas semánticas | OpenTelemetry | OpenLIT | Modelos locales y remotos |

## Arquitectura

```
HOST
├── Claude Code ──ANTHROPIC_BASE_URL──┐
├── Codex ────────OPENAI_BASE_URL─────┤
│                                     ▼
│         ┌──────────────────────────────────────┐
│         │        ESTE COMPOSE                  │
│         │  ┌─────────┐   ┌──────────────────┐  │
│         │  │  Proxy  │   │   Prometheus     │  │
│         │  │ :8585   │   │   (scraper)      │  │
│         │  └────┬────┘   └────────┬─────────┘  │
│         │       │ OTel            │ pull        │
│         │       ▼                 ▼             │
│         │  ┌─────────────────────────────────┐  │
│         │  │  OpenLIT          Grafana        │  │
│         │  └─────────────────────────────────┘  │
│         └──────────────────┬───────────────────┘│
│                  red Docker externa compartida   │
│         ┌────────────────────────────────────┐  │
│         │  OLLAMA COMPOSE (externo)          │  │
│         │  ollama:11434/metrics ─────────────┘  │
│         └────────────────────────────────────┘  │
│                                                  │
└── proxy reenvía ──→ api.anthropic.com / openai   │
```

## Etapas de desarrollo

| # | Etapa | Rama | Estado |
|---|---|---|---|
| 1 | Métricas de hardware | `feat/hardware-metrics` | Pendiente |
| 2 | Métricas de runtime Ollama | `feat/ollama-runtime-metrics` | Pendiente |
| 3 | Trazabilidad semántica — modelos locales | `feat/otel-local` | Pendiente |
| 4 | Trazabilidad semántica — modelos remotos | `feat/otel-remote-proxy` | Pendiente |

### Etapa 1 — Métricas de hardware

Colección de métricas del host (CPU, RAM, GPU AMD) y desglose por contenedor Docker.

- **`node_exporter`** (`prom/node-exporter`) con `--collector.drm`: GPU load %, VRAM usada/total, GTT (RAM de sistema en offload), temperatura, frecuencias CPU/GPU, RAM total del host.
- **`cAdvisor`** (`gcr.io/cadvisor/cadvisor`): CPU y RAM desglosados por contenedor — permite ver qué servicio (ej. `ollama`) está consumiendo qué.
- Ambos son host-level: cubren este compose y cualquier compose externo actual o futuro sin modificarlos.
- Granularidad GPU por modelo concreto (qué modelo dentro de Ollama usa cuánta VRAM): esto es información semántica, se captura en Etapa 3 vía OTel.

### Etapa 2 — Métricas de runtime Ollama

Métricas del propio runtime de Ollama desde su endpoint `/metrics` nativo (`:11434`).

- Tokens por segundo, latencia de inferencia, modelos cargados en VRAM, cola de requests.
- Prometheus scrape directo al contenedor `ollama` a través de la red `monitoring`.
- Depende de Etapa 1 (Prometheus operativo).

### Etapa 3 — Trazabilidad semántica — modelos locales

Captura de trazas OTel de cada llamada de inferencia a modelos locales.

- **OpenLIT** desplegado en este compose, con su OTel Collector embebido.
- Los composes externos instrumentan sus llamadas para enviar spans al collector de OpenLIT.
- Atributos por span: prompt, respuesta, modelo, tokens, latencia, coste estimado.
- Permite diferenciar consumo por modelo concreto (complementa la vista de hardware de Etapa 1).

### Etapa 4 — Trazabilidad semántica — modelos remotos

Extensión de la capa semántica a APIs remotas (Anthropic, OpenAI).

- Proxy HTTP en este compose (`:8585` Anthropic, `:8586` OpenAI).
- Intercepta llamadas de agentes locales (Claude Code, Codex), inyecta span OTel y reenvía de forma transparente.
- Misma UI OpenLIT que Etapa 3 — visión unificada de modelos locales y remotos.

## Stack

- **Prometheus** — scraping de métricas de runtime de Ollama y hardware GPU
- **Grafana** — visualización de dashboards de hardware y runtime
- **OpenLIT** — colección de trazas OTel de modelos locales y llamadas remotas vía proxy, incluye su propio OTel Collector

## Conectar composes externos

Este compose crea y es dueño de la red Docker `monitoring`. Los composes externos se unen a ella:

```yaml
# Compose externo (ej. Ollama)
networks:
  monitoring:
    external: true
```

Este compose debe estar levantado antes que cualquier compose externo.

## Interceptar llamadas a APIs remotas

Apuntar los agentes al proxy local mediante variables de entorno:

```bash
export ANTHROPIC_BASE_URL=http://localhost:8585
export OPENAI_BASE_URL=http://localhost:8586
```

El proxy registra cada request como span OTel en OpenLIT y reenvía de forma transparente a la API real.

## Servicios

| Servicio | Puerto | Descripción |
|---|---|---|
| Grafana | 3000 | Dashboards de hardware y runtime |
| OpenLIT | 3001 | Explorador de trazas semánticas |
| Prometheus | 9090 | Almacenamiento de métricas |
| Proxy (Anthropic) | 8585 | Intercepta llamadas a la API de Anthropic |
| Proxy (OpenAI) | 8586 | Intercepta llamadas a APIs compatibles con OpenAI |
