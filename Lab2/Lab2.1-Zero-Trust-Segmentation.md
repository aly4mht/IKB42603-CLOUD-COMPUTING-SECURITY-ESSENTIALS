# Lab 2.1: Zero Trust Segmentation and Admission Control

# Lab Summary

This Lab 2 addendum extends the secure multi-tenancy controls implemented in Lab 2 by applying Zero Trust principles to Kubernetes workloads.

Lab 2 established logical and network boundaries using namespaces, ResourceQuota and default-deny ingress policies. This addendum focuses on two additional controls:

1. **Default-deny egress and micro-segmentation** — restricting where a compromised workload can communicate.
2. **Admission control using Pod Security Standards** — preventing insecure workloads from running before they are admitted into the cluster.

The main Zero Trust principle demonstrated in this lab is that being inside the Kubernetes cluster is not sufficient to be trusted. A workload must be explicitly authorized to communicate with another destination, and workloads must satisfy security requirements before they are allowed to run.

The addendum demonstrates that effective Zero Trust security requires both network restrictions and preventative workload admission controls.

## Lab Prerequisite

This addendum uses the Kubernetes cluster created in Lab 2.

The following components should already exist:

* kind cluster `ccse-lab2`
* Calico CNI with NetworkPolicy enforcement
* Namespace `tenant-a`
* Namespace `tenant-b`
* Tenant services used for the cross-tenant communication tests

The addendum extends the Lab 2 isolation controls by adding egress restrictions and admission control.

---

# Task Z1: Egress Default-Deny and Micro-Segmentation

## Baseline: Test Default Egress Behaviour

The first step was to confirm that a pod in `tenant-a` could make an outbound connection before an egress NetworkPolicy was applied.

```bash
kubectl -n tenant-a run probe --rm -it --restart=Never --image=busybox:1.36 -- \
  sh -c "wget -qO- --timeout=3 http://api.tenant-b.svc.cluster.local || echo BLOCKED"
```

### Result

Before the egress policy was applied, outbound communication was unrestricted.

This demonstrates that an ingress policy alone does not prevent a compromised workload from initiating connections to other services or destinations.

An attacker who has already compromised a pod is already inside the cluster perimeter. Therefore, restricting only incoming traffic is insufficient for a Zero Trust environment.

### Evidence

<img width="975" height="68" alt="image" src="https://github.com/user-attachments/assets/b6a98cfa-1dfc-4829-8802-f9af0ac662aa" />

---

## Apply Default-Deny Egress Policy

A NetworkPolicy was created to deny all outbound traffic from pods in `tenant-a`.

A second policy was created to explicitly allow only:

* DNS communication to CoreDNS.
* Communication with the required API service.

### Egress Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-api-only
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  - to:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 80
```

The policies were saved as:

```text
egress-policy.yaml
```

The configuration was then applied:

```bash
kubectl apply -f egress-policy.yaml
```

The policies were verified using:

```bash
kubectl -n tenant-a get networkpolicy
```

### Result

The `default-deny-egress` policy denies outbound traffic by default.

The `allow-dns-and-api-only` policy acts as an allow-list by permitting only the required dependencies.

This implements the Zero Trust principle of explicitly authorizing network communication instead of trusting traffic simply because it originates from inside the cluster.

### Evidence
<img width="795" height="1094" alt="image" src="https://github.com/user-attachments/assets/497ac310-cf16-4122-baa4-5440feaeeb43" />

---

## Retest: Cross-Tenant Egress

The cross-tenant connection was tested again after applying the egress policies.

```bash
kubectl -n tenant-a run probe --rm -it --restart=Never --image=busybox:1.36 -- \
  sh -c "wget -qO- --timeout=3 http://api.tenant-b.svc.cluster.local || echo BLOCKED"
```

### Expected Result

```text
BLOCKED
```

### Result

The cross-tenant connection should now fail because the destination is not included in the permitted egress rules.

This demonstrates micro-segmentation. Even if an attacker compromises a workload inside `tenant-a`, the workload cannot freely communicate with other tenants or unauthorized destinations.

### Evidence
<img width="471" height="29" alt="image" src="https://github.com/user-attachments/assets/603874ca-cb2b-4840-a85d-0b504caaa923" />

---

## Verify Permitted In-Namespace Communication

The policy should not block all communication. Legitimate communication with the explicitly allowed service should continue to work.

```bash
kubectl -n tenant-a run probe --rm -it --restart=Never --image=busybox:1.36 -- \
  sh -c "wget -qO- --timeout=3 http://api.tenant-a.svc.cluster.local || echo BLOCKED"
```

### Result

The permitted in-namespace API service should remain reachable.

This demonstrates that the policy is not a blanket block. Instead, it follows the principle of denying unnecessary communication while allowing explicitly authorized traffic.

### Evidence
<img width="466" height="30" alt="image" src="https://github.com/user-attachments/assets/e47c28c7-4dd0-4ccd-9184-33b8b1a4b1db" />

---

## DNS Dependency Test

The DNS rule was deliberately removed to demonstrate the importance of allowing legitimate dependencies when using default-deny controls.

Without DNS egress access, a workload cannot resolve service names such as:

```text
api.tenant-a.svc.cluster.local
```

DNS resolution requires communication with the Kubernetes DNS service using port 53.

### Result

When DNS traffic is blocked, hostname-based communication fails even if the destination service itself would otherwise be permitted.

This demonstrates that security controls must be tested carefully before deployment to production. A deny-by-default policy must account for legitimate dependencies such as DNS.

---

# Task Z2: Admission Control Using Pod Security Standards

## Apply Restricted Pod Security Standard

The `tenant-a` namespace was configured to enforce the Kubernetes restricted Pod Security Standard.

```bash
kubectl label namespace tenant-a \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite
```

The namespace labels were verified using:

```bash
kubectl get namespace tenant-a --show-labels
```

### Result

The `tenant-a` namespace now enforces the restricted Pod Security Standard.

This means workloads that violate the required security restrictions can be rejected during admission before they are allowed to run.

### Evidence

<img width="975" height="163" alt="image" src="https://github.com/user-attachments/assets/de65994f-c50a-4995-a598-6f3f501c6e97" />

---

## Test a Privileged Pod

A deliberately insecure privileged pod was created.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-probe
  namespace: tenant-a
spec:
  containers:
  - name: probe
    image: busybox:1.36
    command: ["sleep", "3600"]
    securityContext:
      privileged: true
```

The configuration was saved as:

```text
privileged-pod.yaml
```

The pod was then submitted:

```bash
kubectl apply -f privileged-pod.yaml
```

### Expected Result

The pod should be rejected by the admission controller.

### Result

The restricted Pod Security Standard prevents the privileged workload from being created.

The admission controller can reject the workload because it violates restricted security requirements such as:

* Privileged container execution.
* Privilege escalation requirements.
* Running as a non-root user.
* Required seccomp profile.
* Dropping unnecessary Linux capabilities.

This is a preventative security control because the insecure workload never runs.

### Evidence
<img width="439" height="339" alt="image" src="https://github.com/user-attachments/assets/e914ff50-b6f5-47aa-8819-275898443156" />

---

## Deploy a Compliant Pod

A compliant pod was created to prove that the Pod Security Standard does not block all workloads.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: compliant-probe
  namespace: tenant-a
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: probe
    image: busybox:1.36
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

The configuration was saved as:

```text
compliant-pod.yaml
```

The pod was applied:

```bash
kubectl apply -f compliant-pod.yaml
```

The pod status was checked using:

```bash
kubectl -n tenant-a get pod compliant-probe
```

### Result

The compliant pod should be successfully admitted and run inside the namespace.

This proves that the Pod Security Standard is enforcing security requirements rather than blocking every workload. Secure workloads that satisfy the restricted requirements are allowed to run.

### Evidence
<img width="659" height="664" alt="image" src="https://github.com/user-attachments/assets/e69a4a14-ba83-463f-adb0-c096c9e5832d" />

---

# Verification Commands

The final Zero Trust configuration can be verified using:

```bash
echo "=== Lab 2 Addendum verification ==="

kubectl -n tenant-a get networkpolicy \
  -o custom-columns=NAME:.metadata.name,TYPES:.spec.policyTypes

kubectl get namespace tenant-a \
  -o jsonpath='{.metadata.labels}' | tr ',' '\n' | grep pod-security

kubectl -n tenant-a get pods
```

### Expected Verification

The NetworkPolicy output should include:

```text
default-deny-egress
allow-dns-and-api-only
```

The namespace labels should include:

```text
pod-security.kubernetes.io/enforce=restricted
```

The pod output should show the compliant pod running.

### Evidence
<img width="975" height="297" alt="image" src="https://github.com/user-attachments/assets/5cf87aa3-8fa9-40d1-b4bf-89593d3480ae" />

---

# Short-Answer Questions

## Q1. Lab 2 gave you default-deny ingress. Explain why default-deny egress is the control an attacker cares about, and name two specific things they can no longer do once it is in place.

Default-deny egress is important because an attacker who compromises a pod is already inside the Kubernetes cluster. Default-deny ingress controls incoming traffic, but it does not prevent a compromised workload from initiating outbound connections.

With default-deny egress, an attacker can no longer freely communicate with unauthorized destinations.

Two things the attacker can no longer do are:

1. Contact a command-and-control server.
2. Exfiltrate sensitive data to unauthorized external systems or services.

---

## Q2. Your first egress policy broke every hostname lookup in the namespace. Explain why at the protocol level, and state what this implies about testing a deny-by-default control before shipping it to production.

When a pod accesses a service using a hostname such as:

```text
api.tenant-a.svc.cluster.local
```

the pod must first perform a DNS lookup to translate the hostname into an IP address.

DNS communication uses port 53. Therefore, when default-deny egress is enabled, DNS traffic is also blocked unless UDP and TCP port 53 are explicitly permitted.

This demonstrates that deny-by-default controls must be thoroughly tested before deployment to production. Testing must include legitimate dependencies such as DNS to ensure that security controls do not unintentionally break normal application functionality.

---

## Q3. Pod Security Standards rejected the privileged pod at admission. Contrast that with detecting a privileged pod after it has started: what does the preventative control give you that the detective one cannot?

Pod Security Standards provide a preventative security control by rejecting an insecure workload during admission.

In this lab, the privileged pod was rejected before it could run.

A detective control would identify the insecure pod only after it had already started. During that time, the workload may already have created a security risk.

Therefore, preventative admission control provides stronger protection because the prohibited workload never runs in the first place.

---

## Q4. Namespaces gave you isolation in Lab 2. Explain why a privileged container defeats that isolation and identify what the two workloads are sharing.

Kubernetes namespaces provide logical isolation between workloads, but containers running on the same node still share the host operating system kernel.

A privileged container receives significantly greater access to the underlying system. If a privileged workload is compromised or exploits the host environment, it may potentially bypass the logical isolation provided by namespaces and affect other workloads.

The workloads ultimately share the **host kernel**.

---

## Q5. Zero Trust is often summarised as "never trust, always verify". Using one example each from Z1 and Z2, state what is being verified and what assumption is being refused.

In **Task Z1**, the NetworkPolicy verifies whether outbound network communication is explicitly authorized.

DNS and required services are allowed, while other outbound communication is denied.

This refuses the assumption that a workload should automatically be trusted simply because it is located inside the Kubernetes cluster.

In **Task Z2**, Pod Security Standards verify whether a workload satisfies the required security restrictions before allowing it to run.

The privileged pod is rejected, while the compliant pod is admitted.

This refuses the assumption that a workload should automatically be trusted simply because it was submitted to an approved namespace.

---

# Security Best-Practices Checklist

* [x] Egress is denied by default, not only ingress.
* [x] Permitted egress is controlled using an explicit allow-list.
* [x] DNS access is explicitly considered as a required dependency.
* [x] Cross-tenant communication is tested after applying egress restrictions.
* [x] Legitimate permitted communication is tested to ensure the policy is not a blanket block.
* [x] DNS dependency failure is tested independently from routing restrictions.
* [x] The namespace enforces the restricted Pod Security Standard.
* [x] A privileged workload is rejected before it runs.
* [x] A compliant workload is successfully admitted.
* [x] Zero Trust controls are applied to both workload communication and workload admission.

---

# Cleanup

After saving all screenshots and evidence, the addendum resources can be removed using:

```bash
kubectl delete -f egress-policy.yaml --ignore-not-found

kubectl delete pod compliant-probe -n tenant-a --ignore-not-found

kubectl label namespace tenant-a \
  pod-security.kubernetes.io/enforce- \
  pod-security.kubernetes.io/enforce-version- \
  pod-security.kubernetes.io/warn- 2>/dev/null

rm -f egress-policy.yaml privileged-pod.yaml compliant-pod.yaml
```

---

# Conclusion

This Lab 2 addendum demonstrated how Zero Trust security extends traditional multi-tenant isolation controls.

Lab 2 established boundaries using namespaces, resource quotas and default-deny ingress policies. However, Zero Trust recognises that a compromised workload may already exist inside those boundaries.

Task Z1 addressed this risk using default-deny egress and micro-segmentation. Workloads in `tenant-a` were prevented from freely communicating with unauthorized destinations and were limited to explicitly permitted services and DNS dependencies.

Task Z2 introduced preventative admission control using Kubernetes Pod Security Standards. Instead of detecting insecure workloads after deployment, the restricted policy rejects prohibited workloads before they are allowed to run. A privileged pod was rejected, while a compliant pod was successfully admitted.

Overall, the lab demonstrates that Zero Trust requires security controls to verify both **where a workload is allowed to communicate** and **whether the workload itself is secure enough to run**. Trust is not granted based solely on network location or namespace membership. Instead, communication and workload admission must be explicitly authorized.
