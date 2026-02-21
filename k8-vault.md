Perfect 🔥 If Vault is running **outside Kubernetes** (like on an EC2 VM), then the setup is **slightly different**, but 100% doable.

Below is a **complete end-to-end working setup**:

✅ Vault (outside)
✅ Kubernetes auth enabled in Vault
✅ ServiceAccount in Kubernetes for token review
✅ Vault Role bound to K8s ServiceAccount
✅ Pod reads secret from Vault

---

# ✅ Architecture (Simple Flow)

1. Pod runs with a ServiceAccount (example: `app-sa`)
2. Pod sends its JWT token to Vault
3. Vault verifies JWT using Kubernetes API (`token_reviewer_jwt`)
4. Vault issues a Vault token
5. Pod uses Vault token to read secrets

---

# 🧩 Step 0 — Requirements

You need:

* `kubectl` access to cluster
* Vault installed on EC2 (or VM)
* Vault reachable from Kubernetes cluster (network)
* Kubernetes API reachable from Vault VM

---

# ✅ Step 1 — Get Kubernetes API Server (kubernetes_host)

On your laptop or VM where kubectl works:

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
echo
```

Example output:

```
https://A1B2C3D4.gr7.ap-south-1.eks.amazonaws.com
```

Save it.

---

# ✅ Step 2 — Get Kubernetes CA Cert (ca.crt)

Because Vault is outside K8s, it won’t have `/var/run/.../ca.crt`.

Run:

```bash
kubectl config view --raw --minify \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > ca.crt
```

Now you have file:

✅ `ca.crt`

Copy this file to your Vault VM.

---

# ✅ Step 3 — Create ServiceAccount for Vault Token Review

We will create a service account only for Vault.

Example namespace: `vault-auth`

```bash
kubectl create namespace vault-auth
kubectl create sa vault-auth -n vault-auth
```

---

# ✅ Step 4 — Give this SA permission to review tokens

Vault needs permission to call Kubernetes TokenReview API.

Run:

```bash
kubectl create clusterrolebinding vault-tokenreview-binding \
  --clusterrole=system:auth-delegator \
  --serviceaccount=vault-auth:vault-auth
```

---

# ✅ Step 5 — Get token_reviewer_jwt (ServiceAccount JWT)

### For modern Kubernetes (v1.24+)

Run:

```bash
kubectl -n vault-auth create token vault-auth
```

This prints a JWT token.

Copy it → this is:

✅ `token_reviewer_jwt`

---

# ✅ Step 6 — Enable Kubernetes Auth in Vault (on Vault VM)

On Vault server:

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
vault login <root-token>
```

Enable auth:

```bash
vault auth enable kubernetes
```

---

# ✅ Step 7 — Configure Kubernetes Auth in Vault (Vault VM)

Copy `ca.crt` to Vault server directory.

Now run:

```bash
vault write auth/kubernetes/config \
  kubernetes_host="https://YOUR-K8S-ENDPOINT" \
  kubernetes_ca_cert=@ca.crt \
  token_reviewer_jwt="PASTE_TOKEN_HERE"
```

---

# ✅ Step 8 — Create a Vault Secret (Test)

Example:

```bash
vault kv put secret/expense username="roboshop" password="roboshop123"
```

Verify:

```bash
vault kv get secret/expense
```

---

# ✅ Step 9 — Create Vault Policy

Create policy file:

```bash
cat > expense-policy.hcl <<EOF
path "secret/data/expense" {
  capabilities = ["read"]
}
EOF
```

Apply policy:

```bash
vault policy write expense-policy expense-policy.hcl
```

---

# ✅ Step 10 — Create Vault Role for Kubernetes

This binds K8s ServiceAccount → Vault Policy.

Example:

* namespace = `expense`
* serviceaccount = `expense-sa`

```bash
vault write auth/kubernetes/role/expense-role \
  bound_service_account_names=expense-sa \
  bound_service_account_namespaces=expense \
  policies=expense-policy \
  ttl=1h
```

---

# ✅ Step 11 — Create Kubernetes Namespace + ServiceAccount for App

```bash
kubectl create namespace expense
kubectl create sa expense-sa -n expense
```

---

# ✅ Step 12 — Create Test Pod and Read Vault Secret

### Create pod yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-test
  namespace: expense
spec:
  serviceAccountName: expense-sa
  containers:
  - name: test
    image: curlimages/curl:latest
    command: ["sh", "-c", "sleep 3600"]
```

Apply:

```bash
kubectl apply -f pod.yaml
```

---

# ✅ Step 13 — Inside pod: Login to Vault using Kubernetes Auth

Exec into pod:

```bash
kubectl exec -it vault-test -n expense -- sh
```

Inside pod:

### 1) Get JWT token

```sh
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

### 2) Login to Vault

```sh
curl -s --request POST \
  --data '{"jwt": "'"$TOKEN"'", "role": "expense-role"}' \
  http://<VAULT-SERVER-IP>:8200/v1/auth/kubernetes/login
```

This returns:

✅ `auth.client_token`

Copy that.

---

# ✅ Step 14 — Read Secret using Vault Token

```sh
VAULT_TOKEN="<client_token_here>"

curl -s \
  --header "X-Vault-Token: $VAULT_TOKEN" \
  http://<VAULT-SERVER-IP>:8200/v1/secret/data/expense
```

You will get:

```json
{
  "data": {
    "data": {
      "username": "roboshop",
      "password": "roboshop123"
    }
  }
}
```

🎉 DONE.

---

# ⚠️ Common Issues (Very Important)

## 1) Vault cannot reach K8s API

Vault VM must reach:

`https://<eks-endpoint>:443`

So open:

* Security group outbound allowed
* Route + DNS works

---

## 2) Pod cannot reach Vault server

Kubernetes nodes must reach Vault VM on:

`8200`

So open inbound in Vault VM SG:

* TCP 8200 from node subnet / VPC CIDR

---

## 3) Wrong CA cert

If CA mismatch → Vault will fail with:
`x509: certificate signed by unknown authority`

---

# ⭐ Production Best Practice (Quick)

Instead of doing curl manually, use:

✅ Vault Agent Injector (but it needs Vault inside cluster)
OR
✅ Vault Agent sidecar
OR
✅ External Secrets Operator + Vault

---

If you want, I can now give you the **best real-world approach**:

### ✅ Vault outside + Kubernetes apps auto-inject secrets using Vault Agent Sidecar

(no manual curl, secrets go into files/env automatically)
