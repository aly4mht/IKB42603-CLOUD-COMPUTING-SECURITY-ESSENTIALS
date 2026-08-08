+-----------------------------------------------------------------------+
| **LAB 0 · SETUP CHEATSHEET**                                          |
|                                                                       |
| **Lab Environment Setup**                                             |
|                                                                       |
| *Install & verify everything ONCE before Lab 1 --- Docker · AWS CLI · |
| Kubernetes (kind) · helper tools*                                     |
+=======================================================================+

# What You Install (once)

Do this on your own laptop before the first lab. Everything runs locally
--- no cloud account, no credit card, no internet after the first
downloads.

+------------------------------------+----------------------------------------+--------------------------+
| **Tool**                           | **What it is for**                     | **Used in**              |
+=========+==========================+========================================+==========================+
| Docker                             | Runs containers and the LocalStack     | All labs                 |
|                                    | cloud simulator                        |                          |
+------------------------------------+----------------------------------------+--------------------------+
| AWS CLI v2                         | Sends AWS commands to LocalStack       | Labs 1, 3, 5             |
+------------------------------------+----------------------------------------+--------------------------+
| kind                               | Runs a local Kubernetes cluster inside | Labs 1, 2, 4             |
|                                    | Docker                                 |                          |
+------------------------------------+----------------------------------------+--------------------------+
| kubectl                            | Controls the Kubernetes cluster        | Labs 1, 2, 4             |
+------------------------------------+----------------------------------------+--------------------------+
| OpenSSL                            | Encryption, keys, certificates         | Lab 3                    |
+------------------------------------+----------------------------------------+--------------------------+
| oathtool                           | Generates MFA / TOTP codes             | Lab 4                    |
+------------------------------------+----------------------------------------+--------------------------+
| Trivy                              | Scans containers for vulnerabilities   | Lab 4                    |
|                                    | (run via Docker --- no install)        |                          |
+---------+--------------------------+----------------------------------------+--------------------------+
|         | **Security tip:** Windows users: after installing Docker, do ALL lab commands inside Git     |
|         | Bash or WSL (Ubuntu). The labs use bash features (heredocs, sha256sum, single-quoting) that  |
|         | do not work in Command Prompt or PowerShell.                                                 |
+---------+----------------------------------------------------------------------------------------------+

# 1. Install Docker

+------------------------------------+---------------------------------------------------------+
| **OS**                             | **How**                                                 |
+====================================+=========================================================+
| Windows 10/11                      | Install Docker Desktop from docker.com. Choose the WSL  |
|                                    | 2 backend when prompted. Reboot.                        |
+------------------------------------+---------------------------------------------------------+
| macOS                              | Install Docker Desktop from docker.com (pick Apple      |
|                                    | Silicon or Intel to match your Mac).                    |
+------------------------------------+---------------------------------------------------------+
| Linux (Ubuntu)                     | Run: curl -fsSL https://get.docker.com \| sh then add   |
|                                    | yourself to the docker group (below).                   |
+------------------------------------+---------------------------------------------------------+
| \# Linux only: run docker without sudo (log out & back in after)                             |
|                                                                                              |
| sudo usermod -aG docker \$USER                                                               |
|                                                                                              |
| \# Verify (all OS) --- should print a version and a friendly message                         |
|                                                                                              |
| docker \--version                                                                            |
|                                                                                              |
| docker run \--rm hello-world                                                                 |
+----------------------------------------------------------------------------------------------+

# ![](media/image1.png){width="4.122700131233596in" height="2.564061679790026in"}

# 2. Install AWS CLI v2

+------------------------------------+---------------------------------------------------------+
| **OS**                             | **How**                                                 |
+====================================+=========================================================+
| Windows                            | Download and run the AWS CLI MSI installer from the AWS |
|                                    | website (search 'install AWS CLI v2 Windows').          |
+------------------------------------+---------------------------------------------------------+
| macOS                              | brew install awscli (or download the .pkg installer     |
|                                    | from AWS).                                              |
+------------------------------------+---------------------------------------------------------+
| Linux                              | See the commands below.                                 |
+------------------------------------+---------------------------------------------------------+
| \# Linux: install AWS CLI v2                                                                 |
|                                                                                              |
| curl 'https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip' -o awscliv2.zip              |
|                                                                                              |
| unzip awscliv2.zip && sudo ./aws/install                                                     |
|                                                                                              |
| \# Verify (all OS)                                                                           |
|                                                                                              |
| aws --version \# expect: aws-cli/2.x ...                                                     |
+----------------------------------------------------------------------------------------------+

# ![](media/image2.png){width="4.1875in" height="0.7083333333333334in"}

# 

# 3. Install kind & kubectl

+-------------------------+-----------------------------+-----------------------------+
| **OS**                  | **kind**                    | **kubectl**                 |
+=========================+=============================+=============================+
| Windows                 | choco install kind          | choco install               |
|                         |                             | kubernetes-cli              |
+-------------------------+-----------------------------+-----------------------------+
| macOS                   | brew install kind           | brew install kubectl        |
+-------------------------+-----------------------------+-----------------------------+
| Linux                   | see commands below          | sudo snap install kubectl   |
|                         |                             | \--classic                  |
+-------------------------+-----------------------------+-----------------------------+
| \# Linux: install kind (Kubernetes-in-Docker)                                       |
|                                                                                     |
| curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64                |
|                                                                                     |
| chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind                               |
|                                                                                     |
| \# Verify (all OS)                                                                  |
|                                                                                     |
| kind \--version                                                                     |
|                                                                                     |
| kubectl version \--client                                                           |
+-------------------------------------------------------------------------------------+

# ![](media/image3.png){width="4.354166666666667in" height="1.1736111111111112in"}

# 

# 

# ![](media/image4.png){width="5.091666666666667in" height="0.6527777777777778in"}4. Helper Tools (OpenSSL, oathtool, Trivy)

+-------------------------------------+---------------------------------------------------------+
| **Tool**                            | **How to install**                                      |
+=====================================+=========================================================+
| OpenSSL                             | Already installed on macOS/Linux. On Windows it comes   |
|                                     | with Git Bash --- no separate install.                  |
+-------------------------------------+---------------------------------------------------------+
| oathtool                            | macOS: brew install oath-toolkit · Linux: sudo apt      |
|                                     | install oathtool · Windows: use WSL, or a phone         |
|                                     | authenticator app.                                      |
+-------------------------------------+---------------------------------------------------------+
| Trivy                               | No install needed --- Lab 4 runs it via Docker: docker  |
|                                     | run \--rm aquasec/trivy image \<name\>.                 |
+-------------------------------------+---------------------------------------------------------+
| \# Verify (where applicable)                                                                  |
|                                                                                               |
| openssl version                                                                               |
|                                                                                               |
| oathtool \--version \# Lab 4 only                                                             |
+-----------------------------------------------------------------------------------------------+

# 5. Start & Stop the Lab Environment

**LocalStack (the local AWS)**

+-----------------------------------------------------------------------+
| \# Start LocalStack (first run downloads the image)                   |
|                                                                       |
| docker run -d \--name localstack -p 4566:4566 localstack/localstack   |
|                                                                       |
| \# Check it is healthy                                                |
|                                                                       |
| curl http://localhost:4566/\_localstack/health                        |
|                                                                       |
| \# Stop / start again / remove                                        |
|                                                                       |
| docker stop localstack                                                |
|                                                                       |
| docker start localstack                                               |
|                                                                       |
| docker rm -f localstack \# remove completely                          |
+=======================================================================+

![](media/image5.jpeg){width="4.868055555555555in" height="2.5625in"}

**Kubernetes cluster (kind)**

+-----------------------------------------------------------------------+
| \# Create a cluster (name it per the lab)                             |
|                                                                       |
| kind create cluster \--name ccse                                      |
|                                                                       |
| \# Check it is up                                                     |
|                                                                       |
| kubectl cluster-info \--context kind-ccse                             |
|                                                                       |
| kubectl get nodes                                                     |
|                                                                       |
| \# Delete the cluster when finished                                   |
|                                                                       |
| kind delete cluster \--name ccse                                      |
+=======================================================================+

# ![](media/image6.png){width="4.805555555555555in" height="0.9236111111111112in"}

# 

# 6. One-Time AWS CLI Configuration

LocalStack accepts any credentials. Set dummy values once so the CLI
stops asking:

+----------------------------------------------------------------------+
| aws configure set aws_access_key_id test                             |
|                                                                      |
| aws configure set aws_secret_access_key test                         |
|                                                                      |
| aws configure set region us-east-1                                   |
|                                                                      |
| \# Save typing: put the endpoint in a variable each session          |
|                                                                      |
| EP=\'\--endpoint-url=http://localhost:4566\'                         |
|                                                                      |
| aws \$EP sts get-caller-identity \# proves the CLI is talking to     |
| LocalStack                                                           |
+======================================================================+

# ![](media/image7.png){width="4.434722222222222in" height="1.301388888888889in"}

# 

# 

# 

# 7. Pre-Lab Verification Checklist

Tick each before Lab 1. If any fails, see Troubleshooting below.

> ✔ docker \--version prints a version, and docker run hello-world
> works.
>
> ✔ aws \--version prints aws-cli/2.x.
>
> ✔ kind \--version and kubectl version \--client both work.
>
> ✔ LocalStack starts and curl \.../health responds.
>
> ✔ aws \$EP sts get-caller-identity returns an identity.
>
> ✔ kind create cluster works and kubectl get nodes shows a node.
>
> ✔ (Windows) You are working inside Git Bash or WSL.

# 8. Troubleshooting

  -----------------------------------------------------------------------
  **Symptom**               **Fix**
  ------------------------- ---------------------------------------------
  'Cannot connect to the    Docker Desktop is not running --- start it.
  Docker daemon'            On Linux, run the usermod command and
                            re-login.

  Docker won\'t start /     Enable virtualization in BIOS. On Windows
  very slow                 enable WSL 2 + Virtual Machine Platform.

  Port 4566 already in use  A LocalStack is already running: docker rm -f
                            localstack, then start again.

  aws: 'Could not connect   LocalStack is not running, or you forgot
  to the endpoint URL'      \--endpoint-url / \$EP. Start LocalStack and
                            retry.

  'aws: command not found'  The tool is not installed or not on PATH.
  / 'kubectl not found'     Re-run the install step; open a new terminal.

  heredoc / sha256sum       You are in PowerShell/CMD. Switch to Git Bash
  errors on Windows         or WSL.

  kind create cluster fails Docker not running, or low memory. Ensure
                            Docker has ≥ 4 GB and is started.

  Image download is slow in Ask the instructor to pre-pull images, or run
  the lab                   the docker run/pull commands before class on
                            Wi-Fi.

  MFA/TOTP code always      Your system clock is out of sync --- enable
  fails (Lab 4)             automatic time, then retry.

  NetworkPolicy not         The cluster needs Calico (the Lab 2 setup
  blocking (Lab 2)          installs it). Wait until calico-node is
                            Ready.
  -----------------------------------------------------------------------

# 
