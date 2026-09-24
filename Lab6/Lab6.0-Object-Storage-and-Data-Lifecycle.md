# IKB42603 Cloud Computing Security Essentials

# LAB 6 · WEEKS 11–12
## Object Storage Security & the Data Security Lifecycle

> **Bucket exposure, resource policies, SSE-KMS, versioning and provable deletion — Amazon S3 on LocalStack**

---

# Lab Learning Outcomes

At the end of this lab, you will be able to:

- Provision object storage, classify the data you place in it, and explain why object storage has a different security model from block or file storage.
- Reproduce the **archetypal cloud breach** — a publicly readable bucket — and remediate it with Block Public Access and a least-privilege bucket policy.
- Distinguish **identity-based** (IAM) from **resource-based** (bucket policy) authorisation, and predict the outcome when the two disagree.
- Enforce **encryption at rest** with SSE-KMS and reason about a policy that mandates encryption on upload.
- Issue **time-bounded delegated access** with a presigned URL and assess its risks.
- Apply **versioning, lifecycle and retention** rules, demonstrate object-level data remanence, and achieve provable deletion through cryptographic erasure.

---

# Course & Assessment Mapping

| Item | Mapping |
|---|---|
| **Course Learning Outcome** | CLO2 — Construct secure cloud operations that safeguard data confidentiality and integrity (VBE3) |
| **Lecture topics** | Week 4 (Data Protection) · Week 10 (Policy, Compliance & Risk) · Week 11 (Compliance Assessment & Reporting) |
| **Value / skill clusters** | VBE3 (Integrity) · SC8 (Integrated Problem-Solving) |
| **CSA CCSK v5 domains** | Domain 5 (Data Security) · Domain 4 (Organisation Management) · Domain 9 (Application Security — resource policy) |
| **Assessment** | Lab report (outputs + short answers) — contributes to the Lab Assignment |

---

# Lab Arrangement (2 Sessions over 2 Weeks)

| Session | Week | Focus |
|---|---|---|
| **Session A** | Week 11 | Object storage, data classification, the public-bucket breach and its remediation, identity vs resource policy (Tasks 1–4) |
| **Session B** | Week 12 | Default encryption, presigned URLs, versioning and remanence, lifecycle and cryptographic erasure (Tasks 5–8), then the report |

> **Note:** Session A is about **who can reach the data**. Session B is about **what state the data is in** — encrypted, versioned, retained, or provably destroyed. Together they trace the full data security lifecycle from Week 4.

---

# Technical Prerequisites

- A laptop with Docker Desktop / Docker Engine and a terminal.
- AWS CLI v2, pointed at LocalStack exactly as in Lab 1.
- A LocalStack auth token configured as in Lab 0.1. The Resource Browser makes bucket state much easier to see.
- The KMS concepts from Lab 3 — this lab reuses envelope encryption and cryptographic erasure at bucket scale.
- `curl` for anonymous requests (pre-installed on macOS/Linux; use Git Bash or WSL on Windows).

> **Security tip:** Object storage is the single most common source of real-world cloud data leaks. Every task in Session A corresponds to a control that, when missing, has produced a headline breach. Read the failures as carefully as the successes.

---

# Session A (Week 11) — Object Storage & the Exposure Problem

## One-Time Environment Setup

Start a clean, activated LocalStack instance and point the CLI at it.

`ENFORCE_IAM=1` asks LocalStack to actually evaluate IAM policies rather than allowing everything — you need this for Task 4.

```bash
# Start clean
docker rm -f localstack 2>/dev/null

docker run -d --name localstack -p 4566:4566 \
  -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN \
  -e ENFORCE_IAM=1 \
  localstack/localstack-pro:latest

# Point the CLI at LocalStack (repeat in every new terminal)
export EP='--endpoint-url=http://localhost:4566'

aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

aws $EP sts get-caller-identity
````

A successful call returns the LocalStack dummy identity (account `000000000000`).

Record that account number — you will need it in the ARNs you write in Task 4.

---

# Task 1 — Classify the Data Before You Store It

Security decisions follow classification, not the other way round.

Create a bucket for a hospital records system and store three objects of different sensitivity, tagging each with its classification.

```bash
export BUCKET=miit-patient-records-$RANDOM

echo $BUCKET
# Write this down - you need it all lab

aws $EP s3api create-bucket --bucket $BUCKET

echo 'Ward visiting hours 10am-8pm' > public-notice.txt

echo 'Staff duty schedule, week 12' > internal-roster.txt

echo 'Patient: Ahmad bin Ali, Diagnosis: confidential' > confidential-record.txt

aws $EP s3api put-object --bucket $BUCKET --key public/notice.txt \
  --body public-notice.txt \
  --tagging 'classification=public'

aws $EP s3api put-object --bucket $BUCKET --key internal/roster.txt \
  --body internal-roster.txt \
  --tagging 'classification=internal'

aws $EP s3api put-object --bucket $BUCKET --key confidential/record.txt \
  --body confidential-record.txt \
  --tagging 'classification=confidential'

aws $EP s3api list-objects-v2 --bucket $BUCKET \
  --query 'Contents[].[Key,Size]' --output table

aws $EP s3api get-object-tagging --bucket $BUCKET \
  --key confidential/record.txt
```

Complete this table in your report.

| Classification   | Who may read it               | Impact if leaked | Control you will apply                          |
| ---------------- | ----------------------------- | ---------------- | ----------------------------------------------- |
| **public**       | Anyone                        | Low              | Public information only                         |
| **internal**     | Authorised staff              | Moderate         | IAM / least privilege access                    |
| **confidential** | Specifically authorised users | High             | Least privilege, encryption & restricted prefix |

> **Note:** Note the key names. A prefix such as `confidential/` is not a folder — object storage has a flat namespace and the slash is just part of the key. That matters because policies grant access by **key prefix**, which is why a sloppy prefix such as `*` exposes everything at once.

---

# Task 2 — Reproduce the Archetypal Breach

Almost every "exposed cloud storage" headline reduces to one resource policy that names `"Principal": "*"`. Build that policy deliberately, then read your own confidential record with no credentials at all.

```bash
cat > public-policy.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadEverything",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*"
  }]
}
JSON

aws $EP s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy file://public-policy.json

aws $EP s3api get-bucket-policy \
  --bucket $BUCKET \
  --query Policy \
  --output text

# The attacker's view: no AWS credentials, no CLI, just a URL

curl -s -o leaked.txt -w 'HTTP %{http_code}\n' \
  http://localhost:4566/$BUCKET/confidential/record.txt

cat leaked.txt
```

> **Caution:** HTTP 200 and the patient record printed in your terminal is the whole breach. There was no exploit, no malware and no vulnerability — only a policy that said `Principal: *`.

In your report, state which single word in the JSON caused the exposure.

---

# Task 3 — Remediate with Block Public Access

Removing the bad policy fixes today's mistake.

**Block Public Access** is the guardrail that stops tomorrow's — it overrides any policy or ACL that would make the bucket public, so a careless colleague cannot re-open it.

## Step 1 — Remove the offending policy

```bash
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```

## Step 2 — Apply the account-level guardrail to the bucket

```bash
aws $EP s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws $EP s3api get-public-access-block --bucket $BUCKET
```

## Step 3 — Try to re-introduce the public policy

The guardrail should refuse it.

```bash
aws $EP s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy file://public-policy.json
```

## Step 4 — Re-test the anonymous read

```bash
curl -s -o /dev/null \
  -w 'anonymous read now: HTTP %{http_code}\n' \
  http://localhost:4566/$BUCKET/confidential/record.txt
```

> **Verify or explain:** LocalStack stores the Block Public Access configuration faithfully but does not always *enforce* it, so step 3 may succeed and step 4 may still return HTTP 200.
>
> If so, capture the `get-public-access-block` output as your evidence and state in your report:
>
> 1. Which of the four flags would have rejected the policy on real AWS.
> 2. Why a preventative guardrail is stronger than a detective control that merely reports the bucket as public.

---

## Least-Privilege Bucket Policy

Now write the policy you *should* have had: read access for your own account only, scoped to the internal prefix.

```bash
cat > least-privilege-policy.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AccountReadInternalOnly",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::000000000000:root"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/internal/*"
  }]
}
JSON

aws $EP s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy file://least-privilege-policy.json

aws $EP s3api get-bucket-policy \
  --bucket $BUCKET \
  --query Policy \
  --output text
```

---

# Task 4 — Identity Policy vs Resource Policy

Cloud storage is governed by two policies at once:

* **Identity-based policy (IAM):** attached to the caller.
* **Resource-based policy (bucket policy):** attached to the bucket.

When they disagree, an **explicit Deny always wins**.

Prove it.

## Create the Analyst

```bash
# An analyst whose IAM policy allows reading everything

aws $EP iam create-user --user-name DataAnalyst
```

Create the IAM policy:

```bash
cat > analyst-iam.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "s3:ListBucket"
    ],
    "Resource": "*"
  }]
}
JSON

aws $EP iam put-user-policy \
  --user-name DataAnalyst \
  --policy-name S3ReadAll \
  --policy-document file://analyst-iam.json
```

Create an access key:

```bash
# Note both values - you will paste them into a named profile

aws $EP iam create-access-key \
  --user-name DataAnalyst \
  --query 'AccessKey.[AccessKeyId,SecretAccessKey]' \
  --output text
```

Copy the two values into these variables:

```bash
# Keep the quotes.
# Do not include angle brackets.

ANALYST_KEY_ID='PASTE_KEY_ID_HERE'

ANALYST_SECRET='PASTE_SECRET_HERE'

aws configure --profile analyst set aws_access_key_id "$ANALYST_KEY_ID"

aws configure --profile analyst set aws_secret_access_key "$ANALYST_SECRET"

aws configure --profile analyst set region us-east-1
```

The analyst's IAM policy says **read anything**.

Now the bucket owner disagrees about one prefix.

## Create the Deny Policy

```bash
cat > deny-confidential.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAnalystInternal",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::000000000000:user/DataAnalyst"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/internal/*"
    },
    {
      "Sid": "DenyAnalystConfidential",
      "Effect": "Deny",
      "Principal": {
        "AWS": "arn:aws:iam::000000000000:user/DataAnalyst"
      },
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::$BUCKET/confidential/*"
    }
  ]
}
JSON

aws $EP s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy file://deny-confidential.json
```

## Test Internal Access

Should **SUCCEED** — allowed by both policies.

```bash
AWS_PROFILE=analyst aws $EP s3api get-object \
  --bucket $BUCKET \
  --key internal/roster.txt \
  analyst-internal.txt && echo "internal: ALLOWED"
```

## Test Confidential Access

Should **FAIL** — IAM allows it, but the bucket policy explicitly denies it.

```bash
AWS_PROFILE=analyst aws $EP s3api get-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  analyst-conf.txt || echo "confidential: DENIED"
```

> **Verify or explain:** If LocalStack was not started with `ENFORCE_IAM=1`, both calls will succeed.
>
> Restart the container with the flag and retry.
>
> If it still does not deny, record both policy documents as evidence and write out the evaluation logic yourself:
>
> **default deny → any explicit Deny → any explicit Allow**
>
> State which statement decides each of the two requests.

> **Caution:** Remove this policy before Session B:
>
> ```bash
> aws $EP s3api delete-bucket-policy --bucket $BUCKET
> ```
>
> A Deny statement scoped to `s3:*` can lock **you** out as well if the principal ARN does not match exactly what you expected. This is itself one of the common resource-policy incidents in production.
>
> If you do lock yourself out, delete the bucket policy or restart the LocalStack container.

> **Note:** End of Session A. Keep your bucket, `$BUCKET` value, and all outputs — Session B builds directly on them.

---

# Session B (Week 12) — Protecting, Retaining and Retiring Data

# Task 5 — Default Encryption at Rest (SSE-KMS)

In Lab 3 you encrypted a file by hand.

Here you make encryption a **property of the bucket**, so that every object is encrypted whether or not the developer who uploads it remembers to ask.

## Create a Dedicated KMS Key

```bash
export KEY_ID=$(aws $EP kms create-key \
  --description 'IKB42603 Lab6 patient records bucket key' \
  --query 'KeyMetadata.KeyId' \
  --output text)

echo $KEY_ID
```

Create the encryption configuration:

```bash
cat > encryption.json <<'JSON'
{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "$KEY_ID"
    },
    "BucketKeyEnabled": true
  }]
}
JSON
```

Apply it to the bucket:

```bash
aws $EP s3api put-bucket-encryption \
  --bucket $BUCKET \
  --server-side-encryption-configuration file://encryption.json

aws $EP s3api get-bucket-encryption --bucket $BUCKET
```

## Upload Without Encryption Flags

The bucket applies the key automatically.

```bash
aws $EP s3api put-object \
  --bucket $BUCKET \
  --key confidential/record-v2.txt \
  --body confidential-record.txt
```

Check the object:

```bash
aws $EP s3api head-object \
  --bucket $BUCKET \
  --key confidential/record-v2.txt \
  --query '[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]' \
  --output text
```

The `head-object` output should report:

* `aws:kms`
* Your key ID

This is evidence that a default control protected an object without the uploader doing anything.

> **Security tip:** `BucketKeyEnabled: true` is the envelope-encryption optimisation from Lab 3 applied at bucket scale: one data key is reused across many objects instead of one KMS call per object. Confidentiality is unchanged; cost and latency fall sharply.

---

# Task 6 — Delegated Access and the Condition-Key Trap

A **presigned URL** grants a specific action on a specific object for a limited time, to someone with no AWS identity at all.

It is the correct answer to "how do I share this file" — and it is frequently misused.

## Create a Time-Bounded Presigned URL

```bash
aws $EP s3 presign s3://$BUCKET/internal/roster.txt \
  --expires-in 60
```

Paste the URL into the variable:

```bash
URL='PASTE_PRESIGNED_URL_HERE'
```

The expiry test below must reuse the **same URL**.

Test it immediately:

```bash
curl -s -w ' <-- HTTP %{http_code}\n' "$URL"
```

Wait for it to expire:

```bash
sleep 65
```

Test again:

```bash
curl -s -o /dev/null \
  -w 'after expiry: HTTP %{http_code}\n' \
  "$URL"
```

> **Verify or explain:** LocalStack may not enforce the expiry and can return HTTP 200 on the second call.
>
> If so, inspect the URL itself and identify:
>
> * `Expires`
> * `X-Amz-Expires`
> * `Signature`
>
> Explain in your report what each one binds and why anyone holding the URL before it lapses is fully authorised.

---

## The SecureTransport Condition-Key Trap

The following policy is copied verbatim from countless security hardening guides — it denies any request that did not arrive over TLS.

Apply it and watch what happens to **every** subsequent command.

```bash
cat > secure-transport.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyUnencryptedTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::$BUCKET",
      "arn:aws:s3:::$BUCKET/*"
    ],
    "Condition": {
      "Bool": {
        "aws:SecureTransport": "false"
      }
    }
  }]
}
JSON

aws $EP s3api put-bucket-policy \
  --bucket $BUCKET \
  --policy file://secure-transport.json
```

Any ordinary call should now be refused:

```bash
aws $EP s3api list-objects-v2 --bucket $BUCKET
```

## Recover Before Continuing

```bash
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```

> **Caution:** You have just locked yourself out of your own bucket, and the policy was correct.
>
> Your LocalStack endpoint is plain **`http://`**, so `aws:SecureTransport` is `false` for every request you make. The Deny therefore matches all of them.
>
> On real AWS the endpoint is HTTPS, the condition evaluates to `true`, and the statement only catches genuinely insecure callers.
>
> In your report, explain why a condition key must always be evaluated against the environment it will run in, not the one it was written for.

---

# Task 7 — Versioning, Delete Markers & Data Remanence

Lab 2 showed remanence inside a container volume.

Object storage has its own version: with versioning enabled, **delete does not delete**.

It writes a **delete marker** over the top and every prior version survives underneath.

## Enable Versioning

```bash
aws $EP s3api put-bucket-versioning \
  --bucket $BUCKET \
  --versioning-configuration Status=Enabled

aws $EP s3api get-bucket-versioning --bucket $BUCKET
```

## Create Two More Revisions

```bash
echo 'Patient: Ahmad bin Ali, Diagnosis: hypertension' > rec-v2.txt

echo 'Patient: [REDACTED], Diagnosis: [REDACTED]' > rec-v3.txt
```

Upload both:

```bash
aws $EP s3api put-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  --body rec-v2.txt \
  --query VersionId \
  --output text

aws $EP s3api put-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  --body rec-v3.txt \
  --query VersionId \
  --output text
```

List the versions:

```bash
aws $EP s3api list-object-versions \
  --bucket $BUCKET \
  --prefix confidential/record.txt \
  --query 'Versions[].[VersionId,IsLatest,Size]' \
  --output table
```

The oldest entry is listed with version ID `null` — that is the copy uploaded in Task 1, before versioning existed.

---

## Delete the Record

```bash
aws $EP s3api delete-object \
  --bucket $BUCKET \
  --key confidential/record.txt
```

A delete marker is now the current version.

Check it:

```bash
aws $EP s3api list-object-versions \
  --bucket $BUCKET \
  --prefix confidential/record.txt \
  --query 'DeleteMarkers[].[VersionId,IsLatest]' \
  --output table
```

To an ordinary reader, the object is gone:

```bash
aws $EP s3api get-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  gone.txt
```

But the original, unredacted record is still there:

```bash
aws $EP s3api get-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  --version-id null \
  recovered.txt

cat recovered.txt
```

> **Caution:** `recovered.txt` contains the original diagnosis — the data you redacted in v3 and then deleted.
>
> This is object-level data remanence, and it is why "we deleted the record" is not an acceptable answer to a data-subject erasure request under a privacy regime such as the PDPA or GDPR.
>
> Removing it for real requires deleting **every version by ID**.

## Permanent, Per-Version Deletion

```bash
aws $EP s3api delete-object \
  --bucket $BUCKET \
  --key confidential/record.txt \
  --version-id null

aws $EP s3api list-object-versions \
  --bucket $BUCKET \
  --prefix confidential/record.txt \
  --query 'Versions[].[VersionId,Size]' \
  --output table
```

---

# Task 8 — Lifecycle, Retention & Cryptographic Erasure

Deleting versions by hand does not scale.

A lifecycle configuration is the automated, auditable expression of your retention policy — and it is exactly the artefact an auditor will ask for in Week 11.

## Create Lifecycle Configuration

```bash
cat > lifecycle.json <<'JSON'
{
  "Rules": [
    {
      "ID": "RetireConfidentialRecords",
      "Filter": {
        "Prefix": "confidential/"
      },
      "Status": "Enabled",
      "Expiration": {
        "Days": 365
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 30
      }
    },
    {
      "ID": "AbortIncompleteUploads",
      "Filter": {
        "Prefix": ""
      },
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
JSON
```

Apply the lifecycle configuration:

```bash
aws $EP s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET \
  --lifecycle-configuration file://lifecycle.json

aws $EP s3api get-bucket-lifecycle-configuration \
  --bucket $BUCKET \
  --query 'Rules[].[ID,Status]' \
  --output table
```

---

## Cryptographic Erasure

Finally, the fastest deletion available in the cloud.

Every object in this bucket is wrapped by one KMS key. Destroy the key and the ciphertext becomes unrecoverable noise, however many copies, versions or backups exist.

Check the key:

```bash
aws $EP kms describe-key \
  --key-id $KEY_ID \
  --query 'KeyMetadata.[KeyId,KeyState,Enabled]' \
  --output text
```

Disable the key:

```bash
aws $EP kms disable-key --key-id $KEY_ID
```

Schedule key deletion:

```bash
aws $EP kms schedule-key-deletion \
  --key-id $KEY_ID \
  --pending-window-in-days 7
```

Check the key state:

```bash
aws $EP kms describe-key \
  --key-id $KEY_ID \
  --query 'KeyMetadata.[KeyState,DeletionDate]' \
  --output text
```

Attempt to read an object encrypted under the disabled key:

```bash
aws $EP s3api get-object \
  --bucket $BUCKET \
  --key confidential/record-v2.txt \
  after-erasure.txt
```

> **Verify or explain:** LocalStack may still return the object because it does not always re-check key state on read.
>
> If your read succeeds, demonstrate the same principle at the KMS layer instead — repeat the Lab 3 Task 6 sequence:
>
> `kms encrypt → disable-key → kms decrypt`
>
> Attach the failed decrypt.
>
> Then explain why cryptographic erasure gives an auditor a stronger assurance than overwriting, given that you do not control the physical media.

---

# Data Classification Table

Complete the four-column table from Task 1, with the control you actually implemented named in the final column.

| Classification | Who may read it               | Impact if leaked | Control                                         |
| -------------- | ----------------------------- | ---------------- | ----------------------------------------------- |
| Public         | Anyone                        | Low              | Public information only                         |
| Internal       | Authorised staff              | Moderate         | IAM / least privilege access                    |
| Confidential   | Specifically authorised users | High             | Least privilege, encryption & restricted prefix |

---

# Short-Answer Questions

## Question 1

**Which single element of the Task 2 policy caused the exposure, and why is `Principal: "*"` more dangerous on a bucket policy than an over-broad IAM policy attached to one user?**

### Answer

The main element that caused the exposure was:

```json
"Principal": "*"
```

It allowed anyone, including unauthenticated users, to read objects in the bucket.

This is more dangerous than an over-broad IAM policy attached to one user because the bucket policy can expose the data to every external user, not just one authenticated account.

---

## Question 2

**Explain the difference between an identity-based policy and a resource-based policy. In Task 4, which one decided each of the analyst's two requests?**

### Answer

An **identity-based policy** is attached to a user, group or role and defines what that identity is allowed to do.

A **resource-based policy** is attached directly to a resource such as an S3 bucket and defines who can access that resource.

In Task 4, the analyst's identity-based IAM policy allowed both requests, but the resource-based bucket policy explicitly denied access to the confidential prefix.

Therefore:

* Internal request → **Allowed**
* Confidential request → **Denied**

The confidential request was denied because **explicit Deny overrides Allow**.

---

## Question 3

**Block Public Access is described as a guardrail rather than a control. What is the difference, and why does the distinction matter for an organisation with many engineers?**

### Answer

A **control** directly enforces a specific security requirement, while a **guardrail** acts as a protective boundary that prevents common unsafe configurations.

Block Public Access is considered a guardrail because it helps prevent engineers from accidentally making S3 data publicly accessible.

This matters in an organization with many engineers because it provides an additional safety layer and reduces the risk of human configuration errors.

---

## Question 4

**Your bucket has default SSE-KMS encryption. Does that protect the confidential record from the analyst in Task 4? Explain precisely what server-side encryption does and does not defend against.**

### Answer

No, not by itself.

Server-side SSE-KMS protects the confidentiality of stored data by encrypting it at rest using a KMS key. However, it does not decide who is authorized to access the object.

Therefore, if the analyst has permission to retrieve the object and the KMS key also allows the required decryption operation, the analyst can still access the plaintext.

**IAM and bucket policies control access, while SSE-KMS protects the stored ciphertext.**

---

## Question 5

**A patient invokes their right to erasure. Using your Task 7 evidence, explain why `delete-object` alone is not compliant, and describe two mechanisms that would make the deletion provable.**

### Answer

`delete-object` alone is not necessarily compliant because Task 7 showed that deleting a versioned object created a delete marker while previous versions remained recoverable.

The original patient record could therefore still be retrieved using its previous version ID.

Two mechanisms that could make deletion more provable are:

1. Permanently delete all object versions and delete markers, then verify using `list-object-versions`.
2. Use an auditable deletion process, such as CloudTrail or equivalent immutable audit logs, to record and prove that all relevant versions were permanently deleted.

---

## Question 6

**You are the auditor in Week 11. Name three commands from this lab whose output you would collect as compliance evidence, and state which control each one evidences.**

### Answer

### 1. Public Access Prevention

```bash
aws $EP s3api get-public-access-block \
  --bucket $BUCKET
```

**Evidence:** Public access prevention — proves the four Block Public Access settings are enabled.

### 2. Encryption at Rest

```bash
aws $EP s3api get-bucket-encryption \
  --bucket $BUCKET
```

**Evidence:** Encryption at rest — proves default SSE-KMS encryption is configured.

### 3. Data Retention / Lifecycle

```bash
aws $EP s3api get-bucket-lifecycle-configuration \
  --bucket $BUCKET
```

**Evidence:** Data retention/lifecycle control — proves lifecycle and retention rules are configured.

---

# Verification Command

Paste the output of the following block to prove the bucket's final security posture.

```bash
echo "=== IKB42603 Lab 6 verification: $BUCKET ==="

aws $EP s3api get-public-access-block \
  --bucket $BUCKET \
  --query 'PublicAccessBlockConfiguration' \
  --output text

aws $EP s3api get-bucket-versioning \
  --bucket $BUCKET \
  --output text

aws $EP s3api get-bucket-encryption \
  --bucket $BUCKET \
  --query 'ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.[SSEAlgorithm,KMSMasterKeyID]' \
  --output text

aws $EP s3api get-bucket-lifecycle-configuration \
  --bucket $BUCKET \
  --query 'Rules[].[ID,Status]' \
  --output text

aws $EP kms describe-key \
  --key-id $KEY_ID \
  --query 'KeyMetadata.KeyState' \
  --output text
```

---

# Security Best-Practices Checklist

* [x] Every object carries a classification tag before any access decision is made.
* [x] No bucket policy names `Principal: "*"`; anonymous access was tested and is refused.
* [x] Block Public Access is enabled on all four flags.
* [x] Access is granted by least privilege and scoped to a key prefix, never to `/*` by default.
* [x] Default encryption at rest is `aws:kms` with a customer-managed key.
* [x] Sharing uses time-bounded presigned URLs, not permanent public objects.
* [x] Versioning is enabled, and the team understands that delete markers do not destroy data.
* [x] A lifecycle configuration expresses the retention policy, and cryptographic erasure is available for provable deletion.

---

# Cleanup & Teardown

A versioned bucket cannot be emptied with `s3 rb --force` — that command ignores non-current versions and delete markers, and the bucket deletion fails with `BucketNotEmpty`.

This is the same lesson as Task 7: **you must remove every version explicitly.**

## Remove the Bucket Policy

```bash
aws $EP s3api delete-bucket-policy --bucket $BUCKET
```

## Delete All Object Versions

```bash
aws $EP s3api delete-objects \
  --bucket $BUCKET \
  --delete "$(aws $EP s3api \
  list-object-versions \
  --bucket $BUCKET \
  --output json \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}')"
```

## Delete All Delete Markers

```bash
aws $EP s3api delete-objects \
  --bucket $BUCKET \
  --delete "$(aws $EP s3api \
  list-object-versions \
  --bucket $BUCKET \
  --output json \
  --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}')"
```

## Verify the Bucket Is Empty

```bash
aws $EP s3api list-object-versions \
  --bucket $BUCKET \
  --output text
```

## Delete the Bucket

```bash
aws $EP s3api delete-bucket --bucket $BUCKET
```

## Remove the IAM User

```bash
aws $EP iam delete-user-policy \
  --user-name DataAnalyst \
  --policy-name S3ReadAll

aws $EP iam delete-user \
  --user-name DataAnalyst
```

## Stop LocalStack

```bash
docker rm -f localstack
```

## Remove Local Files

```bash
rm -f *.json *.txt
```

---

# References

* Course lectures — Week 4 (Data Protection); Week 10 (Policy, Compliance & Risk); Week 11 (Compliance Assessment & Reporting).
* Amazon S3 security best practices — `docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html`
* Amazon S3 versioning and lifecycle — `docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html`
* LocalStack S3 coverage and limitations — `docs.localstack.cloud/references/coverage`
* CSA Security Guidance v5 — Domain 5, Data Security; and the Data Security Lifecycle.
* MCMC MTSFB TC G017:2021 — Information Security Requirements for Cloud Service Providers (data handling clauses).

```