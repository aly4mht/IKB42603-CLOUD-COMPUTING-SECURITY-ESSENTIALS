# LAB 4.1 : Zero Trust: Service Identity with Mutual TLS

*Every service authenticates every caller — binding authorisation to a certificate instead of an IP address*

**NAME:** ALYA LIYANA BINTI MAHAT (L01-B01)

**STUDENT ID:** 52215124600

# Status of This Addendum

> **Caution:** This task was previously listed as an optional *Expansion Idea* in Lab 4. It is now **core and assessed**. Zero Trust is a named domain in CSA CCSK v5 and cannot be left to the students who happen to have spare time. Complete Task Z3 after Lab 4 Session B and fold the evidence into your Lab 4 report.

> **Note:** This is the second of two Zero Trust addenda, and the third of three tasks in the series. Tasks Z1 (egress default-deny) and Z2 (admission control) are in the **Lab 2 Addendum: Zero Trust Micro-Segmentation & Admission Control**. You do not need to have completed them to do this task, but the argument below assumes you have read their framing.

# The Idea You Are Testing

Lab 4 authenticated **users**: a password in Task 1, a second factor in Task 2, an RBAC role in Task 3. Then in Tasks 4 and 5 it segmented the **network**, so that the frontend could not reach the database directly. Notice the asymmetry — users had to prove who they were, but services were trusted purely because of where they sat.

That asymmetry is the gap Zero Trust closes. Its third property, after deny-by-default and verifying what a workload is, is this:

> **Security tip: Bind authorisation to a cryptographic identity the workload holds, not to an address it happens to occupy.** In mutual TLS both ends present a certificate, so the server knows which service is calling and the client knows it is not talking to an impostor. The certificate — not the source IP, not the namespace, not the subnet — is the identity.

This matters more in cloud than anywhere else. Addresses are ephemeral: a pod that dies is replaced at a different IP within seconds, and that address may be reassigned to an entirely different tenant's workload minutes later. A firewall rule written against an address is a rule written against something that does not hold still.

# Course & Assessment Mapping

| **Item**                    | **Mapping**                                                                                                |
| --------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Extends**                 | Lab 4 — Access Control & Network Security (Weeks 7–8)                                                      |
| **Course Learning Outcome** | CLO2 — Construct secure cloud operations (VBE3)                                                            |
| **Lecture topics**          | Week 5 (Enforcing Access Control) · Week 9 (Security Design Patterns II — network security)                |
| **CSA CCSK v5 domains**     | Domain 7 (Infrastructure & Networking) · Domain 12 (Related Technologies — Zero Trust)                     |
| **Assessment**              | Folded into the **Lab 4 report** under the existing evidence quality and conceptual understanding criteria |
| **Builds on**               | Lab 3 (asymmetric keys, certificates and TLS) · Lab 4 Tasks 1–3 (authentication vs authorisation)          |

## Technical Prerequisites

* OpenSSL — pre-installed on macOS and Linux; use Git Bash or WSL on Windows.
* Two terminal windows. The server runs in the foreground in one; the tests run in the other.
* The certificate concepts from Lab 3, Task 2 — key pairs, signing, and the role reversal between public and private keys.

# Task Z3 — Service Identity with Mutual TLS

## Step 1 — Stand up a private certificate authority

In an organisation this is the service mesh's CA, or an internal PKI. Its job is to be the single authority that decides which identities exist.

```bash
mkdir -p ~/ikb42603-mtls && cd ~/ikb42603-mtls

openssl req -x509 -newkey rsa:2048 -nodes -keyout ca.key -out ca.crt -days 365 \
  -subj "/CN=IKB42603 Service CA"

openssl x509 -in ca.crt -noout -subject
```
<img width="875" height="102" alt="image" src="https://github.com/user-attachments/assets/20fb1ded-32b0-4045-ab76-86a0a4102f6b" />
<img width="975" height="153" alt="image" src="https://github.com/user-attachments/assets/5a9323e2-c8ef-4534-9044-a097f65ae906" />


## Step 2 — Issue an identity to each service

Two services: an api service that will accept connections, and a billing service that will make them. Each gets its own key pair and a certificate signed by the CA.

```bash
# The server: the api service

openssl req -newkey rsa:2048 -nodes -keyout server.key -out server.csr \
  -subj "/CN=api.internal"

openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 90

# The client: the billing service

openssl req -newkey rsa:2048 -nodes -keyout client.key -out client.csr \
  -subj "/CN=billing.internal"

openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out client.crt -days 90

openssl x509 -in client.crt -noout -subject -issuer
```

> **Note:** Note the -days 90 on the service certificates against -days 365 on the CA. Short-lived workload identities are deliberate: a stolen certificate expires on its own, which bounds the damage without anyone having to notice the theft. Production service meshes issue certificates lasting hours, not months. In your report, explain the trade-off that makes very short lifetimes practical for services but not for a human's credentials.

<img width="975" height="324" alt="image" src="https://github.com/user-attachments/assets/6e182840-a140-45c1-b16d-760835ce7c34" />


## Step 3 — Require a client certificate

-Verify 1 is the whole point: the server will not serve a caller that cannot prove its identity. Leave this running in your first terminal and open a second one for the tests.

```bash
openssl s_server -accept 8443 -cert server.crt -key server.key \
  -CAfile ca.crt -Verify 1 -www -tls1_2
```

> **Note:** -tls1_2 is pinned deliberately. Under TLS 1.3 the client certificate is sent after the server has finished its own handshake, so authentication failures surface late and asymmetrically. Pinning TLS 1.2 makes the refusal in test 4b immediate and visible from the caller — which is what you want the first time you meet this.

<img width="975" height="215" alt="image" src="https://github.com/user-attachments/assets/a10372f2-3ec4-4b60-85d4-90dc41ec8210" />


# Step 4 — Three callers, three outcomes

```bash
# 4a. The billing service, holding an identity signed by our CA - EXPECT SUCCESS

echo | openssl s_client -connect localhost:8443 -tls1_2 \
  -cert client.crt -key client.key -CAfile ca.crt 2>&1 \
  | grep -E "Verification|Verify return code"

# 4b. An anonymous caller with no identity at all - EXPECT REFUSAL

echo | openssl s_client -connect localhost:8443 -tls1_2 -CAfile ca.crt 2>&1 \
  | grep -iE "alert|handshake failure"

# 4c. An attacker presenting a self-signed identity - CHECK THE SERVER TERMINAL

openssl req -x509 -newkey rsa:2048 -nodes -keyout rogue.key -out rogue.crt \
  -days 30 -subj "/CN=attacker.internal"

echo | openssl s_client -connect localhost:8443 -tls1_2 \
  -cert rogue.crt -key rogue.key -CAfile ca.crt 2>&1 | tail -3
```

### Expected results, verified:

| **Caller**          | **Where the outcome appears** | **What you should see**                                                   |
| ------------------- | ----------------------------- | ------------------------------------------------------------------------- |
| 4a — valid identity | Client terminal               | Verification: OK and Verify return code: 0 (ok)                           |
| 4b — no identity    | Client terminal               | sslv3 alert handshake failure — the server refuses to proceed             |
| 4c — untrusted CA   | **Server** terminal           | verify error:num=18:self-signed certificate naming CN = attacker.internal |

> **Caution:** Case 4c is the instructive one. The client sees a mostly normal-looking handshake, because Verify return code: 0 refers to the client's verification of the **server**, not the server's verification of the client. The rejection is visible only in the server's terminal. Client-side authentication failures surface asymmetrically, and an engineer who debugs only from the caller's side will wrongly conclude the connection succeeded. Capture the server terminal as your evidence.

<img width="975" height="260" alt="image" src="https://github.com/user-attachments/assets/8fce3a10-7e70-4446-a948-d794d8e21f55" />


# Step 5 — What the attacker must steal instead

Under Lab 4's original design, an attacker who compromised any workload on the permitted network segment could call the api service, because reaching it was sufficient to be trusted by it. Under mutual TLS, network reachability buys them nothing. Confirm what has actually changed:

```bash
# The attacker can still reach the service - the network is unchanged

nc -z -v localhost 8443

# But cannot transact with it without a CA-signed private key

ls -l client.key client.crt
```
<img width="975" height="331" alt="image" src="https://github.com/user-attachments/assets/3bc5d44c-be1e-428d-9d15-23e055e46579" />


In your Lab 4 report, answer precisely: an attacker has compromised a workload on the same segment and can reach the api service. State what they must now obtain to call it, and name one control that would limit the damage once they obtain it.

> =To successfully call the api service, the attacker must obtain a valid CA-signed private key and its corresponding certificate since under mutual TLS, network reachability alone grants zero access. For example, client.key and client.crt belonging to an authorized service like billing.internal.
>
> One control that would limit the damage once obtained is short-lived workload certificates such as certificates valid for hours or days with automated rotation. If an attacker manages to steal a private key, short expiration lifetimes automatically bound the window of opportunity for misuse, rendering the stolen key useless once it expires on its own without requiring manual revocation.


# 2. Short-Answer Questions

### 1. In mutual TLS the identity is the certificate, not the IP address. Explain why that distinction matters specifically in a cloud environment where addresses are ephemeral and reassigned between tenants.

= In cloud and containerized environments, IP addresses are very short lived . When a container or pod dies, it is instantly replaced at a different IP address in a matter of seconds. Also, that original IP address could be reallocated to the workload of a totally different tenant within minutes. Traditional firewall rules written against IP addresses attempt to secure an address that is not static. Mutual TLS solves this by binding authorization to a cryptographic certificate held directly by the workload instead of an ephemeral address, ensuring that security rules remain anchored to identity regardless of network changes.

### 2. Lab 4 Task 1 authenticated a user with a password and Task 2 added a second factor. Explain what mTLS is doing that is analogous, and what it is doing that has no user-authentication equivalent.

= Analogous action by mTLS is that just as passwords and second-factor tokens establish the identity of a human user to a system, client certificates in mTLS establish the cryptographic identity of a service to another service before access is granted.

Normal user authentication is generally unidirectional which means the user authenticates to the server, but the user does not present an X.509 certificate to the server. mTLS is mutual cryptographic authentication, where both sides present certificates in the handshake. This ensures that the server authenticates the caller and the client verifies while it's not talking to a fraudster. Plus, mTLS leverages programmatic and non-interactive key management with no human intervention.

### 3. The rogue-certificate rejection was invisible from the client side. Explain what happened at the protocol level, and state what it teaches about where to look when debugging an authentication failure.

= At protocol level, the client verifies the server's certificate first in the TLS 1.2 handshake. As the client trusts the root CA (ca.crt), client-side verification returns Verify return code 0 (ok). However, when the server checks the untrusted certificate of the caller (rogue.crt), it sees an untrusted root/self-signed certificate, kills the session and logs verify error num=18.

When debugging an authentication failure, an engineer who inspects only the caller terminal will see a successful server lookup and wrongly assume the connection succeeded. This teaches that mTLS authentication failures must always be debugged by inspecting server-side logs.

### 4. Service certificates were issued for 90 days and the CA for 365. Explain the security argument for short-lived workload identities, and why the same approach is harder to apply to human credentials.

=The security argument for short-lived workload identities is short-lived certificates (e.g. 90 days or a few hours) reduce the damage window in case of a compromised private key. A stolen certificate will automatically expire quickly on its own, without the need for complex manual revocation processes.

The same approach is harder to apply because short lifetimes work well for workloads such as automated service meshes and PKI platforms that can easily issue and rotate certificates in the background. Short lives for human credentials are much harder as it forcing humans to re-authenticate or re-issue keys manually every few hours creates huge operational friction.

### 5. An attacker compromises a workload on a permitted network segment. Compare precisely what they can do under Lab 4's original segmentation design versus under mTLS, and name what they must steal to close the gap.

= Under original segmentation design, attackers compromising any workload on that permitted segment could directly call and transact with the API service since reaching the API service network segment was sufficient to gain trust. While under mTLS, network reachability alone grants zero authorization or access.

The attacker must steal the legitimate service's private key (client.key) and CA-signed certificate to close the gap.

### 6. Zero Trust is often summarised as 'never trust, always verify'. Using this task, state what is being verified, by whom, and what assumption is being refused.

* **What is being verified:** The cryptographic workload identity (X.509 certificate) signed by a trusted Certificate Authority
* **Who verified it:** verified by both parties in the connection where the server verifies the client's identity and the client verifies the server's identity
* **What assumption is being refused:** Refuses the implicit trust assumption that a service is safe purely because of where it sits on the network such as its IP address, subnet, or network segment.

# 3. Verification Command

```bash
echo "=== Lab 4 Addendum verification ==="

openssl x509 -in ~/ikb42603-mtls/client.crt -noout \
  -subject -issuer -dates

openssl verify -CAfile ~/ikb42603-mtls/ca.crt \
  ~/ikb42603-mtls/client.crt

openssl verify -CAfile ~/ikb42603-mtls/ca.crt \
  ~/ikb42603-mtls/rogue.crt
```

The final two lines are the summary of the whole task: one certificate verifies against your CA and one does not, and that difference — not an address, not a network location — is what grants access.

<img width="975" height="406" alt="image" src="https://github.com/user-attachments/assets/d79df388-254c-43d6-b046-2769ca05c4b3" />


# Security Best-Practices Checklist

✔ Every service holds its own key pair; no key is shared between services.

✔ Certificates are issued by a single trusted authority, and the CA private key is protected.

✔ The server requires and verifies a client certificate, not just the reverse.

✔ A caller with no certificate is refused.

✔ A caller with a certificate from an untrusted authority is refused, and the refusal is logged server-side.

✔ Workload certificate lifetimes are short relative to the CA's.

✔ Network reachability alone grants no access.

# Cleanup

```bash
# Stop the s_server process in your first terminal with Ctrl-C

rm -rf ~/ikb42603-mtls
```
