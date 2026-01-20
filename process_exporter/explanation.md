# Prometheus & Alerting Architecture Guide

This document outlines the configuration changes required to implement **Dynamic CPU Alerting**. This architecture ensures alerts remain accurate regardless of the server's hardware (e.g., changing from 2 cores to 64 cores) by dynamically comparing process CPU usage against the total available server capacity.

---

## 1. Conceptual Understanding

### The Problem: "Isolating the Ports"
We have two separate monitoring agents running on different ports:
1. **Node Exporter (Port 9100/1784):** Knows hardware details (e.g., "I have 2 CPU cores").
2. **Process Exporter (Port 9256):** Knows app details (e.g., "Java is using 100% CPU").

By default, these two agents don't "know" each other. If we try to divide *Process CPU* by *Total Cores*, Prometheus fails because it cannot match the data.

### The Solution: The "Universal Join"
To calculate the true server load percentage, we perform a database join between these two sources using a common key: **`nodename`**.

1. **Label Injection:** We force the `process-exporter` to carry the `nodename` label (via `prometheus.yml`).
2. **Dynamic Math:** We write a query that counts the cores *live* and divides the process usage by that count.

**The Formula:**
$$\frac{\text{Process CPU Usage}}{\text{Total Server Cores}} \times 100 = \text{Actual \% of Capacity}$$

---

## 2. Configuration Changes

### A. `prometheus.yml` (The Bridge)
**Goal:** Ensure `process-exporter` has the `nodename` label so it can be joined with hardware stats.

**Change:** Add a `relabel_configs` block to the `process-exporter` job to copy the EC2 `Name` tag into a `nodename` label.

```yaml
scrape_configs:
  - job_name: "process-exporter"
    ec2_sd_configs:
      - region: "ap-south-1"
        port: 9256
    relabel_configs:
      # 1. Keep only Linux instances
      - source_labels: [__meta_ec2_tag_OS]
        regex: linux
        action: keep
      
      # 2. Fix the Port to 9256 (Process Exporter Port)
      - source_labels: [__meta_ec2_private_ip]
        replacement: ${1}:9256
        target_label: __address__
      
      # 3. CRITICAL: Inject the 'nodename' label
      # This allows us to join this data with Node Exporter data later.
      - source_labels: [__meta_ec2_tag_Name]
        target_label: nodename
```

### B. `EC2-alerts.yml` (The Logic)
**Goal:** Create specific alerts for Java and Python that trigger based on Total Server Capacity, not just single-core usage.

**Key Concepts:**

- `[1m]` Window: Used instead of `[10s]` to ensure we cover at least 2 scrape intervals (preventing "Empty query result").
- `group_left`: Broadcasts the hardware core count (one value) to multiple processes (many values).
- `on(nodename)`: The bridge that connects the two data sources.

```yaml
groups:
  - name: server_alerts
    rules:
      # ---------------------------------------------------------
      # Alert 1: Python Worker
      # Threshold: > 50% of Total Server Capacity
      # ---------------------------------------------------------
      - alert: PythonWorkerHighCPU
        expr: >
          100 * sum by (nodename) (rate(namedprocess_namegroup_cpu_seconds_total{groupname="python_worker"}[1m])) 
          / on (nodename) group_left 
          count by (nodename) (node_cpu_seconds_total{mode="idle", job="node-linux"}) 
          > 50
        for: 10s
        labels:
          severity: critical
        annotations:
          summary: "Python Worker HIGH CPU on {{ $labels.nodename }}"
          description: "Using {{ printf \"%.1f\" $value }}% of Total Server Capacity (Dynamic Core Count)"

      # ---------------------------------------------------------
      # Alert 2: Java Billing
      # Threshold: > 25% of Total Server Capacity
      # ---------------------------------------------------------
      - alert: JavaWorkerHighCPU
        expr: >
          100 * sum by (nodename) (rate(namedprocess_namegroup_cpu_seconds_total{groupname="java_billing"}[1m])) 
          / on (nodename) group_left 
          count by (nodename) (node_cpu_seconds_total{mode="idle", job="node-linux"}) 
          > 25
        for: 10s
        labels:
          severity: trouble
        annotations:
          summary: "Java Worker HIGH CPU on {{ $labels.nodename }}"
          description: "Using {{ printf \"%.1f\" $value }}% of Total Server Capacity (Dynamic Core Count)"
```

### C. `alertmanager.yml` (The Notification)
**Goal:** Route these alerts to a human (via Slack, Email, or PagerDuty).

**Note:** Replace placeholders (like `api_url` or `email_configs`) with your actual credentials.

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'nodename']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'team-notifications'

receivers:
  - name: 'team-notifications'
    # Example 1: Slack Integration
    slack_configs:
      - api_url: ' "https://hoo--------------------------------" '
        channel: '#devops-alerts'
        send_resolved: true
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'

    # Example 2: Email Integration (Optional)
    # email_configs:
    #   - to: 'devops-team@example.com'
    #     from: 'prometheus@example.com'
    #     smarthost: 'smtp.gmail.com:587'
    #     auth_username: 'your_email@gmail.com'
    #     auth_password: 'your_app_password'
```

---

## 3. How to Apply These Changes

**Edit the files:**

```bash
nano /etc/prometheus/prometheus.yml
nano /etc/prometheus/EC2-alerts.yml
nano /etc/alertmanager/alertmanager.yml
```

**Validate syntax (Optional but recommended):**

```bash
promtool check config /etc/prometheus/prometheus.yml
promtool check rules /etc/prometheus/EC2-alerts.yml
```

**Reload Prometheus to apply changes:**

```bash
curl -X POST http://localhost:9090/-/reload
```

**Restart Alertmanager (if you changed `alertmanager.yml`):**

```bash
systemctl restart alertmanager
```

### Next Step
Would you like the commands to **cleanup your server** (stop the 100% CPU loop) n