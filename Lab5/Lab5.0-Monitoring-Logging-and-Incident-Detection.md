# IKB42603 Cloud Computing Security Essentials — Lab 5

**UniKL MIIT · Prof. Dr. Shahrulniza Musa**

## Monitoring, Logging & Incident Detection

Centralised logging, tamper-proof logs, threat detection and incident response using Docker and LocalStack.

**Name:** ALYA LIYANA BINTI MAHAT (L01-B01)  
**Student ID:** 52215124600

---

## Lab Learning Outcomes

At the end of this lab, you will be able to:

1. Collect and centralise logs from multiple services (cloud telemetry).
2. Distinguish logs from events and query logs for security-relevant activity.
3. Build a tamper-evident (hash-chained) log and detect alteration.
4. Detect an incident by correlating events, such as brute-force activity followed by a suspicious action.
5. Execute incident-response steps: detect, contain, collect evidence, and document a timeline.

## Course & Assessment Mapping

| Item | Mapping |
|---|---|
| Course Learning Outcome | CLO2 — Construct secure cloud operations that safeguard data integrity |
| Lecture Topics | Week 6 — Monitoring, Auditing & Management |
| Value / Skill Clusters | VBE3 (Integrity) · SC8 (Integrated Problem-Solving) |
| Assessment | Lab report + short incident report — contributes to the Lab Assignment |

## Lab Arrangement

| Session | Week | Focus |
|---|---|---|
| Session A | Week 9 | Generate and centralise logs; query for failed logins (Tasks 1–3) |
| Session B | Week 10 | Tamper-proof logs, incident detection and response (Tasks 4–6), then the incident report |

> **Note:** Session A builds visibility. Session B turns that visibility into detection and response — the "prevention eventually fails" half of security.

## Technical Prerequisites

- A laptop with Docker and a terminal.
- AWS CLI v2 pointed at LocalStack (as in Lab 1) — provides CloudWatch Logs.
- Standard shell tools: `grep`, `awk`, `sha256sum` (Git Bash / WSL on Windows).

> **Security tip:** You cannot secure — or prove compliance for — what you cannot see. Logs are foundational to detection, forensics, and compliance evidence.

---

# Session A — Week 9: Logging & Centralisation

## Setup — Start LocalStack

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack

EP='--endpoint-url=http://localhost:4566'

aws $EP logs create-log-group --log-group-name /ccse/app

aws $EP logs create-log-stream \
  --log-group-name /ccse/app \
  --log-stream-name auth
````

## Task 1 — Generate Application Logs

Create a small log of authentication events, including some failures representing an attacker probing the system.

```bash
cat > auth.log <<'EOF'
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5
2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9
2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB
EOF

cat auth.log
```

## Task 2 — Centralise Logs (Ship to CloudWatch)

Send each line to the central log service.

```bash
TS=$(date +%s000)

while IFS= read -r line; do
  aws $EP logs put-log-events \
    --log-group-name /ccse/app \
    --log-stream-name auth \
    --log-events timestamp=$TS,message="$line" >/dev/null

  TS=$((TS+1000))
done < auth.log
```

Read the logs back from the central store:

```bash
aws $EP logs get-log-events \
  --log-group-name /ccse/app \
  --log-stream-name auth \
  --query 'events[].message' \
  --output text
```

## Task 3 — Query for Security-Relevant Activity

Count failed logins and identify the IP address:

```bash
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

### Log vs Event

A **log** is a durable record of something that happened.

An **event** is a trigger or security-relevant occurrence that can cause an alert or response.

Example event:

```text
alert: 4 failures from 203.0.113.9
```

> **End of Session A:** Keep `auth.log` and the centralised read-back. Next week, these logs will be made tamper-proof and used to detect an incident.

---

# Session B — Week 10: Tamper-Proofing, Detection & Response

## Task 4 — Tamper-Proof (Hash-Chained) Logs

An attacker may try to edit logs to hide their actions. A hash chain links each line to the previous hash so that changes can be detected.

```bash
PREV=0

while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s\n' "$line" "$PREV"
done < auth.log > auth.chain

cat auth.chain
```

### Simulate Tampering

Change the export size from `500MB` to `5MB`:

```bash
sed 's/500MB/5MB/' auth.log > auth.tampered
```

Recompute the chain from `auth.tampered` and compare the final hash with the original `auth.chain`.

```bash
PREV=0
BROKE=no

paste -d'|' \
  <(cut -d'|' -f1 auth.chain) \
  <(cut -d'|' -f2 auth.chain) >/dev/null
```

A different final hash proves that tampering was detected.

> **Security tip:** Store the final hash, or forward the chain, to a separate append-only location so an attacker who owns the application cannot also rewrite its audit trail.

## Task 5 — Detect the Incident (Correlation)

No single log line was blocked, but together the entries tell a story.

The pattern is:

```text
Repeated failures
      ↓
Successful login
      ↓
Large data export
```

Use the following script:

```bash
IP=203.0.113.9

FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)
SUCCESS=$(grep -c "LOGIN_OK.*$IP" auth.log)
EXPORT=$(grep -c "EXPORT_DATA.*$IP" auth.log)

echo "IP=$IP fails=$FAILS success=$SUCCESS export=$EXPORT"

if [ "$FAILS" -ge 3 ] && [ "$SUCCESS" -ge 1 ] && [ "$EXPORT" -ge 1 ]; then
  echo 'ALERT: probable brute-force -> compromise -> data exfiltration'
fi
```

This demonstrates the idea of a **SIEM**, which correlates events across sources into a single detection that an individual log entry may not reveal.

## Task 6 — Incident Response

The response lifecycle includes containment, evidence collection, and documentation while preserving evidence integrity.

### 1. Contain

Block the attacker IP using an `iptables` rule:

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c \
  'apk add -q iptables; iptables -A INPUT -s 203.0.113.9 -j DROP; iptables -L INPUT -n | tail -2'
```

### 2. Collect Evidence

Create an immutable, timestamped evidence copy and calculate its hash:

```bash
cp auth.log evidence_$(date +%Y%m%d).log

sha256sum evidence_*.log > evidence.sha256

cat evidence.sha256
```

---

# Incident Report

## Detection

The incident was identified through the analysis of authentication logs and the correlation of several security-relevant actions from the same IP address, `203.0.113.9`.

The logs showed four failed login attempts, a successful login, and a 500 MB data export. This pattern triggered an alert for a likely brute-force attack followed by account compromise and data exfiltration.

## Analysis

Multiple failed login attempts indicate that the attacker was trying to access the admin account. A subsequent successful login from the same IP suggested that the account may have been compromised.

Shortly afterwards, there was a large 500 MB data export using the same account and IP address, which could indicate data exfiltration.

## Containment

The suspicious IP address, `203.0.113.9`, was contained by adding an `iptables` rule to drop incoming traffic from that IP address. This was done to prevent further activity from the suspected attacker.

## Evidence & Integrity

The original `auth.log` was copied to a timestamped evidence file, and the SHA-256 hash of the original log was recorded in `evidence.sha256`.

The log was also protected by a hash chain. When the export size was changed from 500 MB to 5 MB in the tampered copy, the final hash changed, demonstrating that the log had been modified.

## Lesson Learned

The incident demonstrates the importance of centralised and tamper-evident logging. Individual log entries may not clearly identify an attack, but correlating multiple activities can reveal a complete attack pattern.

Logs should also be protected so that attackers cannot modify evidence of their actions.

---
## Short-Answer Questions

### Q1. What is the difference between a log and an event? Give an example of each from this lab.

**Answer:**

A **log** is a recorded entry showing something that happened.

Example:

```text
LOGIN_FAIL user=admin ip=203.0.113.9
```

An **event** is a security-relevant occurrence or trigger that can cause an alert or response.

Example:

```text
ALERT: 4 failures from 203.0.113.9
```

### Q2. Why must audit logs be tamper-proof, and how does a hash chain achieve this?

**Answer:**

An attacker who compromises a system may try to modify or delete logs to hide their actions.

A hash chain helps detect this because each hash depends on the previous hash and the current log entry. Changing one entry produces a different hash and breaks the chain.

### Q3. How did correlation detect an incident that no single log line revealed?

**Answer:**

The incident was detected by correlating multiple activities from the same IP address.

Four failed login attempts were followed by a successful login and a large data export. This indicated a probable brute-force attack, account compromise, and possible data exfiltration.

### Q4. List the incident-response steps you performed and the goal of each.

**Answer:**

1. **Detect** — Identify the incident.
2. **Contain** — Stop further attacker activity.
3. **Collect** — Preserve investigation evidence.
4. **Verify integrity** — Detect modification.
5. **Document** — Record what happened and the response.

### Q5. How do the same logs serve both security monitoring and compliance evidence?

**Answer:**

The same logs can be used for security monitoring because they allow analysts to detect suspicious activities such as repeated failed logins and unusual exports.

They can also serve as compliance evidence because they provide a recorded history of activities that can be reviewed during audits.

Hashing and centralised storage help demonstrate that the evidence has integrity.

---

# Verification Commands

Check the LocalStack log groups:

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
```

Verify the evidence hash:

```bash
sha256sum -c evidence.sha256
```

---

# Security Best-Practices Checklist

* [x] Logs are centralised, not left scattered on each host.
* [x] Security-relevant activity such as failed logins can be queried.
* [x] Logs are tamper-evident using a hash chain and forwarded to a separate store.
* [x] An incident is detected by correlating multiple events.
* [x] Incident response is performed: contain, collect evidence, and document.

---

# Cleanup & Teardown

Remove the generated files:

```bash
rm -f auth.log auth.chain auth.tampered evidence_*.log evidence.sha256
```

Stop and remove LocalStack:

```bash
docker stop localstack && docker rm localstack
```