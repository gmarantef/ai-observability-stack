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
