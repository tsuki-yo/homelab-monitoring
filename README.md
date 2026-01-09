# Homelab Monitoring Stack

Complete monitoring solution for homelabs using Prometheus, Grafana, Node Exporter, cAdvisor, and Alertmanager. Monitor system and container metrics across multiple hosts with automatic email alerts and beautiful dashboards.

**Blog Post**: [End-to-End Monitoring Explained for Homelabs: Prometheus, Grafana & Alertmanager](https://tsuki-yo.github.io/PiBlog/posts/homelab-monitoring-prometheus-grafana/)

## Features

- **System Monitoring**: CPU, memory, disk, network, load average, uptime via Node Exporter
- **Container Monitoring**: Per-container resource usage via cAdvisor
- **Multi-Host Support**: Monitor multiple servers from a single Grafana instance
- **Email Alerts**: Automatic notifications via Gmail SMTP with severity-based routing
- **30-Day Retention**: Historical metrics for trend analysis
- **Pre-configured Dashboards**: Ready-to-use Grafana dashboards with PromQL queries
- **Alert Rules**: System and container alerts for disk space, memory, CPU, and uptime

## Architecture

```
┌─────────────┐  ┌─────────────┐
│Node Exporter│  │  cAdvisor   │
│ Host Metrics│  │ Containers  │
└──────┬──────┘  └──────┬──────┘
       │                │
       │  Scrapes every 15s
       │                │
       └────────┬───────┘
                │
         ┌──────▼──────┐
         │ Prometheus  │
         │ Time-Series │
         └──┬────────┬─┘
            │        │
     Alerts │        │ Query
            │        │ (PromQL)
            ▼        ▲
    ┌────────────┐   │
    │Alertmanager│   │    ┌─────────┐
    │   Email    │   └────│ Grafana │
    └────────────┘        │   UI    │
                          └─────────┘
```

## Requirements

### Monitoring Host (where Docker stack runs)
- Docker and Docker Compose
- 2GB+ RAM
- 10GB+ disk space for metrics retention
- Linux (tested on Raspberry Pi 5 with Debian)

### Monitored Hosts
- Linux with systemd
- Network access from monitoring host
- Ports 9100 (Node Exporter) and 8080 (cAdvisor) accessible

## Quick Start

### 1. Clone Repository

```bash
git clone https://github.com/tsuki-yo/homelab-monitoring.git
cd homelab-monitoring
```

### 2. Install Node Exporter on All Hosts

Run on each host you want to monitor:

```bash
# Download Node Exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvfz node_exporter-1.7.0.linux-amd64.tar.gz
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# Create systemd service
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
Type=simple
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

# Start service
sudo useradd -rs /bin/false node_exporter
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

# Verify
curl http://localhost:9100/metrics
```

### 3. Configure Prometheus Targets

Edit `prometheus/config/prometheus.yml` and update the IP addresses:

```yaml
scrape_configs:
  # Replace 192.168.1.201 with your monitoring host IP
  - job_name: 'monitoring-host-node'
    static_configs:
      - targets: ['192.168.1.201:9100']

  # Replace 192.168.1.50 with your additional host IP
  - job_name: 'n305-node'
    static_configs:
      - targets: ['192.168.1.50:9100']
```

### 4. Configure Alertmanager Email

Edit `alertmanager/alertmanager.yml`:

```yaml
global:
  smtp_from: your-email@gmail.com
  smtp_auth_username: your-email@gmail.com
  smtp_auth_password: "your-gmail-app-password"  # Generate at https://myaccount.google.com/apppasswords
```

Replace `your-email@gmail.com` in all receivers as well.

### 5. Create Data Directories

```bash
mkdir -p prometheus/data grafana/data
chmod 777 prometheus/data grafana/data  # Docker containers need write access
```

### 6. Start the Stack

```bash
docker-compose up -d
```

### 7. Access Services

- **Grafana**: http://localhost:3000 (default login: admin/admin)
- **Prometheus**: http://localhost:9090
- **Alertmanager**: http://localhost:9093
- **cAdvisor**: http://localhost:8080

## Configuration Files

### docker-compose.yml
Defines 4 services: Prometheus, Grafana, Alertmanager, and cAdvisor

### prometheus/config/prometheus.yml
- Scrape targets (Node Exporter and cAdvisor endpoints)
- Scrape intervals (15s for most targets)
- Alert rule file locations

### prometheus/config/alerts/system-alerts.yml
System-level alerts:
- DiskSpaceLow (critical <10% free)
- HighMemoryUsage (warning >90%)
- HighCPUUsage (warning >90% for 10 min)
- HostDown (critical, host unreachable)

### prometheus/config/alerts/container-alerts.yml
Container-level alerts:
- ContainerDown (critical, container stopped)
- ContainerHighMemory (warning >8GB)
- CriticalServiceDown (critical services like Bitwarden/Immich)

### alertmanager/alertmanager.yml
Email notification routing:
- Critical alerts: repeat every 1 hour
- Warning alerts: repeat every 4 hours
- Alert grouping by name and severity
- Inhibition rules (critical suppresses warnings)

### grafana/provisioning/datasources/prometheus.yml
Auto-provisions Prometheus as default datasource with UID `PBFA97CFB590B2093`

## Building Dashboards

After starting Grafana:

1. Go to Dashboards > New Dashboard
2. Add panels with PromQL queries from the blog post
3. Configure dashboard variables for multi-host support:
   - `job` = `label_values(node_uname_info, job)`
   - `instance` = `label_values(node_uname_info{job="$job"}, instance)`

Example PromQL queries:

```promql
# CPU Usage %
100 - (avg(rate(node_cpu_seconds_total{mode="idle",instance="$instance",job="$job"}[5m])) * 100)

# Memory Usage %
100 * (1 - ((node_memory_MemAvailable_bytes{instance="$instance",job="$job"}) / node_memory_MemTotal_bytes{instance="$instance",job="$job"}))

# Disk Usage %
100 - ((node_filesystem_avail_bytes{instance="$instance",job="$job",mountpoint="/"} / node_filesystem_size_bytes{instance="$instance",job="$job",mountpoint="/"}) * 100)
```

See the [full blog post](https://tsuki-yo.github.io/PiBlog/posts/homelab-monitoring-prometheus-grafana/) for complete dashboard setup and more queries.

## Customization

### Add More Hosts

1. Install Node Exporter on new host
2. Add scrape target to `prometheus/config/prometheus.yml`:

```yaml
- job_name: 'new-host-node'
  scrape_interval: 15s
  static_configs:
    - targets: ['192.168.1.X:9100']
      labels:
        host: 'new-host'
        instance_name: 'new-host-name'
```

3. Restart Prometheus: `docker-compose restart prometheus`

### Adjust Alert Thresholds

Edit alert files in `prometheus/config/alerts/`:

```yaml
# Example: Change disk space warning from 10% to 20%
- alert: DiskSpaceLow
  expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 20  # Changed from 10
```

### Change Retention Period

Edit `docker-compose.yml`:

```yaml
prometheus:
  command:
    - '--storage.tsdb.retention.time=90d'  # Changed from 30d
```

## Troubleshooting

### Prometheus Not Scraping Targets

Check Prometheus targets page: http://localhost:9090/targets

- **Target is DOWN**: Verify Node Exporter is running (`systemctl status node_exporter`)
- **Network issues**: Check firewall rules, ensure port 9100 is accessible

### Grafana Shows "N/A" for All Panels

- Check datasource UID in `grafana/provisioning/datasources/prometheus.yml` matches dashboard queries
- Verify Prometheus is running: `docker ps | grep prometheus`
- Check Grafana logs: `docker logs monitoring-grafana`

### Alerts Not Sending Emails

- Verify Gmail App Password (not regular password)
- Check Alertmanager logs: `docker logs monitoring-alertmanager`
- Test SMTP: `curl -v telnet://smtp.gmail.com:587`

### High Memory Usage

- Reduce retention period in Prometheus
- Reduce scrape intervals in `prometheus.yml`
- Limit cAdvisor with `--housekeeping_interval=30s`

## Monitoring at Scale

Current setup handles:
- 2 hosts
- 37 containers
- 30 days retention
- ~500MB Prometheus data

For 10+ hosts, consider:
- Prometheus federation or Thanos for long-term storage
- Dedicated metrics server (4GB+ RAM)
- Longer scrape intervals (30s-60s)

## Security Notes

- **Never commit** `alertmanager.yml` with real credentials
- Use Gmail App Passwords, not account passwords
- Restrict access to Grafana/Prometheus with reverse proxy + auth
- Keep Docker images updated for security patches

## Contributing

Issues and pull requests welcome! Please test changes thoroughly before submitting.

## License

MIT License - feel free to use and modify for your homelab.

## Credits

Configuration based on the [End-to-End Monitoring Guide](https://tsuki-yo.github.io/PiBlog/posts/homelab-monitoring-prometheus-grafana/) by Tsuki-yo.

Built with:
- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.com/)
- [Node Exporter](https://github.com/prometheus/node_exporter)
- [cAdvisor](https://github.com/google/cadvisor)
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
