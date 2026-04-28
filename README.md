# ai-observability-stack

Stack de observabilidad self-hosted para runtimes de LLMs locales y remotos.
Métricas de hardware vía Prometheus + Grafana, trazabilidad semántica vía
LiteLLM + OTel Collector + Grafana.

## Visión general

Proyecto Docker Compose independiente diseñado para coexistir con cualquier
runtime de modelos e interceptar llamadas API sin acoplarse al entorno
subyacente.

Dos pipelines corren en paralelo de forma independiente: uno para métricas de
hardware y contenedores, otro para trazas semánticas de llamadas LLM. Ambos
convergen en Grafana como punto único de visualización.

Probado con Ollama (local, AMD RX 6600) y Google Gemini (remoto) vía Open
WebUI. La arquitectura es agnóstica al proveedor, LiteLLM soporta más de
100 modelos y proveedores.

## Arquitectura

```
HOST
│
│  ┌────────────────────────────────────────────────────────────────┐
│  │  docker-compose.yml  ·  red: observability                     │
│  │                                                                │
│  │  ┌─────────────┐   ┌────────────────┐   ┌───────────────────┐  │
│  │  │  Prometheus │   │   ClickHouse   │   │  OTel Collector   │  │
│  │  │  :9090      │   │   (interno)    │   │  :4317 / :4318    │  │
│  │  └──────┬──────┘   └───────▲────────┘   └─────────▲─────────┘  │
│  │         │ pull             │ escribe              │ recibe     │
│  │  ┌──────▼──────────────────┴──────────────────────┘            │
│  │  │   Grafana :3001                                |            │
│  │  │   datasources: Prometheus · ClickHouse         |            │
│  │  └────────────────────────────────────────────────┘            │
│  └────────────────────────────────────────────────────────────────┘
│
│  ┌────────────────────────────────────────────────────────────────┐
│  │  docker-compose.exporters.yml  ·  include: core                │
│  │                                                                │
│  │  ┌─────────────────────────────────────────────────────────┐   │
│  │  │  node-exporter · cAdvisor          → red observability  │   │
│  │  └─────────────────────────────────────────────────────────┘   │
│  └────────────────────────────────────────────────────────────────┘
│
│  ┌────────────────────────────────────────────────────────────────┐
│  │  docker-compose.ai.yml  ·  include: exporters                  │
│  │                                                                │
│  │  ┌──────────────────┐                                          │
│  │  │  LiteLLM :8585   │─── OTel spans HTTP :4318 ──> otelcol     │
│  │  └────────┬─────────┘    (vía red observability)               │
│  │           │ API OpenAI-compatible (red inference)              │
│  │  red propia: inference                                         │
│  └───────────┼────────────────────────────────────────────────────┘
│              │
│  ┌───────────┼────────────────────────────────────────────────────┐
│  │  compose externo (ej. local-ai-lab)  ·  se une a: inference    │
│  │           │                                                    │
│  │  Open WebUI ──OpenAI API──> litellm:4000                       │
│  │  Ollama ──────────────────> litellm (modelos ollama/*)         │
│  └────────────────────────────────────────────────────────────────┘
│
│  Pipeline métricas:  node-exporter / cAdvisor → Prometheus → Grafana
│  Pipeline trazas:    LiteLLM → OTel Collector → ClickHouse → Grafana
```

## Stack

| Herramienta | Capa | Función |
|---|---|---|
| [node-exporter](https://github.com/prometheus/node_exporter) | Colección | Métricas del host: CPU, RAM, GPU (vía DRM) |
| [cAdvisor](https://github.com/google/cadvisor) | Colección | Métricas por contenedor: CPU y RAM |
| [Prometheus](https://prometheus.io/) | Almacenamiento | Scrape y retención de series temporales |
| [ClickHouse](https://clickhouse.com/) | Almacenamiento | Base de datos columnar para trazas OTel |
| [OTel Collector](https://opentelemetry.io/docs/collector/) | Colección | Receptor de spans OTLP; escribe en ClickHouse |
| [LiteLLM](https://github.com/BerriAI/litellm) | Proxy LLM | API OpenAI-compatible; genera spans OTel por llamada |
| [Grafana](https://grafana.com/) | Visualización | Dashboards de métricas y trazas LLM centralizados |

## Servicios

| Servicio | Compose | Puerto host | Descripción |
|---|---|---|---|
| Grafana | `docker-compose.yml` | 3001 | Visualización centralizada (métricas + trazas LLM) |
| Prometheus | `docker-compose.yml` | 9090 | Almacenamiento de métricas |
| ClickHouse | `docker-compose.yml` | — | Storage de trazas OTel (red interna) |
| OTel Collector | `docker-compose.yml` | 4317 / 4318 | Receptor de spans (gRPC / HTTP) |
| node-exporter | `docker-compose.exporters.yml` | — | Métricas de hardware del host |
| cAdvisor | `docker-compose.exporters.yml` | — | Métricas por contenedor |
| LiteLLM | `docker-compose.ai.yml` | 8585 | Proxy LLM + generación de spans OTel |

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

Todas las métricas admiten el label `name` para filtrar por contenedor.

| Métrica | Descripción |
|---|---|
| `rate(container_cpu_usage_seconds_total[1m])` | Uso de CPU por contenedor |
| `container_memory_working_set_bytes` | RAM real en uso (excluye cache) |
| `container_memory_limit_bytes` | Límite de RAM configurado |
| `rate(container_network_receive_bytes_total[1m])` | Tráfico de red entrante por contenedor |
| `rate(container_network_transmit_bytes_total[1m])` | Tráfico de red saliente por contenedor |

### Trazas semánticas LLM — Grafana + ClickHouse

Las trazas se visualizan en Grafana (`:3001`) mediante el datasource ClickHouse.
Cada span generado por LiteLLM incluye:

| Atributo OTel | Descripción |
|---|---|
| `gen_ai.request.model` | Modelo invocado |
| `gen_ai.usage.input_tokens` | Tokens del prompt |
| `gen_ai.usage.output_tokens` | Tokens de la respuesta |
| `gen_ai.usage.total_tokens` | Total de tokens consumidos |
| `Duration` | Latencia de extremo a extremo (nanosegundos) |

Los spans se almacenan en la tabla `otel_traces` de ClickHouse y son consultables
mediante SQL estándar desde Grafana.

## Dashboards de Grafana

| Dashboard | Datasource | Descripción |
|---|---|---|
| Hardware | Prometheus | CPU, RAM, carga del sistema y GPU AMD (node-exporter) |
| cAdvisor | Prometheus | CPU y RAM por contenedor |
| LLM Observability | ClickHouse | Métricas agregadas de llamadas LLM por modelo: requests, tokens, latencia |

El dashboard **LLM Observability** (`grafana/provisioning/dashboards/llm-observability.json`)
incluye dos secciones: **Overview** con selector de modelo para filtrar métricas
agregadas, y **By Model** con una tabla comparativa de todos los modelos activos.

## Levantar el stack

Los composes se encadenan mediante `include`. Cada capa superior levanta
automáticamente las inferiores.

```bash
# Solo el core de observabilidad
docker compose up -d

# Core + exporters de infraestructura (node-exporter, cAdvisor)
docker compose -f docker-compose.exporters.yml up -d

# Stack completo — core + exporters + proxy LLM
docker compose -f docker-compose.ai.yml up -d
```

Levantar `docker-compose.ai.yml` antes que los composes externos es obligatorio,
es quien crea la red `inference` que los composes externos necesitan unirse.

## Adaptar a tu entorno

### Integrar tu entorno externo

Para que las llamadas LLM generen trazas en Grafana, el cliente debe enrutar
las peticiones a través de LiteLLM. Los servicios que necesiten comunicarse
con LiteLLM u Ollama deben unirse a la red `inference`:

```yaml
networks:
  inference:
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
    - inference   # necesaria para resolver litellm y ollama por nombre
```

`OPENAI_API_KEY` puede ser cualquier valor, LiteLLM no la valida en local.

El mismo patrón aplica a cualquier cliente con soporte para API
OpenAI-compatible: agentes LangChain, LlamaIndex, scripts con `openai` SDK, etc.

### Modelos en LiteLLM

LiteLLM no hace autodiscovery; cada modelo debe declararse explícitamente en
`assets/litellm-config.yaml`. Añadir un modelo nuevo requiere una entrada y
recrear el contenedor:

```bash
docker compose -f docker-compose.ai.yml up -d --force-recreate litellm
```

### Proveedores remotos

Añadir la API key al fichero `.env` (ver `.env.example`) y declarar el modelo
en `litellm-config.yaml`. Proveedores soportados: [docs LiteLLM](https://docs.litellm.ai/docs/providers).

### GPU

Las métricas `node_drm_*` funcionan con GPU AMD vía el driver `amdgpu`. El
label `card` (`card0`, `card1`, etc.) puede variar según el sistema, verificar
con `node_drm_card_info` y ajustar las queries de Grafana.

Para NVIDIA, node-exporter no cubre GPU. La alternativa habitual es
[dcgm-exporter](https://github.com/NVIDIA/dcgm-exporter), que expone métricas
bajo el prefijo `DCGM_FI_*`.

## Licencia

[MIT](LICENSE)
