# ai-observability-stack

Stack de observabilidad self-hosted para runtimes de LLMs locales y remotos. Proyecto Docker independiente que puede conectarse a cualquier compose de modelos o interceptar llamadas de agentes en el host.

## Qué construimos

Dos pipelines paralelos e independientes:

**Hardware e infraestructura** → Prometheus scrape → Grafana
- Métricas de GPU, VRAM, temperatura vía nvidia-exporter
- Métricas de runtime de Ollama desde su endpoint `/metrics` nativo
- Solo aplica a modelos locales

**Trazabilidad semántica** → OTel → OpenLIT
- Spans por llamada: prompt, respuesta, tokens, latencia, coste estimado
- Aplica a modelos locales (Ollama vía OTel) y remotos (vía proxy)
- OpenLIT incluye su OTel Collector — no es pieza separada

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
│                  red Docker externa              │
│         ┌────────────────────────────────────┐  │
│         │  OLLAMA COMPOSE                    │  │
│         │  ollama:11434/metrics ─────────────┘  │
│         └────────────────────────────────────┘  │
│                                                  │
└── proxy reenvía ──→ api.anthropic.com / openai  │
```

## Red compartida entre composes

Este compose crea y es dueño de la red `monitoring`. Los composes externos la adoptan como `external: true`.

```yaml
# Este compose — crea la red
networks:
  monitoring:
    name: monitoring
    driver: bridge

# Compose externo (ej. Ollama) — se une
networks:
  monitoring:
    external: true
```

Este compose debe estar levantado antes que los demás — es quien crea la red.

## Proxy para modelos remotos

El compose expone el proxy en el host. Los agentes se configuran vía variables de entorno:

```bash
export ANTHROPIC_BASE_URL=http://localhost:8585
export OPENAI_BASE_URL=http://localhost:8586
```

El proxy intercepta, registra el span OTel en OpenLIT, y reenvía a la API real. Transparente para el agente.

## Stack

| Capa | Herramienta |
|---|---|
| Hardware e infraestructura | Prometheus + Grafana |
| Semántica local y remota | OpenLIT |

Langfuse es la alternativa a OpenLIT si en el futuro se necesita gestión de prompts y evaluaciones a escala. Arize Phoenix es complemento interesante para comparación cualitativa de modelos.

## Contexto

Este proyecto forma parte de un entorno de IA local más amplio. Ver nota del vault: `Observabilidad del entorno de IA local`.
