# Lab 4: Access Control & Network Security

**Course:** IKB42603 Cloud Computing Security Essentials

**Lab:** Lab 4

**Topic:** Authentication, authorization, MFA, network segmentation and container hardening using Docker and Kubernetes

**Environment:** Docker, kind Kubernetes cluster `ccse-lab4`, Docker networks `frontend-net` and `backend-net`

## Lab Summary

This lab demonstrates access control and network security in a cloud environment. The lab distinguishes between authentication (AuthN), which verifies who a user or service is, and authorization (AuthZ), which determines what an authenticated identity is allowed to do.

The first session demonstrates HTTP Basic Authentication, Multi-Factor Authentication (MFA) using Time-Based One-Time Passwords (TOTP), and Kubernetes Role-Based Access Control (RBAC).

The second session demonstrates network segmentation using separate Docker networks, a default-deny firewall model using `iptables`, and container hardening using non-root execution, a read-only filesystem, dropped Linux capabilities and the `no-new-privileges` security option.

The lab also applies the principle of least privilege across identity, network and compute security.

---

# Session A: Authentication & Authorization

## Task 1: Authentication with a Password-Protected Service

Authentication verifies the identity of a user before access is granted.

A password file was created for the user `student`:

```bash
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssw0rd!' > htpasswd.txt
```

An Nginx configuration was created to require HTTP Basic Authentication:

```bash
cat > default.conf <<'EOF'
server {
  listen 80;

  location / {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;

    return 200 'Authenticated OK\n';
  }
}
EOF
```

The authentication service was started:

```bash
docker run --rm -d --name authsvc -p 8080:80 \
  -v $(pwd)/default.conf:/etc/nginx/conf.d/default.conf \
  -v $(pwd)/htpasswd.txt:/etc/nginx/.htpasswd nginx
```

An unauthenticated request was tested:

```bash
curl -s -o /dev/null -w 'no-creds: %{http_code}\n' \
  http://localhost:8080
```

Expected result:

```text
no-creds: 401
```

A request with valid credentials was then tested:

```bash
curl -s -u student:'P@ssw0rd!' http://localhost:8080
```

Expected result:

```text
Authenticated OK
```

### Result

The `401` response demonstrates that unauthenticated users are denied access. The successful response after providing valid credentials demonstrates authentication.

### Evidence

Authentication results should show:

* `401` when no credentials are provided.
* Successful access when valid credentials are provided.
<img width="975" height="797" alt="image" src="https://github.com/user-attachments/assets/75e16223-5d98-4e9e-a265-5eba7696a614" />

---

## Task 2: Multi-Factor Authentication with TOTP

Passwords alone provide only one authentication factor. This task adds a Time-Based One-Time Password (TOTP) as a second factor.

A Base32 secret was generated:

```bash
SECRET=$(head -c20 /dev/urandom | base32)

echo "Enroll this secret in an authenticator app: $SECRET"
```

The current six-digit TOTP code was generated:

```bash
oathtool --totp -b "$SECRET"
```

The user entered a code for validation:

```bash
read -p 'Enter the 6-digit code: ' CODE

[ "$CODE" = "$(oathtool --totp -b "$SECRET")" ] \
  && echo 'MFA OK' || echo 'MFA FAILED'
```

### Result

When the entered TOTP code matches the currently generated code, the output is:

```text
MFA OK
```

This demonstrates a second authentication factor. The password represents something the user knows, while the TOTP code represents a time-sensitive factor generated from a shared secret.

### Evidence

The screenshot should show the generated or entered TOTP code being successfully validated with:

```text
MFA OK
```
<img width="975" height="148" alt="image" src="https://github.com/user-attachments/assets/db017f23-16a8-46fe-879a-394d7ac3285d" />

---

## Task 3: Authorization with Kubernetes RBAC

Authentication verifies identity, while authorization determines what an authenticated identity is allowed to do.

A kind Kubernetes cluster was created:

```bash
kind create cluster --name ccse-lab4
```

An application namespace and ServiceAccount were created:

```bash
kubectl create namespace app

kubectl create serviceaccount dev -n app
```

A Role was created that allows only reading pods:

```bash
kubectl create role dev-role -n app \
  --verb=get,list \
  --resource=pods
```

The Role was assigned to the `dev` ServiceAccount:

```bash
kubectl create rolebinding dev-rb -n app \
  --role=dev-role \
  --serviceaccount=app:dev
```

The ServiceAccount identity was stored:

```bash
SA=system:serviceaccount:app:dev
```

Authorization checks were performed:

```bash
kubectl auth can-i list pods -n app --as=$SA
```

Expected result:

```text
yes
```

Creating deployments was tested:

```bash
kubectl auth can-i create deploy -n app --as=$SA
```

Expected result:

```text
no
```

Deleting pods was also tested:

```bash
kubectl auth can-i delete pods -n app --as=$SA
```

Expected result:

```text
no
```

### Result

The `dev` ServiceAccount is authorized only to read pods. It cannot create deployments or delete pods.

This demonstrates the principle of least privilege because the identity receives only the permissions required for its role.

### Evidence

The RBAC evidence should show:

```text
yes
no
no
```
<img width="975" height="877" alt="image" src="https://github.com/user-attachments/assets/9e7ba23b-aee8-4c4f-adb1-6f7126ab7eb9" />

---

# Session B: Network Security & Container Hardening

## Task 4: Network Segmentation

Two separate Docker networks were created:

```bash
docker network create frontend-net

docker network create backend-net
```

The database was connected only to the backend network:

```bash
docker run -d --name db --network backend-net redis:alpine
```

The application was connected to the backend network:

```bash
docker run -d --name app --network backend-net nginx
```

The application was also connected to the frontend network:

```bash
docker network connect frontend-net app
```

The web container was connected only to the frontend network:

```bash
docker run -d --name web --network frontend-net nginx
```

The web container attempted to reach the database:

```bash
docker exec web sh -c \
  'apk add -q curl; curl -s -m 3 db:6379 || echo BLOCKED'
```

Expected result:

```text
BLOCKED
```

The application container then tested connectivity to the database:

```bash
docker exec app sh -c \
  'apk add -q curl; nc -z -w3 db 6379 && echo REACHABLE'
```

Expected result:

```text
REACHABLE
```

### Result

The web container cannot directly reach the database because they do not share a Docker network.

The application container can reach the database because both are connected to `backend-net`.

This demonstrates network segmentation and reduces the possibility of lateral movement. If the internet-facing web container is compromised, the attacker cannot directly access the database through the network.

### Evidence

The screenshot should show:

```text
BLOCKED
REACHABLE
```
<img width="467" height="374" alt="image" src="https://github.com/user-attachments/assets/73fb4301-27a0-4085-8061-fb274d762953" />

---

## Task 5: Default-Deny Firewall Rules

A default-deny firewall model was configured inside a temporary container.

The container was granted the `NET_ADMIN` capability for firewall configuration:

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c '\
  apk add -q iptables; \
  iptables -P INPUT DROP; \
  iptables -A INPUT -p tcp --dport 443 -j ACCEPT; \
  iptables -A INPUT -i lo -j ACCEPT; \
  iptables -L INPUT -n'
```

### Result

The firewall policy drops incoming traffic by default:

```text
INPUT policy DROP
```

Only TCP port `443` is explicitly allowed, together with loopback traffic.

This demonstrates the default-deny principle. Traffic is blocked unless a specific rule explicitly permits it.

### Security Principle

This approach is similar to cloud security groups and network security rules. Only the ports and protocols required by a service should be exposed.

### Evidence

The screenshot should show the firewall ruleset with:

* Default `DROP` policy.
* Explicit TCP port `443` allow rule.
* Loopback interface allow rule.
<img width="975" height="277" alt="image" src="https://github.com/user-attachments/assets/b155a2fd-9b32-45ce-90e5-06dd8211bd42" />

---

## Task 6: Container and Host Hardening

A hardened Nginx container was started with several security controls:

```bash
docker run -d --name hardened \
  --user 1000:1000 \
  --read-only \
  --cap-drop=ALL \
  --security-opt no-new-privileges \
  --tmpfs /tmp \
  nginxinc/nginx-unprivileged
```

The configuration was inspected:

```bash
docker inspect hardened --format \
  'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'
```

Expected configuration:

```text
User=1000:1000 ReadOnly=true
```

The container image was scanned for known vulnerabilities:

```bash
docker run --rm aquasec/trivy image \
  --severity HIGH,CRITICAL \
  nginx:alpine | head -20
```

### Hardening Measures

#### 1. Non-Root User

```bash
--user 1000:1000
```

The container process runs without root privileges.

This reduces the impact of a compromised application because an attacker does not automatically receive root-level permissions inside the container.

#### 2. Read-Only Root Filesystem

```bash
--read-only
```

The root filesystem cannot be modified.

This reduces the ability of an attacker or malicious process to write persistent scripts, backdoors or modified application files.

#### 3. Drop Linux Capabilities

```bash
--cap-drop=ALL
```

Linux capabilities provide privileged operations that many applications do not require.

Dropping unnecessary capabilities reduces the container's attack surface and limits actions available to a compromised process.

#### 4. Prevent New Privileges

```bash
--security-opt no-new-privileges
```

Processes inside the container cannot gain additional privileges through mechanisms such as privilege escalation using setuid binaries.

#### 5. Temporary Writable Storage

```bash
--tmpfs /tmp
```

Temporary files are stored in memory rather than the persistent container filesystem.

Temporary data is removed when the container stops.

### Result

The hardened configuration applies defence in depth by reducing privileges, limiting filesystem access and reducing the capabilities available to the running process.

The Trivy scan identifies known vulnerabilities in the container image so that vulnerable images can be identified and remediated.

### Evidence

Evidence should include:

* Docker inspection showing the non-root user.
* Read-only filesystem enabled.
* Trivy vulnerability scan output.
<img width="975" height="532" alt="image" src="https://github.com/user-attachments/assets/ed8cfab6-db23-4061-b210-5fc31b50df55" />

---

# Verification Commands

The following commands verify the RBAC configuration and container hardening settings.

Verify the Kubernetes RoleBinding:

```bash
kubectl get rolebinding dev-rb -n app -o yaml
```

Verify dropped Linux capabilities:

```bash
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```

Expected output should include:

```text
["ALL"]
```
<img width="975" height="555" alt="image" src="https://github.com/user-attachments/assets/0918f07d-63b7-46ef-b318-cbe0a68fc113" />

---

# Short-Answer Questions

## Q1. Explain the difference between authentication and authorization using Tasks 1 and 3.

Authentication verifies who a user or identity is. In Task 1, authentication was demonstrated using HTTP Basic Authentication, where valid credentials were required before access was granted. Task 2 further strengthened authentication by validating a TOTP code as an additional authentication factor.

Authorization determines what an authenticated identity is allowed to do. In Task 3, Kubernetes RBAC defined the permissions of the `dev` ServiceAccount. Although the ServiceAccount was recognized as a valid identity, it was allowed only to read pods and was denied permission to create deployments or delete pods.

Therefore, authentication answers **"Who are you?"**, while authorization answers **"What are you allowed to do?"**

## Q2. Why is MFA effective, and which attacks does it help defend against?

Multi-Factor Authentication is effective because access requires more than one authentication factor. A stolen password alone is not sufficient if the attacker does not also possess the second factor.

In this lab, authentication can combine something the user knows, such as a password, with a time-based one-time password generated from a shared secret.

MFA helps reduce the effectiveness of attacks involving stolen or reused credentials, including credential stuffing and password compromise. An attacker who obtains only the password may still be unable to authenticate without the additional factor.

## Q3. How does network segmentation limit the damage of a compromised web server?

Network segmentation reduces the ability of an attacker to move laterally between systems.

In Task 4, the web container was connected only to `frontend-net`, while the database was connected only to `backend-net`. Because the web container and database did not share a network, the web container could not directly reach the database.

If the web server is compromised, network segmentation helps contain the attack by preventing direct access to systems that are not required for the web server's operation.

## Q4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?

A default-deny firewall policy blocks traffic unless an explicit rule allows it.

In Task 5, the default `INPUT` policy was set to `DROP`, and only TCP port `443` was explicitly permitted. This prevents unnecessary or unintended services from being exposed.

The same principle is used by cloud security groups and network security rules, where administrators should permit only the specific ports, protocols and sources required by a workload.

## Q5. List the hardening measures you applied and the attack surface each one removes.

| **Hardening Measure**              | **Attack Surface Reduced**                                                                       |
| ---------------------------------- | ------------------------------------------------------------------------------------------------ |
| `--user 1000:1000`                 | Limits the impact of a compromised process by preventing the application from running as root.   |
| `--read-only`                      | Reduces the ability to write persistent malicious files or modify the container filesystem.      |
| `--cap-drop=ALL`                   | Removes unnecessary privileged Linux capabilities that could be abused by a compromised process. |
| `--security-opt no-new-privileges` | Prevents processes from gaining additional privileges during execution.                          |
| `--tmpfs /tmp`                     | Limits temporary data to memory and reduces persistent filesystem writes.                        |

---

# Security Best-Practices Checklist

* [x] Service requires authentication and unauthenticated requests are rejected.
* [x] MFA or a second authentication factor is generated and validated.
* [x] Authorization is enforced through Kubernetes RBAC.
* [x] Least privilege is applied to the developer ServiceAccount.
* [x] Network segmentation prevents the frontend tier from directly reaching the database.
* [x] Default-deny firewall rules allow only explicitly required traffic.
* [x] Container processes run as a non-root user.
* [x] The root filesystem is configured as read-only.
* [x] Linux capabilities are dropped.
* [x] New privilege escalation is prevented.
* [x] The container image is scanned for known vulnerabilities.

---

# Cleanup

After collecting all screenshots and evidence, the lab environment can be removed.

Remove the containers:

```bash
docker rm -f authsvc db app web hardened 2>/dev/null
```

Remove the Docker networks:

```bash
docker network rm frontend-net backend-net 2>/dev/null
```

Remove the Kubernetes cluster:

```bash
kind delete cluster --name ccse-lab4
```

---

# Conclusion

This lab demonstrated that cloud security requires controls across multiple layers.

Authentication verifies the identity of users and services, while authorization ensures that authenticated identities receive only the permissions required for their role. MFA strengthens authentication by requiring an additional factor.

Network segmentation reduces the attacker's ability to move laterally between application tiers, while a default-deny firewall model ensures that unnecessary traffic is blocked.

Container hardening further reduces attack surface by running applications as non-root users, using read-only filesystems, dropping unnecessary Linux capabilities and preventing privilege escalation.

Together, these controls demonstrate the principle of least privilege across identity, network and compute security.

---
