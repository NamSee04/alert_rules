# alert_rules

A collection of custom [Prometheus](https://prometheus.io/) alerting rules for monitoring infrastructure, applications, and services.

---

## Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Alert Rule Format](#alert-rule-format)
- [Using the Rules](#using-the-rules)
- [Testing Alert Rules](#testing-alert-rules)
- [Contributing](#contributing)

---

## Overview

This repository stores custom Prometheus alerting rules (`.yml` / `.yaml` files) that can be loaded by a Prometheus server to trigger alerts based on time-series metrics. Rules are grouped by category (e.g., host, application, database) to keep them organised and easy to maintain.

---

## Repository Structure

```
alert_rules/
├── README.md
├── host/                  # System / node-level alerts (CPU, memory, disk, …)
│   └── node_alerts.yml
├── application/           # Application-level alerts (error rates, latency, …)
│   └── app_alerts.yml
├── database/              # Database alerts (connection pool, replication, …)
│   └── db_alerts.yml
└── kubernetes/            # Kubernetes cluster alerts (pod restarts, OOMKill, …)
    └── k8s_alerts.yml
```

> **Note:** The directory layout above is a suggested convention. Add subdirectories as needed to match your environment.

---

## Getting Started

### Prerequisites

| Tool | Purpose |
|------|---------|
| [Prometheus](https://prometheus.io/download/) ≥ 2.x | Evaluates alert rules |
| [promtool](https://prometheus.io/docs/prometheus/latest/command-line/promtool/) | Validates and unit-tests rule files (ships with Prometheus) |
| [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) | Routes and delivers alerts |

### Add rules to Prometheus

1. Copy the desired rule files into your Prometheus server (or a directory it can reach).
2. Reference them in `prometheus.yml`:

```yaml
rule_files:
  - "alert_rules/host/*.yml"
  - "alert_rules/application/*.yml"
  - "alert_rules/database/*.yml"
  - "alert_rules/kubernetes/*.yml"
```

3. Reload Prometheus:

```bash
# Send SIGHUP to the running Prometheus process
kill -HUP <prometheus_pid>

# Or use the HTTP API (requires --web.enable-lifecycle flag)
curl -X POST http://localhost:9090/-/reload
```

---

## Alert Rule Format

Each rule file follows the standard Prometheus alerting rule syntax:

```yaml
groups:
  - name: <group_name>            # Logical grouping of related rules
    interval: 1m                  # (optional) Evaluation interval override
    rules:
      - alert: <AlertName>        # PascalCase alert name
        expr: <PromQL expression> # Condition that must be true to fire
        for: <duration>           # How long condition must hold before firing
        labels:
          severity: <critical|warning|info>
          team: <owning_team>
        annotations:
          summary: "Short one-line description"
          description: "Detailed description. Value: {{ $value }}"
          runbook_url: "https://wiki.example.com/runbooks/<AlertName>"
```

### Example — High CPU Usage

```yaml
groups:
  - name: node
    rules:
      - alert: HighCPUUsage
        expr: |
          (
            1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
          ) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ printf \"%.2f\" $value }}% (threshold: 85%) for more than 10 minutes."
```

---

## Using the Rules

### Validate rule files

Use `promtool` to check for syntax errors before deploying:

```bash
promtool check rules path/to/rules/*.yml
```

### Unit-test rule files

Create a test file (e.g., `tests/node_alerts_test.yml`):

```yaml
rule_files:
  - ../host/node_alerts.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'node_cpu_seconds_total{mode="idle", instance="host1"}'
        values: "0 0 0 0 0 0 0 0 0 0"   # 0 idle seconds → 100% CPU
    alert_rule_test:
      - eval_time: 10m
        alertname: HighCPUUsage
        exp_alerts:
          - exp_labels:
              severity: warning
              instance: host1
            exp_annotations:
              summary: "High CPU usage on host1"
```

Run tests:

```bash
promtool test rules tests/node_alerts_test.yml
```

---

## Contributing

1. **Fork** this repository and create a feature branch.
2. Add or modify rule files following the [Alert Rule Format](#alert-rule-format) above.
3. Validate your changes locally with `promtool check rules`.
4. Write unit tests in the `tests/` directory and run `promtool test rules`.
5. Open a **Pull Request** with a clear description of the alert, the metric it targets, and the threshold rationale.

### Naming conventions

| Item | Convention | Example |
|------|-----------|---------|
| Alert name | PascalCase | `HighMemoryUsage` |
| Label values | lowercase snake_case | `severity: critical` |
| Rule file names | snake_case | `node_alerts.yml` |
| Group names | lowercase | `node`, `application` |
