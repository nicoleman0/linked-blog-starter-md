# Log File Download and Analysis Guide

This guide shows you how to set up persistent logging on your DigitalOcean Conpot honeypot and download logs for local analysis.

## Overview

**Strategy:**
- Conpot writes logs to a file on the DO server
- You periodically download logs to your local machine
- Analyze logs locally using Python, grep, or your preferred tools

**Benefits:**
- Free (no additional services needed)
- Full control over analysis
- Works offline once downloaded
- Easy to integrate with research workflow
- Version control your analysis scripts

## Step 1: Deploy Conpot with Log Mounting

When deploying on DigitalOcean, use the `-v` flag to mount logs to a directory:

```bash
# Create log directory
mkdir -p ~/conpot-deployment/logs

# Run Conpot with persistent logging
docker run -d \
  --name conpot \
  --restart unless-stopped \
  -p 8800:8800 \
  -p 102:10201 \
  -p 502:5020 \
  -p 161:16100/udp \
  -p 47808:47808/udp \
  -p 623:6230/udp \
  -p 21:2121 \
  -p 69:6969/udp \
  -p 44818:44818 \
  -v ~/conpot-deployment/logs:/var/log/conpot \
  honeynet/conpot:latest
```

**What this does:**
- Creates `~/conpot-deployment/logs/` directory on the server
- Mounts it into the container at `/var/log/conpot`
- Conpot writes `conpot.log` to this directory
- Logs persist even if container restarts

## Step 2: Verify Logs Are Being Created

SSH into your DO server:

```bash
ssh root@YOUR_DROPLET_IP
```

Check that logs exist:

```bash
# List log files
ls -lh ~/conpot-deployment/logs/

# View last 20 lines
tail -20 ~/conpot-deployment/logs/conpot.log

# Watch logs in real-time
tail -f ~/conpot-deployment/logs/conpot.log
```

You should see entries like:
```
2026-01-25 18:31:49,320 New http session from 172.17.0.1 (8eb21bb3-1a13-40d3-b736-6a762e6c90dc)
2026-01-25 18:31:49,320 HTTP/1.1 GET request from ('172.17.0.1', 47300): ('/', [('Host', 'localhost:8800'), ('User-Agent', 'curl/8.18.0'), ('Accept', '*/*')], None). 8eb21bb3-1a13-40d3-b736-6a762e6c90dc
```

## Step 3: Set Up Local Directory Structure

On your local machine, create a directory for honeypot data:

```bash
# Create analysis workspace
mkdir -p ~/honeypot-research/{logs,scripts,reports}
cd ~/honeypot-research
```

Directory structure:
```
~/honeypot-research/
├── logs/              # Downloaded log files
├── scripts/           # Analysis scripts
└── reports/           # Generated reports and findings
```

## Step 4: Download Logs to Local Machine

### Option A: Using SCP (Simple, one-time download)

```bash
# From your local machine
cd ~/honeypot-research/logs

# Download the log file
scp root@YOUR_DROPLET_IP:~/conpot-deployment/logs/conpot.log ./conpot-$(date +%Y%m%d).log

# Download entire logs directory
scp -r root@YOUR_DROPLET_IP:~/conpot-deployment/logs/ ./
```

### Option B: Using rsync (Recommended - incremental sync)

```bash
# From your local machine
cd ~/honeypot-research

# First time - download everything
rsync -avz root@YOUR_DROPLET_IP:~/conpot-deployment/logs/ ./logs/

# Subsequent runs - only downloads new/changed data
rsync -avz root@YOUR_DROPLET_IP:~/conpot-deployment/logs/ ./logs/
```

**rsync advantages:**
- Only transfers new data
- Faster for regular updates
- Can resume interrupted transfers
- Preserves timestamps

### Option C: Create a Download Script

Save this as `~/honeypot-research/scripts/download-logs.sh`:

```bash
#!/bin/bash

# Configuration
REMOTE_HOST="root@YOUR_DROPLET_IP"
REMOTE_PATH="~/conpot-deployment/logs/"
LOCAL_PATH="$HOME/honeypot-research/logs/"

# Create dated backup before download
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="$HOME/honeypot-research/backups/$DATE"

# Sync logs
echo "Downloading logs from $REMOTE_HOST..."
rsync -avz --progress "$REMOTE_HOST:$REMOTE_PATH" "$LOCAL_PATH"

# Create backup
echo "Creating backup..."
mkdir -p "$BACKUP_DIR"
cp -r "$LOCAL_PATH"* "$BACKUP_DIR/"

echo "Done! Logs saved to $LOCAL_PATH"
echo "Backup created at $BACKUP_DIR"

# Show summary
echo ""
echo "=== Log Summary ==="
wc -l "$LOCAL_PATH"conpot.log
```

Make it executable:

```bash
chmod +x ~/honeypot-research/scripts/download-logs.sh
```

Run it:

```bash
~/honeypot-research/scripts/download-logs.sh
```

## Step 5: Analyzing Logs Locally

### Quick Analysis with Command Line

```bash
cd ~/honeypot-research/logs

# Count total log entries
wc -l conpot.log

# Find unique attacker IPs
grep "session from" conpot.log | awk '{print $7}' | sort -u > unique_ips.txt
cat unique_ips.txt | wc -l

# Top 10 most active IPs
grep "session from" conpot.log | awk '{print $7}' | sort | uniq -c | sort -rn | head -10

# Count HTTP requests
grep "HTTP.*request" conpot.log | wc -l

# Count by protocol
echo "HTTP attacks:"
grep "HTTP.*request" conpot.log | wc -l
echo "Modbus attacks:"
grep -i "modbus" conpot.log | wc -l
echo "S7Comm attacks:"
grep -i "s7comm" conpot.log | wc -l

# Attacks by date
grep "session from" conpot.log | cut -d' ' -f1 | sort | uniq -c

# Find attacks from specific IP
grep "1.2.3.4" conpot.log

# Extract all User-Agent strings
grep "User-Agent" conpot.log | sed "s/.*User-Agent', '//" | sed "s/').*//" | sort | uniq -c | sort -rn
```

### Python Analysis Script

Save as `~/honeypot-research/scripts/analyze_logs.py`:

```python
#!/usr/bin/env python3
"""
Conpot Honeypot Log Analyzer
Parses and analyzes Conpot logs for research purposes
"""

import re
from collections import Counter, defaultdict
from datetime import datetime
import sys

def parse_log_file(filename):
    """Parse Conpot log file and extract attack data"""

    attacks = []
    sessions = {}

    with open(filename, 'r') as f:
        for line in f:
            # Parse new sessions
            session_match = re.search(r'New (\w+) session from ([\d.]+) \(([a-f0-9-]+)\)', line)
            if session_match:
                protocol, ip, session_id = session_match.groups()
                sessions[session_id] = {
                    'protocol': protocol,
                    'ip': ip,
                    'session_id': session_id,
                    'requests': []
                }

            # Parse HTTP requests
            http_match = re.search(r"HTTP/[\d.]+ (GET|POST) request from \('([\d.]+)', (\d+)\): \('([^']+)'.*User-Agent', '([^']+)'", line)
            if http_match:
                method, ip, port, path, user_agent = http_match.groups()
                timestamp = line.split()[0] + ' ' + line.split()[1]

                attacks.append({
                    'timestamp': timestamp,
                    'ip': ip,
                    'port': port,
                    'protocol': 'HTTP',
                    'method': method,
                    'path': path,
                    'user_agent': user_agent
                })

            # Parse Modbus requests
            if 'modbus' in line.lower() and 'request' in line.lower():
                attacks.append({
                    'timestamp': line.split()[0] + ' ' + line.split()[1],
                    'protocol': 'Modbus',
                    'raw': line.strip()
                })

    return attacks, sessions

def generate_report(attacks, sessions):
    """Generate analysis report"""

    print("=" * 60)
    print("CONPOT HONEYPOT ANALYSIS REPORT")
    print("=" * 60)
    print()

    # Total statistics
    print(f"Total attack entries: {len(attacks)}")
    print(f"Total unique sessions: {len(sessions)}")
    print()

    # Attacks by IP
    ip_counter = Counter(a['ip'] for a in attacks if 'ip' in a)
    print("TOP 10 ATTACKING IPs:")
    for ip, count in ip_counter.most_common(10):
        print(f"  {ip:20} - {count:5} attacks")
    print()

    # Attacks by protocol
    protocol_counter = Counter(a['protocol'] for a in attacks)
    print("ATTACKS BY PROTOCOL:")
    for protocol, count in protocol_counter.items():
        print(f"  {protocol:15} - {count:5} attacks")
    print()

    # HTTP-specific analysis
    http_attacks = [a for a in attacks if a.get('protocol') == 'HTTP']
    if http_attacks:
        print("HTTP ANALYSIS:")

        # Most requested paths
        path_counter = Counter(a['path'] for a in http_attacks)
        print("  Most requested paths:")
        for path, count in path_counter.most_common(10):
            print(f"    {path:30} - {count} times")
        print()

        # User agents
        ua_counter = Counter(a['user_agent'] for a in http_attacks)
        print("  Top User-Agents:")
        for ua, count in ua_counter.most_common(10):
            print(f"    {ua[:50]:50} - {count} times")
        print()

    # Session protocols
    session_protocols = Counter(s['protocol'] for s in sessions.values())
    print("SESSIONS BY PROTOCOL:")
    for protocol, count in session_protocols.items():
        print(f"  {protocol:15} - {count:5} sessions")
    print()

    print("=" * 60)

def main():
    if len(sys.argv) < 2:
        print(f"Usage: {sys.argv[0]} <logfile>")
        print(f"Example: {sys.argv[0]} logs/conpot.log")
        sys.exit(1)

    logfile = sys.argv[1]
    print(f"Analyzing {logfile}...")
    print()

    attacks, sessions = parse_log_file(logfile)
    generate_report(attacks, sessions)

if __name__ == '__main__':
    main()
```

Make it executable:

```bash
chmod +x ~/honeypot-research/scripts/analyze_logs.py
```

Run it:

```bash
cd ~/honeypot-research
./scripts/analyze_logs.py logs/conpot.log
```

### Advanced Analysis with Pandas

Save as `~/honeypot-research/scripts/advanced_analysis.py`:

```python
#!/usr/bin/env python3
"""
Advanced Conpot analysis using Pandas
Generates CSV reports and visualizations
"""

import pandas as pd
import re
from collections import Counter
import sys

def parse_to_dataframe(logfile):
    """Parse log file into pandas DataFrame"""

    records = []

    with open(logfile, 'r') as f:
        for line in f:
            # HTTP requests
            http_match = re.search(r"(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}).*HTTP.*request from \('([\d.]+)', (\d+)\): \('([^']+)'", line)
            if http_match:
                timestamp, ip, port, path = http_match.groups()

                ua_match = re.search(r"User-Agent', '([^']+)'", line)
                user_agent = ua_match.group(1) if ua_match else 'Unknown'

                records.append({
                    'timestamp': timestamp,
                    'ip': ip,
                    'port': int(port),
                    'protocol': 'HTTP',
                    'path': path,
                    'user_agent': user_agent
                })

            # Sessions
            session_match = re.search(r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}).*New (\w+) session from ([\d.]+)', line)
            if session_match:
                timestamp, protocol, ip = session_match.groups()
                records.append({
                    'timestamp': timestamp,
                    'ip': ip,
                    'protocol': protocol,
                    'event_type': 'session_start'
                })

    df = pd.DataFrame(records)
    if not df.empty:
        df['timestamp'] = pd.to_datetime(df['timestamp'])
        df['date'] = df['timestamp'].dt.date
        df['hour'] = df['timestamp'].dt.hour

    return df

def main():
    if len(sys.argv) < 2:
        print(f"Usage: {sys.argv[0]} <logfile>")
        sys.exit(1)

    logfile = sys.argv[1]
    print(f"Parsing {logfile} into DataFrame...")

    df = parse_to_dataframe(logfile)

    if df.empty:
        print("No data found in log file")
        return

    print(f"Loaded {len(df)} records")
    print()

    # Attacks by date
    print("ATTACKS BY DATE:")
    print(df.groupby('date').size().to_string())
    print()

    # Attacks by hour
    print("ATTACKS BY HOUR OF DAY:")
    print(df.groupby('hour').size().to_string())
    print()

    # Top IPs
    print("TOP 20 ATTACKING IPs:")
    print(df['ip'].value_counts().head(20).to_string())
    print()

    # Protocol distribution
    print("PROTOCOL DISTRIBUTION:")
    print(df['protocol'].value_counts().to_string())
    print()

    # Export to CSV
    output_file = logfile.replace('.log', '_analysis.csv')
    df.to_csv(output_file, index=False)
    print(f"Full data exported to: {output_file}")

    # Summary statistics
    summary_file = logfile.replace('.log', '_summary.csv')
    summary = df.groupby(['date', 'protocol']).size().reset_index(name='count')
    summary.to_csv(summary_file, index=False)
    print(f"Summary exported to: {summary_file}")

if __name__ == '__main__':
    main()
```

## Step 6: Create Analysis Workflow

### Weekly Research Routine

Create `~/honeypot-research/scripts/weekly-analysis.sh`:

```bash
#!/bin/bash

echo "=== WEEKLY HONEYPOT ANALYSIS ==="
echo "Date: $(date)"
echo ""

# Download latest logs
echo "1. Downloading logs..."
rsync -avz root@YOUR_DROPLET_IP:~/conpot-deployment/logs/ ~/honeypot-research/logs/

# Run analysis
echo ""
echo "2. Running analysis..."
python3 ~/honeypot-research/scripts/analyze_logs.py ~/honeypot-research/logs/conpot.log > ~/honeypot-research/reports/report-$(date +%Y%m%d).txt

# Generate CSV
echo ""
echo "3. Generating CSV exports..."
python3 ~/honeypot-research/scripts/advanced_analysis.py ~/honeypot-research/logs/conpot.log

echo ""
echo "Done! Check ~/honeypot-research/reports/ for results"
```

Make executable and run:

```bash
chmod +x ~/honeypot-research/scripts/weekly-analysis.sh
~/honeypot-research/scripts/weekly-analysis.sh
```

## Step 7: GeoIP Analysis (Optional)

Install GeoIP tools to identify attacker countries:

```bash
# On your local machine (Linux)
sudo apt install geoip-bin geoip-database

# Or on macOS
brew install geoip
```

Analyze attacker locations:

```bash
cd ~/honeypot-research/logs

# Extract unique IPs
grep "session from" conpot.log | awk '{print $7}' | sort -u > ips.txt

# Lookup countries
while read ip; do
    country=$(geoiplookup $ip | cut -d: -f2 | cut -d, -f1)
    echo "$ip - $country"
done < ips.txt | sort -k3 | uniq -c | sort -rn > country_summary.txt

cat country_summary.txt
```

## Step 8: Jupyter Notebook Analysis (Advanced)

For interactive analysis, use Jupyter:

```bash
# Install Jupyter (if not already installed)
pip install jupyter pandas matplotlib seaborn

# Start Jupyter
cd ~/honeypot-research
jupyter notebook
```

Create a new notebook with cells like:

```python
import pandas as pd
import matplotlib.pyplot as plt

# Load your parsed CSV
df = pd.read_csv('logs/conpot_analysis.csv')

# Plot attacks over time
df.groupby('date').size().plot(kind='line', figsize=(12,6))
plt.title('Honeypot Attacks Over Time')
plt.ylabel('Number of Attacks')
plt.show()

# Plot by protocol
df['protocol'].value_counts().plot(kind='bar')
plt.title('Attacks by Protocol')
plt.show()

# Top attacking countries (if you added GeoIP data)
# df['country'].value_counts().head(10).plot(kind='barh')
```

## Regular Maintenance

### Recommended Schedule

**Daily:** Quick check via SSH
```bash
ssh root@YOUR_DROPLET_IP "docker logs --tail 50 conpot"
```

**Weekly:** Download and analyze
```bash
~/honeypot-research/scripts/weekly-analysis.sh
```

**Monthly:** Deep dive analysis
- Review trends
- Update thesis notes
- Generate visualizations for presentation

### Log Rotation on Server

Prevent logs from filling disk:

```bash
# SSH into DO server
ssh root@YOUR_DROPLET_IP

# Create log rotation config
sudo nano /etc/logrotate.d/conpot
```

Add this content:

```
/root/conpot-deployment/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
}
```

## Tips for Research

1. **Version control your scripts**
   ```bash
   cd ~/honeypot-research
   git init
   git add scripts/
   git commit -m "Initial analysis scripts"
   ```

2. **Document findings** in markdown files as you go

3. **Keep raw logs** - don't delete original files

4. **Export important findings** to CSV for Excel/Google Sheets

5. **Take screenshots** of interesting attack patterns for thesis

6. **Correlate with Shodan** - search for your IP to see how attackers find you

## Troubleshooting

### No log file created

```bash
# Check if volume is mounted
docker inspect conpot | grep Mounts -A 10

# Check container logs
docker logs conpot | grep -i log
```

### Download very slow

```bash
# Compress on server first
ssh root@YOUR_DROPLET_IP "tar -czf /tmp/logs.tar.gz ~/conpot-deployment/logs/"
scp root@YOUR_DROPLET_IP:/tmp/logs.tar.gz ./
tar -xzf logs.tar.gz
```

### Can't parse logs

Check log format hasn't changed:
```bash
head -50 logs/conpot.log
```

Update regex patterns in analysis scripts if needed.

## Summary

You now have:
- ✓ Persistent logs on DO server
- ✓ Scripts to download logs
- ✓ Analysis tools (bash, Python, Pandas)
- ✓ Weekly workflow
- ✓ All analysis happens locally (free)

This setup gives you full control over your research data while keeping costs at just $6/month for the server.
