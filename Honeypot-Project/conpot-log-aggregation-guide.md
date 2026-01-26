# Conpot Honeypot Log Aggregation Setup Guide
## Grafana + Loki Stack for Multi-Device Access

### Overview
This guide sets up a complete log aggregation and visualization stack for your Conpot S7-1200 HVAC honeypot, allowing you to analyze attack patterns from any device (phone, laptop, desktop).

**Stack Components:**
- **Loki**: Log aggregation system (like Prometheus, but for logs)
- **Promtail**: Log shipper (reads Conpot logs and sends to Loki)
- **Grafana**: Visualization dashboard (accessible via web browser)

**Benefits:**
- Access logs from any device via web browser
- Real-time log streaming and queries
- Visual dashboards for attack patterns
- Alerting capabilities
- Historical analysis and trends

---

## Prerequisites

- Conpot honeypot already deployed (✓ You have this)
- Docker and Docker Compose installed
- Basic understanding of YAML configuration
- Available ports: 3000 (Grafana), 3100 (Loki)

---

## Step 1: Update Conpot for JSON Logging

JSON-formatted logs are easier to parse and query in Loki.

### 1.1 Modify Conpot Configuration

Edit your Conpot config to enable JSON logging:

```bash
nano ~/conpot-deployment/conpot.cfg
```

Add or update these sections:

```ini
[log_json]
enabled = True
filename = /var/log/conpot/conpot_json.log

[session]
timeout = 300

[fetch_public_ip]
enabled = True
urls = ["https://api.ipify.org/?format=text"]
```

### 1.2 Rebuild Conpot Image

```bash
cd ~/conpot-deployment
docker build -t conpot-hvac:latest .
```

---

## Step 2: Create Loki Configuration

### 2.1 Create Loki Config Directory

```bash
mkdir -p ~/conpot-deployment/loki
```

### 2.2 Create Loki Configuration File

```bash
nano ~/conpot-deployment/loki/loki-config.yml
```

Add this content:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

# Retention - keep logs for 30 days
limits_config:
  retention_period: 720h
```

---

## Step 3: Create Promtail Configuration

Promtail reads your Conpot logs and ships them to Loki.

### 3.1 Create Promtail Config Directory

```bash
mkdir -p ~/conpot-deployment/promtail
```

### 3.2 Create Promtail Configuration File

```bash
nano ~/conpot-deployment/promtail/promtail-config.yml
```

Add this content:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # JSON logs
  - job_name: conpot-json
    static_configs:
      - targets:
          - localhost
        labels:
          job: conpot
          __path__: /var/log/conpot/conpot_json.log
    pipeline_stages:
      - json:
          expressions:
            timestamp: timestamp
            level: level
            message: message
            session_id: session_id
            remote_host: remote_host
      - labels:
          level:
          session_id:
          remote_host:
      - timestamp:
          source: timestamp
          format: RFC3339

  # Standard text logs (fallback)
  - job_name: conpot-text
    static_configs:
      - targets:
          - localhost
        labels:
          job: conpot-text
          __path__: /var/log/conpot/conpot.log
    pipeline_stages:
      - regex:
          expression: '^(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3}) (?P<message>.*)'
      - labels:
          timestamp:
      - timestamp:
          source: timestamp
          format: '2006-01-02 15:04:05'
```

---

## Step 4: Update Docker Compose Configuration

### 4.1 Create Complete Stack Configuration

```bash
nano ~/conpot-deployment/docker-compose.yml
```

Replace with this complete configuration:

```yaml
services:
  conpot-hvac:
    image: conpot-hvac:latest
    container_name: conpot-hvac
    restart: unless-stopped
    command: ["/home/conpot/.local/bin/conpot", "--template", "custom", "--logfile", "/var/log/conpot/conpot.log", "-f", "--temp_dir", "/tmp"]
    ports:
      - "80:8800"         # HTTP
      - "102:102"         # S7comm
      - "502:5020"        # Modbus
      - "161:16100/udp"   # SNMP
    volumes:
      - conpot-logs:/var/log/conpot
    environment:
      - TZ=Europe/London
    networks:
      - honeypot-net

  loki:
    image: grafana/loki:2.9.3
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/tmp/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - honeypot-net

  promtail:
    image: grafana/promtail:2.9.3
    container_name: promtail
    restart: unless-stopped
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml
      - conpot-logs:/var/log/conpot:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    networks:
      - honeypot-net
    depends_on:
      - loki

  grafana:
    image: grafana/grafana:10.2.3
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=ChangeThisPassword123!
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://134.209.179.94:3000
    networks:
      - honeypot-net
    depends_on:
      - loki

volumes:
  conpot-logs:
  loki-data:
  grafana-data:

networks:
  honeypot-net:
    driver: bridge
```

---

## Step 5: Create Grafana Provisioning

This automatically configures Loki as a data source in Grafana.

### 5.1 Create Provisioning Directories

```bash
mkdir -p ~/conpot-deployment/grafana/provisioning/datasources
mkdir -p ~/conpot-deployment/grafana/provisioning/dashboards
```

### 5.2 Configure Loki Data Source

```bash
nano ~/conpot-deployment/grafana/provisioning/datasources/loki.yml
```

Add this content:

```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
    editable: true
```

### 5.3 Create Dashboard Provisioning Config

```bash
nano ~/conpot-deployment/grafana/provisioning/dashboards/default.yml
```

Add this content:

```yaml
apiVersion: 1

providers:
  - name: 'Conpot Honeypot'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

---

## Step 6: Create Honeypot Dashboard

### 6.1 Create Basic Dashboard JSON

```bash
nano ~/conpot-deployment/grafana/provisioning/dashboards/conpot-dashboard.json
```

Add this starter dashboard:

```json
{
  "dashboard": {
    "title": "Conpot Honeypot Analysis",
    "panels": [
      {
        "id": 1,
        "title": "Recent Logs",
        "type": "logs",
        "datasource": "Loki",
        "targets": [
          {
            "expr": "{job=\"conpot\"}"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 24,
          "x": 0,
          "y": 0
        }
      },
      {
        "id": 2,
        "title": "Connections by IP",
        "type": "table",
        "datasource": "Loki",
        "targets": [
          {
            "expr": "sum by (remote_host) (count_over_time({job=\"conpot\"}[24h]))"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 8
        }
      },
      {
        "id": 3,
        "title": "Protocol Distribution",
        "type": "piechart",
        "datasource": "Loki",
        "targets": [
          {
            "expr": "sum by (protocol) (count_over_time({job=\"conpot\"} |~ \"(?i)(http|s7comm|modbus|snmp)\" [24h]))"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 8
        }
      }
    ],
    "refresh": "30s",
    "time": {
      "from": "now-24h",
      "to": "now"
    }
  }
}
```

---

## Step 7: Configure Firewall

Open Grafana port for external access:

```bash
sudo ufw allow 3000/tcp
sudo ufw status
```

**Optional**: If you want Loki accessible externally (not recommended for security):
```bash
# sudo ufw allow 3100/tcp  # Don't do this unless necessary
```

---

## Step 8: Deploy the Stack

### 8.1 Stop Current Conpot

```bash
cd ~/conpot-deployment
docker compose down
```

### 8.2 Deploy Complete Stack

```bash
docker compose up -d
```

### 8.3 Verify All Services Running

```bash
docker compose ps
```

You should see:
- conpot-hvac (Up)
- loki (Up)
- promtail (Up)
- grafana (Up)

### 8.4 Check Logs

```bash
# Conpot logs
docker compose logs -f conpot-hvac

# Loki logs
docker compose logs -f loki

# Promtail logs
docker compose logs -f promtail

# Grafana logs
docker compose logs -f grafana
```

---

## Step 9: Access Grafana Dashboard

### 9.1 Initial Login

1. Open browser (phone, laptop, desktop)
2. Navigate to: `http://134.209.179.94:3000`
3. Login credentials:
   - Username: `admin`
   - Password: `ChangeThisPassword123!` (change this immediately!)

### 9.2 Change Default Password

1. Click on profile icon (bottom left)
2. Select "Change Password"
3. Set a strong password

### 9.3 Verify Loki Connection

1. Go to Configuration → Data Sources
2. Click on "Loki"
3. Click "Test" button at bottom
4. Should see "Data source is working"

---

## Step 10: Create Custom Queries

### Useful LogQL Queries for Honeypot Analysis

#### All S7comm Activity
```logql
{job="conpot"} |~ "(?i)s7comm"
```

#### All Modbus Activity
```logql
{job="conpot"} |~ "(?i)modbus"
```

#### Connections from Specific IP
```logql
{job="conpot", remote_host="1.2.3.4"}
```

#### HTTP Requests with User-Agents
```logql
{job="conpot"} |~ "User-Agent"
```

#### Failed Connection Attempts
```logql
{job="conpot"} |~ "(?i)(error|failed|refused)"
```

#### Count Unique IPs in Last 24h
```logql
count(count by (remote_host) (rate({job="conpot"}[24h])))
```

#### Top 10 Most Active IPs
```logql
topk(10, sum by (remote_host) (count_over_time({job="conpot"}[24h])))
```

---

## Step 11: Set Up Alerts (Optional)

### 11.1 Create Alert for Unusual Activity

In Grafana:
1. Go to Alerting → Alert rules
2. Create new alert rule
3. Query: `count_over_time({job="conpot"} |~ "s7comm" [5m]) > 10`
4. Condition: Alert when query returns value > threshold
5. Set notification channel (email, Slack, etc.)

### 11.2 Example Alert: High S7comm Scanning

```yaml
Alert Name: High S7comm Scanning Activity
Query: rate({job="conpot"} |~ "s7comm" [5m]) > 2
Condition: When rate exceeds 2 requests/second for 5 minutes
Action: Send notification
```

---

## Step 12: Mobile Access Setup

### For iOS (iPhone/iPad)
1. Open Safari
2. Navigate to `http://134.209.179.94:3000`
3. Tap share icon → "Add to Home Screen"
4. Now you have a Grafana app icon

### For Android
1. Open Chrome
2. Navigate to `http://134.209.179.94:3000`
3. Tap menu (three dots) → "Add to Home screen"
4. App icon created on home screen

### Security Consideration
For production use, you should:
1. Set up HTTPS/SSL (using Let's Encrypt)
2. Use strong passwords
3. Consider VPN access instead of public exposure
4. Enable Grafana authentication (OAuth, LDAP)

---

## Maintenance & Troubleshooting

### View Service Logs
```bash
cd ~/conpot-deployment
docker compose logs -f [service_name]
```

### Restart Specific Service
```bash
docker compose restart grafana
docker compose restart loki
docker compose restart promtail
```

### Check Disk Space (logs can grow)
```bash
df -h
docker system df
```

### Clean Old Logs (if needed)
```bash
# Loki retains logs for 30 days by default
# Manual cleanup:
docker compose exec loki rm -rf /tmp/loki/chunks/*
```

### Backup Grafana Dashboards
```bash
docker compose exec grafana grafana-cli admin export-dashboard > dashboard-backup.json
```

---

## Research Analysis Workflow

### Daily Monitoring Routine
1. Check Grafana dashboard on phone/laptop
2. Review "Connections by IP" for new attackers
3. Query specific protocols (S7comm, Modbus) for reconnaissance patterns
4. Export interesting sessions for deeper analysis

### Weekly Analysis
1. Export logs for the week
2. Analyze attack patterns and frequencies
3. Cross-reference attacking IPs with Shodan data
4. Document new attack vectors or tools

### Export Logs for Analysis
```bash
# Export last 7 days to JSON
curl -G -s "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="conpot"}' \
  --data-urlencode 'start=2024-01-01T00:00:00Z' \
  --data-urlencode 'end=2024-01-08T00:00:00Z' \
  > conpot_week_logs.json
```

---

## Advanced Features

### Geolocation Visualization
1. Install GeoIP plugin in Grafana
2. Map attacking IPs on world map
3. Visualize attack origins

### Integration with Threat Intelligence
1. Configure alerts to check IPs against threat feeds
2. Automatic tagging of known malicious IPs
3. Integration with MISP, AlienVault OTX

### Long-term Storage
1. Set up S3 bucket for log archival
2. Configure Loki to ship old logs to S3
3. Keep 90+ days of data for research

---

## Security Hardening

### Grafana Security Best Practices
```bash
# Edit docker-compose.yml to add:
environment:
  - GF_SECURITY_ADMIN_PASSWORD=<strong-password>
  - GF_SERVER_ROOT_URL=https://your-domain.com
  - GF_SECURITY_COOKIE_SECURE=true
  - GF_SECURITY_COOKIE_SAMESITE=strict
  - GF_AUTH_ANONYMOUS_ENABLED=false
```

### Restrict Grafana Access by IP
```bash
# In UFW
sudo ufw allow from YOUR_HOME_IP to any port 3000
sudo ufw deny 3000
```

### Enable HTTPS with Let's Encrypt
```bash
# Install Nginx as reverse proxy
sudo apt install nginx certbot python3-certbot-nginx

# Get SSL certificate
sudo certbot --nginx -d your-domain.com

# Configure Nginx to proxy to Grafana on port 3000
```

---

## Cost Considerations

### Current DO Droplet Resources
- Your honeypot: ~100MB RAM
- Loki: ~200-300MB RAM
- Grafana: ~150-200MB RAM
- Promtail: ~50MB RAM

**Total**: ~500-650MB RAM usage

Your droplet should handle this, but monitor:
```bash
htop
docker stats
```

### Log Storage Growth
- Expect ~100-500MB/day depending on attack volume
- Monitor: `du -sh ~/conpot-deployment/loki/`
- Adjust retention in loki-config.yml if needed

---

## Quick Reference Commands

```bash
# Navigate to deployment directory
cd ~/conpot-deployment

# View all service logs
docker compose logs -f

# View specific service
docker compose logs -f grafana

# Restart all services
docker compose restart

# Stop all services
docker compose down

# Start all services
docker compose up -d

# Check service status
docker compose ps

# View resource usage
docker stats
```

---

## Research Questions to Answer with This Setup

1. **Attack Patterns**: What times of day see most scanning activity?
2. **Geographic Distribution**: Where do attackers originate from?
3. **Protocol Preferences**: Do attackers probe S7comm or Modbus more?
4. **Reconnaissance Techniques**: What fingerprinting tools are used?
5. **Exploit Attempts**: Are there attempts to write to PLCs?
6. **Persistence**: Do attackers return multiple times?
7. **Tooling**: What ICS security tools are seen in User-Agents?

---

## Next Steps for MSc Research

1. **Baseline Period**: Run for 2-4 weeks to establish normal scanning patterns
2. **Data Collection**: Export logs weekly for deeper analysis
3. **Comparative Analysis**: Deploy second honeypot variant (different model) for comparison
4. **Threat Intelligence**: Cross-reference attacker IPs with threat databases
5. **Publication**: Use aggregated data for research paper/thesis

---

## Additional Resources

- Grafana Documentation: https://grafana.com/docs/
- Loki Documentation: https://grafana.com/docs/loki/latest/
- LogQL Query Language: https://grafana.com/docs/loki/latest/logql/
- Conpot Project: https://github.com/mushorg/conpot
- ICS Security Research: https://www.cisa.gov/ics

---

## Support & Community

- Grafana Community: https://community.grafana.com/
- Conpot Issues: https://github.com/mushorg/conpot/issues
- Honeypot Research: /r/netsec, /r/homelab on Reddit

---

**Version**: 1.0  
**Last Updated**: January 2026  
**Maintainer**: Nick - Royal Holloway MSc Research  
**Status**: Production-ready for research deployment
