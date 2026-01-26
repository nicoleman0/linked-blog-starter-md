## Overview

This guide covers how to customize your **Dockerized Conpot deployment** to make it more realistic and harder to fingerprint as a honeypot. Unlike traditional installations, this focuses on Docker-specific workflows for modifying templates while maintaining container best practices.

**Your Current Setup:**
- **Deployment:** DigitalOcean Droplet (134.209.179.94)
- **Container:** honeynet/conpot:latest (Conpot 0.6.0)
- **Template:** default (Technodrome)
- **Location:** London datacenter
- **Log Directory:** ~/conpot-deployment/logs/

## Why Customize Your Docker Deployment?

- **Attackers can fingerprint default Conpot** through timing analysis, response patterns, and protocol deviations
- **Default templates are well-known** and easily identified by sophisticated threat actors
- **Realistic honeypots collect better intelligence** by engaging attackers longer
- **Research value increases** with more authentic interactions targeting your specific ICS simulation

---

## Docker-Specific Approach

Unlike traditional installations where you edit files directly, Docker deployments require:

1. **Extract template from container**
2. **Modify template locally on host**
3. **Mount customized template as volume**
4. **Restart container with new configuration**

This approach keeps your customizations persistent and version-controlled.

---

## Step 0: Understand Your Docker Setup

Your current deployment uses the container's built-in template. To customize, you'll create a local copy and mount it.

```bash
# SSH into your droplet
ssh root@134.209.179.94

# Verify current setup
cd ~/conpot-deployment
docker ps
docker logs conpot | head -20
```

---

## Step 1: Extract the Default Template

### 1.1 Create Local Template Directory

```bash
cd ~/conpot-deployment
mkdir -p templates/custom
```

### 1.2 Copy Template from Running Container

```bash
# Find the template location inside the container
docker exec conpot find / -name "templates" -path "*/conpot/*" 2>/dev/null

# Based on your container, it should be:
# /home/conpot/.local/lib/python3.6/site-packages/conpot-0.6.0-py3.6.egg/conpot/templates

# Copy the default template out
docker cp conpot:/home/conpot/.local/lib/python3.6/site-packages/conpot-0.6.0-py3.6.egg/conpot/templates/default ./templates/custom/
```

### 1.3 Verify Extraction

```bash
ls -la ~/conpot-deployment/templates/custom/default/
```

You should see:
- `conpot.cfg` (main configuration)
- `modbus/` (Modbus protocol data)
- `s7comm/` (S7 protocol data)
- `snmp/` (SNMP configuration)
- `http/` (web interface templates)

---

## Step 2: Backup Before Modifications

```bash
# Create timestamped backup
cd ~/conpot-deployment/templates
cp -r custom/default custom/default.backup.$(date +%Y%m%d)

# Verify backup
ls -la custom/
```

---

## Step 3: Quick Wins - High-Impact Customizations

### 3.1 Device Identity (5 minutes)

Edit the main configuration:

```bash
cd ~/conpot-deployment/templates/custom/default
nano conpot.cfg
```

**Critical changes:**

```ini
[device]
# Change from generic Technodrome to specific BMS
device_name = SIEMENS-BMS-LON-01
device_id = S7-1200/CPU 1214C DC/DC/DC

[s7comm]
system_name = SIMATIC S7
module_type_name = CPU 1214C DC/DC/DC

# Change serial number to realistic but fake value
# Real format: S C-X4U304536019
serial_number = S C-L8N472841053

# Add location context
plant_identification = London Data Center - Building HVAC Control

# Match firmware to your Shodan research findings
firmware_version = V4.4.0

# System location matching your droplet
location = London, UK - Slough Region
```

**Why these values?**
- S7-1200 is a widely deployed modern Siemens PLC (not discontinued)
- CPU 1214C is a common model in building automation
- Serial format matches real Siemens pattern
- Firmware V4.4.0 is current and appears in Shodan results

### 3.2 Add Response Delays (3 minutes)

Real PLCs have network latency and processing delays. Add to `conpot.cfg`:

```ini
[modbus]
# Add realistic response timing
response_delay = 0.05,0.15    # Random 50-150ms delay
timeout = 5.0

[s7comm]
response_delay = 0.08,0.12    # S7 tends to be faster than Modbus
timeout = 10.0

[http]
response_delay = 0.02,0.08
```

**Impact:** This single change makes timing-based fingerprinting significantly harder.

### 3.3 Customize SNMP (5 minutes)

```bash
nano snmp/snmp.cfg
```

Update to match your London deployment:

```ini
[snmp]
# Siemens enterprise OID
enterprise_oid = 1.3.6.1.4.1.4329

# Realistic system description
system_description = Siemens SIMATIC S7-1200, CPU 1214C DC/DC/DC, Firmware V4.4.0

# Match your actual location
system_location = London, Slough Region, UK - Data Center HVAC Room 3F

# Plausible contact (not your real email)
system_contact = facilities@londondc-ops.local

# Match device_name from conpot.cfg
system_name = SIEMENS-BMS-LON-01

# Realistic uptime (~120 days suggests stable production system)
system_uptime = 10368000
```

---

## Step 4: Protocol Data Customization

### 4.1 Realistic Modbus Registers (10 minutes)

```bash
nano modbus/modbus.xml
```

Replace default with realistic HVAC data:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<modbus>
    <slave id="1">
        <!-- Holding Registers: Active sensor readings -->
        <holding_registers>
            <!-- Zone Temperature Sensors (×10 for 0.1°C precision) -->
            <value address="40001">218</value>  <!-- 21.8°C - Zone 1 -->
            <value address="40002">215</value>  <!-- 21.5°C - Zone 2 -->
            <value address="40003">220</value>  <!-- 22.0°C - Zone 3 -->
            <value address="40004">203</value>  <!-- 20.3°C - Return Air -->
            
            <!-- Humidity Sensors (% RH) -->
            <value address="40010">58</value>   <!-- 58% RH - Zone 1 -->
            <value address="40011">62</value>   <!-- 62% RH - Zone 2 -->
            <value address="40012">60</value>   <!-- 60% RH - Zone 3 -->
            
            <!-- Pressure and Air Quality -->
            <value address="40020">1012</value> <!-- Atmospheric pressure (mbar) -->
            <value address="40021">685</value>  <!-- CO2 level (ppm) -->
            <value address="40022">450</value>  <!-- VOC level (ppb) -->
            
            <!-- Control Setpoints -->
            <value address="40100">210</value>  <!-- Temperature setpoint 21.0°C -->
            <value address="40101">60</value>   <!-- Humidity setpoint 60% RH -->
            <value address="40102">1000</value> <!-- CO2 limit (ppm) -->
            
            <!-- Valve/Damper Positions (0-100%) -->
            <value address="40200">42</value>   <!-- Heating valve position -->
            <value address="40201">0</value>    <!-- Cooling valve position -->
            <value address="40202">65</value>   <!-- Fresh air damper -->
            <value address="40203">35</value>   <!-- Return air damper -->
            
            <!-- Fan Speed Control (0-100%) -->
            <value address="40210">75</value>   <!-- Supply fan speed -->
            <value address="40211">70</value>   <!-- Return fan speed -->
        </holding_registers>
        
        <!-- Coils: Equipment Binary Status -->
        <coils>
            <value address="1">1</value>   <!-- AHU-1 Running -->
            <value address="2">0</value>   <!-- General Alarm Active -->
            <value address="3">1</value>   <!-- Supply Fan Status -->
            <value address="4">1</value>   <!-- Return Fan Status -->
            <value address="5">0</value>   <!-- Maintenance Mode -->
            <value address="6">0</value>   <!-- Winter Mode (0=winter, 1=summer) -->
            <value address="7">1</value>   <!-- Auto Mode Enabled -->
            <value address="8">0</value>   <!-- Night Setback Active -->
            <value address="9">1</value>   <!-- Communications OK -->
            <value address="10">0</value>  <!-- Filter Replacement Due -->
        </coils>
        
        <!-- Input Registers: Calculated/Accumulated Values -->
        <input_registers>
            <value address="30001">14562</value>  <!-- Total energy consumption (kWh) -->
            <value address="30002">21840</value>  <!-- Total runtime hours -->
            <value address="30003">97</value>     <!-- System efficiency (%) -->
            <value address="30004">285</value>    <!-- Days since last maintenance -->
            <value address="30005">42</value>     <!-- Current power draw (kW) -->
        </input_registers>
        
        <!-- Discrete Inputs: Read-only Binary Sensors -->
        <discrete_inputs>
            <value address="10001">0</value>  <!-- Filter differential pressure alarm -->
            <value address="10002">1</value>  <!-- Mains power present -->
            <value address="10003">0</value>  <!-- High temperature alarm -->
            <value address="10004">0</value>  <!-- Low temperature alarm -->
            <value address="10005">1</value>  <!-- Network communications OK -->
            <value address="10006">0</value>  <!-- Freeze protection active -->
            <value address="10007">0</value>  <!-- Fire alarm input -->
            <value address="10008">1</value>  <!-- Occupied mode sensor -->
        </discrete_inputs>
    </slave>
</modbus>
```

**Register Address Scheme:**
- **40001-40099:** Real-time sensor values
- **40100-40199:** Setpoints and configuration
- **40200-40299:** Output controls (valves, dampers, VFDs)
- **Coils 1-99:** Equipment binary status flags
- **30001+:** Accumulated/calculated values
- **10001+:** Read-only digital inputs

### 4.2 Update S7comm Data Blocks (10 minutes)

```bash
nano s7comm/s7comm.xml
```

Create realistic S7 memory structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<s7comm>
    <!-- Data Block 1: Real-time Process Values -->
    <datablock id="1" size="256">
        <!-- Temperature Values (REAL = 4 bytes, IEEE 754 float) -->
        <value address="0" type="REAL">21.8</value>   <!-- Zone 1 temp °C -->
        <value address="4" type="REAL">21.5</value>   <!-- Zone 2 temp °C -->
        <value address="8" type="REAL">22.0</value>   <!-- Zone 3 temp °C -->
        <value address="12" type="REAL">20.3</value>  <!-- Return air temp °C -->
        <value address="16" type="REAL">15.2</value>  <!-- Outside air temp °C -->
        
        <!-- Humidity Values (INT = 2 bytes, percentage) -->
        <value address="24" type="INT">58</value>     <!-- Zone 1 RH% -->
        <value address="26" type="INT">62</value>     <!-- Zone 2 RH% -->
        <value address="28" type="INT">60</value>     <!-- Zone 3 RH% -->
        
        <!-- Air Quality (INT = 2 bytes) -->
        <value address="32" type="INT">685</value>    <!-- CO2 ppm -->
        <value address="34" type="INT">450</value>    <!-- VOC ppb -->
        
        <!-- Control Setpoints (REAL) -->
        <value address="40" type="REAL">21.0</value>  <!-- Temperature setpoint -->
        <value address="44" type="INT">60</value>     <!-- Humidity setpoint -->
        
        <!-- Valve Positions (BYTE = 1 byte, 0-100%) -->
        <value address="50" type="BYTE">42</value>    <!-- Heating valve -->
        <value address="51" type="BYTE">0</value>     <!-- Cooling valve -->
        <value address="52" type="BYTE">65</value>    <!-- Fresh air damper -->
        
        <!-- Status Bits (BOOL) -->
        <value address="60" type="BOOL">true</value>  <!-- AHU running -->
        <value address="61" type="BOOL">false</value> <!-- Alarm active -->
        <value address="62" type="BOOL">true</value>  <!-- Auto mode -->
        <value address="63" type="BOOL">true</value>  <!-- Supply fan OK -->
        <value address="64" type="BOOL">false</value> <!-- Maintenance mode -->
    </datablock>
    
    <!-- Data Block 2: System Configuration -->
    <datablock id="2" size="128">
        <value address="0" type="STRING">HVAC-ZONE-NORTH-LON</value>
        <value address="32" type="STRING">London Data Center</value>
        <value address="64" type="STRING">Building 1 - Floor 3</value>
        <value address="96" type="INT">1</value>      <!-- Zone ID -->
        <value address="98" type="INT">3</value>      <!-- Floor number -->
    </datablock>
    
    <!-- Data Block 3: Alarm History -->
    <datablock id="3" size="128">
        <!-- Current Alarms (BOOL) -->
        <value address="0" type="BOOL">false</value>  <!-- High temperature -->
        <value address="1" type="BOOL">false</value>  <!-- Low temperature -->
        <value address="2" type="BOOL">false</value>  <!-- Filter alarm -->
        <value address="3" type="BOOL">false</value>  <!-- Communication loss -->
        <value address="4" type="BOOL">false</value>  <!-- VFD fault -->
        <value address="5" type="BOOL">false</value>  <!-- Freeze protection -->
        
        <!-- Alarm Counters (INT) -->
        <value address="16" type="INT">3</value>      <!-- Alarms this month -->
        <value address="18" type="INT">42</value>     <!-- Alarms this year -->
        <value address="20" type="INT">285</value>    <!-- Days since last critical alarm -->
    </datablock>
    
    <!-- Data Block 4: Energy Monitoring -->
    <datablock id="4" size="64">
        <value address="0" type="REAL">14.562</value>  <!-- Total MWh -->
        <value address="4" type="REAL">42.5</value>    <!-- Current kW -->
        <value address="8" type="INT">21840</value>    <!-- Runtime hours -->
        <value address="10" type="INT">97</value>      <!-- Efficiency % -->
    </datablock>
</s7comm>
```

---

## Step 5: Restart with Custom Template

### 5.1 Stop Current Container

```bash
docker stop conpot
docker rm conpot
```

### 5.2 Launch with Mounted Custom Template

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
  -v ~/conpot-deployment/templates/custom/default:/home/conpot/.local/lib/python3.6/site-packages/conpot-0.6.0-py3.6.egg/conpot/templates/default \
  honeynet/conpot:latest
```

**Key difference:** The second `-v` flag mounts your customized template over the container's default.

### 5.3 Verify Container Started

```bash
# Check container is running
docker ps

# Check logs for errors
docker logs conpot | tail -30

# Look for successful service initialization
docker logs conpot | grep -i "starting\|listening\|serving"
```

---

## Step 6: Testing Your Customizations

### 6.1 Local Testing (from the droplet)

```bash
# Test S7comm device identity
nmap --script s7-info 127.0.0.1 -p 102

# Should show: SIEMENS-BMS-LON-01, S7-1200, firmware V4.4.0

# Test Modbus register (requires modpoll)
# If not installed: apt install libmodbus-dev -y
# Then build modpoll or use python-pymodbus

# Test SNMP
snmpwalk -v2c -c public 127.0.0.1 1.3.6.1.2.1.1

# Should show London location and SIEMENS-BMS-LON-01 name

# Test HTTP interface
curl -s http://127.0.0.1:8800 | grep -i "simatic\|siemens\|technodrome"
```

### 6.2 External Testing (from your local machine)

```bash
# From your laptop/desktop:

# Scan exposed services
nmap -sV -p 102,502,8800,161 134.209.179.94

# Check HTTP interface
curl -I http://134.209.179.94:8800

# Test with browser
# Open: http://134.209.179.94:8800
```

### 6.3 Verify Timing Delays

```bash
# From local machine, time multiple Modbus requests
# If you have modpoll installed locally:

time modpoll -m tcp -a 1 -r 40001 -c 1 134.209.179.94 -p 502
time modpoll -m tcp -a 1 -r 40001 -c 1 134.209.179.94 -p 502
time modpoll -m tcp -a 1 -r 40001 -c 1 134.209.179.94 -p 502

# Response times should vary between 50-150ms due to response_delay setting
```

---

## Step 7: Adding Dynamic Data (Optional - Advanced)

Static sensor values are unrealistic. For true production-grade deception, add variation.

### 7.1 Create Data Generator Script

```bash
cd ~/conpot-deployment
nano update-sensor-data.py
```

```python
#!/usr/bin/env python3
"""
Dynamic sensor data generator for Conpot honeypot
Updates Modbus registers and S7 data blocks with realistic variations
"""

import xml.etree.ElementTree as ET
import random
import time
import math
from datetime import datetime

MODBUS_XML = '/root/conpot-deployment/templates/custom/default/modbus/modbus.xml'
S7_XML = '/root/conpot-deployment/templates/custom/default/s7comm/s7comm.xml'

def update_modbus_registers():
    """Update Modbus registers with time-varying values"""
    tree = ET.parse(MODBUS_XML)
    root = tree.getroot()
    
    # Current hour affects heating/cooling
    hour = datetime.now().hour
    is_night = hour < 6 or hour > 22
    
    # Find holding registers
    for slave in root.findall('.//slave[@id="1"]'):
        for hr in slave.findall('.//holding_registers/value'):
            addr = int(hr.get('address'))
            current_val = int(hr.text)
            
            # Temperature registers (40001-40004) - vary ±0.5°C
            if 40001 <= addr <= 40004:
                base_temp = 215 if not is_night else 210  # Night setback
                new_val = base_temp + random.randint(-5, 5)
                hr.text = str(new_val)
            
            # Humidity (40010-40012) - vary ±3%
            elif 40010 <= addr <= 40012:
                new_val = max(45, min(75, current_val + random.randint(-3, 3)))
                hr.text = str(new_val)
            
            # CO2 (40021) - occupied hours = higher CO2
            elif addr == 40021:
                if 8 <= hour <= 18:  # Business hours
                    new_val = random.randint(650, 850)
                else:
                    new_val = random.randint(400, 550)
                hr.text = str(new_val)
            
            # Valve positions (40200-40203) - respond to conditions
            elif 40200 <= addr <= 40203:
                # Smooth changes, not jumps
                delta = random.randint(-5, 5)
                new_val = max(0, min(100, current_val + delta))
                hr.text = str(new_val)
            
            # Fan speeds (40210-40211) - vary slightly
            elif 40210 <= addr <= 40211:
                new_val = max(50, min(100, current_val + random.randint(-2, 2)))
                hr.text = str(new_val)
    
    tree.write(MODBUS_XML)
    print(f"[{datetime.now()}] Updated Modbus registers")

def update_s7_datablocks():
    """Update S7comm data blocks"""
    tree = ET.parse(S7_XML)
    root = tree.getroot()
    
    for db in root.findall('.//datablock[@id="1"]'):
        for val in db.findall('value'):
            addr = int(val.get('address'))
            val_type = val.get('type')
            
            # Temperature REAL values (addresses 0, 4, 8, 12)
            if val_type == 'REAL' and addr < 20:
                current = float(val.text)
                # Vary ±0.3°C
                new_val = current + random.uniform(-0.3, 0.3)
                val.text = f"{new_val:.1f}"
            
            # Humidity INT values
            elif val_type == 'INT' and 24 <= addr <= 28:
                current = int(val.text)
                new_val = max(45, min(75, current + random.randint(-2, 2)))
                val.text = str(new_val)
    
    tree.write(S7_XML)
    print(f"[{datetime.now()}] Updated S7 data blocks")

def main():
    """Main loop - update every 60 seconds"""
    print(f"[{datetime.now()}] Starting Conpot data generator")
    
    while True:
        try:
            update_modbus_registers()
            update_s7_datablocks()
            
            # Signal Docker container to reload
            # Not needed - Conpot will read on next request
            
            time.sleep(60)  # Update every minute
            
        except KeyboardInterrupt:
            print("\n[{datetime.now()}] Shutting down data generator")
            break
        except Exception as e:
            print(f"[{datetime.now()}] Error: {e}")
            time.sleep(60)

if __name__ == '__main__':
    main()
```

```bash
chmod +x update-sensor-data.py
```

### 7.2 Create Systemd Service for Data Generator

```bash
nano /etc/systemd/system/conpot-data-generator.service
```

```ini
[Unit]
Description=Conpot Dynamic Data Generator
After=docker.service
Requires=docker.service

[Service]
Type=simple
User=root
WorkingDirectory=/root/conpot-deployment
ExecStart=/usr/bin/python3 /root/conpot-deployment/update-sensor-data.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
systemctl daemon-reload
systemctl enable conpot-data-generator
systemctl start conpot-data-generator

# Check status
systemctl status conpot-data-generator
```

**Note:** The data generator modifies the mounted template files directly. Conpot reads these on each protocol request, so changes take effect immediately without container restart.

---

## Step 8: Web Interface Customization (Optional)

### 8.1 Modify HTTP Templates

```bash
cd ~/conpot-deployment/templates/custom/default/http/templates
ls -la
```

The default template likely has basic HTML. You can customize it to look more like a real Siemens interface, but this is time-intensive and low-priority compared to protocol-level authenticity.

**Quick win:** Edit any `index.html` to replace "Technodrome" with your device name:

```bash
# If an index.html exists:
sed -i 's/Technodrome/SIEMENS-BMS-LON-01/g' index.html
```

Then restart the container to pick up changes:

```bash
docker restart conpot
```

---

## Step 9: Verification and Monitoring

### 9.1 Complete System Check

```bash
# Ensure all services running
docker ps
systemctl status conpot-data-generator

# Check logs
docker logs conpot | tail -50

# Verify customizations took effect
docker logs conpot | grep -i "siemens\|s7-1200\|lon-01"

# Monitor live activity
docker logs -f conpot
```

### 9.2 External Verification

From your local machine:

```bash
# Comprehensive port scan
nmap -sV -sC -p 21,69,102,161,502,623,8800,44818,47808 134.209.179.94

# S7 fingerprint
nmap --script s7-info 134.209.179.94 -p 102

# Should show your customized identity
```

### 9.3 Shodan Verification (after 48 hours)

```bash
# Check if indexed (from any machine with browser)
https://www.shodan.io/host/134.209.179.94

# Search for your system
https://www.shodan.io/search?query=SIEMENS-BMS-LON-01
```

---

## Customization Checklist

### Quick Wins (30 minutes)
- [ ] Device identity changed in `conpot.cfg`
- [ ] Serial number updated to realistic format
- [ ] Response delays added to all protocols
- [ ] SNMP information matches London location
- [ ] Firmware version matches current S7-1200 releases

### Protocol Realism (30 minutes)
- [ ] Modbus registers updated with HVAC values
- [ ] Realistic register address ranges (40001-40299)
- [ ] S7comm data blocks configured
- [ ] Data types match real Siemens PLCs (REAL, INT, BOOL)

### Dynamic Behavior (20 minutes) - Optional
- [ ] Data generator script created
- [ ] Systemd service configured
- [ ] Script tested and running
- [ ] Values verified to be changing

### Docker Integration (15 minutes)
- [ ] Custom template extracted from container
- [ ] Backup created
- [ ] Volume mount configured correctly
- [ ] Container restarts with custom template

### Testing (20 minutes)
- [ ] Local protocol tests passed
- [ ] External nmap scan successful
- [ ] Timing delays verified
- [ ] Logs show custom device name
- [ ] HTTP interface accessible

**Total time: ~2 hours for complete customization**

---

## Common Issues and Solutions

### Issue: Container won't start after mounting custom template

**Symptom:**
```bash
docker ps
# Container not listed
docker logs conpot
# Shows errors or permission denied
```

**Solution:**
```bash
# Check file permissions
ls -la ~/conpot-deployment/templates/custom/default/

# Fix permissions if needed
chmod -R 755 ~/conpot-deployment/templates/custom/

# Validate XML syntax
xmllint --noout ~/conpot-deployment/templates/custom/default/modbus/modbus.xml
xmllint --noout ~/conpot-deployment/templates/custom/default/s7comm/s7comm.xml

# Check for typos in volume mount path
docker inspect conpot | grep -A 5 Mounts
```

### Issue: Changes not taking effect

**Symptom:** Testing shows old values (Technodrome, default serial number, etc.)

**Solution:**
```bash
# Verify you're editing the mounted template
ls -la ~/conpot-deployment/templates/custom/default/conpot.cfg

# Verify mount is correct in running container
docker inspect conpot | grep -A 10 Mounts

# Completely recreate container
docker stop conpot
docker rm conpot
# Run the docker run command from Step 5.2 again

# Check logs for which template was loaded
docker logs conpot | grep -i template
```

### Issue: Data generator not updating values

**Symptom:**
```bash
systemctl status conpot-data-generator
# Shows "active (running)" but values stay static
```

**Solution:**
```bash
# Check generator logs
journalctl -u conpot-data-generator -f

# Common issue: Python script has syntax error
python3 -m py_compile update-sensor-data.py

# Common issue: File paths wrong in script
# Verify paths in script match your deployment:
nano update-sensor-data.py
# Check MODBUS_XML and S7_XML paths

# Test script manually
cd ~/conpot-deployment
python3 update-sensor-data.py
# Let it run for 2-3 minutes, then Ctrl+C
```

### Issue: XML syntax errors

**Symptom:**
```bash
docker logs conpot
# Shows XML parsing errors
```

**Solution:**
```bash
# Validate all XML files
xmllint --noout ~/conpot-deployment/templates/custom/default/modbus/modbus.xml
xmllint --noout ~/conpot-deployment/templates/custom/default/s7comm/s7comm.xml

# Common mistakes:
# - Unclosed tags: <value address="40001">215  # Missing </value>
# - Invalid characters in values
# - Duplicate addresses
# - Address not in quotes: address=40001 instead of address="40001"
```

### Issue: Volume mount permission denied

**Symptom:**
```bash
docker logs conpot
# PermissionError: [Errno 13] Permission denied
```

**Solution:**
```bash
# Fix template directory permissions
chmod -R 755 ~/conpot-deployment/templates/

# If that doesn't work, try 777 (less secure but functional for honeypot)
chmod -R 777 ~/conpot-deployment/templates/

# Restart container
docker restart conpot
```

---

## Maintenance and Updates

### Daily Checks

```bash
# Quick health check script
cat > ~/conpot-health-check.sh << 'EOF'
#!/bin/bash
echo "=== Conpot Health Check ==="
echo "Date: $(date)"
echo ""
echo "Container Status:"
docker ps | grep conpot
echo ""
echo "Data Generator:"
systemctl is-active conpot-data-generator
echo ""
echo "Recent Activity (last 10 log lines):"
docker logs --tail 10 conpot
echo ""
echo "Disk Usage:"
du -sh ~/conpot-deployment/logs/
df -h | grep -E 'Filesystem|/$'
EOF

chmod +x ~/conpot-health-check.sh

# Run daily health check
./conpot-health-check.sh
```

### Weekly Review

```bash
# Analyze a week of activity
cd ~/conpot-deployment/logs
grep -c "session" conpot.log  # Count total sessions

# Extract unique attacker IPs
grep "from" conpot.log | awk '{print $NF}' | sort -u > weekly-ips.txt
wc -l weekly-ips.txt

# Most targeted protocols
echo "Modbus requests: $(grep -c 'modbus' conpot.log)"
echo "S7comm requests: $(grep -c 's7' conpot.log)"
echo "HTTP requests: $(grep -c 'GET\|POST' conpot.log)"
```

### Template Version Control (Recommended)

```bash
# Initialize git repo for templates
cd ~/conpot-deployment/templates/custom
git init
git add .
git commit -m "Initial customized template - London deployment"

# After making changes
git diff  # See what changed
git add .
git commit -m "Updated HVAC temperature ranges for realism"

# Rollback if needed
git log  # See commit history
git checkout <commit-hash> -- default/modbus/modbus.xml
```

---

## Research Integration

### Logging Attacker Activity

Your customizations will attract more sophisticated attackers who verify system authenticity before attacking. Document this:

```bash
# Extract detailed session logs
cd ~/conpot-deployment/logs
grep "New session" conpot.log > sessions.log

# Analyze by protocol
grep "modbus.*session" conpot.log | wc -l
grep "s7.*session" conpot.log | wc -l

# Geographic distribution (requires GeoIP lookup)
grep "from" conpot.log | awk '{print $NF}' | sort -u > attacker-ips.txt
```

### Comparing Against Real Deployments

Your Shodan research on real exposed Siemens PLCs provides a baseline:

```bash
# Document in your research notes:
# - Real PLC firmware versions observed
# - Common register ranges in production systems
# - Typical response timing patterns
# - Geographic distribution of real vs honeypot attacks
```

### MSc Thesis Data Points

- **Authenticity impact:** Do customized honeypots receive different attack patterns?
- **Dwell time:** Do attackers spend more time on realistic systems?
- **Attack sophistication:** Are more advanced techniques used against seemingly real targets?
- **Protocol preferences:** Which industrial protocols attract the most attention?

---

## Quick Reference Commands

```bash
# View live Docker logs
docker logs -f conpot

# Restart with new configuration
docker stop conpot && docker rm conpot
# Then run full docker run command from Step 5.2

# Edit main config
nano ~/conpot-deployment/templates/custom/default/conpot.cfg

# Edit Modbus data
nano ~/conpot-deployment/templates/custom/default/modbus/modbus.xml

# Edit S7 data
nano ~/conpot-deployment/templates/custom/default/s7comm/s7comm.xml

# Check container status
docker ps
docker inspect conpot | grep -A 10 Mounts

# Test from local machine
nmap --script s7-info 134.209.179.94 -p 102
curl http://134.209.179.94:8800

# Backup current template
tar -czf ~/conpot-template-backup-$(date +%Y%m%d).tar.gz ~/conpot-deployment/templates/

# Check data generator
systemctl status conpot-data-generator
journalctl -u conpot-data-generator -f
```

---

## Security Reminders

1. **Isolation:** This honeypot should only run on this dedicated droplet
2. **SSH hardening:** Ensure SSH key authentication is configured, consider changing port
3. **Firewall:** UFW should allow only necessary honeypot ports + SSH
4. **No production data:** Never store or process real operational data on this system
5. **Monitoring:** Set up alerts for unusual SSH login attempts or resource exhaustion

---

## Next Steps

1. **Let it run:** 48-72 hours minimum before evaluating results
2. **Monitor Shodan:** Check for indexing after 2 days
3. **Analyze patterns:** Document attacker behavior for your MSc research
4. **Iterate:** Refine based on what you observe
5. **Consider expansion:** Deploy additional instances with different templates/locations for comparison

### Recommended Reading for Further Customization

- Siemens S7-1200 System Manual (TIA Portal)
- Modbus Application Protocol Specification V1.1b3
- S7comm Protocol Analysis (Wireshark dissector documentation)
- "ICS/SCADA Security: Protecting Critical Infrastructure" (research papers)
- Real Shodan results for S7-1200 devices in Europe

---

## Support

If issues arise:

1. Check container logs: `docker logs conpot`
2. Validate XML: `xmllint --noout <file>`
3. Check GitHub issues: https://github.com/mushorg/conpot/issues
4. Review Docker mounts: `docker inspect conpot`
5. Test locally before external exposure

---

**Deployment Details:**
- **Server:** 134.209.179.94 (DigitalOcean London)
- **Container:** honeynet/conpot:latest (Conpot 0.6.0)
- **Template:** Custom (based on default, Siemens S7-1200 themed)
- **Deployment Date:** January 27, 2025
- **Research:** MSc Information Security, Royal Holloway

---

**Last Updated:** January 2025  
**Version:** 3.0 (Docker-Specific Deployment)  
**Status:** Production - 134.209.179.94
