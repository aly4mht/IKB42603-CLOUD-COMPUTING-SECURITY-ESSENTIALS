# IKB42603 Cloud Computing Security Essentials

## Lab 0.1 — Environment Setup (Auth-Token Track)

**Name:** ALYA LIYANA BINTI MAHAT (B01)  
**Student ID:** 52215124600  
**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab:** Lab 0.1 — Environment Setup  
**Topic:** LocalStack with Auth Token + Web Console

---

# Lab Summary

The objective of this lab is to prepare and activate a LocalStack environment using an authentication token and verify that AWS CLI commands can communicate with the local AWS-compatible services.

By completing this lab, the following learning outcomes are achieved:

1. Obtain and safely configure a LocalStack authentication token.
2. Start an activated LocalStack instance in Docker.
3. Confirm licence activation from the command line.
4. Point the AWS CLI at LocalStack.
5. Create AWS resources locally.
6. Inspect the created resources using the LocalStack Web Console Resource Browser.

This lab uses the **Auth-Token Track**. Since 23 March 2026, the standard `localstack/localstack` image requires a licence authentication token to start. A free LocalStack account can be used to obtain the required developer token. :contentReference[oaicite:1]{index=1}

---

# Prerequisites

The following tools are required before starting:

- Docker Desktop
- AWS CLI v2
- Modern web browser
- LocalStack account

The environment was first checked using:

```bash
docker run --rm hello-world
aws --version
docker --version
````

Expected results:

```text
docker run --rm hello-world
```

should successfully run the Docker test container.
<img width="904" height="492" alt="image" src="https://github.com/user-attachments/assets/f727e54a-3562-45a2-b52b-87773b994b96" />

```bash
aws --version
```

should return an AWS CLI v2 version.
<img width="628" height="106" alt="image" src="https://github.com/user-attachments/assets/4f8def62-eab7-4302-b55a-7b64fd744ea1" />

```bash
docker --version
```

should return the installed Docker version. 
<img width="529" height="82" alt="image" src="https://github.com/user-attachments/assets/5c07c37a-e60d-46c9-a578-290c924cb74a" />

---

# Part 1 — Create an Account and Get an Auth Token

## Step 1.1: Create a LocalStack Account

Open the LocalStack sign-up page:

```text
https://app.localstack.cloud/sign-up
```

A free account can be created using:

* Student email
* Google
* GitHub

No credit card is required.

## Step 1.2: Sign In and Confirm the Email

After creating the account, sign in and confirm the email address if prompted.

## Step 1.3: Open the Auth Tokens Page

Open:

```text
https://app.localstack.cloud/workspace/auth-tokens
```

Copy the **Developer Token**.

The token has a format similar to:

```text
ls-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```

The developer token is used to activate the LocalStack image on the local machine.  

> **Security Note:** Never share the LocalStack authentication token or commit it to a code repository.

---

# Part 2 — Set the Token in the Shell

The authentication token was stored as an environment variable in the terminal used for the lab.

For macOS, Linux, Git Bash, or WSL:

```bash
export LOCALSTACK_AUTH_TOKEN="ls-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
```

Replace the placeholder with the actual developer token.

## Step 2.1: Confirm the Token Is Set

```bash
echo $LOCALSTACK_AUTH_TOKEN
```

The command should print the configured token.

The token should only be stored locally and should not be included in screenshots or submitted publicly.
<img width="930" height="692" alt="image" src="https://github.com/user-attachments/assets/47903392-79ad-4aa7-9c64-5a3d1a774f73" />

---

# Part 3 — Start LocalStack

Before starting LocalStack, remove any existing container so that the setup begins from a clean state.

## Step 3.1: Remove Existing LocalStack Container

```bash
docker rm -f localstack 2>/dev/null
```

If there is no existing container, the error can be ignored.

## Step 3.2: Start LocalStack with the Auth Token

Run:

```bash
docker run -d --name localstack \
  -p 4566:4566 \
  -p 4510-4559:4510-4559 \
  -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN \
  -v /var/run/docker.sock:/var/run/docker.sock \
  localstack/localstack
```

The authentication token is passed into the container through:

```bash
-e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN
```

The main LocalStack endpoint is exposed on:

```text
localhost:4566
```

Additional service ports are exposed using:

```text
4510-4559
```

## Step 3.3: Monitor LocalStack Startup

Check the container logs:

```bash
docker logs -f localstack
```

Wait until the logs indicate that LocalStack is ready.

Press Ctrl-C to stop following the logs without stopping the LocalStack container.

The LocalStack startup procedure and required Docker options are specified in the lab document.  
<img width="1014" height="362" alt="image" src="https://github.com/user-attachments/assets/141d085f-0119-4886-a9a7-93513a47d469" />

---

# Part 4 — Verify Licence Activation

Two checks are used to confirm that LocalStack is healthy and activated.

## Step 4.1: Check Licence Activation

Run:

```bash
curl http://localhost:4566/_localstack/info
```

The expected response should contain:

```json
{
  "edition": "pro",
  "is_license_activated": true
}
```

The important field is:

```text
"is_license_activated": true
```

This confirms that the authentication token was successfully accepted and the LocalStack licence is active.

## Step 4.2: Check LocalStack Health

Run:

```bash
curl http://localhost:4566/_localstack/health
```

The health endpoint should return the available LocalStack services.

### Result

The `/ _localstack/info` endpoint confirms licence activation, while the `/ _localstack/health` endpoint confirms that LocalStack services are available. 
<img width="1012" height="259" alt="image" src="https://github.com/user-attachments/assets/c891fc38-f23c-4c79-9ce5-c9f8fccdf90f" />

---

# Part 5 — Point the AWS CLI at LocalStack

LocalStack accepts dummy AWS credentials because the services are running locally.

## Step 5.1: Configure Dummy AWS Credentials

Run:

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
```

## Step 5.2: Configure the AWS Region

The lab uses the `us-east-1` region.

```bash
aws configure set region us-east-1
```

## Step 5.3: Create the LocalStack Endpoint Shortcut

Set the endpoint variable:

```bash
EP='--endpoint-url=http://localhost:4566'
```

This variable allows AWS CLI commands to be redirected to LocalStack.

## Step 5.4: Verify AWS CLI Connectivity

Run:

```bash
aws $EP sts get-caller-identity
```

A successful response returns a dummy identity.

The expected LocalStack result includes:

* Account: `000000000000`
* An example `UserId`
* An ARN ending in `:root`

This confirms that the AWS CLI is communicating with LocalStack instead of the real AWS cloud.  
<img width="536" height="404" alt="image" src="https://github.com/user-attachments/assets/a11a8a3a-ecb9-42aa-971c-7eb7f8cbfbca" />

---

# Part 6 — Open the Web Console and Resource Browser

The LocalStack Web Console can be used to view the local AWS resources created during the lab.

## Step 6.1: Sign In to the Web Console

Open:

```text
https://app.localstack.cloud
```

Sign in using the same LocalStack account created in Part 1.

## Step 6.2: Open the Instances Page

Open the **Instances** page and select the default instance.

The instance endpoint is:

```text
https://localhost.localstack.cloud:4566
```

This endpoint points to the LocalStack container running on the local machine.

## Step 6.3: Check the Status / Stack Overview

The **Status / Stack Overview** page displays the running services and confirms that the web console has connected to the local LocalStack instance.

## Step 6.4: Open Resource Browser

Open the **Resource Browser**.

The Resource Browser allows resources to be viewed by service category, including:

* Storage — S3
* Database — DynamoDB
* Security Identity & Compliance — IAM and KMS
* Compute — Lambda
* Other AWS-compatible services

Set the region dropdown to:

```text
us-east-1
```

This ensures that the resources created during the lab appear in the Resource Browser. 
<img width="1015" height="493" alt="image" src="https://github.com/user-attachments/assets/c0762982-9224-495d-8b03-fe64e9f16f2c" />

---

# Part 7 — Hands-On: Create Resources and View Them in the Console

The purpose of this section is to create several AWS resources using the AWS CLI and then view them in the LocalStack Resource Browser.

This demonstrates the same create-then-view workflow used in the real AWS Console.

---

## Step 7.1: Create an S3 Bucket

Create an S3 bucket named:

```text
ccse-demo-bucket
```

Command:

```bash
aws $EP s3 mb s3://ccse-demo-bucket
```

### Result

The S3 bucket `ccse-demo-bucket` is created inside LocalStack.

---

## Step 7.2: Create a Test File

Create a file containing:

```text
hello cloud
```

Command:

```bash
echo "hello cloud" > hello.txt
```

---

## Step 7.3: Upload the File to S3

Upload the file to the LocalStack S3 bucket:

```bash
aws $EP s3 cp hello.txt s3://ccse-demo-bucket/
```

### Result

The `hello.txt` file is uploaded into the `ccse-demo-bucket` S3 bucket.

---

## Step 7.4: Create a DynamoDB Table

Create a DynamoDB table named `Students`:

```bash
aws $EP dynamodb create-table \
  --table-name Students \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Result

A DynamoDB table named:

```text
Students
```

is created in LocalStack.

The table uses:

```text
id
```

as the partition key with a string (`S`) data type.

The billing mode is:

```text
PAY_PER_REQUEST
```
<img width="697" height="786" alt="image" src="https://github.com/user-attachments/assets/05d13cca-2a52-402b-af6b-e17b371db797" />

---

## Step 7.5: Create an IAM User

An IAM user can also be created through the AWS CLI and then viewed in the Resource Browser.

Example:

```bash
aws $EP iam create-user --user-name LabUser
```

### Result

The IAM user `LabUser` is created in the LocalStack IAM service.
<img width="704" height="242" alt="image" src="https://github.com/user-attachments/assets/64e99909-fea5-4c1c-b8cb-3bad516aa209" />

---

## Step 7.6: View the Resources in Resource Browser

Return to the LocalStack Web Console.

Open:

```text
Resource Browser
```

Set the region to:

```text
us-east-1
```

Refresh the Resource Browser.

The following resources should be visible:

| **Service** | **Resource**       |
| ----------- | ------------------ |
| S3          | `ccse-demo-bucket` |
| S3 Object   | `hello.txt`        |
| DynamoDB    | `Students`         |
| IAM         | `LabUser`          |

This demonstrates the create-then-view workflow using the AWS CLI and LocalStack Web Console. 

---

# Pre-Lab Verification Checklist

✔ docker --version and docker run hello-world both work.
✔LOCALSTACK_AUTH_TOKEN is set (echo prints it).
✔curl .../_localstack/info shows is_license_activated: true.
✔ curl .../_localstack/health responds with services available.
✔aws $EP sts get-caller-identity returns an identity.
✔ Resource Browser shows the default instance and your test resources in us-east-1.

---

# Troubleshooting

| **Symptom**                                                | **Recommended Fix**                                                                                                                    |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Container exits with `55` or `"License activation failed"` | Check `echo $LOCALSTACK_AUTH_TOKEN`, copy the token again from the Auth Tokens page, and confirm a licence is assigned.                |
| `curl` to port `4566` fails immediately                    | Check whether the container is running with `docker ps -a`, then inspect the logs using `docker logs localstack`.                      |
| `is_license_activated` is `false`                          | The token may have started the image but no licence is active. Assign the appropriate role/licence under Workspace → Users & Licenses. |
| `"License server unreachable"` appears in the logs         | Corporate DNS or network restrictions may be blocking the LocalStack licence domain. Check connectivity to `api.localstack.cloud`.     |
| `"Network Failure"` appears in the Web Console             | Ensure LocalStack is running and reload the Web Console.                                                                               |
| Resource Browser shows no resources                        | Check that the region is set to `us-east-1`, then reload the Resource Browser.                                                         |

These troubleshooting steps are based on the troubleshooting section of the uploaded lab document. 

---

# Security Considerations

## Protect the Authentication Token

The LocalStack authentication token is a sensitive credential and should not be committed to GitHub, uploaded publicly, or included in screenshots.

The token should be stored in an environment variable:

```bash
export LOCALSTACK_AUTH_TOKEN="YOUR_TOKEN"
```

## Use Dummy AWS Credentials

LocalStack accepts dummy AWS credentials:

```text
Access Key ID: test
Secret Access Key: test
```

These credentials are used because the AWS services are running locally rather than against a real AWS account.

## Verify the Endpoint

Always make sure AWS CLI commands are pointed to LocalStack:

```bash
EP='--endpoint-url=http://localhost:4566'
```

Then use:

```bash
aws $EP <service> <command>
```

This prevents accidentally sending lab commands to real AWS services.

---

# Cleanup & Teardown

## Stop LocalStack

Stopping the container keeps the container available for future use.

```bash
docker stop localstack
```

## Start LocalStack Again

When continuing the lab later:

```bash
docker start localstack
```

## Remove LocalStack Completely

To remove the container and clear its resources:

```bash
docker rm -f localstack
```

Removing the container is safe because the lab report files, outputs, and screenshots are stored on the local computer rather than inside the LocalStack container. 

---

# Conclusion

The Lab 0.1 Auth-Token Track provides a complete local AWS-compatible environment using LocalStack.

The LocalStack authentication token was configured as an environment variable and used to activate the LocalStack Docker image. Licence activation was verified using the `/_localstack/info` endpoint, while service availability was confirmed through the `/_localstack/health` endpoint.

The AWS CLI was configured with dummy credentials and directed to LocalStack through the `EP` endpoint shortcut. The `sts get-caller-identity` command confirmed that AWS CLI requests were reaching the LocalStack environment.

Several AWS resources were then created locally, including:

* S3 bucket `ccse-demo-bucket`
* S3 object `hello.txt`
* DynamoDB table `Students`
* IAM user `LabUser`

These resources could then be viewed through the LocalStack Web Console Resource Browser using the `us-east-1` region.

Therefore, the LocalStack environment is successfully prepared for subsequent cloud security laboratory activities.

---
