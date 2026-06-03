# 🫀 PulseWat — System Documentation & Reference Guide

Welcome to the comprehensive technical documentation for **PulseWat**, a production-style, real-time infrastructure monitoring and observability platform. This document covers the architecture, component configurations, dashboard structures, alerting thresholds, data persistence mechanics, and operational runbooks.

---

## 🏗️ 1. System Architecture & Flow

PulseWat utilizes a containerized, decoupled architecture to collect, store, and visualize metrics. All services run inside a dedicated Docker bridge network (`pulsewat-network`) to facilitate secure communication and internal DNS resolution.

```
                  ┌──────────────────────────────────────────────┐
                  │              Windows Host System             │
                  └──────────────────────┬───────────────────────┘
                                         │
                                         ▼ (Port Mappings)
┌───────────────────────────────── Docker Bridge Network (pulsewat-network) ────────────────────────────────┐
│                                                                                                          │
│   ┌─────────────────────────┐               ┌─────────────────────────┐               ┌───────────────┐  │
│   │      Node Exporter      │               │        cAdvisor         │               │  Sample App   │  │
│   │ (Host OS Metric Source) │               │   (Container Metrics)   │               │    (Nginx)    │  │
│   │       Port: 9100        │               │  Host Port: 8081 (8080) │               │  Port: 8888   │  │
│   └────────────┬────────────┘               └────────────┬────────────┘               └───────┬───────┘  │
│                │                                         │                                    │          │
│                └────────────────────┬────────────────────┘                                    │          │
│                                     │ (Scraped every 5s)                                      │          │
│                                     ▼                                                         │          │
│                       ┌──────────────────────────┐                                            │          │
│                       │   Prometheus TSDB        │◀───────────────────────────────────────────┘          │
│                       │   Host Port: 9090        │                                                       │
│                       └─────────────┬────────────┘                                                       │
│                                     ▲                                                                    │
│                                     │ (Queries metrics)                                                  │
│                                     │                                                                    │
│                       ┌─────────────┴────────────┐                                                       │
│                       │         Grafana          │                                                       │
│                       │  Host Port: 3001 (3000)  │                                                       │
│                       └──────────────────────────┘                                                       │
│                                                                                                          │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Metrics Lifecycle Flow:
1. **Collectors (Exporters)** gather host and container metrics from system filesystems (`/proc`, `/sys`) and the Docker daemon socket (`/var/run/docker.sock`).
2. **Prometheus** scrapes the collector endpoints every **5 seconds** and stores the data in its time-series database (TSDB). It evaluates alert rules concurrently.
3. **Grafana** queries Prometheus using PromQL (Prometheus Query Language) to dynamically render host status, containers, and alerts.

---

## ⚙️ 2. Core Service Component Matrix

PulseWat runs 5 services orchestrated by Docker Compose:

| Service Name | Docker Container | Port (Host:Container) | Storage / Mounts | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Prometheus** | `pulsewat-prometheus` | `9090:9090` | `prometheus_data` (Named Volume) | Time-series database, alert rule engine, and metrics scraper. |
| **Grafana** | `pulsewat-grafana` | `3001:3000` | `grafana_data` (Named Volume) | Visual analytics server containing pre-provisioned dashboards. |
| **Node Exporter** | `pulsewat-node-exporter` | `9100:9100` | Read-only `/proc`, `/sys`, `/` (rootfs) | Host daemon collecting CPU, RAM, disk, network, and uptime stats. |
| **cAdvisor** | `pulsewat-cadvisor` | `8081:8080` | `/var/run`, `/sys`, `/var/lib/docker/` | Resource usage analyzer for running Docker containers. |
| **Sample App** | `pulsewat-sample-app` | `8888:80` | None (Stateless) | Custom Nginx server representing the monitored microservice. |

---

## 📊 3. Pre-configured Dashboards

Grafana is provisioned automatically with three distinct monitoring environments. The JSON files are loaded from `grafana/dashboards/`:

### A. Infrastructure Overview (`infrastructure-overview.json`)
Provides a single-pane-of-glass overview of host-level virtual machine resources:
*   **KPI Cards**: CPU utilization percentage, RAM utilization percentage, Storage utilization percentage, Host Uptime, Total RAM, and Available RAM.
*   **CPU Performance**: Dynamic line charts tracing Total, User, System, and Idle CPU cycles.
*   **Memory Breakdown**: Deep dive into active memory utilization, cached blocks, and free buffers.
*   **Disk Activity**: Tracking read/write throughput rates (B/s) and I/O load.
*   **Network Throughput**: Tracing bytes transmitted (Tx) and received (Rx) across all active interfaces.
*   **Filesystem Metrics**: Interactive mount tables tracking disk availability per mounted directory.

### B. Docker Container Monitoring (`docker-monitoring.json`)
Maintains deep diagnostic visibility over containerized environments using metrics from cAdvisor:
*   **KPI Cards**: Number of running containers, total CPU utilization across containers, cumulative container memory, and network throughput.
*   **Container Comparison Tables**: Real-time tabular rankings displaying CPU/Memory load per container.
*   **Resource Tracking**: Isolated charts tracking CPU cycles, RAM limits, and Network IO per individual container name.

### C. Alerts & Health (`alerts-health.json`)
Validates that the monitoring infrastructure is functional:
*   **Target Status Check**: Heatmaps and state checks for Node Exporter, cAdvisor, and Prometheus scraper endpoints.
*   **Scrape Performance**: Charts tracing scrape duration and sample counts.
*   **Alert Counters**: Stat blocks indicating total firing and pending alerts.
*   **Prometheus DB Health**: Track TSDB active series, block sizes, and Prometheus container RAM consumption.

---

## 🚨 4. Alerting Threshold Matrix

Prometheus processes the rules inside `prometheus/alert.rules.yml` on every scrape interval (5s). These are grouped into 3 logic blocks:

### 🏠 Host Alerts
*   **HostDown** (`critical`): Triggered if the Node Exporter target is offline/unreachable for `> 1 minute`.
*   **HighCpuUsage** (`warning`): Active if system CPU usage exceeds `80%` for `> 2 minutes`.
*   **CriticalCpuUsage** (`critical`): Active if system CPU usage exceeds `95%` for `> 1 minute`.
*   **HighMemoryUsage** (`warning`): Active if system RAM usage exceeds `85%` for `> 2 minutes`.
*   **CriticalMemoryUsage** (`critical`): Active if system RAM usage exceeds `95%` for `> 1 minute`.
*   **HighDiskUsage** (`warning`): Active if disk space utilization on any mount exceeds `85%` for `> 5 minutes`.
*   **DiskAlmostFull** (`critical`): Active if disk space utilization on any mount exceeds `95%` for `> 1 minute`.

### 📦 Container Alerts
*   **ContainerDown** (`critical`): Active if cAdvisor ceases scraping metrics for `> 1 minute`.
*   **ContainerHighCpu** (`warning`): Triggered if an individual container consumes `> 80%` allocated CPU core resources for `> 2 minutes`.
*   **ContainerRestarting** (`warning`): Triggered if a container restarts `> 3 times` in a `15-minute` window.

### 🎛️ Engine Alerts
*   **PrometheusTargetDown** (`critical`): Global fallback alert if any target configured in `prometheus.yml` goes offline for `> 2 minutes`.

---

## 🛠️ 5. Operational Command Runbook

Run these commands in PowerShell or Command Prompt from the project directory:

### Start the Platform
Launch all 5 containers in daemon (detached) mode:
```powershell
docker compose up -d
```

### Stop the Platform
Shut down all containers and clean up internal networks:
```powershell
docker compose down
```

### Hard Restart (Flush Volumes)
Wipe out stored timeseries data and Grafana runtime cache to perform a fresh start:
```powershell
docker compose down -v
docker compose up -d
```

### Rebuild and Run Sample App
Recompile and restart the custom Nginx server:
```powershell
docker compose up -d --build sample-app
```

### View Service Logs
Stream real-time log aggregates from all containers:
```powershell
docker compose logs -f
```

---

## 🩺 6. Troubleshooting FAQs

#### Q: How do I resolve a Grafana port conflict?
*   **Symptom**: `Bind for 0.0.0.0:3000 failed: port is already allocated` or `exit code 1` on Grafana container startup.
*   **Fix**: PulseWat is pre-configured to bind Grafana to port `3001` on the host side (mapping `3001:3000`). If `3001` is also taken, open `docker-compose.yml` and modify the ports key under `grafana:` (e.g. change `"3001:3000"` to `"3002:3000"`).

#### Q: Why is my Nginx sample app marked "unhealthy" by Docker?
*   **Symptom**: `docker compose ps` shows `Up XX seconds (unhealthy)`.
*   **Fix**: This occurs when `localhost` resolves to an IPv6 address (`::1`) inside the container hosts file, while Nginx listens solely on IPv4 loopback (`127.0.0.1`). PulseWat resolves this by configuring the health check query to target `http://127.0.0.1/` inside the Dockerfile.

#### Q: Why are host CPU/RAM metrics missing on my Windows machine?
*   **Symptom**: Node Exporter graphs display `NaN` or zero metrics.
*   **Fix**: Node Exporter and cAdvisor extract metrics from native Linux kernel paths (`/proc`, `/sys`). On Windows hosts, they scrape the underlying Linux VM running Docker Desktop (via WSL2). If running native Windows containers (non-WSL), these tools cannot query Linux metrics. Ensure Docker Desktop is set to use **Linux Containers** mode.
