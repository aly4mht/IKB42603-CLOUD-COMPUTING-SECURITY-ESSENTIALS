# Lab 0 : Environment Setup
**IKB42603 Cloud Computing Security Essentials** 

# **Lab Environment Setup** 

_Install & verify everything ONCE before Lab 1 — Docker · AWS CLI · Kubernetes (kind) · helper tools_ 

## **What You Install (once)** 

Do this on your own laptop before the first lab. Everything runs locally — no cloud account, no credit card, no internet after the first downloads. 

|**Tool**|**What it is for**|**Used in**|
|---|---|---|
|Docker|Runs containers and the LocalStack cloud simulator|All labs|
|AWS CLI v2|Sends AWS commands to LocalStack|Labs 1, 3, 5|
|kind|Runs a local Kubernetes cluster inside Docker|Labs 1, 2, 4|
|kubectl|Controls the Kubernetes cluster|Labs 1, 2, 4|
|OpenSSL|Encryption, keys, certificates|Lab 3|
|oathtool|Generates MFA / TOTP codes|Lab 4|
|Trivy|Scans containers for vulnerabilities (run via Docker — no<br>install)|Lab 4|
|**Security tip:**Win<br>WSL (Ubuntu). Th<br>work in Command|dows users: after installing Docker, do ALL lab commands in<br>e labs use bash features (heredocs, sha256sum, single-quot<br>Prompt or PowerShell.|side Git Bash or<br>ing) that do not|



## **1. Install Docker** 

|**OS**|**How**|
|---|---|
|Windows 10/11|Install Docker Desktop from docker.com. Choose the WSL 2 backend when<br>prompted. Reboot.|
|macOS|Install Docker Desktop from docker.com (pick Apple Silicon or Intel to match your<br>Mac).|
|Linux (Ubuntu)|Run: curl -fsSL https://get.docker.com | sh   then add yourself to the docker group<br>(below).|



# Linux only: run docker without sudo (log out & back in after) sudo usermod -aG docker $USER 

# Verify (all OS) — should print a version and a friendly message docker --version docker run --rm hello-world 



<!-- Start of picture text -->
$ docker --version<br>docker run --rm hello-world<br>Docker version 29.6.2, build dfc4efb<br>Hello from Docker!<br>This message shows that your installation appears to be working correctly.<br>To generate this message, Docker took the following steps:<br>1. The Docker client contacted the Docker daemon.<br>2. The Docker daemon pulled the "hello-world" image from the Docker Hub.<br>Camd64)<br>3. The Docker daemon created a new container from that image which runs the<br>executable that produces the output you are currently reading.<br>4. The Docker daemon streamed that output to the Docker client, which sent it<br>to your terminal.<br>To try something more ambitious, you can run an Ubuntu container with:<br>$ docker run -it ubuntu bash<br>Share images, automate workflows, and more with a free Docker ID:<br>https://hub.docker.com/<br>For more examples and ideas, visit:<br>https://docs.docker.com/get-started/<br><!-- End of picture text -->



<!-- Start of picture text -->
$ aws --version<br>aws-c1i/2.36.9 Python/3.14.6 Windows/11 exe/AMD64<br><!-- End of picture text -->





$ kind --version kubectl version --client kind version 0.31.0 Client Version: v1.36.3 Kustomize Version: v5.8.1 



<!-- Start of picture text -->
OpenSSL 3.5.6 7 Apr 2026 (Library: OpenSSL 3.5.6 7 Apr 2026)<br>oh ee<br><!-- End of picture text -->



<!-- Start of picture text -->
PS C WINDOWS\system32> docker run localstack 4566:4566 ar/run/docker .sock var/run/docker.sock localstack/localstack:3.8.1<br>126955b699c782¢61¢34aa88240bd80296144a822bead8dl6bfdcae@b6ccF464675<br>IPS C:\WINDOWS\system32> curl http: //localhost :4566/_localstack/health<br>Security Warning: Script Execution Risk<br>parsed<br>RECOMMENDED ACTION<br>Use the -UseBasicParsing switch to avoid script code execution<br>Do you want to continue?<br>[Y] Yes [A] Yes to All [N] No [L] No to All [S] Suspend [?] Help (default is "N"): y<br>IstatusCode 20@<br>iStatusDescription OK<br>content {"services": {"acm": "available", “apigateway": "available", "cloudformation": "available",<br>cloudwatch" “available”, “config”: “available”, “dynamodb": “available”, “dynamodbstreams”<br>"available", "<br>RawContent HTTP/1.1 208 OK<br>Content-Length: 922<br>Content-Type: application/json<br>Date: Wed, 29 Jul 2026 62:48:38 GMT<br>Server: TwistedWeb/24.3.e<br>{"“services": {"acm" “available”, “apigateway” “available”, “cl<br>Forms {}<br>Headers {[Content-Length, 922], [Content-Type, application/json], [Date, Wed, 29 Jul 2026 02:48:38 GMT],<br>[Server, TwistedWeb/24.3.0]}<br>Images {}<br>InputFields {}<br>Links {}<br>ParsedHtml mshtml.HTMLDocumentClass<br>RawContentLength 922<br>$ kubect] cluster-info --context kind-ccse<br>kubect] get nodes<br>Kubernetes control plane is running at https://127.0.0.1:52262<br>CoreDNS is running at https://127.0.0.1:52262/api/v1/namespaces /kube-system/services/kube-dns :dns/proxy<br>To further debug and diagnose cluster problems, use ‘kubect] cluster-info dump’.<br>NAME STATUS ROLES AGE VERSION<br>cCcse-control-plane Ready control-plane 86s v1.35.0<br><!-- End of picture text -->



<!-- Start of picture text -->
$ kubect] cluster-info --context kind-ccse<br>kubect] get nodes<br>Kubernetes control plane is running at https://127.0.0.1:52262<br>CoreDNS is running at https://127.0.0.1:52262/api/v1/namespaces /kube-system/services/kube-dns :dns/proxy<br>To further debug and diagnose cluster problems, use ‘kubect] cluster-info dump’.<br>NAME STATUS ROLES AGE VERSION<br>cCcse-control-plane Ready control-plane 86s v1.35.0<br><!-- End of picture text -->



$ aws $EP sts get-caller-identity tIL “UserId": "“AKIAIOSFODNN7EXAMPLE", "Account": “OO0O0000000000", "Arn": “arn:aws:1am::000000000000:root" 1 J 

Setup Cheatsheet 

|**Symptom**|**Fix**|
|---|---|
|‘aws: command not found’ /<br>‘kubectl not found’|The tool is not installed or not on PATH. Re-run the install step;<br>open a new terminal.|
|heredoc / sha256sum errors on<br>Windows|You are in PowerShell/CMD. Switch to Git Bash or WSL.|
|kind create cluster fails|Docker not running, or low memory. Ensure Docker has ≥ 4 GB<br>and is started.|
|Image download is slow in the lab|Ask the instructor to pre-pull images, or run the docker run/pull<br>commands before class on Wi-Fi.|
|MFA/TOTP code always fails (Lab<br>4)|Your system clock is out of sync — enable automatic time, then<br>retry.|
|NetworkPolicy not blocking (Lab 2)|The cluster needs Calico (the Lab 2 setup installs it). Wait until<br>calico-node is Ready.|

