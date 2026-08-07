# Lab 1 : Account Security and IAM
Task 1 — Map the Cloud Identity Landscape
Before creating anything, understand the building blocks of cloud identity. Complete the table in your report by filling the ‘Purpose’ column in your own words.


Task 2 — Create a Least-Privilege Admin (Stop Using Root)
The root user is a liability. Create a dedicated admin identity and grant permissions through a group, never directly to the user.

EP='--endpoint-url=http://localhost:4566'

# 2.1 Create a group and attach an admin policy to the GROUP aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
--policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 2.2 Create your personal admin user (replace YOURNAME) 
aws $EP iam create-user --user-name CloudAdmin_YOURNAME

# 2.3 Put the user in the group (permissions flow from the group) aws $EP iam add-user-to-group --group-name Admins \
--user-name CloudAdmin_YOURNAME

# 2.4 Verify the membership
aws $EP iam get-group --group-name Admins
Security tip: Attaching policies to groups (not users) is how you keep permissions manageable and auditable at scale — change the group once, and every member updates.
<img width="973" height="715" alt="image" src="https://github.com/user-attachments/assets/5a9dd433-1050-4395-9851-01b395c24748" />



Task 3 — Enforce Least Privilege with a Scoped Policy
Now create a read-only user for a teammate who should never modify data. This demonstrates fine-grained authorization.
 
In your report, explain: if the Analyst account were stolen, why is the damage limited compared to a stolen admin account? Connect your answer to blast-radius reduction.
= If the Analyst account was stolen, the damage would be limited because it only has the AmazonS3ReadOnlyAccess policy. The attacker could only view S3 resources and would not be able to create, modify or delete data or manage other AWS resources. In contrast, a stolen administrator account has full permission and could make significant changes or delete critical resources. This demonstrates blast-radius reduction, where limiting permissions ensures that the impact of a compromised account is contained and potential damage is minimized.
<img width="975" height="378" alt="image" src="https://github.com/user-attachments/assets/c5269a5e-de63-4312-a36c-9d93f4ff2e15" />

Task 4 — Credential Hygiene & Access Keys
Programmatic access uses access keys. Create one, then reason about the risk of long-lived keys.

# 4.1 Create an access key for the Analyst
aws $EP iam create-access-key --user-name Analyst_YOURNAME

# 4.2 List access keys (note the AccessKeyId and status) aws $EP iam list-access-keys --user-name Analyst_YOURNAME

# 4.3 Rotate: deactivate the old key (paste the AccessKeyId) aws $EP iam update-access-key --user-name Analyst_YOURNAME \
--access-key-id <PASTE_KEY_ID> --status Inactive
Caution: In real AWS, never create access keys on the root user and never commit keys to code repositories. Prefer short-lived roles over long-lived keys.
Note: End of Session A. Save your terminal outputs and screenshots. Stop the container if you wish (docker stop localstack) and restart it next week with docker start localstack.
<img width="974" height="369" alt="image" src="https://github.com/user-attachments/assets/e5f5d697-f8d6-46ae-a0f3-c726d70f162e" />


Session B (Week 2) — Enforced Access Control with Kubernetes RBAC

LocalStack teaches the mechanics of IAM, but it does not fully *enforce* policies. Kubernetes RBAC does — so in this session you will see access control actually block an unauthorised action.
Setup — Create a Local Kubernetes Cluster

# Create a throwaway cluster (runs inside Docker) kind create cluster --name ccse-lab1

# Confirm it is up
kubectl cluster-info --context kind-ccse-lab1 kubectl get nodes
<img width="971" height="119" alt="image" src="https://github.com/user-attachments/assets/b40f80b5-24fb-4598-aae5-8d3d976a765c" />

Task 5 — Separate Environments with Namespaces

kubectl create namespace dev kubectl create namespace prod kubectl get namespaces

 <img width="953" height="187" alt="image" src="https://github.com/user-attachments/assets/41d6ff0b-f3af-49f2-8fc0-2589f4e24374" />

Task 6 — Define a Role and Bind It (Least Privilege)
Create a role that can only read pods in dev, and bind it to a test service account. This is RBAC: a role (permissions) plus a role-binding (who gets them).

<img width="972" height="235" alt="image" src="https://github.com/user-attachments/assets/ff46f325-37f1-4573-be71-751826f5ea88" />

Task 7 — Test That Access Control Works
Use kubectl auth can-i to prove the boundary. Record every result.

SA=system:serviceaccount:dev:dev-user

# Should be YES — reading pods in dev is allowed kubectl auth can-i list pods -n dev --as=$SA

# Should be NO — deleting pods is not granted kubectl auth can-i delete pods -n dev --as=$SA

# Should be NO — the role does not extend to prod kubectl auth can-i list pods -n prod --as=$SA
Security tip: This is least privilege enforced by the platform: the developer can do exactly what the role permits and nothing more — even in the same cluster, prod is off-limits.
In your report, relate the three can-i results to authentication versus authorization: which step is the service account passing, and which step is blocking the delete and the prod access?
= The service account successfully passed the authentication step because Kubernetes recognized the identity system:serviceaccount:dev:dev-user. After authentication, Kubernetes performed authorization by checking the Role and RoleBinding assigned to that service account. The command kubectl auth can-i list pods -n dev returned yes because the Role allowed the list permission on pods in the dev namespace. However, kubectl auth can-i delete pods -n dev returned no because the Role did not grant the delete permission. Likewise, kubectl auth can-i list pods -n prod returned no because the RoleBinding applied only to the dev namespace and gave no permissions in prod. This
 
demonstrates that authentication verifies who the service account is, while authorization determines what actions it is allowed to perform.

 <img width="942" height="147" alt="image" src="https://github.com/user-attachments/assets/b5a0caba-5e06-43f3-aed1-18f4c31bfa00" />

Deliverables & Assessment

1.	Screenshots (label each clearly)
•	Output of sts get-caller-identity showing your operating identity.
<img width="983" height="130" alt="image" src="https://github.com/user-attachments/assets/a489aa79-c8bb-4658-a702-0539f92a03ff" />

•	get-group Admins output showing your CloudAdmin user as a member.
<img width="975" height="306" alt="image" src="https://github.com/user-attachments/assets/cdf4a42b-3a5c-4ece-a711-f952fd5bdff3" />

•	list-attached-user-policies for the Analyst showing only the read-only policy.
<img width="977" height="149" alt="image" src="https://github.com/user-attachments/assets/b5dd464c-69fa-4843-8a42-c43fe91760f8" />

•	The three kubectl auth can-i results (YES / NO / NO).
 <img width="975" height="153" alt="image" src="https://github.com/user-attachments/assets/6323dad8-75f6-4ec5-90cd-e20b26cc3cd5" />

2.	Short-Answer Questions
Q1. Why is attaching policies to groups better than attaching them directly to users?
= Attaching policies to groups simplifies permission management because permissions only need to be assigned once to the group. Any user added to the group automatically receives the appropriate permissions. This makes administration easier, ensures consistency, and improves auditing.
 
Q2.What is the difference between an IAM User and an IAM Role?
= An IAM User is a permanent identity for a person or application with long-term credentials. An IAM Role is a temporary identity that provides permissions when assumed by trusted users or services and does not require permanent credentials.

Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.
= The Analyst account was granted only the Amazon S3 ReadOnly policy. It can view S3 resources but cannot create, modify, or delete them. If the account is compromised, the attacker has very limited permissions, reducing the potential damage and minimizing the blast radius.

Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?
= A Role defines the permissions that are allowed within a namespace, such as reading pods. A RoleBinding assigns those permissions to a specific user, group, or service account, allowing them to use the Role.

Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?
= The service account was granted permissions only within the dev namespace through a RoleBinding. No permissions were assigned in the prod namespace, so access was denied. This demonstrates the principle of least privilege and namespace isolation by restricting access only to the resources required.
3.	Verification Command
Paste the output of the following to prove your cluster RBAC is in place:

<img width="980" height="394" alt="image" src="https://github.com/user-attachments/assets/dd259e9e-fa58-46d9-a63a-2369156232f3" />

 
Security Best-Practices Checklist

✔Root user is not used for daily tasks (a dedicated admin identity exists).
✔ Permissions are granted via groups/roles, not directly to individual users.
✔ At least one least-privilege (read-only) identity was created and tested.
✔ Access keys were listed and a rotation (deactivate) was demonstrated.
✔ Kubernetes RBAC blocks an unauthorised action (delete / cross-namespace).

Cleanup & Teardown


Expansion Ideas (Advanced Students)
•	Infrastructure as Code: recreate the IAM group, user and policy attachment using a Terraform script pointed at LocalStack.
•	Policy conditions: write a custom IAM policy JSON that denies all actions unless a condition (e.g. aws:MultiFactorAuthPresent) is met, and attach it.
•	RBAC depth: create a ClusterRole + ClusterRoleBinding and compare its scope to a namespaced Role.
•	Policy-as-code guardrails: install OPA Gatekeeper and write a policy that blocks any pod running as root.
References

•	Course lectures — Week 1 (Fundamentals), Week 2 (Security Architecture), Week 5 (Access Control), Week 7 (Identity Management).
•	LocalStack documentation — docs.localstack.cloud
•	Kubernetes RBAC — kubernetes.io/docs/reference/access-authn-authz/rbac
•	CSA Security Guidance v5 — Domain on Identity & Access Management.
