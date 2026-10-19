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
| **Grafana** | `grafana/grafana:11.3.0` | ~50 - 70 MB | Web visualization UI |
| **Total** | | **< 230 MB** | Lightweight, ideal for single VPS |

---

## 7. Upcoming Observability Roadmap

- [x] **Stages 1 - 5**: Structured logging (`log/slog`), Chi correlation middleware, Loki & Promtail stack.
- [ ] **Stage 6**: Metrics fundamentals — introducing Prometheus, RED method (Rate, Errors, Duration), and instrumenting HTTP latency.
- [ ] **Stage 7**: Production monitoring & alerting — SLI/SLO/SLA definitions, Prometheus alert rules, and Grafana alert dashboards.
- [ ] **Stage 8**: Distributed tracing & profiling — OpenTelemetry integration and Go `pprof`.