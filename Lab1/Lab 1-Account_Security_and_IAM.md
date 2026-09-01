# Identity and Access Control

## Session A — Cloud Identity and IAM

## Task 1 — Map the Cloud Identity Landscape

Before creating anything, understand the building blocks of cloud identity. Complete the table by filling in the **Purpose** column in your own words.

| **Concept** | **AWS Term** | **Purpose** |
|---|---|---|
| All-powerful owner | Root User | The main account with full control over all AWS resources. It should only be used for initial setup and emergency tasks because it has unrestricted permission. |
| Human/app identity | IAM User | A permanent identity created for a person or application to securely access AWS services using its own credentials. |
| Permission bundle | IAM Policy | A set of rules that defines which actions are allowed or denied on AWS resources. Policies are attached to users, groups, or roles. |
| Collection of users | IAM Group | A collection of IAM users that share the same permissions, making it easier to manage access for multiple users. |
| Temporary identity | IAM Role | A temporary identity that provides permissions without long-term credentials. It can be assumed by users, applications, or AWS services when needed. |

---

## Task 2 — Create a Least-Privilege Admin (Stop Using Root)

The root user is a liability. Create a dedicated admin identity and grant permissions through a group, never directly to the user.

EP='--endpoint-url=http://localhost:4566'

### Step 2.1: Create the Admin Group
```bash
aws $EP iam create-group --group-name Admins
````

### Step 2.2: Attach the Administrator Policy

```bash
aws $EP iam attach-group-policy \
  --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

### Step 2.3: Create the CloudAdmin User

```bash
aws $EP iam create-user \
  --user-name CloudAdmin_YOURNAME
```

### Step 2.4: Put the User in the Group

Permissions flow from the group.

```bash
aws $EP iam add-user-to-group \
  --group-name Admins \
  --user-name CloudAdmin_YOURNAME
```

### Step 2.5: Verify the Membership

```bash
aws $EP iam get-group \
  --group-name Admins
```

### Result

The `CloudAdmin_YOURNAME` user is added to the `Admins` group. The administrator permissions are inherited through the group rather than being attached directly to the user.

### Security Tip

Attaching policies to groups rather than users keeps permissions manageable and auditable at scale. Changes can be made once at the group level, and every member of the group receives the updated permissions.

<img width="973" height="715" alt="image" src="https://github.com/user-attachments/assets/cd7c30db-f5a6-4293-8e74-58c452822e37" />

---

## Task 3 — Enforce Least Privilege with a Scoped Policy

Now create a read-only user for a teammate who should never modify data. This demonstrates fine-grained authorization.

### Step 3.1: Create the Analyst User

```bash
aws $EP iam create-user \
  --user-name Analyst_YOURNAME
```

### Step 3.2: Attach the Read-Only Policy

```bash
aws $EP iam attach-user-policy \
  --user-name Analyst_YOURNAME \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

### Step 3.3: Verify the Attached Policy

```bash
aws $EP iam list-attached-user-policies \
  --user-name Analyst_YOURNAME
```

### Result

The `Analyst_YOURNAME` account was granted only the `AmazonS3ReadOnlyAccess` policy. This allows the account to view S3 resources without granting permissions to modify or delete data.

### Least-Privilege and Blast-Radius Explanation

If the Analyst account was stolen, the damage would be limited because it only has the `AmazonS3ReadOnlyAccess` policy. The attacker could only view S3 resources and would not be able to create, modify or delete data or manage other AWS resources.

In contrast, a stolen administrator account has full permission and could make significant changes or delete critical resources.

This demonstrates blast-radius reduction, where limiting permissions ensures that the impact of a compromised account is contained and potential damage is minimized.

<img width="975" height="378" alt="image" src="https://github.com/user-attachments/assets/ecc5bdaa-e2fd-4e9c-a06b-1cf04b605efa" />


---

## Task 4 — Credential Hygiene & Access Keys

Programmatic access uses access keys. Create one, then reason about the risk of long-lived keys.

### Step 4.1: Create an Access Key for the Analyst

```bash
aws $EP iam create-access-key \
  --user-name Analyst_YOURNAME
```

### Step 4.2: List Access Keys

Note the `AccessKeyId` and status.

```bash
aws $EP iam list-access-keys \
  --user-name Analyst_YOURNAME
```

### Step 4.3: Rotate the Access Key

Deactivate the old key by pasting the `AccessKeyId`.

```bash
aws $EP iam update-access-key \
  --user-name Analyst_YOURNAME \
  --access-key-id <PASTE_KEY_ID> \
  --status Inactive
```
<img width="974" height="369" alt="image" src="https://github.com/user-attachments/assets/f57d9068-2714-4c48-87c1-165e81a67fda" />


### Security Notes

In real AWS, never create access keys on the root user and never commit keys to code repositories.

Short-lived roles should be preferred over long-lived access keys whenever possible.

### End of Session A

Save all terminal outputs and screenshots.

The LocalStack container can be stopped at the end of the session if required:

```bash
docker stop localstack
```

It can be restarted in the next session using:

```bash
docker start localstack
```

---

# Session B — Enforced Access Control with Kubernetes RBAC

LocalStack teaches the mechanics of IAM, but it does not fully enforce policies. Kubernetes RBAC does. In this session, access control is used to actually block an unauthorised action.

---

## Setup — Create a Local Kubernetes Cluster

Create a temporary Kubernetes cluster that runs inside Docker.

### Create the Cluster

```bash
kind create cluster --name ccse-lab1
```

### Confirm the Cluster Is Running

```bash
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```
### Result

The local kind cluster `ccse-lab1` was created and kubectl was configured to use the `kind-ccse-lab1` context.

<img width="971" height="119" alt="image" src="https://github.com/user-attachments/assets/b5aefaea-8955-4368-98e7-5ab183c7608c" />

---

# Task 5 — Separate Environments with Namespaces

Create separate namespaces for development and production.

### Create the Namespaces

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

### Result

The namespaces `dev` and `prod` were created and listed as `Active`.

<img width="953" height="187" alt="image" src="https://github.com/user-attachments/assets/ff1d9a36-6b7d-43ed-98dd-91f44104b0d8" />

---

# Task 6 — Define a Role and Bind It

Create a role that can only read pods in the `dev` namespace and bind it to a test service account.

This demonstrates Kubernetes RBAC:

* A **Role** defines the permissions.
* A **RoleBinding** determines who receives those permissions.

## Step 6.1: Create the Service Account

```bash
kubectl create serviceaccount dev-user -n dev
```

### Result

The service account `dev-user` was created in the `dev` namespace.

## Step 6.2: Create the Pod Reader Role

```bash
kubectl create role pod-reader -n dev \
    --verb=get,list,watch \
    --resource=pods
```

### Result

The `pod-reader` Role was created in the `dev` namespace.

It allows only the following actions on pods:

* `get`
* `list`
* `watch`

Delete permission was not granted.

## Step 6.3: Create the RoleBinding

```bash
kubectl create rolebinding dev-user-binding -n dev \
    --role=pod-reader \
    --serviceaccount=dev:dev-user
```

### Result

The `dev-user-binding` RoleBinding binds the `pod-reader` Role to the `dev-user` service account.

<img width="972" height="235" alt="image" src="https://github.com/user-attachments/assets/5c6ac384-c8c7-4aec-8439-dfa0afc7942d" />

---

# Task 7 — Test That Access Control Works

Use `kubectl auth can-i` to prove the access-control boundary.

The service account identity was stored in a shell variable:

```bash
SA=system:serviceaccount:dev:dev-user
```

This represents the Kubernetes service account `dev-user` in the `dev` namespace.

---

## Test 1: List Pods in Dev

Reading pods in `dev` should be allowed.

### Command

```bash
kubectl auth can-i list pods -n dev --as=$SA
```

### Result

```text
yes
```

### Explanation

The service account can list pods in `dev` because the `pod-reader` Role allows the `list` action on pods in the `dev` namespace.

---

## Test 2: Delete Pods in Dev

Deleting pods should not be allowed because delete permission was not granted.

### Command

```bash
kubectl auth can-i delete pods -n dev --as=$SA
```

### Result

```text
no
```

### Explanation

The service account cannot delete pods because the Role only grants `get`, `list`, and `watch`. Delete permission was not granted.

---

## Test 3: List Pods in Prod

The service account should not be able to access pods in the `prod` namespace.

### Command

```bash
kubectl auth can-i list pods -n prod --as=$SA
```

### Result

```text
no
```
<img width="942" height="147" alt="image" src="https://github.com/user-attachments/assets/057fd1d5-9709-43dd-8add-e52cccdea24e" />


### Explanation

The service account cannot list pods in `prod` because the Role and RoleBinding are namespaced to `dev`. The permission does not extend to the `prod` namespace.

---

## Authentication vs Authorization

The service account successfully passed the authentication step because Kubernetes recognized the identity:

```text
system:serviceaccount:dev:dev-user
```

After authentication, Kubernetes performed authorization by checking the Role and RoleBinding assigned to that service account.

The command:

```bash
kubectl auth can-i list pods -n dev --as=$SA
```

returned `yes` because the Role allowed the `list` permission on pods in the `dev` namespace.

However:

```bash
kubectl auth can-i delete pods -n dev --as=$SA
```

returned `no` because the Role did not grant the `delete` permission.

Likewise:

```bash
kubectl auth can-i list pods -n prod --as=$SA
```

returned `no` because the RoleBinding applied only to the `dev` namespace and gave no permissions in `prod`.

This demonstrates that **authentication verifies who the service account is, while authorization determines what actions it is allowed to perform**.

### Security Tip

This is least privilege enforced by the platform: the developer can do exactly what the Role permits and nothing more. Even in the same cluster, the `prod` environment is off-limits.

---

# RBAC Verification Command

Use the following command to prove that the cluster RBAC configuration is in place:

```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```
The output should confirm that:

* The RoleBinding is named `dev-user-binding`.
* The namespace is `dev`.
* The referenced Role is `pod-reader`.
* The subject is the `dev-user` service account.
* The service account belongs to the `dev` namespace.

### Expected Output

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
```
<img width="980" height="394" alt="image" src="https://github.com/user-attachments/assets/04b27d3e-ed56-4090-9f7d-27f753955422" />


### Explanation

This confirms that the `dev-user-binding` RoleBinding connects the `dev-user` service account to the `pod-reader` Role in the `dev` namespace.

---

# 2. Short-Answer Questions

## Q1. Why is attaching policies to groups better than attaching them directly to users?

Attaching policies to groups simplifies permission management because permissions only need to be assigned once to the group. Any user added to the group automatically receives the appropriate permissions.

This makes administration easier, ensures consistency, and improves auditing.

---

## Q2. What is the difference between an IAM User and an IAM Role?

An IAM User is a permanent identity for a person or application with long-term credentials.

An IAM Role is a temporary identity that provides permissions when assumed by trusted users or services and does not require permanent credentials.

---

## Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.

The Analyst account was granted only the `AmazonS3ReadOnlyAccess` policy. It can view S3 resources but cannot create, modify, or delete them.

If the account is compromised, the attacker has very limited permissions, reducing the potential damage and minimizing the blast radius.

---

## Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?

A Role defines the permissions that are allowed within a namespace, such as reading pods.

A RoleBinding assigns those permissions to a specific user, group, or service account, allowing them to use the Role.

---

## Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?

The service account was granted permissions only within the `dev` namespace through a RoleBinding.

No permissions were assigned in the `prod` namespace, so access was denied.

This demonstrates the principle of **least privilege** and **namespace isolation** by restricting access only to the resources required.

---

# Security Best-Practices Checklist

* [x] Root user is not used for daily tasks; a dedicated admin identity exists.
* [x] Permissions are granted via groups/roles, not directly to individual users.
* [x] At least one least-privilege read-only identity was created and tested.
* [x] Access keys were listed and a rotation/deactivation was demonstrated.
* [x] Kubernetes RBAC blocks an unauthorised action, including delete and cross-namespace access.

---

# Cleanup & Teardown

After completing the lab, the temporary Kubernetes cluster can be deleted with:

```bash
kind delete cluster --name ccse-lab1
```

The LocalStack container can be stopped with:

```bash
docker stop localstack
```

If the LocalStack environment is required again later, it can be restarted with:

```bash
docker start localstack
```

---

# Expansion Ideas (Advanced Students)

## Infrastructure as Code

Recreate the IAM group, user, and policy attachment using a Terraform script pointed at LocalStack.

## Policy Conditions

Write a custom IAM policy JSON that denies all actions unless a condition, such as `aws:MultiFactorAuthPresent`, is met, and attach it.

## RBAC Depth

Create a `ClusterRole` and `ClusterRoleBinding` and compare their scope to a namespaced `Role`.

## Policy-as-Code Guardrails

Install OPA Gatekeeper and write a policy that blocks any pod running as root.

---

# References

* Course lectures — Week 1 (Fundamentals), Week 2 (Security Architecture), Week 5 (Access Control), Week 7 (Identity Management).
* LocalStack documentation — `docs.localstack.cloud`
* Kubernetes RBAC — `kubernetes.io/docs/reference/access-authn-authz/rbac`
* CSA Security Guidance v5 — Domain on Identity & Access Management.

```
```
