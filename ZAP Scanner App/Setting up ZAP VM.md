**VM Setup Checklist:**

**1. Create VM in virt-manager:**

```md
Name: zap-scanner (or whatever you prefer)
OS: Ubuntu 24.04 Server or Desktop
  - Server: lighter, headless-ready
  - Desktop: easier if you want GUI for initial testing
Memory: 4096 MB (4GB)
CPUs: 2
Disk: 40 GB
Network: Default (NAT) for now
```

**2. During Ubuntu installation:**

- Create a user (e.g., `scanner` or your preferred username)
- Enable OpenSSH server (makes it easier to work from your host)
```bash
sudo apt install openssh-server
```
- Minimal installation is fine

**3. First boot - get SSH access from your host:**

Inside the VM:

```bash
# Get the VM's IP address
ip addr show
# Look for the inet address on your main interface (probably enp1s0 or similar)
```

IP - 192.168.122.142

From the host laptop:
```bash
# SSH into the VM (easier than using virt-manager console)
ssh zapper@192.168.122.142

# If you want, set up key-based auth
ssh-copy-id zapper@192.168.122.142
```

**4. Install dependencies:**

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Java (required for ZAP)
sudo apt install openjdk-21-jdk wget curl -y

# Verify Java installation
java -version

# Optional but useful: install pip for Python testing
sudo apt install python3-pip python3-venv -y
```

**5. Install ZAP**
```bash
# Create directory for apps
mkdir -p ~/tools
cd ~/tools

# Download latest ZAP (v2.17.0 - January 2025 release)
wget https://github.com/zaproxy/zaproxy/releases/download/v2.17.0/ZAP_2.17.0_Linux.tar.gz

# Extract
tar -xvf ZAP_2.17.0_Linux.tar.gz

# Clean up
rm ZAP_2.17.0_Linux.tar.gz

# Create symlink for easier access (optional)
sudo ln -s ~/tools/ZAP_2.17.0/zap.sh /usr/local/bin/zap
```

**6. Test ZAP in daemon mode:**

```bash
# Start ZAP headless
cd ~/tools/ZAP_2.15.0
./zap.sh -daemon -host 0.0.0.0 -port 8080 -config api.disablekey=true

# Should see output like:
# INFO [main] ZAP is now listening on 0.0.0.0:8080
```

Keep that terminal open. From another SSH session or from the host:
```bash
# Test the API (from your EndeavourOS host)
curl http://VM_IP:8080/JSON/core/view/version/

# Should return JSON with ZAP version info
```

**7. Set up Python environment for testing:**

Back in the VM (or via SSH):

```bash
# Create project directory
mkdir -p ~/scanner-project
cd ~/scanner-project

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install ZAP Python API client
pip install python-owasp-zap-v2.4

# Create test script
cat > test_zap.py << 'EOF'
from zapv2 import ZAPv2
import time

# Connect to ZAP
zap = ZAPv2(proxies={'http': 'http://127.0.0.1:8080', 'https': 'http://127.0.0.1:8080'})

print(f"ZAP Version: {zap.core.version}")

# Test with a safe target
target = 'http://testphp.vulnweb.com'
print(f"\nStarting spider scan on: {target}")

zap.spider.scan(target)

# Monitor spider progress
while int(zap.spider.status()) < 100:
    status = zap.spider.status()
    print(f"Spider progress: {status}%")
    time.sleep(2)

print("\nSpider complete!")
print(f"URLs found: {len(zap.core.urls())}")
EOF

# Run test
python test_zap.py
```

**8. Make ZAP run as a service (optional, but useful):**

```bash
# Create systemd service file
sudo tee /etc/systemd/system/zap.service << 'EOF'
[Unit]
Description=OWASP ZAP Daemon
After=network.target

[Service]
Type=simple
User=YOUR_USERNAME
WorkingDirectory=/home/YOUR_USERNAME/tools/ZAP_2.15.0
ExecStart=/home/YOUR_USERNAME/tools/ZAP_2.15.0/zap.sh -daemon -host 0.0.0.0 -port 8080 -config api.disablekey=true
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# Replace YOUR_USERNAME with your actual username
sudo sed -i "s/YOUR_USERNAME/$(whoami)/g" /etc/systemd/system/zap.service

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable zap
sudo systemctl start zap

# Check status
sudo systemctl status zap
```

**9. Set up a test target:**

For safe testing, run OWASP Juice Shop in the same VM:

```bash
# Install Docker
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER

# Log out and back in for group membership, or:
newgrp docker

# Run Juice Shop
docker run -d -p 3000:3000 bkimminich/juice-shop

# Now you can scan http://localhost:3000 from ZAPc
```

**Next steps after basic setup:**

Once ZAP is responding to API calls, you'll want to:

1. Test different scan types (spider, active scan, passive scan)
2. Understand ZAP contexts and authentication
3. Start building your FastAPI wrapper
4. Set up proper API key authentication (replace `api.disablekey=true`)

**Troubleshooting tips:**

- If ZAP won't start: Check Java version with `java -version`
- If API isn't accessible: Check firewall with `sudo ufw status`
- If scans fail: Check ZAP logs in `~/.ZAP/zap.log`
- Memory issues: Monitor with `htop` during scans

Once ZAP responds to API calls, the next step is to build the Python automation layer.