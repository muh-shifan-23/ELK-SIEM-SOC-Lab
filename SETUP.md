# Setup Guide - ELK SIEM SOC Lab

This guide will help you set up the entire SOC lab on AWS.

## Prerequisites
- AWS Account with EC2 access
- Basic Linux/Windows command line knowledge
- ~30GB storage for Elasticsearch
- Patience (setup takes 2-3 hours)

---

## Step 1: Launch EC2 Instances

### Instance 1: ELK Server (Elasticsearch + Kibana)
```
AMI: Ubuntu 24.04 LTS
Type: m7i.large (8GB RAM)
Storage: 30GB
Security Group: Allow 9200, 5601, 8220 (inbound)
Elastic IP: Assign one
```

### Instance 2: Windows Victim
```
AMI: Windows Server 2022 Base
Type: t3.micro
Storage: 30GB
Security Group: Allow 3389 (RDP), 22 (SSH)
Elastic IP: Assign one
```

### Instance 3: osTicket Server
```
AMI: Windows Server 2022 Base
Type: t3.micro
Storage: 20GB
Security Group: Allow 80 (HTTP), 3306 (MySQL)
Elastic IP: Assign one
```

### Instance 4: Linux Agent
```
AMI: Ubuntu 24.04 LTS
Type: t2.micro
Storage: 10GB
Security Group: Allow 22 (SSH)
Elastic IP: Optional
```

### Instance 5: Mythic C2 (Optional - can be different AWS account)
```
AMI: Ubuntu 24.04 LTS
Type: t2.micro
Storage: 10GB
Security Group: Allow 7443, 80
Elastic IP: Assign one
```

---

## Step 2: Install Elasticsearch & Kibana (ELK Server)

SSH into your ELK server:
```bash
ssh -i your-key.pem ubuntu@ELK-IP
```

### Add Elastic Repository
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
```

### Install Elasticsearch
```bash
sudo apt update
sudo apt install elasticsearch -y
```

### Configure Elasticsearch
```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Add these lines:
```yaml
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
xpack.security.enabled: true
```

### Start Elasticsearch
```bash
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
sudo systemctl status elasticsearch
```

Wait 30 seconds for startup to complete.

### Install Kibana
```bash
sudo apt install kibana -y
```

### Configure Kibana
```bash
sudo nano /etc/kibana/kibana.yml
```

Add these lines:
```yaml
server.host: "0.0.0.0"
server.port: 5601
server.publicBaseUrl: "http://YOUR-ELK-IP:5601"
elasticsearch.hosts: ["http://localhost:9200"]
```

### Start Kibana
```bash
sudo systemctl start kibana
sudo systemctl enable kibana
```

### Access Kibana
Open browser: `http://YOUR-ELK-IP:5601`

---

## Step 3: Install Fleet Server (Separate Ubuntu Instance)

SSH into your Fleet Server instance:
```bash
ssh -i your-key.pem ubuntu@FLEET-SERVER-IP
```

### Download Elastic Agent
```bash
cd ~
wget https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.4.2-linux-x86_64.tar.gz
tar xzf elastic-agent-9.4.2-linux-x86_64.tar.gz
cd elastic-agent-9.4.2-linux-x86_64
```

### In Kibana, Generate Enrollment Token

On your ELK server, go to Kibana:
```
1. Go to Management → Fleet → Fleet Servers
2. Click "Add Fleet Server"
3. Follow the instructions to get enrollment token and service token
```

### Install Fleet Server

Run on Fleet Server instance:
```bash
sudo ./elastic-agent install --url=https://FLEET-SERVER-IP:8220 \
  --enrollment-token=YOUR-ENROLLMENT-TOKEN \
  --insecure \
  --fleet-server-es=https://ELK-SERVER-IP:9200 \
  --fleet-server-service-token=YOUR-SERVICE-TOKEN \
  -f
```

Verify:
```bash
sudo systemctl status elastic-agent
```

**Note**: Port 8220 must be open in Security Group for agents to connect

---

## Step 4: Enroll Agents to Fleet Server

### Windows Victim - RDP into Windows machine
```
Use public IP with your .pem key to get password
```

### Download Elastic Agent (Windows)
```powershell
# On your Windows machine, download from Elastic:
# https://www.elastic.co/downloads/elastic-agent

# Or use PowerShell:
Invoke-WebRequest -Uri https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.4.2-windows-x86_64.zip -OutFile agent.zip
Expand-Archive agent.zip -DestinationPath "C:\Program Files\Elastic\"
cd "C:\Program Files\Elastic\Agent"
```

### Enroll to Fleet Server
```powershell
.\elastic-agent.exe enroll --url=https://FLEET-SERVER-IP:8220 `
  --enrollment-token=YOUR-TOKEN `
  --insecure `
  --force

Restart-Service "Elastic Agent"
```

### Verify in Kibana
```
Fleet → Agents
Should show: EC2AMAZ-XXX (Healthy)
```

### Install Elastic Defend
```
In Kibana:
1. Integrations → Elastic Defend
2. Add Elastic Defend
3. Select Windows Policy
4. Deploy
```

Wait 2-3 minutes for deployment.

---

## Step 5: Install Sysmon (Windows)

### Download Sysmon
```powershell
cd C:\
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile Sysmon.zip
Expand-Archive Sysmon.zip -DestinationPath C:\Sysmon
```

### Download Config
```powershell
cd C:\Sysmon
Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile sysmonconfig.xml
```

### Install
```powershell
.\sysmon.exe -i sysmonconfig.xml -accepteula
```

Verify:
```powershell
Get-Service Sysmon | select status
```

---

## Step 6: Install osTicket (osTicket Server)

### RDP into osTicket Windows machine

### Download XAMPP
```
From: https://www.apachefriends.org/
Download XAMPP for Windows
Run installer (default installation)
```

### Configure Apache
```powershell
notepad C:\xampp\apache\conf\httpd.conf
```

Find and change:
```
Listen 80
to
Listen 0.0.0.0:80
```

Save and restart Apache from XAMPP Control Panel.

### Set MySQL Password
```powershell
cd C:\xampp\mysql\bin
.\mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'osticket123';"
```

### Create osTicket Database
```powershell
.\mysql -u root -posticket123
CREATE DATABASE osticket;
EXIT;
```

### Download & Extract osTicket
```powershell
cd C:\xampp\htdocs

# Download from: https://osticket.com/download/
# Extract to: C:\xampp\htdocs\osticket\
```

### Run Installer
```
Open browser: http://localhost/osticket/upload/setup/install.php

Fill in:
- MySQL Host: localhost
- MySQL Database: osticket
- MySQL Username: root
- MySQL Password: osticket123
- Admin Email: your-email@example.com
- Admin Username: shifan
- Admin Password: Your-Secure-Password

Click: Install Now
```

### Access osTicket
```
Admin Panel: http://localhost/osticket/upload/scp
Staff Panel: http://localhost/osticket/upload
```

### Create API Key
```
In Admin Panel:
1. Manage → API Keys
2. Click "Add New API Key"
3. IP Address: YOUR-KIBANA-IP
4. Create
5. Copy the key
```

---

## Step 7: Enroll Linux Agent

### SSH into Linux machine
```bash
ssh -i your-key.pem ubuntu@LINUX-IP
```

### Download Elastic Agent
```bash
cd ~
wget https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.4.2-linux-x86_64.tar.gz
tar xzf elastic-agent-9.4.2-linux-x86_64.tar.gz
cd elastic-agent-9.4.2-linux-x86_64
```

### Enroll to Fleet Server
```bash
sudo ./elastic-agent enroll https://FLEET-SERVER-IP:8220 \
  --enrollment-token=YOUR-TOKEN \
  --insecure

sudo systemctl restart elastic-agent
```

---

## Step 8: Verify Everything is Running

### In Kibana:
```
1. Fleet → Agents
   ✅ All agents should show "Healthy"

2. Integrations → System
   ✅ Should see Windows and Linux agents

3. Discover
   ✅ Should see logs coming in
```

### From Terminal:
```bash
# Check Elasticsearch
curl http://YOUR-ELK-IP:9200

# Check Kibana
curl http://YOUR-ELK-IP:5601
```

---

## Troubleshooting

**Agent shows "Offline"**
- Restart agent: `sudo systemctl restart elastic-agent`
- Check Fleet Server URL in policy
- Verify port 8220 is open

**No logs in Discover**
- Wait 2-3 minutes after agent enrollment
- Check agent is "Healthy" status
- Verify integrations are deployed

**Elasticsearch won't start**
- Check disk space: `df -h`
- Check memory: `free -h`
- View logs: `sudo journalctl -u elasticsearch`

**osTicket 404 error**
- Verify Apache is running in XAMPP Control Panel
- Check MySQL password is correct
- Verify database `osticket` exists

---

## Next Steps
1. Create detection rules (see RULES.md)
2. Set up osTicket webhook (see README.md)
3. Run attack simulations
4. Investigate incidents

**Total Setup Time: 1-2 hours**
