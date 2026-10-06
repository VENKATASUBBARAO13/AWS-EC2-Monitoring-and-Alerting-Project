# AWS EC2 Monitoring & Alerting Architecture

## Production-style monitoring flow

    AWS Cloud / VPC
    │
    ├── Monitored EC2
    │     └── Linux OS
    │           └── Node Exporter :9100
    │                  │
    │                  │ /metrics
    │                  ▼
    └── Monitoring EC2
          ├── Prometheus :9090
          │     ├── Scrapes Node Exporter
          │     └── Evaluates alert_rules.yml
          │            │
          │            ▼
          │       Alertmanager
          │            │
          │            ▼
          │        PagerDuty
          │
          └── Grafana :3000
                ├── Prometheus data source
                └── CloudWatch data source
                         │
                         ▼
                    Amazon CloudWatch

## Main data flow

1. Node Exporter exposes Linux host metrics on the monitored EC2 instance.
2. Prometheus scrapes the Node Exporter endpoint every 15 seconds.
3. Prometheus evaluates InstanceDown, HighCPUUsage, and HighDiskUsage rules.
4. Firing alerts are routed through Alertmanager to PagerDuty.
5. Grafana reads Prometheus metrics for infrastructure dashboards.
6. Grafana can also read AWS metrics from CloudWatch using IAM permissions.

## Alert pipeline

    Node Exporter
          │
          ▼
      Prometheus
          │
          ▼
      Alert rules
          │
          ▼
     Alertmanager
          │
          ▼
       PagerDuty
          │
          ▼
     Incident / Recovery

## Failure and recovery test

    Normal
      │
      ▼
    Target UP
      │
      ▼
    Stop Node Exporter
      │
      ▼
    Target DOWN
      │
      ▼
    InstanceDown fires
      │
      ▼
    PagerDuty incident triggered
      │
      ▼
    Start Node Exporter
      │
      ▼
    Target UP
      │
      ▼
    Alert resolves

## Recommended network controls

| Component | Port | Recommended source |
|---|---:|---|
| Node Exporter | 9100 | Monitoring EC2 security group only |
| Prometheus | 9090 | Trusted admin / monitoring network |
| Grafana | 3000 | Trusted admin / monitoring network |
| Alertmanager | 9093 | Monitoring components only |

Do not expose ports 9100, 9090, 3000, or 9093 to the public Internet for a normal deployment.

## Repository mapping

    prometheus/
    ├── prometheus.yml
    └── alert_rules.yml

    alertmanager/
    └── alertmanager.example.yml

    node-exporter/
    └── node_exporter.service

    grafana/
    └── provisioning/
        └── datasources/datasources.yml

## Why this architecture is better

- Separates the monitored workload from the monitoring stack.
- Centralizes metrics collection in Prometheus.
- Separates visualization from alert routing.
- Uses PagerDuty for incident lifecycle handling.
- Combines infrastructure metrics from Node Exporter with AWS metrics from CloudWatch.
- Keeps monitoring ports restricted through security groups.