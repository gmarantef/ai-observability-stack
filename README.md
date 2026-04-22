# ai-observability-stack

Stack de observabilidad self-hosted para runtimes de LLMs locales y remotos.
Métricas de hardware vía Prometheus + Grafana, trazabilidad semántica vía
LiteLLM + OpenLIT.

## Visión general

Proyecto Docker Compose independiente diseñado para coexistir con cualquier
runtime de modelos e interceptar llamadas API sin acoplarse al entorno
subyacente.

Dos pipelines corren en paralelo de forma independiente: uno para métricas de
hardware y contenedores, otro para trazas semánticas de llamadas LLM.

Probado con Ollama (local, AMD RX 6600) y Google Gemini (remoto) vía Open
WebUI. La arquitectura es agnóstica al proveedor, LiteLLM soporta más de
100 modelos y proveedores.

## Arquitectura

```
        ┌──────────────────────────────────────────────────────┐
        │  docker-compose.yml            (red: monitoring)     │
        │  ┌──────────────┐  ┌────────────────────────────┐    │
        │  │  Prometheus  │  │  node-exporter             │    │
        │  │  :9090       │<─│  cAdvisor                  │    │
        │  └──────┬───────┘  └────────────────────────────┘    │
        │         │ pull                                       │
        │  ┌──────▼───────┐                                    │
        │  │  Grafana     │                                    │
        │  │  :3001       │                                    │
        │  └──────────────┘                                    │
        └──────────────────────────────────────────────────────┘

        ┌──────────────────────────────────────────────────────┐
        │  docker-compose.tracing.yml  (red: tracing)          │
        │  ┌────────────────────────┐                          │
        │  │  LiteLLM :8585         │── OTel spans (:4318) ──┐ │
        │  │  (API OpenAI-compat)   │                        │ │
        │  └────────────────────────┘                        │ │
        │  ┌──────────────────────────────────────────────┐  │ │
        │  │  OpenLIT :3002 (UI + OTel Collector)         │<─┘ │
        │  └──────────────┬───────────────────────────────┘    │
        │  ┌──────────────▼───────────────────────────────┐    │
        │  │  ClickHouse (interno, red tracing)           │    │
        │  └──────────────────────────────────────────────┘    │
        └──────────────────────────────────────────────────────┘

        ┌──────────────────────────────────────────────────────┐
        │  compose externo (ej. local-ai-lab)                  │
        │                                                      │
        │  Open WebUI ──OpenAI API──> LiteLLM ──> Ollama       │
        │                                 │ OTel spans         │
        │                            OpenLIT:4318              │
        └──────────────────────────────────────────────────────┘

  Pipeline métricas:  node-exporter / cAdvisor → Prometheus → Grafana
  Pipeline trazas:    cliente → LiteLLM → modelo → OTel → OpenLIT
```

## Stack

| Herramienta | Capa | Función |
|---|---|---|
| [node-exporter](https://github.com/prometheus/node_exporter) | Colección | Métricas del host: CPU, RAM, GPU (vía DRM) |
| [cAdvisor](https://github.com/google/cadvisor) | Colección | Métricas por contenedor: CPU y RAM |
| [Prometheus](https://prometheus.io/) | Almacenamiento | Scrape y retención de series temporales |
| [Grafana](https://grafana.com/) | Visualización | Dashboards de hardware y contenedores |
| [LiteLLM](https://github.com/BerriAI/litellm) | Proxy LLM | API OpenAI-compatible; genera spans OTel por llamada |
| [ClickHouse](https://clickhouse.com/) | Almacenamiento | Base de datos columnar para trazas OTel |
| [OpenLIT](https://github.com/openlit/openlit) | Trazabilidad | UI de trazas semánticas + OTel Collector embebido |

## Servicios

| Servicio | Compose | Puerto host | Descripción |
|---|---|---|---|
| Grafana | `docker-compose.yml` | 3001 | Dashboards de hardware y contenedores |
| Prometheus | `docker-compose.yml` | 9090 | Almacenamiento de métricas |
| node-exporter | `docker-compose.yml` | — | Métricas de hardware del host |
| cAdvisor | `docker-compose.yml` | — | Métricas por contenedor |
| LiteLLM | `docker-compose.tracing.yml` | 8585 | Proxy LLM + generación de spans OTel |
| OpenLIT | `docker-compose.tracing.yml` | 3002 | UI de trazas semánticas |
| OTel Collector (OpenLIT) | `docker-compose.tracing.yml` | 4317 / 4318 | Receptor de spans (gRPC / HTTP) |
| ClickHouse | `docker-compose.tracing.yml` | — | Storage de trazas (red interna `tracing`) |

## Métricas principales

### Hardware del host — node-exporter

Métricas expuestas en `:9100/metrics`, scrapeadas por Prometheus cada 15 s.

| Métrica | Descripción |
|---|---|
| `rate(node_cpu_seconds_total{mode!="idle"}[1m])` | Uso de CPU del host |
| `node_memory_MemAvailable_bytes` | RAM disponible |
| `node_load1` / `node_load5` | Carga del sistema (1 y 5 minutos) |
| `node_drm_gpu_busy_percent{card="card1"}` | Utilización de la GPU durante inferencia |
| `node_drm_memory_vram_used_bytes{card="card1"}` | VRAM ocupada por modelos cargados |
| `node_drm_memory_vram_size_bytes{card="card1"}` | VRAM total disponible |
| `node_drm_memory_gtt_used_bytes{card="card1"}` | GTT: RAM del sistema usada como overflow de VRAM |

Las métricas `node_drm_*` requieren el flag `--collector.drm` en node-exporter
y son específicas de GPU AMD (driver `amdgpu`). El label `card` puede variar
según el sistema, verificar con `node_drm_card_info`.

### Contenedores — cAdvisor

Todas las métricas admiten el label `name` para filtrar por contenedor:
`{name="ollama"}`.

| Métrica | Descripción |
|---|---|
| `rate(container_cpu_usage_seconds_total[1m])` | Uso de CPU por contenedor |
| `container_memory_working_set_bytes` | RAM real en uso (excluye cache — mejor indicador de presión) |
| `container_memory_limit_bytes` | Límite de RAM configurado |
| `rate(container_network_receive_bytes_total[1m])` | Tráfico de red entrante por contenedor |
| `rate(container_network_transmit_bytes_total[1m])` | Tráfico de red saliente por contenedor |

### Trazas semánticas — OpenLIT

Las trazas se exploran desde la UI de OpenLIT (`:3002`), no vía PromQL. Cada
span generado por LiteLLM incluye:

| Atributo | Descripción |
|---|---|
| `gen_ai.usage.input_tokens` | Tokens del prompt |
| `gen_ai.usage.output_tokens` | Tokens de la respuesta |
| `gen_ai.usage.total_tokens` | Total de tokens consumidos |
| `gen_ai.request.model` | Modelo invocado |
| `gen_ai.response.finish_reason` | Motivo de finalización (`stop`, `length`, etc.) |
| Duración del span | Latencia de extremo a extremo de la llamada LLM |
| `gen_ai.usage.cost` | Coste estimado (aplica a modelos remotos con pricing conocido) |

## Etapas de desarrollo

| # | Etapa | Estado |
|---|---|---|
| 1 | Métricas de hardware — node-exporter + cAdvisor + Prometheus | Completada |
| 2 | Métricas de runtime Ollama — proxy sidecar ollama-metrics | Completada |
| 3 | Dashboards Grafana — hardware, cAdvisor | Completada |
| 4 | Infraestructura OTel — ClickHouse + OpenLIT | Completada |
| 5 | Proxy LiteLLM — trazabilidad semántica local y remota | Completada |
| 6 | Simplificación — eliminación del proxy ollama-metrics | Completada |

La Etapa 2 (proxy ollama-metrics) fue explorada y descartada en la Etapa 6 — sus
métricas quedaron cubiertas con mayor detalle por LiteLLM + OpenLIT.

## Levantar el stack

Orden de arranque obligatorio: primero el stack de métricas (crea `monitoring`), luego el de trazas (crea `tracing`), luego los composes externos.

```bash
# Stack de métricas (crea la red monitoring)
docker compose up -d

# Stack de trazas (crea la red tracing)
docker compose -f docker-compose.tracing.yml up -d
```

## Adaptar a tu entorno

### Integrar tu entorno externo

Para que las llamadas LLM generen trazas en OpenLIT, el cliente debe apuntar
a LiteLLM en lugar de llamar directamente al modelo. Los servicios que necesiten
comunicarse con LiteLLM u Ollama deben unirse a la red `tracing`:

```yaml
networks:
  tracing:
    external: true
```

Ejemplo con Open WebUI:

```yaml
open-webui:
  image: ghcr.io/open-webui/open-webui:main
  environment:
    - OPENAI_API_BASE_URL=http://litellm:4000
    - OPENAI_API_KEY=sk-local
  networks:
    - ai-lab
    - tracing   # necesaria para resolver litellm por nombre
```

`OPENAI_API_KEY` puede ser cualquier valor, LiteLLM no la valida en local.

El mismo patrón aplica a cualquier cliente con soporte para API
OpenAI-compatible: agentes LangChain, LlamaIndex, scripts con `openai` SDK, etc.

### Modelos en LiteLLM

LiteLLM no hace autodiscovery, cada modelo debe declararse explícitamente en
`assets/litellm-config.yaml`. Añadir un modelo nuevo requiere una entrada y
recrear el contenedor:

```bash
docker compose -f docker-compose.tracing.yml up -d --force-recreate litellm
```

### Proveedores remotos

Añadir la API key al fichero `.env` (ver `.env.example`) y declarar el modelo
en `litellm-config.yaml`. Proveedores soportados por [LiteLLM](https://docs.litellm.ai/docs/providers).

### GPU

Las métricas `node_drm_*` funcionan con GPU AMD vía el driver `amdgpu`. El
label `card` (`card0`, `card1`, etc.) puede variar según el sistema, verificar
con `node_drm_card_info` y ajustar las queries de Grafana.

Para NVIDIA, node-exporter no cubre GPU. La alternativa habitual es
[dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter), que expone métricas
bajo el prefijo `DCGM_FI_*`.
