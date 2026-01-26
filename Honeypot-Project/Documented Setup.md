## Conpot Honeypot #1 Setup

**Deployment Date:** January 27, 2025  
**Server IP:** 134.209.179.94  
**Location:** DigitalOcean - London datacenter  
**Hostname:** backup-node-london

### Infrastructure Specs

- **OS:** Ubuntu 22.04 LTS
- **Resources:** 1GB RAM, 1 vCPU, 25GB SSD
- **Cost:** $6/month

### Honeypot Configuration

- **Platform:** Conpot 0.6.0 (Docker)
- **Template:** default (Technodrome)
- **Container Name:** conpot
- **Restart Policy:** unless-stopped

### Exposed Services & Ports

|Service|Port|Protocol|Purpose|
|---|---|---|---|
|HTTP|8800|TCP|Web interface|
|S7Comm|102|TCP|Siemens PLC protocol|
|Modbus|502|TCP|Industrial protocol|
|SNMP|161|UDP|Network monitoring|
|BACnet|47808|UDP|Building automation|
|IPMI|623|UDP|Server management|
|FTP|21|TCP|File transfer|
|TFTP|69|UDP|Trivial file transfer|
|EtherNet/IP|44818|TCP|Industrial Ethernet|

### Log Storage

- **Host Directory:** ~/conpot-deployment/logs/
- **Container Path:** /var/log/conpot/
- **Backup Schedule:** Daily at 2:00 AM (30-day retention)
- **Backup Location:** ~/backups/

### Access URLs

- **Web Interface:** http://134.209.179.94:8800
- **Shodan Lookup:** https://www.shodan.io/host/134.209.179.94

### Research Timeline

- **Deployment Start:** January 27, 2025
- **Expected First Index:** January 28-29, 2025 (24-48 hours)
- **Expected First Activity:** January 28-30, 2025 (24-72 hours)

### Monitoring Commands

```bash
# Check container status
docker ps

# View live logs
docker logs -f conpot

# Count total sessions
docker logs conpot | grep -i "new.*session" | wc -l

# View recent connections
docker logs conpot | grep -i "session" | tail -20

# Check log file size
du -sh ~/conpot-deployment/logs/

# Check disk usage
df -h
```

### Research Objectives

- Analyze attacker behavior targeting ICS/SCADA systems
- Document protocols and exploits attempted
- Identify geographic distribution of attacks
- Study attack patterns on industrial control systems
- Gather data for MSc Information Security thesis at Royal Holloway

### Notes

- First Shodan check scheduled for: January 29, 2025
- Monitor memory usage given 1GB constraint
- Consider upgrade to 2GB if sustained high activity observed