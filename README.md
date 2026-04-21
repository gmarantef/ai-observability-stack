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
| 1 | Métricas de hardware | `feat/hardware-metrics` | Completada |
| 2 | Métricas de runtime Ollama | `feat/ollama-runtime-metrics` | En progreso |
| 3 | Trazabilidad semántica — modelos locales | `feat/otel-local` | Pendiente |
| 4 | Trazabilidad semántica — modelos remotos | `feat/otel-remote-proxy` | Pendiente |

### Etapa 1 — Métricas de hardware

Colección de métricas del host (CPU, RAM, GPU AMD) y desglose por contenedor Docker.

- **`node_exporter`** (`prom/node-exporter`) con `--collector.drm`: GPU load %, VRAM usada/total, GTT (RAM de sistema en offload), temperatura, frecuencias CPU/GPU, RAM total del host.
- **`cAdvisor`** (`gcr.io/cadvisor/cadvisor`): CPU y RAM desglosados por contenedor — permite ver qué servicio (ej. `ollama`) está consumiendo qué.
- Ambos son host-level: cubren este compose y cualquier compose externo actual o futuro sin modificarlos.
- Granularidad GPU por modelo concreto (qué modelo dentro de Ollama usa cuánta VRAM): esto es información semántica, se captura en Etapa 3 vía OTel.

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

**Requisito:** los clientes deben apuntar a `ollama-metrics:8082` en vez de a `ollama:11434` directamente. Las métricas de tokens y latencia solo se capturan para el tráfico que pasa por el proxy.

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

## Almacenamiento de métricas a largo plazo

Por defecto Prometheus almacena métricas en local con retención de 15 días — suficiente para el uso habitual del stack. Si en el futuro se necesita retención larga, alta disponibilidad o backups gestionados, las opciones evaluadas son:

| Opción | Descripción | Cuándo considerar |
|---|---|---|
| **VictoriaMetrics** | Recibe `remote_write` de Prometheus, muy ligera, drop-in replacement. Es en sí misma un sistema de métricas completo. | Retención larga + HA multi-nodo con mínima complejidad |
| **ClickHouse** | BD columnar OLAP con integración oficial `remote_write` y datasource nativo en Grafana. Más pesada operativamente. | Análisis histórico avanzado (comparativas semana vs mes, etc.) |

Postgres vanilla no es una opción viable — el modelo relacional no encaja con series temporales de alta cardinalidad.

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
| Grafana | 3001 | Dashboards de hardware y runtime |
| OpenLIT | 3002 | Explorador de trazas semánticas |
| Prometheus | 9090 | Almacenamiento de métricas |
| Proxy (Anthropic) | 8585 | Intercepta llamadas a la API de Anthropic |
| Proxy (OpenAI) | 8586 | Intercepta llamadas a APIs compatibles con OpenAI |
