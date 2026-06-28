# Incident Investigation Guide - ELK SIEM SOC Lab

This guide shows how to investigate security incidents like a professional SOC analyst.

---

## Investigation Framework

### 5-Step Incident Investigation

```
1. ALERT TRIAGE
   ↓
2. EVIDENCE GATHERING
   ↓
3. THREAT ASSESSMENT
   ↓
4. TIMELINE ANALYSIS
   ↓
5. DOCUMENTATION & CLOSURE
```

---

## Step 1: Alert Triage

When an alert triggers, your first job is **determine if it's real**.

### In Kibana - Alerts Tab

1. **Go to**: Security → Alerts
2. **Find your alert** in the list
3. **Check basic info**:
   - Timestamp
   - Rule name
   - Severity
   - Status

### Questions to Ask

- **Is this alert real or false positive?**
  - Check if source IP is known/expected
  - Verify it's not internal testing
  
- **How severe is it?**
  - Critical = Confirmed attack (immediate action)
  - High = Suspicious activity (investigate now)
  - Medium = Potential threat (schedule investigation)
  - Low = Informational (log and monitor)

- **What triggered it?**
  - Read rule name
  - Check threshold
  - Review matching fields

### Example: SSH Brute Force Alert

```
Alert #12345
Rule: ssh bruteforce attempt ubuntu
Severity: Medium
Source IP: 81.163.20.41
Target: ubuntu user
Failed attempts: 8
Time: 2026-06-27 @ 19:30 UTC
```

**Decision**: ✅ This is REAL - 8 failed attempts is definitely brute force
**Action**: Proceed to evidence gathering

---

## Step 2: Evidence Gathering

### 2A: Click the Alert to View Details

In Kibana Alerts page, click your alert:

**Fields you'll see**:
```
source.ip: 81.163.20.41
source.geo.country_name: Netherlands
user.name: ubuntu
event.count: 8
@timestamp: 2026-06-27T19:30:45.000Z
host.name: ip-172-31-3-64
```

### 2B: Investigate in Discover

Click **Investigate** or go to **Discover** tab

**Run query to see ALL events from this source**:
```kql
source.ip: 81.163.20.41 and @timestamp: [now-24h TO now]
```

**What you're looking for**:
- How many failed attempts total?
- Are there ANY successful logins?
- Did attacker try other usernames?
- What time period was active?

### 2C: Check Authentication Timeline

Filter for all failed logins:
```kql
source.ip: 81.163.20.41 and system.auth.ssh.event: Failed
```

**Create table view**:
- Columns: @timestamp, user.name, source.ip, message
- Sort by: @timestamp ascending
- Look for patterns

**Example output**:
```
19:30:10  ubuntu   81.163.20.41  Failed password for ubuntu
19:30:15  ubuntu   81.163.20.41  Failed password for ubuntu
19:30:20  ubuntu   81.163.20.41  Failed password for ubuntu
19:30:25  ubuntu   81.163.20.41  Failed password for ubuntu
19:30:30  ubuntu   81.163.20.41  Failed password for ubuntu
19:30:35  ubuntu   81.163.20.41  Failed password for ubuntu
19:30:40  ubuntu   81.163.20.41  Failed password for ubuntu
19:30:45  ubuntu   81.163.20.41  Failed password for ubuntu
```

---

## Step 3: Threat Assessment

Now determine: **Did the attack succeed?**

### Check for Successful Login

Query:
```kql
source.ip: 81.163.20.41 and event.code: 4624
```

**If ZERO results**: ✅ Attack failed (no compromise)
**If ANY results**: 🔴 Attack succeeded (compromise confirmed)

### If Attack Succeeded - What Did They Do?

```kql
source.ip: 81.163.20.41 and @timestamp: [successful_login_time TO now]
```

Look for:
- [ ] Process execution (Event 1)
- [ ] Network connections (Event 3)
- [ ] File creation (Event 11)
- [ ] Registry modification (Event 13)
- [ ] Commands executed

### Compromise Timeline

If successful:
```
19:30:45 - Failed login attempt #8
19:31:00 - SUCCESSFUL LOGIN (event.code: 4624)
19:31:15 - cmd.exe launched (process execution)
19:31:30 - Downloaded malware file
19:31:45 - Established connection to C2 server
19:32:00 - Defender disabled
```

---

## Step 4: Timeline Analysis

### Create a Master Timeline

Use **Discover** with this query:
```kql
host.name: EC2AMAZ-QPF036S and @timestamp: [now-1h TO now]
| sort @timestamp asc
```

**Export to CSV and organize**:

| Time | Event | Details | Severity |
|------|-------|---------|----------|
| 19:30:10 | SSH Brute Force | Failed login attempt | Medium |
| 19:30:45 | SSH Brute Force | 8th failed attempt | Medium |
| 19:31:00 | SSH Success | Successful login | High |
| 19:31:15 | Process Created | cmd.exe started | Medium |
| 19:31:30 | File Download | malware.exe downloaded | High |
| 19:31:45 | Network Conn | Outbound to 192.168.1.100:4444 | Critical |
| 19:32:00 | Defender Disabled | Windows Defender stopped | Critical |

### Correlate Events

Look for patterns:
- Did brute force + successful login happen close together?
- What happened immediately after successful login?
- Are there multiple attack vectors used?

---

## Step 5: Documentation & Closure

### Document Your Findings

In osTicket ticket or notepad:

**INCIDENT REPORT**
```
Incident ID: #12345
Date Discovered: 2026-06-27 @ 19:30 UTC
Reporting System: ELK SIEM

SUMMARY:
SSH brute force attack from Netherlands (81.163.20.41)
Resulted in successful compromise of ubuntu account
Attacker downloaded and executed malware payload

TIMELINE:
19:30-19:31   SSH brute force (8 failed attempts)
19:31:00      Successful login as ubuntu
19:31:15      cmd.exe launched
19:31:30      Malware downloaded from attacker server
19:31:45      C2 connection established
19:32:00      Defender disabled

EVIDENCE:
- Failed login attempts: 8
- Source IP: 81.163.20.41 (Netherlands)
- Successful login time: 2026-06-27 19:31:00 UTC
- Malware hash: [hash from Sysmon]
- C2 server: 192.168.1.100:4444

IMPACT:
- Compromised: ubuntu user account
- Host affected: ip-172-31-3-64
- Data at risk: Unknown
- Lateral movement: Not detected

REMEDIATION:
- Reset ubuntu password
- Scan system for backdoors
- Block source IP 81.163.20.41
- Update SSH security rules
- Review other accounts for compromise

RECOMMENDATIONS:
1. Disable password auth, use SSH keys only
2. Implement fail2ban for brute force protection
3. Deploy MFA for critical accounts
4. Increase logging verbosity
5. Set up rate limiting on SSH port
```

### Close the Incident

1. **In osTicket**: Set ticket status to "Closed"
2. **Add resolution note**: Document the findings
3. **Link to documentation**: Add links to Kibana queries
4. **Lessons learned**: What would you do differently?

---

## Real-World Investigation Example

### Attack Simulation #1: SSH Brute Force

**Alert Triggered**:
```
Rule: ssh bruteforce attempt ubuntu
Time: 2026-06-27 10:53:26
Source: 81.163.20.41
Attempts: 6+
```

**Investigation Steps**:

1. **Triage**
   - 6 failed attempts = Definite brute force
   - Source IP in Netherlands (unusual for organization)
   - Decision: Real attack, investigate immediately

2. **Evidence**
   ```kql
   source.ip: 81.163.20.41 and user.name: ubuntu
   ```
   - Found 8 failed login attempts over 45 seconds
   - No successful login detected ✅

3. **Threat Assessment**
   - Attack failed (no successful login)
   - Risk level: Low (unsuccessful)
   - No further action needed beyond monitoring

4. **Timeline**
   ```
   19:30:10 - Failed attempt
   19:30:15 - Failed attempt
   ... (6 more)
   19:30:45 - Failed attempt #8
   19:31:00 - Attacker gave up
   ```

5. **Documentation**
   - Ticket #377131 created
   - Status: Resolved
   - Resolution: Attack unsuccessful, no compromise

---

### Attack Simulation #2: RDP Brute Force

**Alert Triggered**:
```
Rule: rdp bruteforce attempt administrator
Time: 2026-06-27 10:53:26
Attempts: 5+
```

**Key Finding**: This one SUCCEEDED (had valid credentials)

**Investigation**:

1. **Query for successful login**:
   ```kql
   source.ip: 81.163.20.41 and event.code: 4624
   ```
   ⚠️ FOUND SUCCESSFUL LOGIN

2. **What happened after login**:
   ```kql
   source.ip: 81.163.20.41 and @timestamp: [login_time TO now]
   ```
   
   Found:
   - PowerShell launched
   - System discovery commands run
   - whoami, ipconfig, net user commands executed

3. **Threat Level**: 🔴 HIGH - Confirmed compromise

4. **Actions**:
   - ✅ Elastic Defend isolated host
   - ✅ All further activity blocked
   - Incident contained

---

### Attack Simulation #3: C2 Malware

**Alert Triggered**:
```
Rule: apollo c2 detection
Time: 2026-06-27 10:58:26
Severity: Critical
File: zoro.exe
```

**Investigation**:

1. **Verify file execution**:
   ```kql
   event.code: 1 and image: *zoro.exe*
   ```
   ✅ Confirmed file executed

2. **Check network activity**:
   ```kql
   destination.ip: 13.61.205.240 and event.code: 3
   ```
   ✅ Found connection to C2 server

3. **Timeline**:
   ```
   10:58:26 - zoro.exe executed
   10:58:35 - Process injects into memory
   10:58:45 - DNS lookup to C2 domain
   10:59:00 - TCP connection established
   10:59:15 - First command received
   ```

4. **Elastic Defend Response**:
   - ✅ File blocked from execution
   - ✅ Host isolated automatically
   - ✅ No data exfiltration possible

5. **Conclusion**: Malware detected and contained

---

## Investigation Tools & Queries

### Discovery Queries

**Find all events from an IP**:
```kql
source.ip: 81.163.20.41
```

**Find all process executions on a host**:
```kql
host.name: EC2AMAZ-QPF036S and event.code: 1
```

**Find all network connections**:
```kql
event.code: 3 and destination.ip: 13.61.205.240
```

**Find failed authentications**:
```kql
event.code: 4625 or system.auth.ssh.event: Failed
```

**Find file creation by attacker**:
```kql
event.code: 11 and source.ip: 81.163.20.41
```

### Aggregation Queries

**Count failed logins by IP**:
```kql
system.auth.ssh.event: Failed | stats count() by source.ip
```

**Top attacked users**:
```kql
event.code: 4625 | stats count() by user.name
```

**Timeline of events**:
```kql
source.ip: 81.163.20.41 | stats count() by @timestamp
```

---

## Investigation Checklist

When investigating any alert:

- [ ] Verify alert is real (not false positive)
- [ ] Determine attack success (compromise confirmed?)
- [ ] Create complete timeline of events
- [ ] Identify all affected systems/users
- [ ] Document evidence with timestamps
- [ ] Determine attacker identity/origin (if possible)
- [ ] Assess business impact
- [ ] Determine containment needed
- [ ] Plan remediation steps
- [ ] Close ticket with full documentation

---

## Common Mistakes to Avoid

❌ **Assuming alert = real threat**
- Always verify in Discover first

❌ **Not checking for successful compromise**
- Just because login attempt doesn't mean it failed
- Always check for event code 4624 (success)

❌ **Ignoring post-exploitation activity**
- Attackers don't stop at just logging in
- Look for what they did AFTER gaining access

❌ **Inadequate documentation**
- You'll forget details by tomorrow
- Write down everything today

❌ **Not correlating events**
- Don't look at failed login in isolation
- Connect it to what happened afterward

---

## Escalation Procedures

When to escalate incident:

| Situation | Action |
|-----------|--------|
| Successful login + malware | Escalate to Security Team immediately |
| Multiple hosts compromised | Full incident response team |
| Data access suspected | Notify management + legal |
| Ransomware detected | Activate incident response playbook |
| C2 callback confirmed | Block all outbound traffic |

---

## Interview Talking Points

When discussing investigations:

> "When an alert fires, I first verify it's real by checking the actual logs in Kibana. Then I assess if the attack was successful - for brute force, I check if there's a successful authentication event. I create a timeline of all events to understand the attack progression. For the SSH brute force attack, I found 8 failed attempts but no successful login, so I documented it as an unsuccessful attack. For the C2 detection, I correlated process execution, network connections, and Elastic Defend alerts to confirm compromise, and the system was automatically isolated."

---

That's how you investigate incidents like a real SOC analyst! 🎯
