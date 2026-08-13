# LAB 0 · SETUP CHEATSHEET

# Lab Environment Setup

Install & verify everything **ONCE before Lab 1** — Docker · AWS CLI · Kubernetes (kind) · helper tools.

---

## What You Install (Once)

Do this on your own laptop before the first lab. Everything runs locally — no cloud account, no credit card, and no internet after the first downloads.

| **Tool** | **What it is for** | **Used in** |
|---|---|---|
| Docker | Runs containers and the LocalStack cloud simulator | All labs |
| AWS CLI v2 | Sends AWS commands to LocalStack | Labs 1, 3, 5 |
| kind | Runs a local Kubernetes cluster inside Docker | Labs 1, 2, 4 |
| kubectl | Controls the Kubernetes cluster | Labs 1, 2, 4 |
| OpenSSL | Encryption, keys, certificates | Lab 3 |
| oathtool | Generates MFA / TOTP codes | Lab 4 |
| Trivy | Scans containers for vulnerabilities (run via Docker — no install) | Lab 4 |

> **Security Tip:** Windows users should do **all lab commands inside Git Bash or WSL (Ubuntu)** after installing Docker. The labs use bash features such as heredocs, `sha256sum`, and single-quoting that do not work correctly in Command Prompt or PowerShell.

---

# 1. Install Docker

## Operating System Installation

| **OS** | **How** |
|---|---|
| Windows 10/11 | Install Docker Desktop from `docker.com`. Choose the WSL 2 backend when prompted, then reboot. |
| macOS | Install Docker Desktop from `docker.com`. Choose Apple Silicon or Intel to match your Mac. |
| Linux (Ubuntu) | Run `curl -fsSL https://get.docker.com \| sh`, then add yourself to the Docker group. |

### Linux Only

Run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
````

Log out and back in after running the command.

## Verify Docker

The following commands should print a Docker version and a friendly confirmation message:

```bash
docker --version
docker run --rm hello-world
```

### Expected Result

* `docker --version` prints the installed Docker version.
* `docker run --rm hello-world` successfully runs the test container.

---

# 2. Install AWS CLI v2

AWS CLI v2 is used to send AWS commands to LocalStack during the labs.

## Operating System Installation

| **OS**  | **How**                                                              |
| ------- | -------------------------------------------------------------------- |
| Windows | Download and run the AWS CLI MSI installer from the AWS website.     |
| macOS   | Run `brew install awscli` or download the `.pkg` installer from AWS. |
| Linux   | Follow the Linux installation commands below.                        |

## Linux Installation

Download the AWS CLI v2 package:

```bash
curl 'https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip' -o awscliv2.zip
```

Extract and install:

```bash
unzip awscliv2.zip && sudo ./aws/install
```

## Verify AWS CLI

Run:

```bash
aws --version
```

### Expected Result

The command should display an AWS CLI v2 version similar to:

```text
aws-cli/2.x ...
```

---

# 3. Install kind & kubectl

`kind` runs a local Kubernetes cluster inside Docker, while `kubectl` is used to control the Kubernetes cluster.

## Operating System Installation

| **OS**  | **kind**             | **kubectl**                           |
| ------- | -------------------- | ------------------------------------- |
| Windows | `choco install kind` | `choco install kubernetes-cli`        |
| macOS   | `brew install kind`  | `brew install kubectl`                |
| Linux   | See commands below   | `sudo snap install kubectl --classic` |

## Linux: Install kind

Download kind:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
```

Make it executable and move it into the system path:

```bash
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
```

## Verify kind and kubectl

Run:

```bash
kind --version
kubectl version --client
```

### Expected Result

* `kind --version` should display the installed kind version.
* `kubectl version --client` should display the installed kubectl client version.

---

# 4. Helper Tools

The helper tools used throughout the later labs are OpenSSL, oathtool, and Trivy.

| **Tool** | **How to Install**                                                                                                       |
| -------- | ------------------------------------------------------------------------------------------------------------------------ |
| OpenSSL  | Already installed on macOS/Linux. On Windows it comes with Git Bash — no separate installation is required.              |
| oathtool | macOS: `brew install oath-toolkit` · Linux: `sudo apt install oathtool` · Windows: use WSL or a phone authenticator app. |
| Trivy    | No installation required. Lab 4 runs Trivy through Docker.                                                               |

## OpenSSL

OpenSSL is used for encryption, keys, and certificates in Lab 3.

Verify the installation with:

```bash
openssl version
```

## oathtool

oathtool is used to generate MFA / TOTP codes in Lab 4.

Verify the installation with:

```bash
oathtool --version
```

> **Note:** oathtool is required for Lab 4 only.

## Trivy

Trivy does not need to be installed separately because it is run through Docker:

```bash
docker run --rm aquasec/trivy image <name>
```

---

# 5. Start & Stop the Lab Environment

## LocalStack — The Local AWS Environment

LocalStack provides a local AWS-compatible environment for the labs.

### Start LocalStack

On the first run, Docker downloads the LocalStack image:

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack
```

### Check LocalStack Health

Run:

```bash
curl http://localhost:4566/_localstack/health
```

The command should return the LocalStack health information.

### Stop LocalStack

```bash
docker stop localstack
```

### Start LocalStack Again

```bash
docker start localstack
```

### Remove LocalStack Completely

```bash
docker rm -f localstack
```

---

# Kubernetes Cluster — kind

kind is used to create and run a local Kubernetes cluster inside Docker.

## Create a Cluster

Create the cluster using the name required by the lab:

```bash
kind create cluster --name ccse
```

## Check the Kubernetes Cluster

Check the cluster information:

```bash
kubectl cluster-info --context kind-ccse
```

Check the available nodes:

```bash
kubectl get nodes
```

### Expected Result

The Kubernetes cluster should be running and `kubectl get nodes` should show a node in the `Ready` state.

## Delete the Kubernetes Cluster

When finished with the lab:

```bash
kind delete cluster --name ccse
```

---

# 6. One-Time AWS CLI Configuration

LocalStack accepts any credentials. Dummy credentials can therefore be configured so the AWS CLI does not repeatedly ask for credentials.

## Configure Dummy Credentials

Run:

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```

These are dummy credentials for LocalStack and are not real AWS credentials.

## Configure the LocalStack Endpoint

Save the LocalStack endpoint in a variable for each terminal session:

```bash
EP='--endpoint-url=http://localhost:4566'
```

## Verify AWS CLI Connection to LocalStack

Run:

```bash
aws $EP sts get-caller-identity
```

This proves that the AWS CLI is communicating with LocalStack instead of a real AWS account.

### Expected Result

The command should return an identity from LocalStack.

---

# 7. Pre-Lab Verification Checklist

Complete each check before starting Lab 1. If any check fails, refer to the troubleshooting section below.

* [ ] `docker --version` prints a version.
* [ ] `docker run --rm hello-world` works successfully.
* [ ] `aws --version` prints an `aws-cli/2.x` version.
* [ ] `kind --version` works.
* [ ] `kubectl version --client` works.
* [ ] LocalStack starts successfully.
* [ ] `curl http://localhost:4566/_localstack/health` responds.
* [ ] `aws $EP sts get-caller-identity` returns an identity.
* [ ] `kind create cluster --name ccse` works.
* [ ] `kubectl get nodes` shows a node.
* [ ] Windows users are working inside Git Bash or WSL.

---

# 8. Troubleshooting

| **Symptom**                                             | **Fix**                                                                                                                     |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `Cannot connect to the Docker daemon`                   | Docker Desktop is not running. Start Docker Desktop. On Linux, run the `usermod` command and log in again.                  |
| Docker will not start / is very slow                    | Enable virtualization in BIOS. On Windows, enable WSL 2 and Virtual Machine Platform.                                       |
| Port `4566` is already in use                           | A LocalStack container may already be running. Remove it with `docker rm -f localstack`, then start LocalStack again.       |
| AWS CLI reports `Could not connect to the endpoint URL` | LocalStack is not running, or the `--endpoint-url` / `$EP` option was forgotten. Start LocalStack and retry.                |
| `aws: command not found` / `kubectl not found`          | The tool is not installed or is not available in PATH. Re-run the installation step and open a new terminal.                |
| Heredoc / `sha256sum` errors on Windows                 | The commands are being run in PowerShell or Command Prompt. Switch to Git Bash or WSL.                                      |
| `kind create cluster` fails                             | Docker may not be running or Docker may have insufficient memory. Ensure Docker is started and has at least 4 GB available. |
| Image download is slow in the lab                       | Ask the instructor to pre-pull images or run the Docker `run` / `pull` commands before class using a Wi-Fi connection.      |
| MFA/TOTP code always fails in Lab 4                     | The system clock may be out of sync. Enable automatic time synchronization and retry.                                       |
| NetworkPolicy is not blocking traffic in Lab 2          | The cluster requires Calico. The Lab 2 setup installs it. Wait until `calico-node` is in the `Ready` state.                 |

---

# Lab 0 Completion Summary

The purpose of Lab 0 is to prepare and verify the local environment before starting the remaining IKB42603 Cloud Computing Security Essentials labs.

The required environment consists of:

1. **Docker** — runs containers and LocalStack.
2. **AWS CLI v2** — sends AWS commands to LocalStack.
3. **kind** — runs Kubernetes inside Docker.
4. **kubectl** — controls the Kubernetes cluster.
5. **OpenSSL** — provides encryption, key, and certificate functionality.
6. **oathtool** — generates MFA/TOTP codes for Lab 4.
7. **Trivy** — scans containers for vulnerabilities through Docker.

LocalStack runs on:

```text
http://localhost:4566
```

The Kubernetes cluster is created using:

```bash
kind create cluster --name ccse
```

The environment is considered ready for Lab 1 when all items in the **Pre-Lab Verification Checklist** have been successfully completed.

---

# Conclusion

Lab 0 establishes the local environment required for the IKB42603 Cloud Computing Security Essentials course. Docker provides the container environment, LocalStack provides a local AWS-compatible environment, and kind provides a local Kubernetes cluster.

AWS CLI v2 is configured to communicate with LocalStack using dummy credentials, while kubectl is configured to manage the local Kubernetes cluster. OpenSSL, oathtool, and Trivy provide additional security tools required in later labs.

Completing these setup and verification steps ensures that the required tools are installed, accessible from the terminal, and ready to be used in the following cloud security labs.
