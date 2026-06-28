# Architecture Guide - ELK SIEM SOC Lab

## High-Level Architecture

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
│                                   FLEET SERVER              │
│                                   (ELK Server)              │
│                                   Port 8220                 │
│                                          │                  │
│                                          ▼                  │
│                          ┌────────────────────────────────┐ │
│                          │   ELASTICSEARCH (Port 9200)    │ │
│                          │   Storage & Indexing           │ │
│                          │   8GB RAM                      │ │
│                          └────────────────────────────────┘ │
│                                          │                  │
│                                          ▼                  │
│                          ┌────────────────────────────────┐ │
│                          │   KIBANA (Port 5601)           │ │
│                          │   Dashboards & Visualization   │ │
│                          │   Detection Rules              │ │
│                          │   Alerting                     │ │
│                          └────────────────────────────────┘ │
│                                          │                  │
│                          Detection Rule Triggers            │
│                                          │                  │
│                                          ▼                  │
│                          ┌────────────────────────────────┐ │
│                          │   osTicket Server              │ │
│                          │   (Windows 2022 + XAMPP)       │ │
│                          │   Port 80 (HTTP)               │ │
│                          │   Ticket Management            │ │
│                          └────────────────────────────────┘ │
│                                                               │
│  LINUX AGENT                                                 │
│  (Ubuntu 24.04)                                              │
│  ├─ SSH Logs (system.auth)                                  │
│  ├─ System Logs                                             │
│  └─ Network Traffic                                         │
│         │                                                    │
│         └─────────► Fleet Server ► Elasticsearch             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Details

### 1. ELK Server (Central Hub)

**Role**: Central logging, analysis, and alerting platform

**Components**:
- **Elasticsearch**: Full-text search engine, stores all logs
- **Kibana**: Web UI for searching, analyzing, and visualizing logs
- **Fleet Server**: Endpoint management and agent enrollment

**Why Elastic Stack?**
- Industry standard SIEM platform
- Enterprise-grade security features
- Used by Fortune 500 companies
- Powerful KQL query language for threat hunting

**Key Files**:
```
/etc/elasticsearch/elasticsearch.yml    - Elasticsearch config
/etc/kibana/kibana.yml                  - Kibana config
/etc/systemd/system/elastic-agent.service - Fleet Server
```

---

### 2. Windows Victim (EC2AMAZ-QPF036S)

**Role**: Target machine for attack simulation and detection

**Installed Software**:
- **Elastic Agent**: Collects logs and sends to Fleet Server
- **Elastic Defend**: Endpoint protection (blocks malware)
- **Sysmon**: Advanced process monitoring

**Monitored Events**:
```
Event ID 1   - Process creation (zoro.exe execution)
Event ID 3   - Network connections (C2 callbacks)
Event ID 5001- Windows Defender disabled
Event ID 4624- Successful logon (RDP)
Event ID 4625- Failed logon (brute force)
```

**Attack Surface**:
- RDP (Port 3389) - Brute force target
- Network access - C2 callbacks
- File system - Malware execution

---

### 3. Linux Agent (Ubuntu)

**Role**: Diversified monitoring across operating systems

**Monitored Logs**:
- `/var/log/auth.log` - SSH authentication attempts
- `/var/log/syslog` - System events
- Network traffic - Firewall logs

**Integration**:
- Sends logs via Elastic Agent to Fleet Server
- Same detection rules apply across platforms

---

### 4. osTicket (Ticketing System)

**Role**: Incident management and ticketing

**Technology Stack**:
- Windows Server 2022 (Host OS)
- XAMPP (Apache + MySQL stack)
- PHP (Application)
- osTicket v1.18.3 (Ticketing software)

**Integration with SIEM**:
```
Detection Rule Triggered (Kibana)
         │
         ▼
   Create Alert
         │
         ▼
   HTTP Webhook POST
   (Kibana Connector)
         │
         ▼
   osTicket API Endpoint
   /api/tickets.xml
         │
         ▼
   New Ticket Created
   (Auto-assigned to Support)
```

**API Key Security**:
- Each IP address gets unique API key
- Keys are IP-restricted for security
- Prevents unauthorized ticket creation

---

### 5. Mythic C2 (Attacker Infrastructure)

**Role**: Simulates real C2 server for malware testing

**Components**:
- Mythic Framework (C2 platform)
- Apollo Agent (Windows malware payload)
- HTTP C2 Profile (Command channel)
- RabbitMQ (Message broker)

**Attack Flow**:
```
1. Generate zoro.exe payload (Apollo agent)
2. Host on HTTP server (python3 -m http.server 9999)
3. Victim downloads: Invoke-WebRequest
4. Victim executes: .\zoro.exe
5. Agent calls home to C2 (http://13.61.205.240:80)
6. SIEM detects via Sysmon events
7. Elastic Defend blocks execution
8. Alert triggers → Ticket created
9. Host isolation executed
```

---

## Data Flow

### Attack Simulation Flow

```
┌─────────────────────────┐
│  Attacker (Kali Linux)  │
│  Hydra brute force      │
│  C2 malware execution   │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Target (Windows)       │
│  • SSH/RDP connection   │
│  • Malware execution    │
│  • Network traffic      │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Windows Event Logs     │
│  Sysmon Events          │
│  • Process creation     │
│  • Network connections  │
│  • Registry changes     │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Elastic Agent (Windows)│
│  Collects & forwards    │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Fleet Server           │
│  (Port 8220)            │
│  Agent management       │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Elasticsearch          │
│  Indexes logs           │
│  (Port 9200)            │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Kibana                 │
│  Detects threat         │
│  (Detection Rules)      │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Alert Generated        │
│  Webhook triggered      │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  osTicket API           │
│  (Port 80)              │
│  Ticket created         │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Ticket #XXXXXX         │
│  In osTicket            │
│  Awaiting investigation │
└─────────────────────────┘
```

---

## Network Isolation & Security

### Security Group Rules (AWS)

**ELK Server**:
```
Inbound:
- Port 9200 (Elasticsearch) - From agents only
- Port 5601 (Kibana) - From your IP
- Port 8220 (Fleet Server) - From agent IPs

Outbound:
- All traffic allowed
```

**Windows Victim**:
```
Inbound:
- Port 3389 (RDP) - From your IP
- Port 22 (SSH) - From attacker IP (for simulation)

Outbound:
- All traffic allowed (for C2 callbacks)
```

**osTicket Server**:
```
Inbound:
- Port 80 (HTTP) - From Kibana server
- Port 3306 (MySQL) - From localhost only
- Port 3389 (RDP) - From your IP

Outbound:
- All traffic allowed
```

---

## Log Storage & Retention

### Elasticsearch Indexes

Logs are stored in daily indexes:
```
logs-system.auth-*              (SSH/Linux auth)
logs-windows.sysmon_operational (Windows Sysmon)
logs-endpoint.events.*          (Elastic Defend events)
```

**Default Retention**: 30 days
**Storage Size**: ~1GB per day (depends on volume)

### Index Management

In Kibana:
```
Stack Management → Index Management → Lifecycle Policies
```

---

## Agent Communication

### Agent → Fleet Server → Elasticsearch

1. **Agent Configuration**: Stored in Fleet policies
2. **Policy Update**: Fleet Server pushes configs every 30 seconds
3. **Log Forwarding**: Agent ships logs every 10 seconds
4. **Enrollment**: Agents authenticate with enrollment tokens

### Enrollment Flow

```
Agent starts
     │
     ▼
Request enrollment token
     │
     ▼
Contact Fleet Server (port 8220)
     │
     ▼
Exchange token for client cert
     │
     ▼
Get policy configuration
     │
     ▼
Start collecting logs
     │
     ▼
Forward to Elasticsearch
```

---

## Detection & Response Flow

### How Detection Works

1. **Rule Evaluation**
   - Every 1 minute, rules run
   - Query recent 1 minute of logs
   - Match against conditions (threshold, fields, etc.)

2. **Alert Triggering**
   - If match found → Alert created
   - Alert details stored in Elasticsearch
   - Webhook POSTs to osTicket

3. **Automated Response**
   - Elastic Defend isolated host
   - Network connectivity blocked
   - Investigation begins

---

## Scalability Considerations

**Current Setup**: Single-node lab
- 1 Elasticsearch node
- 1 Kibana instance
- 1 Fleet Server
- 2-3 Elastic Agents

**Production Setup** would include:
- Elasticsearch cluster (3+ nodes)
- Kibana dedicated node
- Fleet Server cluster
- Hundreds of agents
- SOAR platform (Elasticsearch SOAR)
- Centralized alerting (PagerDuty, etc.)

---

## Key Architecture Decisions

| Decision | Why | Trade-off |
|----------|-----|-----------|
| Single-node Elasticsearch | Simplicity, learning | Not highly available |
| m7i.large ELK instance | Enough CPU/RAM | Higher AWS cost |
| Elastic IPs on all instances | Prevents IP drift | Small monthly cost |
| osTicket on separate instance | Isolation, scalability | More instances to manage |
| Mythic in separate AWS account | Security isolation | More complex setup |

---

## Monitoring the Lab

### Key Metrics to Watch

**Elasticsearch Health**:
```
Kibana → Stack Management → Stack Health
```

**Agent Status**:
```
Fleet → Agents
All should show "Healthy" status
```

**Rule Execution**:
```
Security → Rules → Detection rules
Check execution status in "Execution results" tab
```

**osTicket Tickets**:
```
Admin Panel → Dashboard
Shows new tickets, trending issues
```

---

## Disaster Recovery

### Backup Strategy

**Elasticsearch Snapshots**:
```bash
# In Kibana Dev Tools:
PUT /_snapshot/my-backup
{
  "type": "fs",
  "settings": {
    "location": "/mnt/backup"
  }
}
```

**Configuration Backup**:
```bash
# Backup configs
tar -czf elasticsearch-config.tar.gz /etc/elasticsearch/
tar -czf kibana-config.tar.gz /etc/kibana/
```

### Recovery Procedure

If Elasticsearch fails:
1. Restore from snapshot
2. Restart Elasticsearch
3. Verify agent connections
4. Resume investigations

---

## Performance Tuning

### Common Issues & Solutions

**Slow Kibana**: Increase Kibana heap memory
```bash
nano /etc/kibana/kibana.yml
Add: NODE_OPTIONS="--max-old-space-size=2048"
```

**High CPU**: Reduce rule execution frequency
```
Rules → Edit rule → Schedule to 5 minutes instead of 1
```

**Disk Full**: Delete old indexes
```
Index Management → Delete old indexes
```

---

This architecture provides a **solid foundation for SOC operations** while remaining cost-effective for learning and demonstrations.
