# Secure Isolation and Multi-Tenancy

**Environment:** kind Kubernetes cluster `ccse-lab2`, Calico CNI, Docker volume `ccse-vol`

## Lab Summary

This lab demonstrates secure isolation in a multi-tenant cloud environment. Two tenants are represented using separate Kubernetes namespaces, `tenant-a` and `tenant-b`, running on the same shared cluster. The lab first demonstrates the default-open behaviour of Kubernetes networking, where workloads in different namespaces can communicate unless network policies are applied.

Security controls were then introduced to improve tenant isolation. ResourceQuota was used to control shared compute resources, NetworkPolicy was used to restrict network communication, and namespace-scoped RBAC was used to protect tenant secrets.

The lab also demonstrates data remanence in container storage by creating and deleting sensitive data in a Docker volume. An overwrite-before-delete technique was then used to demonstrate a more secure deletion approach. In real cloud environments, cryptographic erasure is generally preferred because customers do not directly control the underlying physical storage blocks.

## Setup: Cluster with Policy Enforcement

The lab cluster was created using kind with the default CNI disabled. Calico was then installed so that Kubernetes NetworkPolicy rules could be enforced.

### Commands

```bash
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s
```

### Result

The `ccse-lab2` kind cluster was created with the default CNI disabled. Calico was subsequently installed to provide NetworkPolicy enforcement for the isolation experiments.

### Evidence

<img width="781" height="529" alt="image" src="https://github.com/user-attachments/assets/5417d6c0-4f74-4707-a440-19b6bc5f3048" />
<img width="975" height="560" alt="image" src="https://github.com/user-attachments/assets/26d4ddcd-8735-4899-a78b-aea503cc95ad" />

## Task 1: Two Tenants on One Cluster

Two customers were modelled as separate Kubernetes namespaces:

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
```

Each namespace was given a simple Nginx web deployment and ClusterIP service.

```bash
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx

kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80

kubectl get pods,svc -n tenant-a
kubectl get pods,svc -n tenant-b
```

### Result

Both tenants were successfully deployed on the same Kubernetes cluster. Kubernetes namespaces provide logical separation between the workloads while allowing both tenants to share the underlying cluster infrastructure.

The Nginx pods and services were created successfully in both `tenant-a` and `tenant-b`.

### Evidence
<img width="910" height="657" alt="image" src="https://github.com/user-attachments/assets/7217e305-311f-4bf9-b661-e217be665803" />

## Task 2: Observe the Default-Open Risk

The ClusterIP address of the Tenant B web service was first obtained.

```bash
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo
```

### Observed Tenant B Service IP

```text
10.96.22.249
```

A temporary curl pod was then launched inside `tenant-a` to access the Tenant B service.

```bash
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://10.96.22.249 -o /dev/null -w 'HTTP %{http_code}\n'
```

### Observed Output

```text
HTTP 200
```

### Result

The `HTTP 200` response demonstrates that a pod in `tenant-a` was able to reach the web service in `tenant-b`.

This demonstrates the default-open network behaviour in Kubernetes. Namespace separation by itself does not automatically prevent network communication between tenants. In a multi-tenant cloud environment, this creates a security risk because a compromised workload belonging to one tenant could potentially communicate with or attack workloads belonging to another tenant.

### Evidence
<img width="975" height="90" alt="image" src="https://github.com/user-attachments/assets/750e02d6-d79e-4976-a7c3-fb8dccef4ea3" />

## Task 3: Contain the Noisy Neighbour with ResourceQuota

A ResourceQuota was applied to `tenant-a` to limit the amount of shared cluster resources that the tenant could request.

### Command

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF
```

The quota was then inspected using:

```bash
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

### Observed Quota

```text
pods              Used: 1   Hard: 5
requests.cpu      Used: 0   Hard: 1
requests.memory   Used: 0   Hard: 512Mi
```

### Result

The ResourceQuota limits `tenant-a` to a maximum of five pods, one CPU core of total requested CPU and 512 MiB of total requested memory.

This provides compute and resource isolation by preventing one tenant from consuming an excessive amount of shared cluster capacity. It helps reduce the risk of a noisy-neighbour situation affecting other workloads.

### Evidence
<img width="491" height="304" alt="image" src="https://github.com/user-attachments/assets/070e9706-259f-4efc-a740-acfe58c82abd" />

## Task 4: Default-Deny Network Isolation

A default-deny ingress NetworkPolicy was applied to `tenant-b`.

### Command

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
```

### Result

The NetworkPolicy selects all pods in `tenant-b` using:

```yaml
podSelector: {}
```

Because the policy contains no ingress allow rules, incoming traffic to the selected pods is denied by default.

This implements the **default-deny principle**, where traffic is blocked unless it is explicitly permitted.

### Evidence
<img width="710" height="297" alt="image" src="https://github.com/user-attachments/assets/4531df73-7eed-4b90-a4d6-8197b368312e" />


### Retest After NetworkPolicy

The lab requires the same cross-tenant probe from Task 2 to fail after the NetworkPolicy is applied.

However, the first retest produced the following error:

```text
Error from server (Forbidden): pods "probe" is forbidden: failed quota: tenant-a-quota: must specify requests.cpu; requests.memory
```

This error demonstrates that the ResourceQuota was enforcing its resource requirements, but it does **not** prove that the NetworkPolicy blocked network traffic.

To perform a valid NetworkPolicy test, the temporary probe pod must specify CPU and memory requests:

```bash
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  --requests='cpu=100m,memory=64Mi' \
  -- curl -s -m 5 http://10.96.22.249 -o /dev/null -w 'HTTP %{http_code}\n'
```

### Expected Result

```text
HTTP 000
```

or a timeout/error from curl.

A failed request after the policy is applied, together with the earlier `HTTP 200` result, provides the required before-and-after evidence of network isolation.

### Evidence
<img width="975" height="100" alt="image" src="https://github.com/user-attachments/assets/4a557b36-c6a6-41c2-84d1-66c0ff0b6218" />

> **Evidence note:** The existing `4.1-check-network.png` screenshot shows the ResourceQuota rejection rather than a network timeout. Therefore, it should not be presented as definitive proof that the NetworkPolicy blocked the traffic. A corrected probe with CPU and memory requests should be captured for the strongest final evidence.

## Task 5: Storage and Secret Isolation

Each tenant was provided with its own Kubernetes Secret.

### Create Tenant Secrets

```bash
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B
```

A ServiceAccount was then created for `tenant-a`.

```bash
kubectl -n tenant-a create serviceaccount app-a
```

A namespace-scoped Role allowing the ServiceAccount to read Secrets was created:

```bash
kubectl -n tenant-a create role reader --verb=get --resource=secrets
```

The Role was bound to the ServiceAccount:

```bash
kubectl -n tenant-a create rolebinding rb \
  --role=reader \
  --serviceaccount=tenant-a:app-a
```

### Authorization Verification

The ServiceAccount identity was defined as:

```bash
SA=system:serviceaccount:tenant-a:app-a
```

The following commands were used to test authorization:

```bash
kubectl auth can-i get secrets -n tenant-a --as=$SA
kubectl auth can-i get secrets -n tenant-b --as=$SA
```

### Expected Result

```text
yes
no
```

### Result

The ServiceAccount is granted permission to read Secrets within `tenant-a` because the Role and RoleBinding are namespace-scoped.

The same ServiceAccount does not receive permission to read Secrets in `tenant-b`. This demonstrates tenant-scoped access control using Kubernetes RBAC.

The result shows that namespace separation combined with RBAC can prevent a tenant identity from accessing another tenant's secrets.

### Evidence
<img width="975" height="604" alt="image" src="https://github.com/user-attachments/assets/d5c6224e-a085-4fae-a5c2-126d019543a5" />

### RBAC Note

The lab guide specifies the ServiceAccount name `app-a`. The final report should use `app-a` consistently in the RoleBinding and `can-i` verification.

If the screenshot contains a different ServiceAccount name such as `appa`, the screenshot should be checked against the actual configuration before final submission.

## Task 6: Data Remanence and Secure Deletion

Data remanence refers to the possibility that data may remain recoverable after normal deletion. This task demonstrated the concept using a Docker volume.

### Normal Deletion and Remanence Scan

Sensitive data was created inside the Docker volume:

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
   grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```

### Observed Output

```text
scan-done
```

The file was removed using `rm`, and no matching content was found in the remaining visible files.

However, normal deletion does not necessarily guarantee that the underlying storage blocks have been overwritten. Therefore, the absence of the string from the visible directory should not be interpreted as proof that all underlying storage copies or physical blocks have been securely destroyed.

### Secure Wipe Demonstration

The second command overwrites the file with zero bytes before deleting it:

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
   dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; \
   echo wiped'
```

### Observed Output

```text
1+0 records in
1+0 records out
1024 bytes copied
wiped
```

### Result

The first command demonstrates normal file deletion. The `rm` operation removes the file reference but does not intentionally overwrite the data that previously occupied the file.

The second command demonstrates an overwrite-before-delete technique. The contents of `phi2.txt` are overwritten with zeros before the file is removed.

In cloud environments, however, customers generally do not control the underlying physical storage blocks, replicas, snapshots or storage infrastructure. Therefore, **cryptographic erasure** is the more practical cloud solution. By destroying the encryption key, encrypted data that remains on underlying storage becomes computationally unusable.

### Evidence
<img width="975" height="267" alt="image" src="https://github.com/user-attachments/assets/6d57c9af-a091-4e45-b6f5-de0c60342fdb" />

## Verification Commands

The final configuration can be verified using the following commands:

```bash
kubectl get networkpolicy -A

kubectl describe resourcequota tenant-a-quota -n tenant-a
```

### Expected NetworkPolicy Verification

```text
tenant-b   default-deny-ingress
```

### Expected ResourceQuota Verification

```text
Name:            tenant-a-quota
Namespace:       tenant-a

pods             Hard: 5
requests.cpu     Hard: 1
requests.memory  Hard: 512Mi
```

These commands provide final evidence that the network isolation policy and resource quota were configured in the cluster.
<img width="332" height="143" alt="image" src="https://github.com/user-attachments/assets/4a774f0d-efda-4c05-b477-08bf8b86b2ba" />

## Short-Answer Questions

### Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?

Kubernetes namespaces provide logical separation of resources, but they do not automatically prevent network communication between namespaces. Without an appropriate NetworkPolicy, pods can communicate across namespace boundaries.

This is dangerous in a multi-tenant cloud because a compromised workload belonging to one tenant could potentially discover, access or attack workloads belonging to another tenant. This increases the risk of unauthorized access, information disclosure and lateral movement.

### Q2. Explain the default-deny principle and how your NetworkPolicy implements it.

The default-deny principle means that network traffic should be blocked unless it is explicitly allowed.

The NetworkPolicy applied to `tenant-b` uses:

```yaml
podSelector: {}
policyTypes:
- Ingress
```

The empty `podSelector` selects all pods in `tenant-b`. Because no ingress rules are defined, incoming traffic is denied by default.

Therefore, a workload outside `tenant-b`, such as the probe in `tenant-a`, should no longer be able to reach the Tenant B web service after the policy is enforced.

### Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?

Virtual machines generally provide stronger isolation because each VM has its own guest operating system and kernel. Containers are lighter and more efficient but share the host operating system kernel.

Because containers share the kernel, a serious kernel-level vulnerability could potentially affect multiple workloads on the same host. A VM boundary should therefore be considered when tenants are untrusted, workloads handle highly sensitive or regulated information, stronger tenant separation is required, or the risks associated with a shared kernel are unacceptable.

### Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?

Data remanence is the possibility that information remains recoverable on storage media after a file has been deleted.

In cloud environments, customers generally do not control the physical storage blocks, replicas, snapshots or hardware used by the provider. As a result, simply deleting a file may not guarantee that every copy of the underlying data has been physically destroyed.

Cryptographic erasure addresses this problem by destroying the encryption key used to protect the data. Even if encrypted remnants remain on storage, the data becomes unreadable without the key.

### Q5. Which of the three isolation dimensions did each task exercise?

| **Task**                                    | **Isolation Dimension**              | **Explanation**                                                                                                                    |
| ------------------------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| Task 1 – Two Tenants on One Cluster         | Compute Isolation                    | Tenants were separated into different Kubernetes namespaces while sharing the same cluster infrastructure.                         |
| Task 2 – Default-Open Risk                  | Network Isolation Risk               | The successful `HTTP 200` response demonstrated that namespace separation alone does not block cross-tenant network communication. |
| Task 3 – ResourceQuota                      | Compute and Resource Isolation       | ResourceQuota limited CPU, memory and pod usage to reduce the risk of one tenant exhausting shared resources.                      |
| Task 4 – Default-Deny NetworkPolicy         | Network Isolation                    | NetworkPolicy was used to deny incoming traffic to `tenant-b` unless explicitly allowed.                                           |
| Task 5 – Secret Isolation                   | Storage and Secret Isolation         | Namespace-scoped RBAC restricted the ServiceAccount's access to secrets so it could access `tenant-a` but not `tenant-b`.          |
| Task 6 – Data Remanence and Secure Deletion | Storage Isolation and Data Lifecycle | Normal deletion and overwrite-before-delete were demonstrated to explain data remanence and secure deletion concepts.              |

## Security Best-Practices Checklist

* [x] Tenants are separated into distinct Kubernetes namespaces.
* [x] ResourceQuota limits shared CPU, memory and pod consumption.
* [x] Per-tenant secrets are protected using namespace-scoped RBAC.
* [x] Default-deny NetworkPolicy is configured for `tenant-b`.
* [x] Cross-tenant communication was demonstrated before applying the NetworkPolicy.
* [ ] A corrected post-policy probe should be captured to provide definitive evidence that cross-tenant traffic is blocked.
* [x] Normal deletion and overwrite-before-delete were demonstrated for data remanence.
* [x] Cryptographic erasure was identified as the preferred approach for cloud storage.

## Cleanup

After all evidence has been saved, the lab environment can be removed using:

```bash
kind delete cluster --name ccse-lab2

docker volume rm ccse-vol
```

## Conclusion

This lab demonstrated that secure multi-tenancy requires several complementary isolation controls rather than relying on namespace separation alone.

Kubernetes namespaces provide logical compute separation, but the Task 2 experiment showed that network communication between tenants remains possible when no NetworkPolicy is applied. ResourceQuota provides additional compute isolation by limiting the resources that a tenant can consume. NetworkPolicy applies network segmentation through a default-deny approach, while namespace-scoped RBAC protects tenant secrets from unauthorized access.

The data remanence exercise also demonstrated that normal file deletion does not necessarily guarantee complete removal of underlying data. Although overwrite-before-delete can reduce recoverability in controlled environments, cryptographic erasure is more appropriate for cloud environments where the customer does not control the physical storage infrastructure.

Overall, the lab shows that effective cloud multi-tenancy depends on **layered isolation across compute, network and storage**, supported by appropriate access controls and secure data lifecycle practices.
