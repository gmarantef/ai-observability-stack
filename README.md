# ai-observability-stack

Stack de observabilidad self-hosted para runtimes de LLMs locales y remotos — Prometheus + Grafana para métricas de hardware y runtime, LiteLLM + OpenLIT para trazas semánticas.

## Visión general

Proyecto Docker Compose independiente que se conecta a cualquier runtime de modelos local e intercepta llamadas API de agentes. Diseñado para funcionar junto a entornos de modelos separados sin acoplarse a ellos.

Dos pipelines independientes:

| Pipeline | Protocolo | Herramientas | Alcance |
|---|---|---|---|
| Hardware y runtime | Prometheus scrape | Prometheus + Grafana | Modelos locales |
| Trazas semánticas | OpenTelemetry | LiteLLM + OpenLIT | Local y remoto |

## Arquitectura

```
│         ┌──────────────────────────────────────────────────────┐
│         │  docker-compose.yml            (red: monitoring)     │
│         │  ┌──────────────┐  ┌────────────────────────────┐   │
│         │  │  Prometheus  │  │  node-exporter             │   │
│         │  │  :9090       │  │  cAdvisor                  │   │
│         │  └──────┬───────┘  │  ollama-metrics :8082      │   │
│         │         │ pull     └────────────────────────────┘   │
│         │  ┌──────▼───────┐                                   │
│         │  │  Grafana     │                                   │
│         │  │  :3001       │                                   │
│         │  └──────────────┘                                   │
│         └──────────────────────────────────────────────────────┘
│
│         ┌──────────────────────────────────────────────────────┐
│         │  docker-compose.tracing.yml  (redes: tracing +       │
│         │                               monitoring)            │
│         │  ┌────────────────────────┐                         │
│         │  │  LiteLLM :8585         │── OTel spans (:4318) ──┐│
│         │  │  (API OpenAI-compat)   │                        ││
│         │  └────────────────────────┘                        ││
│         │  ┌──────────────────────────────────────────────┐  ││
│         │  │  OpenLIT :3002 (UI + OTel Collector)         │◄─┘│
│         │  └──────────────┬─────────────────────────────── ┘  │
│         │  ┌──────────────▼───────────────────────────────┐   │
│         │  │  ClickHouse (interno, red tracing)           │   │
│         │  └──────────────────────────────────────────────┘   │
│         └──────────────────────────────────────────────────────┘
│
│         ┌──────────────────────────────────────────────────────┐
│         │  local-ai-lab compose          (red: monitoring)     │
│         │                                                      │
│         │  Open WebUI ──OpenAI API──► litellm:4000             │
│         │        │                         │ OTel spans        │
│         │        └────Ollama API──► ollama-metrics:8082        │
│         │                                  │ Prometheus scrape │
│         │                           ollama:11434               │
│         └──────────────────────────────────────────────────────┘
│
│  Pipeline métricas:  Open WebUI → ollama-metrics → Ollama → Prometheus → Grafana
│  Pipeline trazas:    Open WebUI → LiteLLM → ollama-metrics → Ollama → OTel → OpenLIT
```

## Etapas de desarrollo

| # | Etapa | Rama | Estado |
|---|---|---|---|
| 1 | Métricas de hardware | `feat/hardware-metrics` | Completada |
| 2 | Métricas de runtime Ollama | `feat/ollama-runtime-metrics` | Completada |
| 3 | Dashboards Grafana | `feat/grafana-dashboards` | Completada |
| 4 | Trazabilidad semántica — infraestructura OTel | `feat/otel-local` | Completada |
| 5 | Trazabilidad semántica — proxy LiteLLM | `feat/otel-remote-proxy` | Completada |

### Etapa 1 — Métricas de hardware

Colección de métricas del host (CPU, RAM, GPU AMD) y desglose por contenedor Docker.

- **`node_exporter`** (`prom/node-exporter`) con `--collector.drm`: GPU load %, VRAM usada/total, GTT (RAM de sistema en offload), temperatura, frecuencias CPU/GPU, RAM total del host.
- **`cAdvisor`** (`gcr.io/cadvisor/cadvisor`): CPU y RAM desglosados por contenedor — permite ver qué servicio (ej. `ollama`) está consumiendo qué.
- Ambos son host-level: cubren este compose y cualquier compose externo actual o futuro sin modificarlos.
- Granularidad GPU por modelo concreto (qué modelo dentro de Ollama usa cuánta VRAM): esto es información semántica, se captura en Etapa 4 vía OTel.

#### Métricas clave disponibles vía node_exporter (AMD RX 6600)

| Métrica | Descripción |
|---|---|
| `node_drm_gpu_busy_percent{card="card1"}` | Utilización de la GPU durante inferencia |
| `node_drm_memory_vram_used_bytes{card="card1"}` | VRAM consumida por modelos cargados |
| `node_drm_memory_vram_size_bytes{card="card1"}` | VRAM total disponible (8 GB) |
| `node_drm_memory_gtt_used_bytes{card="card1"}` | GTT: RAM del sistema usada como overflow cuando el modelo no cabe en VRAM |
| `node_drm_card_info{card="card1",...}` | Metadatos de la GPU (vendor, fabricante de memoria) |
| `rate(node_cpu_seconds_total{mode!="idle"}[1m])` | Uso de CPU del host |
| `node_memory_MemAvailable_bytes` | RAM libre del host |
| `node_load1` / `node_load5` | Presión general del sistema |

#### Métricas clave disponibles vía cAdvisor (por contenedor)

| Métrica | Descripción |
|---|---|
| `rate(container_cpu_usage_seconds_total[1m])` | Uso de CPU por contenedor (núcleos) |
| `container_memory_usage_bytes` | RAM total consumida por el contenedor (incluye cache) |
| `container_memory_working_set_bytes` | RAM real en uso (excluye cache — mejor indicador de presión de memoria) |
| `container_memory_limit_bytes` | Límite de RAM configurado para el contenedor |
| `rate(container_network_receive_bytes_total[1m])` | Tráfico de red entrante por contenedor |
| `rate(container_network_transmit_bytes_total[1m])` | Tráfico de red saliente por contenedor |
| `rate(container_fs_reads_bytes_total[1m])` | I/O de disco — lecturas por contenedor |
| `rate(container_fs_writes_bytes_total[1m])` | I/O de disco — escrituras por contenedor |

Todas admiten el label `name` para filtrar por contenedor: `{name="ollama"}`, `{name="prometheus"}`, etc.

#### Observaciones de validación

**llama 3.2 3B con AMD RX 6600** — `node_drm_memory_vram_used_bytes` mostró un pico de ~3,7 GB (≈ 4.000.000.000 bytes) al procesar la primera petición al modelo. El valor persistió tras la respuesta, confirmando que Ollama mantiene el modelo cargado en VRAM entre peticiones para evitar el coste de recarga.

### Etapa 2 — Métricas de runtime Ollama

Métricas del runtime de Ollama instrumentadas vía proxy sidecar.

#### Limitación: Ollama no expone un endpoint `/metrics` nativo

Ollama no tiene soporte nativo de Prometheus en ninguna versión actual. El endpoint `/metrics` en `:11434` devuelve 404. Existe un [issue abierto](https://github.com/ollama/ollama/issues/3144) solicitando esta funcionalidad, pero no está implementada.

Opciones evaluadas para obtener métricas de Ollama:

| Opción | Descripción | Decisión |
|---|---|---|
| **[ollama-metrics](https://github.com/NorskHelsenett/ollama-metrics)** | Proxy sidecar en Go. Intercepta el tráfico hacia Ollama e instrumenta cada request. También expone métricas de estado (modelos cargados, RAM) via polling de `/api/ps`. | **Elegida** |
| **[ollama-exporter](https://github.com/frcooper/ollama-exporter)** | Proxy sidecar en Python (FastAPI). Solo instrumenta `/api/chat` y `/api/generate`. | Descartada — cobertura parcial y mayor peso |

#### Implementación

`ollama-metrics` se despliega en este compose y actúa como proxy entre los clientes y Ollama:

```
Open WebUI → ollama-metrics:8082 → ollama:11434
                    ↓
              /metrics (Prometheus)
```

Métricas disponibles:

| Métrica | Tipo | Descripción |
|---|---|---|
| `ollama_loaded_models` | Gauge | Número de modelos cargados en VRAM |
| `ollama_model_loaded{model}` | Gauge | Estado de carga por modelo |
| `ollama_model_ram_mb{model}` | Gauge | RAM consumida por modelo (MB) |
| `ollama_prompt_tokens_total{model}` | Counter | Tokens de prompt procesados |
| `ollama_generated_tokens_total{model}` | Counter | Tokens generados |
| `ollama_request_duration_seconds{model}` | Histogram | Duración total del request |
| `ollama_time_per_token_seconds{model}` | Histogram | Tiempo por token generado |

**Requisito:** los clientes deben apuntar a `ollama-metrics:8082` en vez de a `ollama:11434` directamente. Las métricas de tokens y latencia solo se capturan para el tráfico que pasa por el proxy — peticiones que lleguen directamente a Ollama no quedan instrumentadas.

**Nota:** `ollama_loaded_models`, `ollama_model_loaded` y `ollama_model_ram_mb` se obtienen por polling de `/api/ps` y están disponibles siempre, independientemente de si el tráfico pasa por el proxy.

### Etapa 3 — Dashboards Grafana

Visualización unificada en Grafana de las métricas de hardware (Etapa 1) y runtime de Ollama (Etapa 2). Todos los dashboards se provisionan como código en `grafana/provisioning/dashboards/` — sin configuración manual.

#### Dashboards disponibles

| Fichero | Título | Origen |
|---|---|---|
| `hardware.json` | Node Exporter Full | [rfmoz/grafana-dashboards](https://github.com/rfmoz/grafana-dashboards) (id: 1860) + sección **AMD GPU** añadida |
| `ollama.json` | Ollama Metrics Dashboard | [NorskHelsenett/ollama-metrics](https://github.com/NorskHelsenett/ollama-metrics) — dashboard oficial del proxy |
| `cadvisor.json` | cadvisor dashboard | [grafana.com/dashboards/19792](https://grafana.com/grafana/dashboards/19792-cadvisor-dashboard/) — dashboard de comunidad |

#### Navegación: Node Exporter Full (`hardware.json`)

El dashboard es exhaustivo y puede resultar denso. Las secciones más relevantes para el uso habitual:

| Sección | Qué muestra |
|---|---|
| **Quick CPU / Mem / Disk** | Fila de stats instantáneos — punto de entrada rápido al estado del host |
| **Basic CPU / Mem / Net / Disk** | Series temporales de CPU, RAM, red y disco del host con histórico |
| **AMD GPU (RX 6600)** | Sección añadida: utilización GPU, VRAM usada vs máximo (8 GB), GTT usado vs máximo, temperatura |
| **Memory Meminfo** | Desglose completo de RAM — útil para diagnóstico de presión de memoria |
| **Storage Disk** | I/O por dispositivo — latencias de lectura/escritura, throughput |

El resto de secciones (Vmstat, Timesync, Systemd, Netstat, etc.) son para diagnóstico puntual y pueden ignorarse en el uso habitual.

#### Navegación: cAdvisor Dashboard (`cadvisor.json`)

Permite filtrar por `compose_project` y `container_name` desde las variables del dashboard. Secciones más relevantes:

| Sección | Qué muestra |
|---|---|
| **misc** | Stats instantáneos del contenedor seleccionado: CPU %, RAM, uptime |
| **cpu** | Uso de CPU por contenedor a lo largo del tiempo |
| **memory** | RAM usada, working set (RAM real sin cache), límite configurado |
| **network** | Tráfico de red entrante y saliente por contenedor |
| **blkio** | I/O de disco por contenedor |

#### Navegación: Ollama Metrics Dashboard (`ollama.json`)

Dashboard directo, sin necesidad de guía — todos sus paneles son relevantes para el uso habitual (tokens, latencia, tiempo por token, modelos cargados).

### Etapa 4 — Trazabilidad semántica — infraestructura OTel

Despliegue de la infraestructura de trazabilidad semántica en `docker-compose.tracing.yml`, independiente del pipeline de métricas.

- **ClickHouse** — base de datos columnar que almacena las trazas OTel. Solo accesible dentro de la red `tracing`.
- **OpenLIT** — UI de exploración de trazas + OTel Collector embebido. Recibe spans en `:4317` (gRPC) y `:4318` (HTTP), almacena en ClickHouse y los expone en su UI en `:3002`.
- Tablas OTel inicializadas en ClickHouse: `otel_traces`, `otel_logs`, `otel_metrics_*`.

#### Lección aprendida

La imagen `ghcr.io/openlit/openlit:latest` no incluye `curl` ni `wget`. El healthcheck debe hacerse con `node -e "fetch(...)"` (Node 18+ con fetch nativo).

### Etapa 5 — Trazabilidad semántica — proxy LiteLLM

Proxy LiteLLM en `docker-compose.tracing.yml` que intercepta las llamadas LLM, genera spans OTel y los envía al collector embebido de OpenLIT.

#### Implementación

LiteLLM expone una API OpenAI-compatible en `:8585`. Los clientes (Open WebUI) se configuran para enviar por esta vía en lugar de llamar directamente a Ollama:

```
Open WebUI ──OpenAI API──► LiteLLM:4000 ──► ollama-metrics:8082 ──► Ollama
                                │
                          OTel spans (HTTP :4318)
                                │
                           OpenLIT
```

La pipeline de métricas Prometheus no se interrumpe: LiteLLM reenvía a `ollama-metrics`, que sigue exponiendo sus métricas al scraper de Prometheus.

#### Configuración del cliente Open WebUI

```yaml
# en el servicio open-webui del compose local-ai-lab
environment:
  - OPENAI_API_BASE_URL=http://litellm:4000/v1
  - OPENAI_API_KEY=dummy
```

#### Routing de modelos (`assets/litellm-config.yaml`)

| Prefijo del modelo | Destino | Requisito |
|---|---|---|
| `ollama/<model>` | `ollama-metrics:8082` → Ollama | Modelo declarado explícitamente (ver nota) |
| `anthropic/<model>` | `api.anthropic.com` | `ANTHROPIC_API_KEY` en `.env` |
| `openai/<model>` | `api.openai.com` | `OPENAI_API_KEY` en `.env` |

**Nota — modelos Ollama:** a diferencia de Anthropic y OpenAI (wildcard `*`), los modelos de Ollama deben declararse individualmente en `litellm-config.yaml`. LiteLLM no descubre los modelos disponibles en Ollama automáticamente — solo expone a Open WebUI los modelos que están definidos en el config. Añadir un nuevo modelo Ollama requiere agregar una entrada y reiniciar LiteLLM (`docker compose -f docker-compose.tracing.yml restart litellm`).

#### Lecciones aprendidas

- El exporter gRPC (`OTEL_EXPORTER=otlp_grpc`) falla con error SSL porque el collector de OpenLIT no tiene TLS. Usar HTTP: `OTEL_EXPORTER=otlp_http`, `OTEL_ENDPOINT=http://openlit:4318`.
- La imagen de LiteLLM no incluye `curl`. El healthcheck debe usar `python3 -c "import urllib.request; urllib.request.urlopen(...)"`.
- `ollama-metrics` solo usa `expose` (no `ports`) — no es accesible desde el host. LiteLLM debe estar en la red `monitoring` para alcanzarlo por nombre de contenedor.
- Open WebUI habla con Ollama en API nativa (no OpenAI-compatible). Para que LiteLLM intercepte, hay que añadirlo como conexión OpenAI separada en Open WebUI — no reemplaza la conexión Ollama existente.

## Levantar el stack

Los dos compose files son independientes. `docker-compose.yml` debe arrancarse primero porque crea la red `monitoring`.

```bash
# Pipeline de métricas (hardware + runtime Ollama) — crea la red monitoring
docker compose up -d

# Pipeline de trazabilidad (LiteLLM + OpenLIT + ClickHouse)
docker compose -f docker-compose.tracing.yml up -d
```

## Almacenamiento de métricas a largo plazo

Por defecto Prometheus almacena métricas en local con retención de 15 días — suficiente para el uso habitual del stack. Si en el futuro se necesita retención larga, alta disponibilidad o backups gestionados, las opciones evaluadas son:

| Opción | Descripción | Cuándo considerar |
|---|---|---|
| **VictoriaMetrics** | Recibe `remote_write` de Prometheus, muy ligera, drop-in replacement. Es en sí misma un sistema de métricas completo. | Retención larga + HA multi-nodo con mínima complejidad |
| **ClickHouse** | BD columnar OLAP con integración oficial `remote_write` y datasource nativo en Grafana. Más pesada operativamente. | Análisis histórico avanzado (comparativas semana vs mes, etc.) |

Postgres vanilla no es una opción viable — el modelo relacional no encaja con series temporales de alta cardinalidad.

## Servicios

| Servicio | Compose | Puerto host | Descripción |
|---|---|---|---|
| Grafana | `docker-compose.yml` | 3001 | Dashboards de hardware y runtime |
| Prometheus | `docker-compose.yml` | 9090 | Almacenamiento de métricas |
| ollama-metrics | `docker-compose.yml` | — | Proxy sidecar Ollama (red `monitoring`) |
| LiteLLM | `docker-compose.tracing.yml` | 8585 | Proxy LLM + generación de spans OTel |
| OpenLIT | `docker-compose.tracing.yml` | 3002 | UI de trazas semánticas |
| OTel Collector (OpenLIT) | `docker-compose.tracing.yml` | 4317 / 4318 | Receptor de spans (gRPC / HTTP) |
| ClickHouse | `docker-compose.tracing.yml` | — | Storage de trazas (red `tracing`) |

## Conectar composes externos

`docker-compose.yml` crea y es dueño de la red Docker `monitoring`. Los composes externos se unen a ella:

```yaml
# Compose externo (ej. Ollama, local-ai-lab)
networks:
  monitoring:
    external: true
```
