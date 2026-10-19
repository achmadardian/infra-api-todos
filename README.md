# Observability Infrastructure (`infra-api-todos`)

Production-ready, decoupled monitoring and log aggregation pipeline built with **Grafana Loki 3.x**, **Grafana Promtail 3.x**, and **Grafana 11.x**.

This stack collects, indexes, and visualizes structured JSON logs emitted by [api-go-todos](https://github.com/achmadardian/api-go-todos) with **zero HTTP overhead** on the application runtime.

---

## 1. Architecture & Data Flow

```mermaid
flowchart LR
    subgraph AppContainer ["App Container (api-go-todos)"]
        App["Go API (log/slog)"] -->|stdout (JSON)| DockerEngine["Docker json-file driver"]
    end

    subgraph Host ["Host Filesystem"]
        DockerEngine -->|writes to| LogFiles["/var/lib/docker/containers/*/*-json.log"]
    end

    subgraph MonitoringStack ["Monitoring Stack (infra-api-todos)"]
        Promtail["Promtail 3.x\n(docker_sd_configs)"]
        Loki["Grafana Loki 3.x\n(TSDB Index + Chunks)"]
        Grafana["Grafana 11.x\n(:3000 Web UI)"]

        Promtail -->|reads asynchronously via ro volume| LogFiles
        Promtail -->|discovers metadata via| DockerSock["/var/run/docker.sock"]
        Promtail -->|pushes streams (:3100)| Loki
        Grafana -->|queries LogQL| Loki
    end
```

### Key Architectural Decisions

1. **Decoupled Lifecycle**: The monitoring infrastructure lives in its own repository and lifecycle. Application redeployments or restarts never disrupt the log collection pipeline.
2. **Zero-Overhead Log Shipping**: The Go API does **not** push logs over HTTP to Loki. It outputs standard structured JSON to `os.Stdout` (12-Factor App methodology). Promtail reads Docker container logs from the host asynchronously. If Loki or Promtail is restarting, the application continues to run without latency or memory degradation.
3. **Cardinality Safety**:
   - **Indexed Loki Stream Labels** (low cardinality): `service`, `env`, `level`.
   - **Payload Fields** (high cardinality): `request_id`, `status`, `duration_ms`, `error`, `operation` remain inside the JSON payload and are parsed dynamically via LogQL (`| json`). This prevents TSDB index explosion.
4. **Automated Data Retention**: Configured with a strict **14-day auto-purge** policy executed by the Loki compactor.
5. **Security Isolation**: Loki port `3100` is bound to `127.0.0.1` and communicated over the internal Docker network `monitoring-net`; it is **not** exposed to the public internet.

---

## 2. Directory Structure

```text
.
├── docker-compose.yml                         # Service definitions (Loki, Promtail, Grafana)
├── loki-config.yml                            # Loki TSDB & 14-day retention configuration
├── promtail-config.yml                        # Docker SD & 2-stage JSON extraction pipeline
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasources.yml                # Zero-click Loki datasource provisioning
├── .env.example                               # Default environment variables
├── .gitignore
└── README.md
```

---

## 3. Quickstart

### Step 1: Prepare Environment
Copy `.env.example` to `.env`:
```bash
cp .env.example .env
```

### Step 2: Start Monitoring Stack
Launch all services in detached mode:
```bash
docker compose up -d
```

Verify service status:
```bash
docker compose ps
```

Check Loki readiness:
```bash
curl -s http://127.0.0.1:3100/ready
# Output should be: ready
```

---

## 4. Connecting `api-go-todos` to the Pipeline

To allow Promtail and the monitoring stack to discover and network with `api-go-todos`:

### A. Ensure Shared Network in `api-go-todos/docker-compose.yml`
Add the `monitoring-net` network to your application's `docker-compose.yml`:

```yaml
services:
  api:
    container_name: api-todos
    # ... your existing configuration ...
    networks:
      - default
      - monitoring-net

networks:
  monitoring-net:
    external: true
```

### B. Ensure JSON Logging Driver
Docker's default logging driver is `json-file`. Ensure your container has container name or labels:
```yaml
services:
  api:
    container_name: api-todos
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"
```

Once started, Promtail will automatically discover the container via `/var/run/docker.sock` and tail its logs.

---

## 5. Grafana & LogQL Query Guide

1. Open Grafana in your browser: `http://localhost:3000`
2. Login with credentials from `.env` (default: `admin` / `admin`).
3. Navigate to **Explore** (compass icon on the sidebar).
4. Loki is already pre-configured as the default datasource.

### Essential LogQL Queries

#### 1. View All Logs for `api-todos`
```logql
{service="api-todos"}
```

#### 2. Filter Only Application Errors (5xx / Internal Server Errors)
```logql
{service="api-todos"} | json | level="ERROR"
```

#### 3. Trace a Specific Request by Correlation ID (`request_id`)
```logql
{service="api-todos"} | json | request_id="a1b2c3d4-e5f6-7890-abcd-ef1234567890"
```

#### 4. Find Slow HTTP Requests (> 200ms)
```logql
{service="api-todos"} | json | duration_ms > 200
```

#### 5. Find Failed Database Operations
```logql
{service="api-todos"} | json | operation =~ "TaskRepository.*"
```

#### 6. Calculate Log Rate / Second by Level
```logql
sum(rate({service="api-todos"} | json [1m])) by (level)
```

---

## 6. Resource Footprint & Limits

| Component | Image | Typical RAM Usage | Notes |
| :--- | :--- | :--- | :--- |
| **Loki** | `grafana/loki:3.2.1` | ~80 - 120 MB | TSDB index, 14d retention compactor |
| **Promtail** | `grafana/promtail:3.2.1` | ~25 - 40 MB | Async host tailer |
| **Prometheus** | `prom/prometheus:v2.54.1` | ~40 - 70 MB | Scrapes `/metrics` every 15s, 15d retention |
| **Grafana** | `grafana/grafana:11.3.0` | ~50 - 70 MB | Web visualization UI with auto-provisioned dashboards |
| **Total** | | **< 300 MB** | Extremely lightweight, ideal for single VPS |

---

## 7. RED Method Dashboard & Alert Rules

### Auto-Provisioned Grafana Dashboard
Access Grafana at `http://localhost:3000` -> **Dashboards** -> folder **Observability** -> **"API Todos - Overview (RED Method)"**:
- **Rate (R)**: Request Rate (RPS by Route) via `sum(rate(http_requests_total[1m])) by (path)`
- **Errors (E)**: Error Rate (% 5xx) via `sum(rate(http_requests_total{status=~"5.."}[1m]))`
- **Duration (D)**: Latency distribution (p50, p95, p99) via `histogram_quantile(..., http_request_duration_seconds_bucket)`
- **Saturation**: In-Flight Requests (`http_requests_in_flight`) & Go Goroutines (`go_goroutines`)
- **Forensic Logs**: Embedded live stream of `{service="api-todos"}` logs.

### Prometheus Alert Rules (`alerts.yml`)
1. **`ApiHighErrorRate`**: Triggers if 5xx error rate > 5% for 3 minutes.
2. **`ApiHighLatency`**: Triggers if p95 latency > 1s for 3 minutes.
3. **`ApiInstanceDown`**: Triggers if target is unreachable (`up{job="api-todos"} == 0`) for 1 minute.

---

## 8. Upcoming Observability Roadmap

- [x] **Stages 1 - 5**: Structured logging (`log/slog`), Chi correlation middleware, Loki & Promtail stack.
- [x] **Stage 6**: Metrics fundamentals — RED method (Rate, Errors, Duration), Go runtime metrics, Prometheus scraping.
- [x] **Stage 7**: Production monitoring & alerting — Prometheus alert rules, Grafana auto-provisioned RED dashboards.
- [ ] **Stage 8**: Distributed tracing & profiling — OpenTelemetry integration and Go `pprof`.