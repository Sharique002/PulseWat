# 🫀 PulseWat

> **Monitor the pulse of your infrastructure.**

PulseWat is a real-time infrastructure monitoring and observability platform. It aggregates host-level and container-level metrics, visualizes them via pre-configured Grafana dashboards, and manages alert rules — all packaged into a simple, production-ready Docker Compose stack.

---

## 🚀 Quick Start

### 1. Prerequisites
Ensure you have [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/) installed on your machine.

### 2. Run the Stack
Clone the repository and spin up the services:
```bash
git clone https://github.com/Sharique002/PulseWat.git
cd PulseWat
docker compose up -d
```

### 3. Service Endpoints
Once the stack is running, you can access the following endpoints:

| Service | Access URL | Port (Host:Container) | Credentials / Details |
| :--- | :--- | :--- | :--- |
| **Grafana** | [http://localhost:3000](http://localhost:3000) | `3000:3000` | User: `admin` / Password: `pulsewat` |
| **Prometheus** | [http://localhost:9090](http://localhost:9090) | `9090:9090` | Alerting & metric querying engine |
| **cAdvisor** | [http://localhost:8081](http://localhost:8081) | `8081:8080` | Container resource analytics |
| **Sample App** | [http://localhost:8888](http://localhost:8888) | `8888:80` | Simple Nginx website (monitored target) |
| **Node Exporter** | *Internal Only* | `9100:9100` | Host metrics collector |

---

## 🏗️ Architecture

```
┌───────────────────────────────── Docker Bridge Network ────────────────────────────────┐
│                                                                                        │
│  ┌─────────────────┐       ┌─────────────────┐                                         │
│  │  Node Exporter  │       │    cAdvisor     │                                         │
│  │  (Host Metrics) │       │ (Container Mt.) │                                         │
│  └────────┬────────┘       └────────┬────────┘                                         │
│           │                         │                                                  │
│           └───────────┬─────────────┘                                                  │
│                       │ Scrapes every 5s                                               │
│                       ▼                                                                │
│            ┌────────────────────┐          Auto-provisions                             │
│            │     Prometheus     │◄──────────────────────────────────┐                  │
│            │  (Alerting Engine) │                                   │                  │
│            └──────────┬─────────┘                                   │                  │
│                       │                                             │                  │
│                       │ Queries metrics                             │                  │
│                       ▼                                             │                  │
│            ┌────────────────────┐                                   │                  │
│            │      Grafana       ├───────────────────────────────────┘                  │
│            │ (Visual Dashboard) │                                                      │
│            └──────────┬─────────┘                                                      │
│                       │                                                                │
│                       ▼ (Monitors)                                                     │
│            ┌────────────────────┐                                                      │
│            │  Nginx Sample App  │                                                      │
│            └────────────────────┘                                                      │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Pre-configured Dashboards

PulseWat auto-provisions three production-grade dashboards in Grafana on startup:

1. **Infrastructure Overview**: High-level and detailed host performance metrics (CPU usage breakdown, Memory available/used, Disk space and I/O, Network traffic, System uptime).
2. **Docker Container Monitoring**: Resource utilization for all running containers mapped via cAdvisor (Container CPU %, Memory usage limits, Network Rx/Tx, restart counters).
3. **Alerts & Health**: Scrape health of Prometheus targets, job duration, active/firing alert instances, and internal TSDB health.

---

## 🚨 Pre-configured Alert Rules

Prometheus evaluates the following alert rules every 5 seconds (defined in `prometheus/alert.rules.yml`):

| Alert Name | Condition | Severity |
| :--- | :--- | :--- |
| **HostDown** | Node Exporter target is offline for > 1m | `critical` |
| **ContainerDown** | cAdvisor target is offline for > 1m | `critical` |
| **HighCpuUsage** | System CPU usage > 80% for > 2m | `warning` |
| **CriticalCpuUsage** | System CPU usage > 95% for > 1m | `critical` |
| **HighMemoryUsage** | System RAM usage > 85% for > 2m | `warning` |
| **CriticalMemoryUsage** | System RAM usage > 95% for > 1m | `critical` |
| **HighDiskUsage** | Disk space usage > 85% for > 5m | `warning` |
| **DiskAlmostFull** | Disk space usage > 95% for > 1m | `critical` |
| **ContainerHighCpu** | A container uses > 80% CPU for > 2m | `warning` |
| **ContainerRestarting** | A container has restarted > 3 times in 15m | `warning` |

---

## 📂 Project Structure

```
PulseWat/
├── docker-compose.yml              # Service orchestration & mappings
├── prometheus/
│   ├── prometheus.yml              # Scrape configurations & targets
│   └── alert.rules.yml             # Prometheus alerting threshold rules
├── grafana/
│   ├── provisioning/
│   │   ├── dashboards/
│   │   │   └── dashboards.yml      # Auto-loads JSON dashboards
│   │   └── datasources/
│   │       └── datasource.yml      # Auto-registers Prometheus datasource
│   └── dashboards/
│       ├── infrastructure-overview.json
│       ├── docker-monitoring.json
│       └── alerts-health.json
├── sample-app/
│   ├── Dockerfile                  # Monitored Nginx custom container
│   ├── nginx.conf                  # Nginx stub_status configuration
│   └── index.html                  # Simple dark-themed landing page
└── README.md                       # Documentation
```

---

## 🛠️ Management Commands

*   **Start Stack**: `docker compose up -d`
*   **Stop Stack**: `docker compose down`
*   **Wipe Data & Restart (Fresh configuration)**: `docker compose down -v && docker compose up -d`
*   **View Logs**: `docker compose logs -f`
