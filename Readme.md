# Monitoring Project Documentation

## Overview
This project sets up a robust monitoring solution using **Prometheus** and **Grafana** to monitor a server running multiple Docker containers and the host operating system. Metrics from the host (CPU, memory, disk) and containers (resource usage, health) are collected, visualized in Grafana dashboards, and monitored with alerting mechanisms via Alertmanager. The setup uses Docker Compose to manage services, ensuring easy deployment and scalability.

---

## Project Components
- **Prometheus**: A time-series database that collects and stores metrics from configured targets.
- **Node Exporter**: Exposes host OS metrics (e.g., CPU, memory, disk usage).
- **cAdvisor**: Monitors Docker container metrics (e.g., CPU, memory, network).
- **Grafana**: Visualizes metrics through customizable dashboards and supports alerting.
- **Alertmanager**: Handles alerts from Prometheus, sending notifications (e.g., via Slack or email).

## Architecture
- **Prometheus** scrapes metrics from:
  - Itself (`localhost:9090`)
  - Node Exporter (`node-exporter:9100`)
  - cAdvisor (`cadvisor:8080`)
- **Grafana** connects to Prometheus as a data source for visualization.
- **Alertmanager** receives alerts from Prometheus and sends notifications to configured receivers (e.g., Slack).
- All services run as Docker containers on a shared `monitoring` network.

## Prerequisites
- **Server**: Ubuntu 22.04 LTS (or similar Linux distribution).
- **Software**:
  - Docker: `sudo apt install docker.io`
  - Docker Compose: `sudo apt install docker-compose`
- **Ports**:
  - Prometheus: 9090
  - Grafana: 3000
  - Node Exporter: 9100
  - cAdvisor: 8080
  - Alertmanager: 9093
- **Disk Space**: At least 10GB for Prometheus and Grafana data volumes.

## Directory Structure
The project uses the following structure:
```
monitoring-project
├── docker-compose.yml
├── prometheus
│   ├── prometheus.yml
│   └── alerts.yml
├── grafana
│   └── grafana.ini
└── alertmanager
    └── alertmanager.yml
```

### Creating the Directory Structure
Run the following script to set up the directory structure:

```bash
#!/bin/bash
PROJECT_DIR="monitoring-project"
mkdir -p "$PROJECT_DIR"/{prometheus,grafana,alertmanager}
touch "$PROJECT_DIR"/{docker-compose.yml,prometheus/prometheus.yml,prometheus/alerts.yml,grafana/grafana.ini,alertmanager/alertmanager.yml}
chmod -R 755 "$PROJECT_DIR"
chmod 644 "$PROJECT_DIR"/{docker-compose.yml,prometheus/*,grafana/*,alertmanager/*}
echo "Directory structure created at $PROJECT_DIR"
```

Save as `setup_directory_structure.sh`, make executable (`chmod +x setup_directory_structure.sh`), and run (`./setup_directory_structure.sh`).

## Configuration Files

### 1. Docker Compose (`docker-compose.yml`)
Defines services for Prometheus, Node Exporter, cAdvisor, Grafana, and Alertmanager.

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml
      - prometheus_data:/prometheus
    command: 
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/host/root:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/host/root'
    ports:
      - "9100:9100"
    networks:
      - monitoring
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.0
    container_name: cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.1.0
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/grafana.ini:/etc/grafana/grafana.ini
    ports:
      - "3000:3000"
    networks:
      - monitoring
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.25.0
    container_name: alertmanager
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"
    networks:
      - monitoring
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

### 2. Prometheus Configuration (`prometheus/prometheus.yml`)
Configures Prometheus to scrape metrics and send alerts to Alertmanager.

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "alerts.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

### 3. Prometheus Alert Rules (`prometheus/alerts.yml`)
Defines alerts for high CPU usage and container downtime.

```yaml
groups:
- name: example
  rules:
  - alert: HighCPUUsage
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage detected on {{ $labels.instance }}"
      description: "{{ $labels.instance }} has CPU usage above 80% for 5 minutes."
  - alert: ContainerDown
    expr: absent(container_cpu_usage_seconds_total)
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Container is down"
      description: "A container has been down for more than 1 minute."
```

### 4. Alertmanager Configuration (`alertmanager/alertmanager.yml`)
Configures Alertmanager to send notifications (e.g., via Slack).

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'slack'

receivers:
- name: 'slack'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/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'
    channel: '#alerts'
    send_resolved: true
    title: '{{ .CommonAnnotations.summary }}'
    text: '{{ .CommonAnnotations.description }}'
```

**Note**: Replace the Slack webhook URL with your actual URL or configure an alternative receiver (e.g., email).

### 5. Grafana Configuration (`grafana/grafana.ini`)
Customizes Grafana settings, such as disabling anonymous access.

```ini
[auth]
disable_login_form = false
disable_signout_menu = false

[auth.anonymous]
enabled = false

[security]
allow_embedding = true
```

## Setup Instructions
1. **Create Directory Structure**:
   - Run the `setup_directory_structure.sh` script (see above) to create the directory structure.
   - Copy the configuration files into their respective directories.

2. **Deploy the Monitoring Stack**:
   - Navigate to the project directory: `cd monitoring-project`
   - Start services: `docker-compose up -d`
   - Verify containers are running: `docker ps`

3. **Access Services**:
   - **Prometheus**: `http://<server-ip>:9090`
   - **Grafana**: `http://<server-ip>:3000` (default login: admin/admin)
   - **Alertmanager**: `http://<server-ip>:9093`
   - **Node Exporter**: `http://<server-ip>:9100`
   - **cAdvisor**: `http://<server-ip>:8080`

## Configuring Grafana
### Adding Prometheus as a Data Source
1. Log in to Grafana (`http://<server-ip>:3000`).
2. Navigate to **Configuration > Data Sources** (gear icon in the sidebar).
3. Click **Add data source** and select **Prometheus**.
4. Configure the data source:
   - **Name**: `Prometheus`
   - **URL**: Use `http://prometheus:9090` (container name, preferred for Docker networking) instead of `http://localhost:9090`.
   - **Access**: Set to `Server (default)`.
5. Click **Save & Test**. A green “Data source is working” message confirms success.
6. **Note**: If `localhost:9090` fails (e.g., due to Docker network isolation), using the container name (`prometheus:9090`) ensures Grafana resolves the Prometheus service within the `monitoring` network.

### Importing Dashboards
1. Go to **Dashboards > Import**.
2. Import pre-built dashboards:
   - **Node Exporter Dashboard**: ID `1860` (host metrics).
   - **cAdvisor Dashboard**: ID `14282` (container metrics).
3. Select the `Prometheus` data source and save.
4. Customize dashboards as needed for specific metrics (e.g., container memory usage).

### Setting Up Grafana Alerts
1. Open a dashboard and edit a panel (e.g., for container memory usage).
2. Add an alert rule:
   - **Query**: `container_memory_usage_bytes / container_memory_limit_bytes * 100`
   - **Condition**: `> 80` (alert if memory usage exceeds 80%).
   - **Evaluate every**: `1m`, **For**: `5m`.
3. Configure a notification channel:
   - Go to **Alerting > Notification channels**.
   - Add a Slack or email channel, matching the Alertmanager receiver settings.
4. Save the alert and test by simulating high memory usage (e.g., `stress --vm 2 --vm-bytes 512M`).

## Alerting Setup
- **Prometheus Alerts**: Defined in `alerts.yml` (e.g., `HighCPUUsage`, `ContainerDown`).
- **Alertmanager**: Sends notifications to the configured receiver (e.g., Slack).
- **Testing Alerts**:
  - Simulate high CPU usage: `stress --cpu 4 --timeout 600`
  - Check Prometheus alerts: `http://<server-ip>:9090/alerts`
  - Verify Alertmanager: `http://<server-ip>:9093/#/alerts`
  - Confirm notifications in the Slack channel (or other receiver).

## Maintenance
- **Backup**: Regularly back up `prometheus_data` and `grafana_data` volumes:
  ```bash
  docker volume ls
  tar -czf prometheus_data_backup.tar.gz /var/lib/docker/volumes/monitoring-project_prometheus_data
  tar -czf grafana_data_backup.tar.gz /var/lib/docker/volumes/monitoring-project_grafana_data
  ```
- **Updates**: Update Docker images:
  ```bash
  docker-compose pull && docker-compose up -d
  ```
- **Scaling**: Add exporters for other applications (e.g., MySQL Exporter) by extending `docker-compose.yml` and `prometheus.yml`.

## Security Considerations
- Restrict port access using a firewall (e.g., `ufw allow 3000,9090,9093,9100,8080`).
- Enable HTTPS for Grafana and Alertmanager in production.
- Use strong passwords for Grafana (update via `grafana.ini` or admin settings).

## Troubleshooting
### General Issues
- **Container Not Starting**:
  - Check logs: `docker logs <container-name>`
  - Verify `docker-compose.yml` syntax: `docker-compose config`
  - Ensure sufficient disk space: `df -h`
- **Prometheus Not Scraping Metrics**:
  - Check targets: `http://<server-ip>:9090/targets`
  - Verify container connectivity: `docker exec prometheus curl http://node-exporter:9100/metrics`
  - Ensure `prometheus.yml` targets use container names (e.g., `node-exporter:9100`).
- **Alerts Not Firing**:
  - Verify alert rules: `http://<server-ip>:9090/alerts`
  - Check PromQL query in Prometheus UI: `100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
  - Lower threshold for testing (e.g., `> 50` in `alerts.yml`) and reload: `curl -X POST http://<server-ip>:9090/-/reload`
- **Notifications Not Sent**:
  - Check Alertmanager logs: `docker logs alertmanager`
  - Validate `alertmanager.yml`: `docker run -v $(pwd)/alertmanager/alertmanager.yml:/alertmanager.yml prom/alertmanager amtool check-config /alertmanager.yml`
  - Test Slack webhook: `curl -X POST -d '{"text":"Test"}' <your-slack-webhook-url>`

### Adding Prometheus Data Source in Grafana
If you encounter issues adding Prometheus as a data source in Grafana:
1. **Problem**: `localhost:9090` fails with a connection error.
   - **Cause**: Docker containers run in isolated networks, and `localhost` may not resolve to the Prometheus container.
   - **Solution**: Use the container name and port: `http://prometheus:9090`.
     - In Grafana’s data source settings, set **URL** to `http://prometheus:9090`.
     - The `monitoring` network in `docker-compose.yml` ensures containers can communicate by name.
2. **Problem**: “Data source is not working” after saving.
   - **Cause**: Network issues, incorrect URL, or Prometheus container not running.
   - **Solution**:
     - Verify Prometheus is running: `docker ps | grep prometheus`
     - Test connectivity from Grafana container: `docker exec grafana curl http://prometheus:9090`
     - Check Grafana logs: `docker logs grafana`
3. **Problem**: Metrics not appearing in Grafana dashboards.
   - **Cause**: Incorrect PromQL queries or mismatched data source.
   - **Solution**:
     - Ensure dashboards use the `Prometheus` data source (check dashboard settings).
     - Validate queries in Prometheus UI first (e.g., `node_cpu_seconds_total`).
     - Re-import dashboards (IDs: 1860 for Node Exporter, 14282 for cAdvisor).

## Stress Testing
To test alerts:
1. Install `stress`: `sudo apt install stress`
2. Simulate high CPU usage: `stress --cpu 4 --timeout 600`
3. Monitor:
   - Prometheus: `http://<server-ip>:9090/graph` (query CPU metrics)
   - Alertmanager: `http://<server-ip>:9093/#/alerts` (check `HighCPUUsage`)
   - Slack: Verify notification in the configured channel
4. Stop stress test: Ctrl+C or wait for timeout.

## Zipping the Project for Backup or Sharing
To create a compressed archive of the `monitoring-project` folder:
```bash
zip -r monitoring-project.zip monitoring-project
```
Verify contents: `unzip -l monitoring-project.zip`

## Extensibility
- Add exporters for specific applications (e.g., `mysql-exporter` for MySQL databases).
- Integrate additional notification channels (e.g., PagerDuty, email).
- Customize Grafana dashboards for application-specific metrics.

## Conclusion
This monitoring setup provides a scalable, production-ready solution for tracking server and container performance. Regularly monitor dashboards, test alerts, and back up data to ensure reliability. For further assistance, consult the troubleshooting section or contact the system administrator.

*Last Updated: June 19, 2025*