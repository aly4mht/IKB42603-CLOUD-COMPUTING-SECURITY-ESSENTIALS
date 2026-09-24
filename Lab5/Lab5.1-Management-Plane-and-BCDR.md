# IKB42603 Cloud Computing Security Essentials — Lab 5 Addendum
**Name:** ALYA LIYANA BINTI MAHAT  
**Student ID:** 52215124600

## Management Plane Audit, Backup & the Restore Drill

> Who changed the cloud itself, and can you actually get the data back?

This lab focuses on reconstructing the audit trail, versioning, backup, and timed recovery using LocalStack.

---

### 1. Management Plane Audit Trail

The management plane audit trail records **who reconfigured the cloud itself**, rather than what happened inside a workload.

Examples:

- Deleting a bucket
- Disabling a key
- Attaching an administrator policy

These actions may leave no application log.

### 2. Tested Recovery

Backup is a claim; restore is a measurement.

Until you have timed a restore, you do not know your **Recovery Time Objective (RTO)**. You only have an aspiration.

> **Note:** This addendum is supervised homework. Complete Tasks A1–A4, bring your evidence and measured numbers to the debrief session, and be prepared to defend your RTO and RPO figures.

---

# Course & Assessment Mapping

| Item | Mapping |
|---|---|
| Extends | Lab 5 — Monitoring, Logging & Incident Detection (Weeks 9–10) |
| Course Learning Outcome | CLO2 — Construct secure cloud operations, extended to resilience and recovery (VBE3) |
| Lecture Topics | Week 2 — Security Design & Architecture (management plane) · Week 6 — Monitoring, Auditing & Management |
| CSA CCSK v5 Domains | Domain 6 — Security Monitoring · Domain 11 — Incident Response & Resilience |
| Assessment | Evidence + measured RTO/RPO submitted with the Lab 5 report and discussed at the debrief |

---

# Before You Start

Remove any existing LocalStack container:

```bash
docker rm -f localstack 2>/dev/null
````

Start LocalStack with API request logging enabled:

```bash
docker run -d --name localstack -p 4566:4566 \
  -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN \
  -e DEBUG=1 \
  localstack/localstack-pro:latest
```

Wait until the LocalStack services are ready:

```bash
until curl -sf http://localhost:4566/_localstack/health >/dev/null; do
  sleep 2
done
```

Set the LocalStack endpoint:

```bash
export EP='--endpoint-url=http://localhost:4566'
```

Verify the AWS identity:

```bash
aws $EP sts get-caller-identity
```

---

# Task A1 — Reconstruct the Management Plane Audit Trail

Every administrative API call is a **management plane event**.

Examples include:

* `Create`
* `Delete`
* `PutPolicy`
* `ScheduleKeyDeletion`

In production, a trail records these events to object storage so that the record can survive tampering with the account being audited.

In this task, the trail is reconstructed from LocalStack's own request log and then protected using a cryptographic digest.

## Why Not CloudTrail?

CloudTrail is not included in the LocalStack licence used in this course.

Therefore, the lab reconstructs the same type of trail from the platform's own request log.

The mechanism built in this task is similar to what CloudTrail automates.

---

## Create the Audit Trail Store

The audit trail needs its own storage location.

In production, this bucket would ideally be located in a **different account**, creating a separate trust boundary from the account being audited.

Create the audit bucket:

```bash
aws $EP s3api create-bucket --bucket miit-audit-trail
```

Enable versioning:

```bash
aws $EP s3api put-bucket-versioning \
  --bucket miit-audit-trail \
  --versioning-configuration Status=Enabled
```

Record the current LocalStack log size:

```bash
BEFORE=$(docker logs localstack 2>&1 | wc -l)

echo "baseline: $BEFORE lines"
```

---

## Generate Administrative Activity

Generate administrative actions that change the cloud management plane:

```bash
aws $EP s3api create-bucket --bucket miit-throwaway
```

Create a temporary IAM user:

```bash
aws $EP iam create-user --user-name TempContractor
```

Attach AdministratorAccess:

```bash
aws $EP iam attach-user-policy \
  --user-name TempContractor \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Delete the temporary bucket:

```bash
aws $EP s3api delete-bucket --bucket miit-throwaway
```

---

## Extract the Management Plane Events

Extract the AWS API calls recorded after the baseline:

```bash
docker logs localstack 2>&1 | tail -n +$((BEFORE+1)) \
  | grep -E 'AWS [a-z0-9-]+\.[A-Za-z]+ => ' > mgmt-trail.log
```

Count the records:

```bash
wc -l mgmt-trail.log
```

Filter for security-relevant management plane actions:

```bash
grep -E '\.(CreateUser|AttachUserPolicy|DeleteUser|CreateBucket|DeleteBucket|PutBucketPolicy|ScheduleKeyDeletion) =>' mgmt-trail.log
```

If `mgmt-trail.log` is empty, restart LocalStack using:

```bash
-e LS_LOG=trace
```

instead of:

```bash
-e DEBUG=1
```

Then repeat the activity and extraction steps.

---

# Seal the Management Trail

The management trail must be protected against tampering.

This uses the same basic idea as the hash chain from Lab 5 Task 4.

Calculate the SHA-256 digest:

```bash
sha256sum mgmt-trail.log > mgmt-trail.sha256
```

Display the digest:

```bash
cat mgmt-trail.sha256
```

> **Important:** The digest should be stored in the audit store rather than beside the log. Otherwise, an attacker who modifies the log could also modify the digest.

---

## Upload the Trail to the Audit Store

Upload the management trail:

```bash
aws $EP s3 cp mgmt-trail.log s3://miit-audit-trail/
```

Upload the SHA-256 digest:

```bash
aws $EP s3 cp mgmt-trail.sha256 s3://miit-audit-trail/
```

---

## Simulate Tampering

Remove the `AttachUserPolicy` event to simulate an attacker hiding a privilege escalation:

```bash
grep -v 'AttachUserPolicy' mgmt-trail.log > t.log && mv t.log mgmt-trail.log
```

Download the original digest from the audit store:

```bash
aws $EP s3 cp \
  s3://miit-audit-trail/mgmt-trail.sha256 \
  ./check.sha256
```

Verify the modified trail:

```bash
sha256sum -c check.sha256
```

Expected result:

```text
mgmt-trail.log: FAILED
```

This demonstrates that the alteration was detected.

---

# Real CloudTrail Record

The reconstructed trail proves that an API call happened, but it does not show all the information available in a real CloudTrail record.

A real CloudTrail record contains fields such as:

```json
{
  "eventVersion": "1.09",
  "userIdentity": {
    "type": "IAMUser",
    "arn": "arn:aws:iam::123456789012:user/j.tan",
    "accountId": "123456789012",
    "userName": "j.tan",
    "sessionContext": {
      "mfaAuthenticated": "false"
    }
  },
  "eventTime": "2026-03-01T09:01:40Z",
  "eventSource": "iam.amazonaws.com",
  "eventName": "AttachUserPolicy",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.9",
  "userAgent": "aws-cli/2.15.0 Python/3.11 Linux/6.5",
  "requestParameters": {
    "userName": "TempContractor",
    "policyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
  },
  "errorCode": null,
  "readOnly": false,
  "managementEvent": true,
  "eventID": "1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d"
}
```

### Important Fields

The real record provides information that the reconstructed trail does not, such as:

* **User identity** — identifies the account or IAM identity responsible.
* **Event time** — establishes when the action happened.
* **Source IP address** — identifies where the request originated.
* **Request parameters** — shows exactly what resource or policy was affected.

The `sourceIPAddress` can also connect a management-plane investigation with another investigation, such as the brute-force activity from Lab 5.

---

# Management Plane vs Application Logs

Application logs and management plane logs provide different visibility.

For example:

```text
Application Logs
       ↓
Login attempts
User activity
Application requests
       ↓
Workload
```

Whereas:

```text
Management Plane Logs
       ↓
IAM changes
Bucket creation/deletion
Policy changes
Key deletion
       ↓
Cloud infrastructure
```

An action such as attaching `AdministratorAccess` may not produce any application traffic or application log.

Therefore:

> The management plane requires separate telemetry collection, storage, and alerting.

---

# Task A2 — Backup and Why Versioning Is Not One

Set up a primary storage bucket and a separate backup destination.

Create the primary bucket:

```bash
aws $EP s3api create-bucket --bucket miit-primary
```

Create the disaster-recovery backup bucket:

```bash
aws $EP s3api create-bucket --bucket miit-dr-backup
```

Enable versioning on the primary bucket:

```bash
aws $EP s3api put-bucket-versioning \
  --bucket miit-primary \
  --versioning-configuration Status=Enabled
```

Enable versioning on the backup bucket:

```bash
aws $EP s3api put-bucket-versioning \
  --bucket miit-dr-backup \
  --versioning-configuration Status=Enabled
```

---

## Create Test Data

Create 200 records:

```bash
for i in $(seq 1 200); do
  echo "patient record $i - $(date)" > /tmp/rec$i.txt
done
```

Upload them to the primary bucket:

```bash
aws $EP s3 sync /tmp/ s3://miit-primary/records/ \
  --exclude '*' \
  --include 'rec*.txt'
```

Count the objects:

```bash
aws $EP s3 ls s3://miit-primary/records/ | wc -l
```

Expected result:

```text
200
```

---

## Take the Backup

Synchronise the primary bucket to the separate backup bucket:

```bash
aws $EP s3 sync s3://miit-primary s3://miit-dr-backup
```

Verify the backup:

```bash
aws $EP s3 ls s3://miit-dr-backup/records/ | wc -l
```

Expected result:

```text
200
```

---

# Versioning vs Backup

Versioning is **not the same as backup**.

Versioning protects object versions inside the same bucket.

It does not necessarily survive:

* Bucket deletion
* Account compromise
* An attacker with `s3:DeleteObjectVersion`

A backup is a copy stored in a **different trust boundary**.

### Key Difference

```text
Versioning
    ↓
Same bucket
    ↓
Previous object versions


Separate Backup
    ↓
Different bucket
    ↓
Separate recovery location
```

---

# Task A3 — The Restore Drill

Simulate a destructive incident and recover the data.

The measured restore time becomes the **measured RTO** for this dataset size.

## Simulate the Incident

Delete all objects from the primary bucket:

```bash
aws $EP s3 rm s3://miit-primary/records/ --recursive
```

Check the number of remaining objects:

```bash
aws $EP s3 ls s3://miit-primary/records/ | wc -l
```

Expected result:

```text
0
```

---

## Perform the Recovery

Start the timer:

```bash
START=$(date +%s)
```

Restore the data from the backup bucket:

```bash
aws $EP s3 sync s3://miit-dr-backup s3://miit-primary
```

Stop the timer:

```bash
END=$(date +%s)
```

Verify the restored objects:

```bash
echo "Objects restored: $(aws $EP s3 ls s3://miit-primary/records/ | wc -l)"
```

Calculate the measured RTO:

```bash
echo "MEASURED RTO (seconds): $((END - START))"
```

---

# RTO and RPO Results

Record the following values:

| Measure          | How You Obtain It                                            |              Value |
| ---------------- | ------------------------------------------------------------ | -----------------: |
| Measured RTO     | Elapsed seconds for restoring 200 objects                    |      **3 seconds** |
| Extrapolated RTO | Scale measurement to 1,000,000 objects and state assumptions | **15,000 seconds** |
| RPO              | Time between last S3 sync and the incident                   |      **0 seconds** |

### Extrapolated RTO

```text
15,000 seconds
≈ 4.17 hours
```

> **Important:** The extrapolation assumes recovery time scales linearly. This is an assumption and may not hold in a real production environment.

A real environment may have:

* Larger datasets
* Network delays
* Encryption overhead
* Dependencies
* Recovery procedures
* Different storage throughput

---

# Understanding RTO and RPO

## RTO — Recovery Time Objective

RTO is the amount of time required to restore the service or data after an incident.

```text
Incident
   ↓
Recovery starts
   ↓
Data/service restored
```

### In this lab:

```text
Measured RTO = 3 seconds
```

for 200 objects.

---

## RPO — Recovery Point Objective

RPO represents the amount of data or time that could be lost between the last backup and the incident.

```text
Last Backup
     ↓
     ↓  Possible data loss
     ↓
  Incident
```

### In this lab:

```text
RPO = 0 seconds
```

because the backup was taken immediately before the simulated incident.

### Key Difference

> **More frequent backups improve RPO.**

> **Faster restoration improves RTO.**

---

# Task A4 — Compare the Two Recovery Paths

There are two recovery methods:

1. Versioning within the primary bucket.
2. Restoring from the separate backup bucket.

They are not equivalent.

---

## Path 1 — Versioning

The delete operation creates delete markers while previous object versions remain in the bucket.

Check delete markers:

```bash
aws $EP s3api list-object-versions \
  --bucket miit-primary \
  --prefix records/ \
  --query 'length(DeleteMarkers)'
```

Check object versions:

```bash
aws $EP s3api list-object-versions \
  --bucket miit-primary \
  --prefix records/ \
  --query 'length(Versions)'
```

---

## Path 2 — Separate Backup

The second recovery path is restoring from the separate backup bucket:

```bash
aws $EP s3 sync s3://miit-dr-backup s3://miit-primary
```

---

# Recovery Path Comparison

| Factor                                     | Versioning (In-Place)                                                                                               | Separate Backup Bucket                                                                                            |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Recovery speed                             | Faster because previous versions can be recovered within the same bucket.                                           | Slower because data must be restored/copied from the backup bucket.                                               |
| Survives bucket deletion?                  | **No.** Versions are stored in the same bucket.                                                                     | **Yes**, if the separate backup bucket is not deleted.                                                            |
| Survives compromised admin credentials?    | No / less resilient. An attacker with sufficient admin privileges may modify or delete the bucket and its versions. | Better protection, especially if the backup is separately protected from the compromised credentials.             |
| Survives cryptographic erasure of KMS key? | No, if the stored versions depend on the deleted KMS key.                                                           | Depends on the backup's encryption and KMS key. It survives only if the backup does not depend on the erased key. |
| Cost profile                               | Lower additional cost because storage is within the existing bucket, although previous versions consume storage.    | Higher cost because an additional bucket and duplicate data require additional storage and management.            |

---

# Short-Answer Questions

## Q1. Name three administrative actions that would appear in a management plane trail but produce no application log at all. For each, state what an attacker gains by performing it.

### Answer

### 1. Create an IAM User

An attacker gains a new identity that can be used to access cloud resources.

### 2. Attach AdministratorAccess Policy

The attacker gains full administrative privileges and can control cloud resources.

### 3. Delete a Bucket

The attacker can destroy data, disrupt services, and potentially remove evidence.

These actions affect the cloud management plane, so they may not appear in the application's own logs.

---

## Q2. CloudTrail log file validation, the digest sealed in Task A1, and the hash chain from Lab 5 Task 4 all solve the same problem. Explain the mechanism and why the digest must be stored in a different trust boundary.

### Answer

CloudTrail validation, the A1 digest, and the Lab 5 hash chain all use **cryptographic hashing**.

A hash is calculated from the original log data and stored separately.

If the log is changed, its new hash will not match the original hash, revealing that tampering occurred.

The digest must be stored in a different trust boundary because an attacker who can modify the audited account could otherwise modify both the log and its digest, hiding the tampering.

---

## Q3. Distinguish RTO from RPO using your own measured figures. Which is improved by taking backups more frequently, and which is improved by restoring faster?

### Answer

My measured RTO was **3 seconds** for restoring 200 objects.

**RTO** is the time required to restore the service or data after an incident.

**RPO** is the amount of data or time that could be lost between the last backup and the incident.

In this lab, the backup was taken immediately before the simulated incident, so the RPO was approximately **0 seconds**.

> More frequent backups improve **RPO**, while faster restoration improves **RTO**.

---

## Q4. Your measured RTO was a few seconds. Explain why you should not report that number to a board, and what you would report instead.

### Answer

The measured **3 seconds** only represents a small 200-object LocalStack lab test.

It does not represent a real production environment with:

* Millions of objects
* Network delays
* Larger data sizes
* Encryption
* Dependencies
* Recovery procedures

I would report the measured result together with an appropriately scaled estimate and its assumptions rather than presenting 3 seconds as the expected production RTO.

For example:

```text
Measured RTO:
3 seconds for 200 objects

Extrapolated RTO:
15,000 seconds
≈ 4.17 hours for 1,000,000 objects

Assumption:
Recovery time scales linearly with the number of objects.
```

---

## Q5. Using your A4 table, state one incident that versioning survives and the separate backup does not, and one incident that the backup survives and versioning does not.

### Answer

### Incident that versioning survives but a separate backup may not

An accidental deletion or overwrite of an object can be recovered using a previous version.

If the backup was taken before the latest change, it may not contain the most recent version.

### Incident that a separate backup survives but versioning does not

Deletion of the entire primary bucket.

Versioning cannot protect versions if the bucket itself is deleted, while a separate backup bucket can still contain the data.

### Key Difference

> **Versioning** provides recovery within the same bucket.

> **Separate backup** provides a separate recovery location and trust boundary.

---

# Debrief Session

The following questions are for discussion during the 30-minute debrief session:

* What is your extrapolated RTO?
* What assumption is your extrapolated RTO most sensitive to?
* Your organisation promises clients a four-hour RTO. Based on your numbers, is that promise defensible?
* What would you need to change to make the RTO target achievable?
* Who should be alerted when `attach-user-policy` grants administrator rights?
* How quickly should the alert be generated?
* If an attacker obtains an administrator credential, which controls still hold?
* How long would those controls remain effective?

---

# Cleanup

Remove the disaster-recovery backup bucket:

```bash
aws $EP s3 rb s3://miit-dr-backup --force
```

Remove the audit trail bucket:

```bash
aws $EP s3 rb s3://miit-audit-trail --force
```

Detach AdministratorAccess from the temporary user:

```bash
aws $EP iam detach-user-policy \
  --user-name TempContractor \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

Delete the temporary user:

```bash
aws $EP iam delete-user --user-name TempContractor
```

Remove LocalStack:

```bash
docker rm -f localstack
```

Remove generated files:

```bash
rm -f /tmp/rec*.txt \
  mgmt-trail.log \
  mgmt-trail.sha256 \
  check.sha256
```

---

# References

* Course lectures — Week 2 (Security Design & Architecture)
* Course lectures — Week 6 (Monitoring, Auditing & Management)
* AWS CloudTrail Log File Integrity Validation
* CSA Security Guidance v5 — Domain 6 (Security Monitoring) and Domain 11 (Incident Response & Resilience)
* Lab 5 — Monitoring, Logging & Incident Detection
* Lab 6 — Object Storage Security (versioning and delete markers)

```
```
