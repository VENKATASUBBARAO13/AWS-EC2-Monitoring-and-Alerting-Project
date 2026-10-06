# AWS EC2 Monitoring & Alerting with Prometheus, Grafana, Node Exporter, CloudWatch & PagerDuty

A hands-on cloud monitoring project demonstrating Linux server metrics collection, Prometheus alerting, Grafana dashboards, AWS CloudWatch integration, and PagerDuty incident handling.

> **Security note:** This repository intentionally contains no AWS credentials, PagerDuty integration keys, account IDs, or live public IP addresses. Replace placeholders locally and never commit secrets.

## Architecture

```text
                              AWS Cloud / VPC

     ┌──────────────────────┐          ┌──────────────────────────────┐
     │    Monitored EC2     │          │       Monitoring EC2        │
     │                      │          │                              │
     │ Linux Host           │          │ Prometheus :9090             │
     │      │               │          │      │                       │
     │ Node Exporter :9100  │─────────▶│      ├── Alert rules         │
     │                      │ /metrics  │      │        │               │
     └──────────────────────┘          │      ▼        ▼               │
                                       │ Alertmanager  Grafana :3000  │
                                       │      │        │      │        │
                                       └──────┼────────┼──────┼────────┘
                                              │        │      │
                                              ▼        │      ▼
                                          PagerDuty    │  CloudWatch
                                                       │  data source
                                                       ▼
                                                AWS / EC2 metrics
```

### Monitoring Flow

```text
Monitored EC2
     │
     └── Node Exporter :9100
             │
             ▼
       Prometheus :9090
             │
             ├── Alert rules → Alertmanager → PagerDuty
             │
             └── Metrics → Grafana :3000
                            └── CloudWatch data source
```

See [architecture/architecture.md](architecture/architecture.md) for the detailed flow, alert pipeline, recovery test, and recommended network controls.

## Features

- EC2 monitoring setup with IAM and security-group access
- Node Exporter for Linux host metrics
- Prometheus scrape configuration and alert rules
- Grafana dashboards using Prometheus and CloudWatch data sources
- PagerDuty incident workflow validation
- Failure simulation by stopping Node Exporter, then recovery verification

## Repository layout

```text
.
├── prometheus/
│   ├── prometheus.yml
│   └── alert_rules.yml
├── alertmanager/
│   └── alertmanager.example.yml
├── node-exporter/
│   └── node_exporter.service
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/datasources.yml
│   │   └── dashboards/dashboards.yml
│   └── dashboards/README.md
├── docs/
│   └── screenshots/       # Add sanitized screenshots here
├── .gitignore
└── README.md
```

## Prerequisites

- Two Linux EC2 instances (or equivalent): one monitoring host and one monitored host
- Prometheus, Grafana, Node Exporter, and optionally Alertmanager installed
- AWS IAM role/permissions for CloudWatch read access in Grafana
- PagerDuty integration configured in Alertmanager or your chosen alert receiver
- Security groups restricted to trusted sources

## Setup overview

### 1. Node Exporter

Install Node Exporter on each Linux host to be monitored. Review `node-exporter/node_exporter.service`, update the binary path/user if needed, then install it as a systemd service.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

Check locally on the monitored EC2:

```bash
curl http://localhost:9100/metrics
```

### 2. Prometheus

Copy `prometheus/prometheus.yml` to your Prometheus configuration directory and replace `MONITORED_PRIVATE_IP` with the monitored EC2 private IP.

Copy `prometheus/alert_rules.yml` beside it, then validate and restart Prometheus:

```bash
promtool check config /etc/prometheus/prometheus.yml
promtool check rules /etc/prometheus/alert_rules.yml
sudo systemctl restart prometheus
```

Open Prometheus Targets and confirm the Node Exporter target is **UP**.

### 3. Grafana

Configure the Prometheus and CloudWatch data sources. A sample provisioning file is included for Prometheus; CloudWatch IAM permissions and authentication must be configured for your environment.

Import a Node Exporter dashboard or create panels for CPU, memory, filesystem, network, and uptime. Add CloudWatch panels for the AWS/EC2 metrics you enabled.

### 4. Alerting and PagerDuty

Configure Alertmanager (or your existing alerting integration) with a PagerDuty integration URL/key stored securely as a secret—not in Git. Ensure Prometheus is configured to send alerts to Alertmanager if using this route.

Test the `InstanceDown` alert by stopping Node Exporter on the monitored host:

```bash
sudo systemctl stop node_exporter
```

Verify the target becomes **DOWN**, the alert transitions through pending/firing according to its `for` duration, and PagerDuty receives the incident. Restore service:

```bash
sudo systemctl start node_exporter
```

Confirm the target returns to **UP**, then resolve/close the test incident according to your PagerDuty workflow.

## Example PromQL

Node Exporter target health:

```promql
up{job="ec2-node-exporters"}
```

CPU usage percentage (non-idle):

```promql
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{job="ec2-node-exporters",mode="idle"}[5m])))
```

Memory usage percentage:

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

Filesystem usage percentage (excluding common pseudo filesystems):

```promql
100 * (1 - (
  node_filesystem_avail_bytes{fstype!~"tmpfs|devtmpfs|overlay"}
  /
  node_filesystem_size_bytes{fstype!~"tmpfs|devtmpfs|overlay"}
))
```

## Security considerations

- Never commit AWS access keys, secret keys, PagerDuty integration keys, `.env` files, or private SSH keys.
- Do not expose ports `3000`, `9090`, or `9100` publicly. Restrict them to your trusted IP or monitoring security group.
- Prefer IAM roles over long-lived AWS access keys.
- Use HTTPS/reverse proxy and authentication for any internet-facing monitoring UI.
- Sanitize screenshots: hide public IPs, account IDs, incident keys, email addresses, and other identifying details.

## Evidence / Screenshots

### 1. Node Exporter
![Node Exporter](docs/screenshots/node-exporter.png.jpeg)

### 2. PagerDuty Resolved
![PagerDuty Resolved](docs/screenshots/Pagerduty_resolved.jpeg)

### 3. PagerDuty Triggered
![PagerDuty Triggered](docs/screenshots/Pagerduty_triggered.jpeg)

### 4. Adding Prometheus Data Source
![Adding Prometheus Data Source](docs/screenshots/adding_prometheus_data_resource.jpeg)

### 5. Additional Grafana Dashboard
![Additional Dashboard](docs/screenshots/additional-dashboard.png.jpeg)

### 6. Checking Prometheus Targets
![Prometheus Targets](docs/screenshots/checking_prometheus_targets.jpeg)

### 7. Prometheus After Instance Restart
![Prometheus After Restart](docs/screenshots/after_instanace_stopped_and_started_checking_prometheus.jpeg)

### 8. CloudWatch Data Source
![CloudWatch Data Source](docs/screenshots/cloudwatch-datasource.png.jpeg)

### 9. Grafana Dashboard
![Grafana Dashboard](docs/screenshots/grafana-dashboard.png.jpeg)

### 10. Node Server Running
![Node Server Running](docs/screenshots/node_server_running.jpeg)

### 11. Node Server After Stop
![Node Server After Stop](docs/screenshots/Running_node_server_after_stopped.jpeg)

### 12. Instance Down on Prometheus
![Instance Down on Prometheus](docs/screenshots/showing_instance_down_on_prometheus.jpeg)

### 13. Updating Data on Prometheus
![Updating Data on Prometheus](docs/screenshots/updating_data_on_prometheus.jpeg)

### 14. CloudWatch CPU Credit Usage Check
![CloudWatch CPU Credit Usage](docs/screenshots/with_cloudwatch_cpucreditusage_checking.jpeg)

### 15. CloudWatch CPU Utilization Check
![CloudWatch CPU Utilization](docs/screenshots/with_cloudwatch_cpuutilization_checking.jpeg)

## What I learned

- Deploying a basic metrics monitoring stack on AWS EC2
- Scraping Linux metrics with Node Exporter and Prometheus
- Writing Prometheus alert rules and validating alert states
- Visualizing metrics with Grafana and CloudWatch
- Testing alert delivery and incident lifecycle with PagerDuty
- Troubleshooting target health and verifying recovery

## Disclaimer

This is a learning project. Adapt ports, paths, IAM policies, alert thresholds, and receiver configuration to your own environment.
