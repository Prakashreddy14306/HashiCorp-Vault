Perfect — this is a **very real production-style scenario**, exactly how secure teams run Terraform from Jenkins on EC2 👍
I’ll explain **concept → architecture → Vault config → Jenkins config → full Jenkinsfile**, step by step.

---

# 🔐 Scenario You Described (Rewritten Clearly)

* **Jenkins master** is running on an **EC2 instance**
* **Jenkins agents** run pipelines (EC2 / SSH / dynamic agents)
* **Terraform** is used to create AWS infrastructure
* **AWS access key & secret key must NOT be stored**
* **Vault should generate AWS credentials ONLY during infra creation**
* After pipeline → credentials should **expire automatically**

✅ This is exactly what **Vault AWS Secrets Engine** is built for.

---

## 🏗 Architecture Overview (Real World)

![Image](https://miro.medium.com/1%2AQ2ULHJB28fJvPQdmR-QlWg.jpeg)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1400/1%2AoLZ2UnwWRnlFb4XchPjOMQ.png)

![Image](https://www.datocms-assets.com/2885/1682348441-vault-backed-dynamic-creds.png)

Entities involved:

* **Jenkins**
* **HashiCorp Vault**
* **Terraform**
* **Amazon Web Services**

---

## 🔄 End-to-End Flow (Understand This First)

```
Developer → Git Push
        → Jenkins Agent (EC2)
        → Jenkins authenticates to Vault (AppRole)
        → Vault generates AWS Access Key & Secret Key
        → Terraform uses credentials
        → AWS infrastructure created
        → Credentials expire / revoked
```

👉 **AWS credentials exist only during Terraform execution**

---

# 🔐 PART 1: Vault Configuration (One-Time Setup)

> This is done by a **Vault admin**, not by Jenkins.

---

## 1️⃣ Enable AWS Secrets Engine

```bash
vault secrets enable aws
```

This enables:

```
aws/
├── config/
├── roles/
└── creds/
```

---

## 2️⃣ Configure Vault with AWS “Manager” IAM User

```bash
vault write aws/config/root \
  access_key="AKIA-MANAGER-KEY" \
  secret_key="MANAGER-SECRET" \
  region="us-east-1"
```

### What is this IAM user?

* This is **NOT** used by Jenkins
* Vault uses it to **create temporary IAM users**
* Permissions needed:

  * `iam:CreateUser`
  * `iam:DeleteUser`
  * `iam:CreateAccessKey`
  * `iam:DeleteAccessKey`
  * `iam:PutUserPolicy`

📌 This key is stored **only in Vault**

---

## 3️⃣ Create AWS Role for Terraform

```bash
vault write aws/roles/terraform-role \
  credential_type=iam_user \
  policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "s3:*",
        "vpc:*"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```

### What this role means

* Vault will:

  * Create a **temporary IAM user**
  * Attach this policy
  * Generate access key & secret
  * Revoke after TTL

Apps will request creds using:

```bash
vault read aws/creds/terraform-role
```

---

## 4️⃣ Enable AppRole Auth for Jenkins

```bash
vault auth enable approle
```

---

## 5️⃣ Create Vault Policy for Jenkins

📄 `jenkins-policy.hcl`

```hcl
path "aws/creds/terraform-role" {
  capabilities = ["read"]
}
```

Apply policy:

```bash
vault policy write jenkins-policy jenkins-policy.hcl
```

---

## 6️⃣ Create AppRole for Jenkins

```bash
vault write auth/approle/role/jenkins \
  token_policies="jenkins-policy" \
  token_ttl=15m \
  token_max_ttl=30m
```

---

## 7️⃣ Get Role ID & Secret ID

```bash
vault read auth/approle/role/jenkins/role-id
vault write -f auth/approle/role/jenkins/secret-id
```

👉 Store these in **Jenkins credentials**:

* `vault-role-id`
* `vault-secret-id`

---

# 🔐 PART 2: Jenkins Agent Requirements

On the **Jenkins agent EC2**, install:

```bash
terraform
vault
awscli
jq
```

---

# 🔐 PART 3: Terraform Code (Minimal Example)

### `provider.tf`

```hcl
provider "aws" {
  region = "us-east-1"
}
```

### `main.tf`

```hcl
resource "aws_s3_bucket" "demo" {
  bucket = "vault-terraform-demo-${random_id.rand.hex}"
}

resource "random_id" "rand" {
  byte_length = 4
}
```

---

# 🚀 PART 4: FULL Jenkinsfile (What You Asked For)

This Jenkinsfile:

* Runs on EC2 agent
* Logs into Vault
* Fetches AWS creds
* Runs Terraform
* Leaves **no secrets behind**

---

## ✅ Jenkinsfile (Production-style)

```groovy
pipeline {
  agent any

  environment {
    VAULT_ADDR = "http://vault-server:8200"
    VAULT_ROLE_ID = credentials('vault-role-id')
    VAULT_SECRET_ID = credentials('vault-secret-id')
  }

  stages {

    stage('Checkout Code') {
      steps {
        git url: 'https://github.com/your-org/terraform-repo.git', branch: 'main'
      }
    }

    stage('Authenticate to Vault') {
      steps {
        sh '''
          echo "Logging into Vault using AppRole..."

          vault write auth/approle/login \
            role_id=$VAULT_ROLE_ID \
            secret_id=$VAULT_SECRET_ID \
            -format=json > vault_token.json

          export VAULT_TOKEN=$(jq -r .auth.client_token vault_token.json)
          echo $VAULT_TOKEN > .vault-token
        '''
      }
    }

    stage('Get Dynamic AWS Credentials') {
      steps {
        sh '''
          export VAULT_TOKEN=$(cat .vault-token)

          echo "Requesting AWS credentials from Vault..."
          vault read aws/creds/terraform-role -format=json > aws_creds.json

          export AWS_ACCESS_KEY_ID=$(jq -r .data.access_key aws_creds.json)
          export AWS_SECRET_ACCESS_KEY=$(jq -r .data.secret_key aws_creds.json)

          echo "AWS credentials injected for Terraform"
        '''
      }
    }

    stage('Terraform Init') {
      steps {
        sh '''
          terraform init
        '''
      }
    }

    stage('Terraform Plan') {
      steps {
        sh '''
          terraform plan
        '''
      }
    }

    stage('Terraform Apply') {
      steps {
        sh '''
          terraform apply -auto-approve
        '''
      }
    }
  }

  post {
    always {
      sh '''
        rm -f aws_creds.json vault_token.json .vault-token
      '''
    }
  }
}
```

---

# 🔐 What Happens at Runtime (Critical Understanding)

1. Jenkins agent **logs into Vault**
2. Vault issues **short-lived Vault token**
3. Jenkins requests AWS creds
4. Vault creates **temporary IAM user**
5. Terraform uses creds
6. TTL expires → IAM user deleted

👉 **No AWS credentials stored anywhere**

---

## 🧠 Interview-Ready Explanation (Use This)

> “Jenkins authenticates to Vault using AppRole. Vault generates temporary AWS credentials using the AWS secrets engine. Terraform consumes those credentials to provision infrastructure. Once the job completes, Vault automatically revokes the credentials, ensuring zero long-lived AWS keys.”

---

## ❌ What You Are Avoiding (Very Important)

❌ AWS keys in Jenkins credentials
❌ AWS keys in Git repo
❌ Long-lived IAM users
❌ Manual rotation

---

## ✅ Production Enhancements (Next Level)

| Enhancement         | Benefit         |
| ------------------- | --------------- |
| Use `assumed_role`  | Faster, cleaner |
| Vault TLS           | Secure traffic  |
| Jenkins K8s agents  | Isolation       |
| Vault audit logs    | Compliance      |
| IAM least privilege | Security        |

---

## 🧩 One-Line Mental Model

> **Vault owns secrets. Jenkins borrows them. Terraform uses them. AWS never sees static keys.**

---

If you want next 🚀

* 🔄 **STS assumed_role version**
* ☸ **Jenkins agent on EKS**
* 🧪 **Terraform destroy with Vault revoke**
* 🎯 **Vault + Jenkins interview mock**

Just tell me — you’re very close to senior-level Vault usage 👊
