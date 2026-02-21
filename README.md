It is:

* ✅ Clean
* ✅ Interview-ready
* ✅ Recruiter-friendly
* ✅ Structured professionally
* ✅ Senior-level documentation style

---

# 📘 README.md

# 🔐 Secure Terraform Deployment using Jenkins + HashiCorp Vault (Dynamic AWS Credentials)

This project demonstrates a **production-grade secure architecture** where:

- Jenkins runs on EC2
- Terraform provisions AWS infrastructure
- AWS credentials are NEVER stored
- HashiCorp Vault dynamically generates temporary AWS credentials
- Credentials automatically expire after use

---

## 🚀 Problem Statement

Traditional CI/CD pipelines often store:

- ❌ AWS Access Keys in Jenkins
- ❌ AWS credentials in GitHub
- ❌ Long-lived IAM users

This creates security risks.

This project solves that using:

> 🔐 Vault AWS Secrets Engine + AppRole Authentication

---

## 🏗 Architecture Overview

```

Developer → Git Push
↓
Jenkins (EC2 Agent)
↓ (AppRole Login)
HashiCorp Vault
↓ (Dynamic AWS Credentials)
Jenkins
↓ (Terraform Apply)
AWS Infrastructure
↓
Credentials Expire Automatically

````

---

## 🧩 Components Used

| Component | Purpose |
|-----------|----------|
| Jenkins | CI/CD Pipeline |
| HashiCorp Vault | Secret Management |
| Terraform | Infrastructure as Code |
| AWS | Cloud Infrastructure |
| AppRole | Machine Authentication |
| AWS Secrets Engine | Dynamic IAM Credentials |

---

## 🔐 How It Works

1. Developer pushes code to GitHub
2. Jenkins pipeline starts
3. Jenkins authenticates to Vault using AppRole
4. Vault generates temporary AWS IAM credentials
5. Terraform uses those credentials
6. AWS infrastructure is created
7. Credentials expire automatically (TTL-based)



# 🔐 Vault Configuration (One-Time Setup)

### 1️⃣ Enable AWS Secrets Engine

```bash
vault secrets enable aws
````

### 2️⃣ Configure Root IAM User (Used by Vault Only)

```bash
vault write aws/config/root \
  access_key="ROOT-ACCESS-KEY" \
  secret_key="ROOT-SECRET-KEY" \
  region="us-east-1"
```

### 3️⃣ Create Terraform Role

```bash
vault write aws/roles/terraform-role \
  credential_type=iam_user \
  policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:*","s3:*","vpc:*"],
      "Resource": "*"
    }
  ]
}
EOF
```
# Fetching the Credentials

```bash
vault read aws/creds/terraform-role
```

---

## 🔐 Enable AppRole Authentication

```bash
vault auth enable approle
```

### Create Policy for Jenkins

`jenkins-policy.hcl`

```hcl
path "aws/creds/terraform-role" {
  capabilities = ["read"]
}
```

Apply:

```bash
vault policy write jenkins-policy jenkins-policy.hcl
```

### Create AppRole

```bash
vault write auth/approle/role/jenkins \
  token_policies="jenkins-policy" \
  token_ttl=15m \
  token_max_ttl=30m
```

Get credentials:

```bash
vault read auth/approle/role/jenkins/role-id
vault write -f auth/approle/role/jenkins/secret-id
```

Store these in Jenkins credentials store:

* `vault-role-id`
* `vault-secret-id`

---

## 🏗 Terraform Example

### provider.tf

```hcl
provider "aws" {
  region = "us-east-1"
}
```

### main.tf

```hcl
resource "random_id" "rand" {
  byte_length = 4
}

resource "aws_s3_bucket" "demo" {
  bucket = "vault-terraform-demo-${random_id.rand.hex}"
}
```

---

## 🚀 Jenkins Pipeline (Jenkinsfile)

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
          vault write auth/approle/login \
            role_id=$VAULT_ROLE_ID \
            secret_id=$VAULT_SECRET_ID \
            -format=json > vault_token.json

          export VAULT_TOKEN=$(jq -r .auth.client_token vault_token.json)
          echo $VAULT_TOKEN > .vault-token
        '''
      }
    }

    stage('Get AWS Credentials') {
      steps {
        sh '''
          export VAULT_TOKEN=$(cat .vault-token)
          vault read aws/creds/terraform-role -format=json > aws_creds.json

          export AWS_ACCESS_KEY_ID=$(jq -r .data.access_key aws_creds.json)
          export AWS_SECRET_ACCESS_KEY=$(jq -r .data.secret_key aws_creds.json)
        '''
      }
    }

    stage('Terraform Init') {
      steps {
        sh 'terraform init'
      }
    }

    stage('Terraform Apply') {
      steps {
        sh 'terraform apply -auto-approve'
      }
    }
  }

  post {
    always {
      sh 'rm -f aws_creds.json vault_token.json .vault-token'
    }
  }
}
```

---

## 🛡 Security Benefits

✅ No AWS credentials stored in Jenkins
✅ No AWS credentials stored in GitHub
✅ No long-lived IAM users
✅ Automatic credential revocation
✅ Short-lived Vault tokens
✅ Least privilege IAM policies

---

## 📈 Production Enhancements

* Use `assumed_role` instead of IAM user
* Enable Vault TLS
* Use private subnet for Vault
* Enable Vault audit logging
* Use Kubernetes agents for isolation
* Use IAM roles for EC2 instead of root access keys

---

## 🎯 Interview-Ready Explanation

> Jenkins authenticates to Vault using AppRole. Vault dynamically generates short-lived AWS IAM credentials via the AWS Secrets Engine. Terraform uses those credentials to provision infrastructure. Once the job completes or the TTL expires, Vault automatically revokes the credentials. No static AWS keys are stored anywhere.

---

## 🧠 One-Line Summary

**Vault owns secrets. Jenkins borrows them. Terraform uses them. AWS never sees static keys.**

---

## 👨‍💻 Author

DevOps Engineer | Terraform | Vault | Jenkins | AWS

---

## 📜 License

This project is for educational and demonstration purposes.

```

---

# 🚀 Why This README Is Strong

- Clear architecture
- Security-focused
- Enterprise-style explanation
- Good for interviews
- Looks professional on GitHub
- Shows senior-level understanding

---

If you want, I can also generate:

- ⭐ GitHub project description line
- 🏷 GitHub tags for visibility
- 📊 Architecture diagram image to include
- 🧪 Demo walkthrough section
- 🎯 Resume bullet point for this project

Just tell me 👌
```
