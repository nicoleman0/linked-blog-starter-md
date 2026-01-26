# DigitalOcean Conpot Deployment Guide

This guide will walk you through deploying Conpot honeypot on DigitalOcean, replicating the exact setup tested locally.

## Prerequisites

- DigitalOcean account
- SSH key configured
- Local terminal access

## Step 1: Create DigitalOcean Droplet

### Via Web Interface:

1. Log into DigitalOcean
2. Create Droplet
3. Choose image: **Ubuntu 22.04 LTS**
4. Choose size: **Basic - $6/month** (1GB RAM, 1 vCPU, 25GB SSD)
   - For production research: Consider $12/month (2GB RAM)
5. Choose datacenter region (pick one relevant to your research)
6. Add your SSH key
7. Hostname: `conpot-honeypot` (or your preference)
8. Click **Create Droplet**

### Via CLI (Optional):

```bash
# Install doctl if not already installed
# snap install doctl
# doctl auth init

# Create droplet
doctl compute droplet create conpot-honeypot \
  --image ubuntu-22-04-x64 \
  --size s-1vcpu-1gb \
  --region nyc1 \
  --ssh-keys YOUR_SSH_KEY_ID
```

Wait for the droplet to be created and note the IP address.

## Step 2: Initial Server Setup

SSH into your droplet:

```bash
ssh root@YOUR_DROPLET_IP
```

Update the system:

```bash
apt update && apt upgrade -y
```

Create a non-root user (recommended):

```bash
adduser conpot
usermod -aG sudo conpot
```

Configure firewall (UFW):

```bash
# Enable UFW
ufw allow OpenSSH
ufw allow 8800/tcp  # HTTP interface
ufw allow 102/tcp   # S7Comm
ufw allow 502/tcp   # Modbus
ufw allow 161/udp   # SNMP
ufw allow 47808/udp # BACnet
ufw allow 623/udp   # IPMI
ufw allow 21/tcp    # FTP
ufw allow 69/udp    # TFTP
ufw allow 44818/tcp # EtherNet/IP
ufw enable
```

## Step 3: Install Docker

Install Docker using the official script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

Add user to docker group (if using non-root):

```bash
usermod -aG docker conpot
```

Verify Docker installation:

```bash
docker --version
docker run hello-world
```

## Step 4: Pull Conpot Image

```bash
docker pull honeynet/conpot:latest
```

## Step 5: Create Deployment Structure

Create directories for logs and configuration:

```bash
mkdir -p ~/conpot-deployment/logs
cd ~/conpot-deployment
```

## Step 6: Run Conpot (Production)

### Quick Start (Testing):

Run Conpot interactively to verify it works:

```bash
docker run -d --rm \
  --name conpot-test \
  -p 8800:8800 \
  -p 102:10201 \
  -p 502:5020 \
  -p 161:16100/udp \
  -p 47808:47808/udp \
  -p 623:6230/udp \
  -p 21:2121 \
  -p 69:6969/udp \
  -p 44818:44818 \
  honeynet/conpot:latest
```

Check logs:

```bash
docker logs -f conpot-test
```

Test from your local machine:

```bash
curl http://YOUR_DROPLET_IP:8800
```

You should see a Siemens SIMATIC S7-200 web interface.

Stop test container:

```bash
docker stop conpot-test
```

### Production Deployment with Persistent Logs:

Create a production container with log persistence:

```bash
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

## Step 7: Verify Deployment

### Check Container Status:

```bash
docker ps
```

### View Real-time Logs:

```bash
docker logs -f conpot
```

### Check Persistent Logs:

```bash
ls -lh ~/conpot-deployment/logs/
tail -f ~/conpot-deployment/logs/conpot.log
```

### Test Services:

From your local machine:

```bash
# Test HTTP
curl http://YOUR_DROPLET_IP:8800

# Test with nmap
nmap -p 102,502,8800,21,161 YOUR_DROPLET_IP

# Test Modbus (if you have modbus tools)
# modpoll -m tcp -a 1 -r 1 -c 10 YOUR_DROPLET_IP -p 502
```

## Step 8: Monitoring and Maintenance

### View Active Connections:

```bash
docker logs conpot | grep "New.*session"
```

### Check Attack Logs:

```bash
grep -i "request\|session" ~/conpot-deployment/logs/conpot.log
```

### Container Management:

```bash
# Stop honeypot
docker stop conpot

# Start honeypot
docker start conpot

# Restart honeypot
docker restart conpot

# Remove container
docker rm -f conpot
```

### Backup Logs Daily:

Create a backup script:

```bash
cat > ~/backup-logs.sh << 'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d)
tar -czf ~/backups/conpot-logs-$DATE.tar.gz ~/conpot-deployment/logs/
find ~/backups/ -name "conpot-logs-*.tar.gz" -mtime +30 -delete
EOF

chmod +x ~/backup-logs.sh
mkdir -p ~/backups
```

Add to crontab:

```bash
crontab -e
# Add this line:
# 0 2 * * * /root/backup-logs.sh
```

## Step 9: Verify on Shodan

After 24-48 hours, check if your honeypot is indexed:

1. Go to [https://www.shodan.io](https://www.shodan.io)
2. Search: `ip:YOUR_DROPLET_IP`
3. You should see services detected (S7, Modbus, HTTP, etc.)

## Port Mapping Reference

| Service       | Host Port | Container Port | Protocol |
|---------------|-----------|----------------|----------|
| HTTP          | 8800      | 8800           | TCP      |
| S7Comm (PLC)  | 102       | 10201          | TCP      |
| Modbus        | 502       | 5020           | TCP      |
| SNMP          | 161       | 16100          | UDP      |
| BACnet        | 47808     | 47808          | UDP      |
| IPMI          | 623       | 6230           | UDP      |
| FTP           | 21        | 2121           | TCP      |
| TFTP          | 69        | 6969           | UDP      |
| EtherNet/IP   | 44818     | 44818          | TCP      |

## Security Considerations

1. **Isolation**: This VM should ONLY run the honeypot - no other services
2. **SSH Hardening**:
   - Disable password authentication
   - Use SSH keys only
   - Consider changing SSH port
3. **Monitoring**: Set up alerts for unusual activity
4. **Data Collection**: Logs contain attacker IPs and tactics - handle responsibly
5. **Legal**: Ensure honeypot deployment complies with your institution's policies

## Troubleshooting

### Container won't start:

```bash
# Check docker logs
docker logs conpot

# Check if ports are already in use
sudo netstat -tulpn | grep -E '(102|502|8800)'
```

### Can't access from outside:

```bash
# Verify firewall
sudo ufw status

# Check if container is running
docker ps

# Check container network
docker inspect conpot | grep IPAddress
```

### No attacks/connections:

- Wait 24-48 hours for Shodan/Censys to index
- Verify ports are open: `nmap -p 102,502,8800 YOUR_DROPLET_IP`
- Check firewall allows inbound traffic

### High CPU/Memory usage:

```bash
# Check container stats
docker stats conpot

# If needed, restart
docker restart conpot
```

## Next Steps

1. **Custom Templates**: Modify Conpot templates to simulate specific ICS environments
2. **Log Analysis**: Set up ELK stack or similar for log aggregation
3. **Alerting**: Configure alerts for specific attack patterns
4. **Research Data**: Begin analyzing attacker behavior patterns

## Useful Commands Cheat Sheet

```bash
# View live logs
docker logs -f conpot

# Check what's running
docker ps

# Restart honeypot
docker restart conpot

# View connections
docker logs conpot | grep session

# Download logs to local machine
scp -r root@YOUR_DROPLET_IP:~/conpot-deployment/logs ./local-logs/

# Check disk usage
du -sh ~/conpot-deployment/logs/
```

## Resources

- Conpot GitHub: https://github.com/mushorg/conpot
- Conpot Docker Hub: https://hub.docker.com/r/honeynet/conpot
- Shodan: https://www.shodan.io
- DigitalOcean Docs: https://docs.digitalocean.com
