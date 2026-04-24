# Infragate — Production Testing Guide

Real deployment testing on infrastructure. Two paths: **k3s on a single OCI VM** and **existing OKE cluster**. This guide tracks current validation status and next test phases (BYON and OKE-hosted Infragate) so test scope is explicit.

---

## Table of Contents

1. [Choose Your Test Path](#1-choose-your-test-path)
2. [Path A: k3s on OCI VM](#2-path-a-k3s-on-oci-vm)
3. [Path B: Existing OKE Cluster](#3-path-b-existing-oke-cluster)
4. [Deploy Infragate (Both Paths)](#4-deploy-infragate-both-paths)
5. [Configure Keycloak](#5-configure-keycloak)
6. [Smoke Tests](#6-smoke-tests)
7. [Cluster Lifecycle Testing](#7-cluster-lifecycle-testing)
8. [Template & RBAC Testing](#8-template--rbac-testing)
9. [Security Testing](#9-security-testing)
10. [Resilience Testing](#10-resilience-testing)
11. [Teardown](#11-teardown)
12. [Checklist](#12-checklist)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Choose Your Test Path

| | **Path A: k3s (single-node VM)** | **Path B: Existing OKE cluster** |
|---|---|---|
| **Best for** | Quick validation, Always Free tier, dev/test | Production-like testing, enterprise deployment |
| **Runtime** | k3s on a single OCI VM | Multi-node OKE cluster |
| **Ingress** | traefik (bundled with k3s — zero setup) | ingress-nginx with OCI Load Balancer (installed separately) |
| **Storage** | local-path | oci-bv (OCI Block Volume) |
| **Values file** | `values-oci.yaml` | `values-oke.yaml` |
| **Setup time** | ~15 min | ~30 min (if cluster exists) |

Both paths target the same Infragate functionality. Steps 4-10 define the test plan for both paths, while the status table below tracks what is already validated vs. pending.

### Architecture

**Path A: k3s on single VM**

```
┌──────────────────────────────────────────────────────────┐
│  OCI VM (VM.Standard.E5.Flex or similar)                 │
│  ├── k3s (single-node Kubernetes)                        │
│  │   ├── infragate-api         (FastAPI backend)         │
│  │   ├── infragate-frontend    (nginx + static portal)   │
│  │   ├── infragate-postgresql  (bundled database)        │
│  │   ├── infragate-keycloak    (bundled OIDC provider)   │
│  │   └── traefik               (ingress controller)      │
│  └── Helm 3.x                                           │
└──────────────┬───────────────────────────────────────────┘
               │ Infragate provisions OKE clusters via OCI API
               ▼
┌──────────────────────────────────────────────────────────┐
│  OCI Tenancy                                             │
│  ├── OKE Cluster (created by test deploy via portal)     │
│  └── Object Storage (infragate-tfstate bucket)           │
└──────────────────────────────────────────────────────────┘
```

**Path B: Existing OKE cluster**

```
┌──────────────────────────────────────────────────────────┐
│  OKE Cluster (existing, managed by OCI)                  │
│  ├── infragate namespace                                 │
│  │   ├── infragate-api         (FastAPI backend)         │
│  │   ├── infragate-frontend    (nginx + static portal)   │
│  │   ├── infragate-postgresql  (bundled, oci-bv PVC)     │
│  │   ├── infragate-keycloak    (bundled OIDC provider)   │
│  │   └── ingress-nginx         (OCI flexible LB)         │
│  └── ingress-nginx namespace                             │
│      └── ingress-nginx-controller (OCI Load Balancer)    │
└──────────────┬───────────────────────────────────────────┘
               │ Infragate provisions OKE clusters via OCI API
               ▼
┌──────────────────────────────────────────────────────────┐
│  OCI Tenancy                                             │
│  ├── OKE Cluster (created by test deploy via portal)     │
│  └── Object Storage (infragate-tfstate bucket)           │
└──────────────────────────────────────────────────────────┘
```

### Current validation status (as of 2026-04-24)

| Area | Status | Notes |
|---|---|---|
| Path A - k3s host + full lifecycle (deploy/scale/destroy) | Completed | Officially tested and working end-to-end |
| BYON lifecycle | Planned next | Add Advanced override test matrix (section 7f) |
| Path B - deploy Infragate into existing OKE | Planned after BYON | Needed to validate production-like runner topology |

### OCI prerequisites (both paths)

Complete these steps in OCI Console before starting either path. Infragate uses an OCI service account to provision OKE clusters, networking, and compartments via Terraform.

#### Step 1 — Create a parent compartment for Infragate

Infragate creates child compartments for each cluster it provisions. You need a parent compartment where those children will live.

1. **OCI Console** > **Identity & Security** > **Compartments**
2. Click **Create Compartment**
3. Name: `infragate` (or `infragate-test` for testing)
4. Description: `Parent compartment for Infragate-managed OKE clusters`
5. Parent: your root compartment (or your preferred hierarchy)
6. Click **Create**
7. **Copy the compartment OCID** — you'll need it as `parentCompartmentOcid`

#### Step 2 — Create the infragate-svc service account

A dedicated user that Infragate uses to call OCI APIs. Never use your personal account.

OCI now manages users under **Identity Domains** (per compartment). Each tenancy has a **Default** domain.

1. **OCI Console** > **Identity & Security** > **Domains**
2. Click the **Default** domain (or whichever domain your tenancy uses)
3. In the left sidebar, click **Users**
4. Click **Create user**
5. First name: `Infragate`, Last name: `Service`
6. Username: `infragate-svc`
7. Email: use a shared team address (required by Domains — e.g. `infragate-svc@yourdomain.com`)
8. Uncheck **Use the email address as the username**
9. Click **Create**
10. **Copy the user OCID** from the user details page — you'll need it as `userOcid`

> If your tenancy still uses the legacy Identity (no Domains), go to **Identity & Security** > **Users** > **Create User** > select **IAM User** instead.

#### Step 3 — Create an API key for infragate-svc

1. In the Default domain, go to **Users** > click `infragate-svc`
2. In the left sidebar under **Resources**, click **API keys**
3. Click **Add API key**
4. Select **Generate API key pair**
5. Click **Download private key** — save as `oci_api_key.pem`
6. Click **Add**
7. **Copy the fingerprint** shown (format: `aa:bb:cc:dd:...`) — you'll need it as `fingerprint`
8. Note the **Tenancy OCID** from the config file preview — you'll need it as `tenancyOcid`

> Keep `oci_api_key.pem` safe. You'll upload it as a Kubernetes secret in step 4a.

#### Step 4 — Create a group and IAM policies for infragate-svc

Infragate needs permissions to manage OKE clusters, networking, compute, and compartments within the parent compartment. Use **group-based policies** — the older `any-user where request.user.id = <ocid>` pattern silently fails for the S3-compatible Object Storage API on Identity-Domain tenancies and produces misleading `NoSuchBucket` errors against a bucket that clearly exists.

##### Step 4a — Create the group and add the user

1. **OCI Console** > **Identity** > **Domains** > **Default** > **Groups**
2. Click **Create group**
3. Name: `InfragateSvcGroup`
4. Description: `Infragate service account group`
5. Click **Create**
6. Open the new group, click **Assign user to group**, select `infragate-svc`, and save.

> Legacy IAM (no Domains): **Identity & Security** > **Groups** > **Create Group**, then add `infragate-svc` as a member.

##### Step 4b — Create the policy

1. **OCI Console** > **Identity & Security** > **Policies**
2. Click **Create Policy**
3. Name: `infragate-svc-policy`
4. Description: `Permissions for Infragate service account`
5. Compartment: **root compartment** (policies must be at root or above the target compartment)
6. Switch to **Manual Editor** and paste:

```
Allow group InfragateSvcGroup to manage cluster-family in compartment infragate
Allow group InfragateSvcGroup to manage virtual-network-family in compartment infragate
Allow group InfragateSvcGroup to manage instance-family in compartment infragate
Allow group InfragateSvcGroup to manage compartments in compartment infragate
Allow group InfragateSvcGroup to manage object-family in compartment infragate
Allow group InfragateSvcGroup to inspect compartments in tenancy
Allow group InfragateSvcGroup to use tag-namespaces in tenancy
```

> Replace `infragate` with your compartment name if different. If you created the group inside an Identity Domain other than `Default`, qualify the name: `Allow group 'MyDomain'/'InfragateSvcGroup' to ...`
>
> `inspect compartments in tenancy` is required so the S3-compat gateway can resolve the bucket's parent compartment chain — without it, Terraform gets `NoSuchBucket` even against a valid bucket.
> `use tag-namespaces in tenancy` is only needed if any resources apply defined tags; harmless to include.

7. Click **Create**

**Verify policies are correct** — if any policy is missing, Terraform will fail with `404 NotAuthorizedOrNotFound` (or `NoSuchBucket` for state backend issues) during cluster provisioning.

#### Step 5 — Create the Terraform state bucket

Infragate stores Terraform state for each provisioned cluster in OCI Object Storage using the S3-compatible API.

1. **OCI Console** > **Storage** > **Object Storage** > **Buckets**
2. Make sure you're in the **correct compartment** (root or the one where you want the bucket)
3. Click **Create Bucket**
4. Bucket Name: `infragate-tfstate`
5. Default Storage Tier: **Standard**
6. Encryption: **Oracle-Managed Keys** (default)
7. Click **Create**
8. **Note the Namespace** shown on the bucket details page — you'll need it as `namespace`

#### Step 6 — Generate S3 compatibility credentials

Terraform's S3 backend needs Customer Secret Keys to access the Object Storage bucket.

1. **OCI Console** > **Identity & Security** > **Domains** > **Default** domain
2. Click **Users** > click `infragate-svc`
3. In the left sidebar under **Resources**, click **Customer secret keys**
4. Click **Generate secret key**
5. Name: `infragate-tfstate-s3`
6. Click **Generate secret key**
7. **Copy the Secret Key immediately** — it's shown only once. This is your `s3SecretKey`.
8. **Copy the Access Key** from the table — this is your `s3AccessKey`.

> If using legacy Identity (no Domains): **Identity & Security** > **Users** > `infragate-svc` > **Customer Secret Keys**.

#### Summary — values you should have

| Value | Where it goes | Example |
|---|---|---|
| Tenancy OCID | `api.oci.tenancyOcid` | `ocid1.tenancy.oc1..aaaaaa...` |
| infragate-svc User OCID | `api.oci.userOcid` | `ocid1.user.oc1..aaaaaa...` |
| API Key Fingerprint | `api.oci.fingerprint` | `aa:bb:cc:dd:ee:ff:00:11:...` |
| API Key PEM file | Kubernetes secret `infragate-oci-key` | `oci_api_key.pem` |
| Parent Compartment OCID | `api.oci.parentCompartmentOcid` | `ocid1.compartment.oc1..aaaaaa...` |
| Object Storage Namespace | `api.oci.namespace` | `axk2bz5example` |
| S3 Access Key | `api.terraform.s3AccessKey` | `abcdef1234567890` |
| S3 Secret Key | `api.terraform.s3SecretKey` | `ABCDEF...long-base64...` |
| Region | `api.oci.region` | `eu-frankfurt-1` |

---

## 2. Path A: k3s on OCI VM

### 2a. VM requirements

| Setting | Recommended |
|---|---|
| Shape | VM.Standard.A1.Flex (ARM, Always Free eligible), E4.Flex, or E5.Flex |
| OCPU | 2+ |
| RAM | 16 GB+ |
| Boot volume | 50 GB+ |
| OS | Oracle Linux 8/9 or Ubuntu 22.04/24.04 |
| Region | eu-frankfurt-1 (or your preferred region) |

### 2b. Configure VM networking (OCI Console)

The VM needs inbound access for HTTP and SSH, plus full outbound for pulling images and reaching OCI APIs.

**VCN Security List — Ingress rules (all stateful):**

| Stateless | Source CIDR | Protocol | Dest Port | Description |
|---|---|---|---|---|
| No | `0.0.0.0/0` | TCP | 22 | SSH access |
| No | `0.0.0.0/0` | TCP | 80 | HTTP (Infragate portal) |
| No | `0.0.0.0/0` | TCP | 443 | HTTPS (if using TLS later) |
| No | `0.0.0.0/0` | TCP | 6443 | k3s API (optional — only if you need remote kubectl) |

**VCN Security List — Egress rules (all stateful):**

| Stateless | Dest CIDR | Protocol | Dest Port | Description |
|---|---|---|---|---|
| No | `0.0.0.0/0` | TCP | 443 | GHCR image pulls, OCI API, Terraform registry |
| No | `0.0.0.0/0` | TCP | 80 | HTTP redirects |

> **Stateful vs Stateless:** Use stateful (Stateless = No). Stateful rules automatically allow return traffic for established connections. Stateless requires manually defining matching rules in both directions — unnecessary complexity for this use case.
>
> **Tip:** OCI default security lists allow all egress. If you're using a custom security list, make sure outbound 443 is open.

**OS firewall (on the VM itself):**

Oracle Linux 8/9 has `firewalld` enabled by default. Open the required ports:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=6443/tcp

# CRITICAL: Allow inter-pod traffic on CNI interfaces
# Without this, traefik will return 502 Bad Gateway because
# firewalld blocks pod-to-pod communication on the CNI bridge
sudo firewall-cmd --zone=trusted --add-interface=cni0 --permanent
sudo firewall-cmd --zone=trusted --add-interface=flannel.1 --permanent
sudo firewall-cmd --reload
```

Ubuntu uses `ufw` (usually inactive on OCI images). If active:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 6443/tcp
```

### 2c. SSH into the VM and install dependencies

```bash
ssh -i ~/.ssh/your_key opc@<VM_PUBLIC_IP>
```

**Install k3s (traefik ingress ships bundled and starts automatically):**

```bash
curl -sfL https://get.k3s.io | sh -

# Wait for k3s to be ready
sudo systemctl status k3s

# Copy kubeconfig to your user (k3s.yaml is root-owned by default)
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc

# Verify
kubectl get nodes
# Expected: 1 node, STATUS=Ready
```

**Install git:**

```bash
# Oracle Linux / RHEL
sudo dnf install -y git

# Ubuntu / Debian
# sudo apt-get install -y git

git --version
```

**Install Helm:**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
# Expected: v3.x
```

### 2d. Verify the VM is ready

```bash
# Kubernetes running
kubectl get nodes
# Expected: 1 node, Ready

# Traefik ingress controller running (bundled with k3s)
kubectl get pods -n kube-system | grep traefik
# Expected: traefik pod Running

# Disk space (need at least 10 GB free)
df -h /

# Outbound connectivity — can reach GHCR?
curl -sI https://ghcr.io | head -1
# Expected: HTTP/2 200 or HTTP/2 301

# Outbound connectivity — can reach OCI API?
curl -sI https://containerengine.eu-frankfurt-1.oci.oraclecloud.com | head -1
# Expected: HTTP/1.1 400 or similar (reachable, just no valid request)
```

### 2e. Configure DNS

Use nip.io for testing (no DNS setup needed):

```bash
# Your domain will be:
echo "infragate.<VM_PUBLIC_IP>.nip.io"
# e.g. infragate.129.159.42.10.nip.io
```

For production: create an A record `infragate.yourdomain.com → <VM_PUBLIC_IP>`.

### 2f. Clone the repository

The repository is private. Use a GitHub Personal Access Token (PAT) with `repo` scope.

Generate one at: **GitHub** > **Settings** > **Developer settings** > **Personal access tokens** > **Tokens (classic)** > **Generate new token** — select `repo` scope.

```bash
git clone https://<YOUR_GITHUB_TOKEN>@github.com/SolviaLab/infragate.git
cd infragate
git checkout DEV
```

**Continue to [Step 4 — Deploy Infragate](#4-deploy-infragate-both-paths).**

---

## 3. Path B: Existing OKE Cluster

### 3a. OKE cluster requirements

| Setting | Recommended |
|---|---|
| Node pool | 2+ nodes (E4.Flex, 2 OCPU / 16 GB RAM each) |
| K8s version | v1.32+ |
| Network type | Flannel or VCN-native |
| Region | eu-frankfurt-1 (or your preferred region) |

### 3b. Configure kubectl access

```bash
# From your workstation (requires OCI CLI installed)
oci ce cluster create-kubeconfig \
  --cluster-id ocid1.cluster.oc1..YOUR_CLUSTER_OCID \
  --file ~/.kube/config \
  --region eu-frankfurt-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT

kubectl get nodes
# Expected: 2+ nodes, STATUS=Ready

helm version
# Expected: v3.x
```

### 3c. Verify OKE storage class

```bash
kubectl get storageclass
# Must include: oci-bv (OCI Block Volume)
# If missing, your OKE cluster may need the OCI CSI driver
```

### 3d. Install ingress-nginx

OKE does not come with an ingress controller.

```bash
# Check if already installed
kubectl get pods -n ingress-nginx 2>/dev/null
# If you see ingress-nginx-controller pods, skip this step

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.annotations."oci\.oraclecloud\.com/load-balancer-type"=lb \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape"=flexible \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape-flex-min"=10 \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape-flex-max"=100
```

### 3e. Get the load balancer IP

```bash
# Wait for the LB to be provisioned (1-3 minutes)
kubectl get svc -n ingress-nginx ingress-nginx-controller -w
# Wait until EXTERNAL-IP changes from <pending> to an IP

LB_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Load Balancer IP: $LB_IP"
```

### 3f. Configure DNS

- **For testing:** `infragate.<LB_IP>.nip.io`
- **For production:** A record `infragate.yourdomain.com → <LB_IP>`

### 3g. Clone the repository

The repository is private — use a GitHub PAT with `repo` scope (same as Path A step 2f).

```bash
git clone https://<YOUR_GITHUB_TOKEN>@github.com/SolviaLab/infragate.git
cd infragate
git checkout DEV
```

---

## 4. Deploy Infragate (Both Paths)

CI builds images on every push to `DEV` or `main` and pushes to GHCR. No local builds needed.

| Branch | Image tag | Example |
|---|---|---|
| `DEV` | `dev-latest` or `dev-<sha>` | `ghcr.io/solvialab/infragate-api:dev-latest` |
| `main` | `latest` or `<sha>` | `ghcr.io/solvialab/infragate-api:latest` |

### 4a. Transfer the OCI API key to the target machine

The `oci_api_key.pem` is the **API key generated for the `infragate-svc` service account** (prerequisite Step 3) — not your personal user's key. Infragate uses this key for all Terraform operations. You need to transfer it to the machine where `kubectl` runs.

**Path A (k3s) — copy the PEM from your workstation to the VM:**

```bash
# Run this on your WORKSTATION (not the VM)
scp -i ~/.ssh/your_key /path/to/oci_api_key.pem opc@<VM_PUBLIC_IP>:~/oci_api_key.pem

# Then on the VM — verify the file arrived and looks correct
ls -la ~/oci_api_key.pem
# Expected: file exists, size ~1.6-1.7 KB

head -1 ~/oci_api_key.pem
# Expected: -----BEGIN RSA PRIVATE KEY----- (or -----BEGIN PRIVATE KEY-----)

chmod 600 ~/oci_api_key.pem
```

**Path B (OKE) — the PEM is already on your workstation where kubectl runs.** No transfer needed.

The OCI key secret will be created **after** the Helm install (which creates the namespace). See step 4c.

### 4b. Create your values file

**Path A (k3s):**

```bash
cp deploy/helm/values-oci.yaml deploy/helm/values-local.yaml
```

**Path B (OKE):**

```bash
cp deploy/helm/values-oke.yaml deploy/helm/values-local.yaml
```

Edit `deploy/helm/values-local.yaml` with your actual values.

<details>
<summary><strong>Path A (k3s) — full annotated values-local.yaml</strong></summary>

```yaml
global:
  domain: <YOUR_DOMAIN>                    # e.g. infragate.129.159.42.10.nip.io

api:
  replicaCount: 1
  image:
    repository: ghcr.io/solvialab/infragate-api
    tag: dev-latest
    pullPolicy: Always
  env:
    APP_ENV: production
    CORS_ORIGINS: "http://<YOUR_DOMAIN>"
  oidc:
    issuer: http://<YOUR_DOMAIN>/auth/realms/infragate
    clientId: infragate-portal
  oci:
    tenancyOcid: "ocid1.tenancy.oc1..YOUR_VALUE"
    userOcid: "ocid1.user.oc1..YOUR_VALUE"
    fingerprint: "aa:bb:cc:dd:ee:ff:00:11:22:33:44:55:66:77:88:99"
    region: eu-frankfurt-1
    namespace: "your_obj_storage_namespace"
    parentCompartmentOcid: "ocid1.compartment.oc1..YOUR_VALUE"
  terraform:
    stateBucket: infragate-tfstate
    s3AccessKey: "YOUR_S3_ACCESS_KEY"
    s3SecretKey: "YOUR_S3_SECRET_KEY"

frontend:
  replicaCount: 1
  image:
    repository: ghcr.io/solvialab/infragate-frontend
    tag: dev-latest
    pullPolicy: Always

postgresql:
  enabled: true
  storage:
    size: 20Gi

keycloak:
  enabled: true

ingress:
  enabled: true
  className: traefik
  tls:
    enabled: false
```

</details>

<details>
<summary><strong>Path B (OKE) — full annotated values-local.yaml</strong></summary>

```yaml
global:
  domain: <YOUR_DOMAIN>                    # e.g. infragate.<LB_IP>.nip.io

api:
  replicaCount: 1
  image:
    repository: ghcr.io/solvialab/infragate-api
    tag: dev-latest
    pullPolicy: Always
  env:
    APP_ENV: production
    CORS_ORIGINS: "http://<YOUR_DOMAIN>"
  oidc:
    issuer: http://<YOUR_DOMAIN>/auth/realms/infragate
    clientId: infragate-portal
  oci:
    tenancyOcid: "ocid1.tenancy.oc1..YOUR_VALUE"
    userOcid: "ocid1.user.oc1..YOUR_VALUE"
    fingerprint: "aa:bb:cc:dd:ee:ff:00:11:22:33:44:55:66:77:88:99"
    region: eu-frankfurt-1
    namespace: "your_obj_storage_namespace"
    parentCompartmentOcid: "ocid1.compartment.oc1..YOUR_VALUE"
  terraform:
    stateBucket: infragate-tfstate
    s3AccessKey: "YOUR_S3_ACCESS_KEY"
    s3SecretKey: "YOUR_S3_SECRET_KEY"

frontend:
  replicaCount: 1
  image:
    repository: ghcr.io/solvialab/infragate-frontend
    tag: dev-latest
    pullPolicy: Always

postgresql:
  enabled: true
  storage:
    size: 50Gi                             # OCI Block Volume minimum is 50 GB
    storageClassName: oci-bv

keycloak:
  enabled: true

ingress:
  enabled: true
  className: nginx
  tls:
    enabled: false
```

**Key differences from k3s:**
- `ingress.className: nginx` (not `traefik`) — OKE has no bundled ingress, nginx-ingress is installed separately and is the conventional choice
- `postgresql.storage.storageClassName: oci-bv` (OCI Block Volume, not local-path)
- `postgresql.storage.size: 50Gi` (OCI BV minimum is 50 GB)
- No control-plane tolerations (OKE has dedicated worker nodes)

</details>

### 4c. Deploy

This step creates the namespace, secrets, and deploys Infragate in one go using a deploy script. The script avoids long command-line issues with terminal wrapping.

**Before running:** You need a GitHub PAT with `read:packages` scope to pull images from GHCR. Create one at: **GitHub** > **Settings** > **Developer settings** > **Personal access tokens** > **Tokens (classic)**. You can reuse the same PAT you used to clone the repo if it also has `read:packages` scope.

**Create the deploy script:**

```bash
cat > /tmp/deploy-infragate.sh << 'SCRIPT'
#!/bin/bash
set -e

GHCR_USER="$1"
GHCR_TOKEN="$2"
VALUES_FILE="$3"

DB_PASS=$(openssl rand -base64 24)
KC_PASS=$(openssl rand -base64 24)

echo "DB_PASS=$DB_PASS" > ~/.infragate-creds
echo "KC_PASS=$KC_PASS" >> ~/.infragate-creds
chmod 600 ~/.infragate-creds
echo "DB: $DB_PASS"
echo "KC: $KC_PASS"

# 1. Deploy (Helm creates namespace, pods start but can't pull yet)
helm upgrade --install infragate deploy/helm/ \
  -n infragate --create-namespace \
  -f "$VALUES_FILE" \
  --set "global.imagePullSecrets[0].name=ghcr-pull-secret" \
  --set "postgresql.auth.password=$DB_PASS" \
  --set "postgresql.auth.postgresPassword=$DB_PASS" \
  --set "keycloak.admin.password=$KC_PASS" \
  --timeout 10m

# 2. Create secrets (namespace now exists, pods will retry automatically)
kubectl create secret docker-registry ghcr-pull-secret \
  -n infragate \
  --docker-server=ghcr.io \
  --docker-username="$GHCR_USER" \
  --docker-password="$GHCR_TOKEN"

kubectl create secret generic infragate-oci-key \
  --from-file=oci_api_key.pem=$HOME/oci_api_key.pem \
  -n infragate

rm -f $HOME/oci_api_key.pem

echo ""
echo "Deploy started. Secrets created."
echo "Pods will retry image pulls automatically."
echo "Watch progress with: kubectl get pods -n infragate -w"
SCRIPT
chmod +x /tmp/deploy-infragate.sh
```

**Run the deploy script:**

```bash
# Path A (k3s):
/tmp/deploy-infragate.sh <GITHUB_USERNAME> <GITHUB_TOKEN> deploy/helm/values-oci.yaml

# Path B (OKE):
/tmp/deploy-infragate.sh <GITHUB_USERNAME> <GITHUB_TOKEN> deploy/helm/values-oke.yaml
```

### 4d. Watch pods come up

```bash
kubectl get pods -n infragate -w
```

Wait until all 4 pods show `Running` and `1/1` Ready:

```
NAME                                  READY   STATUS    RESTARTS   AGE
infragate-api-xxx                     1/1     Running   0          2m
infragate-frontend-xxx                1/1     Running   0          2m
infragate-postgresql-0                1/1     Running   0          2m
infragate-keycloak-xxx                1/1     Running   0          3m
```

Keycloak takes 2-3 minutes on first boot (JVM startup + database migration).

### 4e. Verify services are accessible

```bash
DOMAIN="<YOUR_DOMAIN>"

# API health
curl -s http://$DOMAIN/api/v1/health | python3 -m json.tool
# Expected: {"status": "ok", "env": "production", "db": true}

# Frontend
curl -sI http://$DOMAIN/ | head -1
# Expected: HTTP/1.1 200 OK
```

**Path B (OKE) — additional checks:**

```bash
# PVC bound with oci-bv
kubectl get pvc -n infragate
# Expected: STATUS=Bound, STORAGECLASS=oci-bv

# Ingress routing
kubectl get ingress -n infragate
# Expected: CLASS=nginx, ADDRESS=<LB_IP>

# Load Balancer healthy
kubectl get svc -n ingress-nginx ingress-nginx-controller
# Expected: EXTERNAL-IP shows the LB IP, not <pending>
```

---

## 5. Configure Keycloak

### 5a. Access Keycloak admin console

```bash
# Via ingress (if routing is working)
open http://<YOUR_DOMAIN>/auth/

# Or via port-forward
kubectl port-forward svc/infragate-keycloak 9090:8080 -n infragate &
open http://localhost:9090
```

Login: `admin` / the `KC_PASS` from step 4c (saved in `~/.infragate-creds`).

### 5b. Create the infragate realm

1. Hover over "master" in the top-left dropdown
2. Click **Create Realm**
3. Realm name: `infragate`
4. Click **Create**

### 5c. Create the infragate-portal client

1. **Clients** > **Create client**
2. Client ID: `infragate-portal`
3. Client type: **OpenID Connect**
4. Click **Next**
5. Client authentication: **OFF** (public client — Infragate uses PKCE)
6. Click **Next**
7. Valid redirect URIs: `http://<YOUR_DOMAIN>/*`
8. Valid post logout redirect URIs: `http://<YOUR_DOMAIN>/*`
9. Web origins: `http://<YOUR_DOMAIN>`
10. Click **Save**

> **If using TLS (Cloudflare, cert-manager, etc.):** Use `https://` in all URIs above, and ensure your `values-dev.yaml` also uses `https://` for `CORS_ORIGINS` and `api.oidc.issuer`.

### 5d. Create realm roles

| Role | Purpose |
|---|---|
| `admin` | Grants access to the Infragate admin panel |
| `testing` | *(optional)* Access to TEST cluster templates |
| `uat` | *(optional)* Access to UAT cluster templates |
| `production` | *(optional)* Access to PROD cluster templates |

To create each role: **Realm roles** > **Create role** > enter name > **Save**.

### 5e. Create test users

**Regular user:**

1. **Users** > **Add user**
2. Username: `testuser`, Email: `testuser@test.com`
3. **Create** > **Credentials** tab > **Set password** (turn off "Temporary")

**Admin user:**

1. **Users** > **Add user**
2. Username: `testadmin`, Email: `testadmin@test.com`
3. **Create** > **Credentials** tab > **Set password** (turn off "Temporary")
4. **Role mapping** tab > **Assign role** > select `admin` > **Assign**

### 5f. Verify OIDC discovery

```bash
curl -s http://<YOUR_DOMAIN>/auth/realms/infragate/.well-known/openid-configuration \
  | python3 -m json.tool | head -5
# Expected: issuer matches "http://<YOUR_DOMAIN>/auth/realms/infragate"
```

Stop port-forward when done: `kill %1`

---

## 6. Smoke Tests

All 5 must pass before continuing to cluster lifecycle testing.

```bash
DOMAIN="<YOUR_DOMAIN>"

# 1. API health
echo "=== 1. API Health ==="
curl -s http://$DOMAIN/api/v1/health | python3 -m json.tool
# Expected: {"status":"ok","env":"production","db":true}

# 2. Frontend
echo "=== 2. Frontend ==="
curl -sI http://$DOMAIN/ | head -1
# Expected: HTTP/1.1 200 OK

# 3. OIDC discovery
echo "=== 3. OIDC Discovery ==="
curl -s http://$DOMAIN/auth/realms/infragate/.well-known/openid-configuration \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['issuer'])"
# Expected: "http://<YOUR_DOMAIN>/auth/realms/infragate"

# 4. Auth enforcement
echo "=== 4. Auth Enforcement ==="
curl -s -o /dev/null -w "%{http_code}" http://$DOMAIN/api/v1/users/me
# Expected: 403

# 5. Internal connectivity
echo "=== 5. Internal Connectivity ==="
kubectl exec -n infragate deploy/infragate-api -- curl -s http://localhost:8000/health
# Expected: {"status":"ok","env":"production","db":true}
```

---

## 7. Cluster Lifecycle Testing

The most critical test — deploying, scaling, and destroying an OKE cluster through the Infragate portal end-to-end.

### 7a. Configure the platform (Admin panel)

Open `http://<YOUR_DOMAIN>` and log in as `testadmin`.

Go to **Admin** > **Configuration** and set:

| Setting | Value | Notes |
|---|---|---|
| CIDR pool | `10.120.0.0/24` (add 2-3 more for multi-cluster tests) | Each cluster consumes one /24 |
| VM shapes | `VM.Standard.E4.Flex` | Must be available in your region |
| K8s versions | `v1.35.0` | Check OCI Console > OKE for available versions |
| Node images | *(leave empty)* | Auto-selects latest OKE-compatible image |
| Cluster limit | `3` | Allows testing multiple simultaneous clusters |
| Cluster tier | `BASIC_CLUSTER` | Free; use `ENHANCED_CLUSTER` for production tests |

Save — changes take effect immediately.

### 7b. Deploy a cluster

1. Go to **Deploy**
2. Enter cluster name: `test-cluster-1`
3. Select CIDR, K8s version, VM shape
4. Configure 1 node pool: 1 node, 1 OCPU, 16 GB RAM, 50 GB storage
5. Review the **Deployment Summary** — verify cost estimate appears
6. Click **Deploy**

**What to watch for:**

- SSE log stream opens with real-time Terraform output
- You see `terraform init`, `terraform plan`, `terraform apply`
- Plan shows ~12-15 resources (compartment, VCN, subnet, IGW, route table, security list, OKE cluster, node pool, SSH key, etc.)
- Apply takes **8-15 minutes** — this is normal for OKE
- Final message: "Cluster deployed successfully"

**After deployment:**

1. Go to **My Clusters** — verify the cluster card shows: Running status, cost estimate, node pool info
2. Click **Details** — verify: cluster info, node pools, cost breakdown, kubeconfig download button, SSH key download button

3. Download kubeconfig and test:

```bash
# Requires OCI CLI on the machine where you run kubectl
export KUBECONFIG=~/downloaded-kubeconfig.yaml
kubectl get nodes
# Should show the OKE worker node(s) with STATUS=Ready
```

### 7c. Scale the cluster

**Scale up nodes:**

1. Click **Scale** on the running cluster
2. Change node count from 1 to 2
3. Review change preview, click **Apply**
4. Watch SSE stream — takes **5-10 minutes**
5. Dashboard reflects updated node count

**Add a node pool:**

1. Click **Scale** > **Add Pool**
2. "pool-2": 1 node, 1 OCPU, 16 GB RAM, 50 GB storage
3. Apply and verify — dashboard should show 2 pools

**Remove a node pool:**

1. Click **Scale** > remove "pool-2"
2. Apply — Terraform destroys the pool resources

### 7d. Destroy the cluster

1. Click **Destroy** on the cluster
2. Confirm destruction
3. Watch SSE stream — `terraform plan -destroy` then `terraform destroy`
4. Takes **5-10 minutes**
5. Final message: "Cluster destroyed successfully"

**Verify in OCI Console:**

- [ ] **Developer Services > OKE** — cluster gone or "Deleting"
- [ ] **Networking > VCNs** — VCN removed
- [ ] **Identity > Compartments** — compartment removed or empty
- [ ] **Object Storage > infragate-tfstate** — state file still exists (for audit)
- [ ] Back in Infragate admin > CIDRs — CIDR released and available

### 7e. Deploy a second cluster (concurrent test)

1. Deploy `test-cluster-2` using a different CIDR
2. While it's deploying, verify `test-cluster-1`'s destroy completed cleanly
3. This validates CIDR pool isolation and concurrent Terraform operations
4. Destroy `test-cluster-2` when done

### 7f. BYON lifecycle test matrix (next phase)

Use these scenarios to validate Advanced-tab overrides end-to-end.

| Scenario | Advanced overrides to set | Expected behavior |
|---|---|---|
| BYON-1: Compartment only | `Existing Compartment OCID` | Reuses only the compartment. Infragate still creates a new VCN/subnets/network stack inside that compartment. No auto-discovery of existing VCN/subnet objects. |
| BYON-2: Full BYON network | `Existing Compartment OCID` + `Existing VCN OCID` + `Existing Subnet OCID` | Reuses your existing network objects; Terraform should not create Infragate VCN/subnet/route/security-list resources. |
| BYON-3: Network prerequisites negative test | Same as BYON-2, but with missing NAT/SGW routes | Node pool create should fail with node registration timeout. Confirms route prerequisites are enforced by reality, not hidden fallback logic. |

Recommended execution order:

1. Run BYON-1 deploy -> scale -> destroy and capture logs/screenshots.
2. Run BYON-2 deploy -> scale -> destroy and verify only OKE resources are created/destroyed.
3. Run BYON-3 once as a controlled failure, then fix routes and re-run to green.

Validation checks per BYON run:

- [ ] Deploy succeeds and cluster reaches `ACTIVE` (except planned BYON-3 failure case)
- [ ] Scale up/down works from UI and state remains consistent
- [ ] Destroy succeeds and releases CIDR allocation
- [ ] Terraform state path remains isolated per cluster/user
- [ ] OCI resources match intended ownership (managed vs reused)

---

## 8. Template & RBAC Testing

### 8a. Create and deploy from a template

1. **Admin > Cluster Templates > Create**:
   - Name: `Dev Small`
   - K8s version: `v1.35.0`, VM shape: `VM.Standard.E4.Flex`
   - 1 pool: "default", 1 node, 1 OCPU, 16 GB RAM, 50 GB storage
   - Tier: Basic, TTL: 24h, Destroy protection: OFF
   - Required role: *(empty — visible to all)*
2. Verify template appears in admin table with estimated cost
3. Go to **Deploy** — verify template card appears
4. Select the template — form fields pre-filled and locked, user can still set cluster name and CIDR
5. Deploy from template — verify cluster matches template specs

### 8b. Role-based template gating

1. Create template `Prod HA` with required role: `production`
2. Log in as `testuser` (no `production` role)
3. **Deploy** — `Prod HA` must **NOT** appear
4. In Keycloak: assign `production` role to `testuser`
5. Refresh — `Prod HA` must now appear
6. Remove the `production` role from `testuser` after testing

### 8c. Destroy protection

1. Create a template with **Destroy Protection** enabled
2. Deploy a cluster from it
3. As `testuser` — try to destroy → **blocked** (button disabled or 403)
4. As `testadmin` — force-destroy → **succeeds**

### 8d. TTL visibility

1. Deploy a cluster from a template with TTL (e.g. 24 hours)
2. Verify on dashboard:
   - TTL badge appears (green >24h, orange <24h, red <4h)
   - Detail page shows expiry timestamp and remaining time

---

## 9. Security Testing

### 9a. Container security

```bash
# All containers must run as non-root
for pod in $(kubectl get pods -n infragate -o jsonpath='{.items[*].metadata.name}'); do
  echo -n "$pod: "
  kubectl exec -n infragate $pod -- whoami 2>/dev/null || echo "N/A"
done
# Expected: infragate (API), nginx (frontend), postgres (PostgreSQL)

# Verify security contexts
kubectl get pods -n infragate -o json | \
  python3 -c "
import json, sys
pods = json.load(sys.stdin)['items']
for pod in pods:
    name = pod['metadata']['name']
    for c in pod['spec']['containers']:
        sc = c.get('securityContext', {})
        print(f\"{name}/{c['name']}: allowPrivilegeEscalation={sc.get('allowPrivilegeEscalation', 'NOT SET')}, readOnlyRootFilesystem={sc.get('readOnlyRootFilesystem', 'NOT SET')}\")
"
```

### 9b. CORS validation

```bash
DOMAIN="<YOUR_DOMAIN>"

# Unauthorized origin — no CORS headers
curl -s -H "Origin: https://evil.com" -I http://$DOMAIN/api/v1/users/me 2>&1 | grep -i "access-control"
# Expected: no output

# Authorized origin — CORS headers present
curl -s -H "Origin: http://$DOMAIN" -I http://$DOMAIN/api/v1/users/me 2>&1 | grep -i "access-control"
# Expected: Access-Control-Allow-Origin: http://<YOUR_DOMAIN>
```

### 9c. Role-based access control

```bash
DOMAIN="<YOUR_DOMAIN>"

# Get a regular user token
TOKEN=$(curl -s -X POST "http://$DOMAIN/auth/realms/infragate/protocol/openid-connect/token" \
  -d "grant_type=password" \
  -d "client_id=infragate-portal" \
  -d "username=testuser" \
  -d "password=<TESTUSER_PASSWORD>" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Regular user can access own data
curl -s -H "Authorization: Bearer $TOKEN" http://$DOMAIN/api/v1/users/me | python3 -m json.tool
# Expected: 200 with user data

# Regular user CANNOT access admin endpoints
curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" http://$DOMAIN/api/v1/admin/config
# Expected: 403

curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" http://$DOMAIN/api/v1/admin/users
# Expected: 403

curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" http://$DOMAIN/api/v1/admin/clusters
# Expected: 403
```

### 9d. Secrets not exposed

```bash
# No passwords in ConfigMaps
kubectl get configmap -n infragate -o yaml | grep -iE "password|secret_key|private_key" \
  | grep -v "secretName\|secretKeyRef\|privateKey\|passwordKey"
# Expected: no output

# Secrets stored as Kubernetes Secrets
kubectl get secrets -n infragate
# Should show: infragate-oci-key, postgresql credentials, keycloak credentials
```

### 9e. API authentication enforcement

```bash
DOMAIN="<YOUR_DOMAIN>"

# No token
curl -s -o /dev/null -w "%{http_code}" http://$DOMAIN/api/v1/users/me
# Expected: 403

# Invalid token
curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer invalid-token" http://$DOMAIN/api/v1/users/me
# Expected: 403
```

---

## 10. Resilience Testing

### 10a. Pod auto-recovery

```bash
kubectl delete pod -n infragate -l app.kubernetes.io/component=api
kubectl get pods -n infragate -w
# Should be back to Running/Ready within 60 seconds

sleep 10
curl -s http://<YOUR_DOMAIN>/api/v1/health | python3 -m json.tool
# Expected: {"status":"ok","env":"production","db":true}
```

### 10b. Database persistence

```bash
# Note current state: clusters, config, users in the dashboard

kubectl delete pod -n infragate -l app.kubernetes.io/component=postgresql
kubectl get pods -n infragate -w
# Should restart with data intact (PVC-backed storage)

sleep 15
kubectl exec -n infragate deploy/infragate-api -- curl -s http://localhost:8000/health | python3 -m json.tool
# Expected: {"status":"ok","db":true}

# Check dashboard — all clusters and configuration must still be there
```

### 10c. Keycloak recovery

```bash
kubectl delete pod -n infragate -l app.kubernetes.io/component=keycloak
kubectl get pods -n infragate -w
# Wait 2-3 minutes for JVM restart

curl -s http://<YOUR_DOMAIN>/auth/realms/infragate/.well-known/openid-configuration \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['issuer'])"
# Expected: "http://<YOUR_DOMAIN>/auth/realms/infragate"

# Log in via the portal — must still work with existing users
```

### 10d. Helm upgrade (rolling update)

```bash
helm upgrade infragate deploy/helm/ -n infragate \
  -f deploy/helm/values-local.yaml \
  --set api.image.tag=dev-latest \
  --set frontend.image.tag=dev-latest \
  --set api.image.pullPolicy=Always \
  --set frontend.image.pullPolicy=Always \
  --set postgresql.auth.password="$(grep DB_PASS ~/.infragate-creds | cut -d= -f2)" \
  --set postgresql.auth.postgresPassword="$(grep DB_PASS ~/.infragate-creds | cut -d= -f2)" \
  --set keycloak.admin.password="$(grep KC_PASS ~/.infragate-creds | cut -d= -f2)" \
  --wait --timeout 5m

kubectl rollout status deployment/infragate-api -n infragate
# Expected: "successfully rolled out"

kubectl rollout status deployment/infragate-frontend -n infragate
# Expected: "successfully rolled out"

# Dashboard must still work, all data preserved
curl -s http://<YOUR_DOMAIN>/api/v1/health | python3 -m json.tool
```

### 10e. Full pod restart (simulate node reboot)

```bash
# Delete ALL infragate pods at once
kubectl delete pods -n infragate --all

# Watch them all come back
kubectl get pods -n infragate -w
# All 4 pods should be Running/Ready within 3-5 minutes

# Verify everything works
curl -s http://<YOUR_DOMAIN>/api/v1/health | python3 -m json.tool
curl -sI http://<YOUR_DOMAIN>/ | head -1

# Dashboard data must be intact
```

---

## 11. Teardown

### Destroy managed clusters first

**Before uninstalling Infragate:** Destroy every OKE cluster created through the portal. Check the dashboard and destroy each one. If you uninstall Infragate first, you'll have orphaned OCI resources.

### Uninstall Infragate (both paths)

```bash
helm uninstall infragate -n infragate
kubectl delete pvc -n infragate --all
kubectl delete namespace infragate

kubectl get all -n infragate
# Expected: "No resources found in infragate namespace."
```

**Path A (k3s) — clear cached images:**

```bash
sudo k3s crictl rmi ghcr.io/solvialab/infragate-api:dev-latest 2>/dev/null || true
sudo k3s crictl rmi ghcr.io/solvialab/infragate-frontend:dev-latest 2>/dev/null || true
```

**Path B (OKE) — remove ingress-nginx (optional):**

```bash
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
# Wait for OCI Load Balancer to deprovision (check OCI Console)
```

### Verify OCI cleanup

Check OCI Console:

- [ ] No orphaned OKE clusters (Developer Services > OKE)
- [ ] No orphaned VCNs/subnets (Networking > VCNs)
- [ ] No orphaned compartments (Identity > Compartments)
- [ ] No dangling load balancers (Networking > Load Balancers)
- [ ] No unused block volumes (Storage > Block Volumes)
- [ ] Terraform state files in Object Storage (keep for audit or delete)

---

## 12. Checklist

Copy this for each test run.

### OCI Prerequisites

- [ ] Parent compartment created (e.g. `infragate`)
- [ ] Service account `infragate-svc` created (IAM User, not federated)
- [ ] API key generated for `infragate-svc`, PEM downloaded
- [ ] Group `InfragateSvcGroup` created, `infragate-svc` added as a member
- [ ] IAM policies created as **group-based** (`manage cluster-family`, `virtual-network-family`, `instance-family`, `compartments`, `object-family` in compartment + `inspect compartments in tenancy` + `use tag-namespaces in tenancy`)
- [ ] Object Storage bucket `infragate-tfstate` created
- [ ] S3 compatibility credentials generated for `infragate-svc`
- [ ] All OCI values collected (tenancy OCID, user OCID, fingerprint, namespace, parent compartment OCID, S3 keys)

### Environment Setup

- [ ] **(A)** VM provisioned, firewall configured, k3s + Helm installed
- [ ] **(B)** OKE cluster running, kubectl configured, ingress-nginx installed
- [ ] DNS configured (nip.io or real domain)
- [ ] CI pipeline green, images on GHCR

### Deployment

- [ ] OCI API key secret created in namespace
- [ ] Helm install completes without errors
- [ ] All 4 pods Running and Ready (API, frontend, PostgreSQL, Keycloak)
- [ ] **(B)** PostgreSQL PVC bound with `oci-bv`
- [ ] No error logs in API pod

### Keycloak

- [ ] Realm `infragate` created
- [ ] Client `infragate-portal` created (public, PKCE, correct redirect URIs)
- [ ] `admin` realm role created
- [ ] Test users created (regular + admin)
- [ ] OIDC discovery returns correct issuer

### Smoke Tests

- [ ] API health: `{"status":"ok","env":"production","db":true}`
- [ ] Frontend: HTTP 200
- [ ] OIDC discovery: issuer matches domain
- [ ] Unauthenticated API access returns 403
- [ ] Internal API-to-DB connectivity works

### Cluster Lifecycle (most critical)

- [ ] Admin config set (CIDRs, shapes, K8s versions, limits)
- [ ] **DEPLOY**: Cluster deploys with SSE streaming
- [ ] **DEPLOY**: Terraform plan shows ~12-15 resources
- [ ] **DEPLOY**: Terraform apply completes (8-15 min)
- [ ] **DEPLOY**: Dashboard shows Running cluster with cost estimate
- [ ] **DEPLOY**: Kubeconfig downloads and `kubectl get nodes` works
- [ ] **DEPLOY**: SSH key downloads
- [ ] **SCALE**: Node count increase works
- [ ] **SCALE**: Add node pool works
- [ ] **SCALE**: Remove node pool works
- [ ] **DESTROY**: Cluster destroyed successfully
- [ ] **DESTROY**: All OCI resources removed (verified in Console)
- [ ] **DESTROY**: CIDR released and available
- [ ] **CONCURRENT**: Second cluster deploys while first is destroyed
- [ ] **BYON-1**: Existing Compartment only -> deploy/scale/destroy succeeds and creates new network in that compartment
- [ ] **BYON-2**: Existing Compartment + VCN + Subnet -> deploy/scale/destroy succeeds without creating new VCN/subnet stack
- [ ] **BYON-3**: Existing VCN/subnet with missing NAT/SGW routes fails as expected, then passes after route fix

### Templates & RBAC

- [ ] Template created with all fields
- [ ] Template appears on deploy form with cost estimate
- [ ] Deploy from template — form pre-filled and locked
- [ ] Deployed cluster matches template specs
- [ ] Role-based gating: restricted template hidden without role
- [ ] Role-based gating: template visible after role assigned
- [ ] Destroy protection blocks regular user
- [ ] Destroy protection allows admin force-destroy
- [ ] TTL badge visible on dashboard

### Security

- [ ] All pods run as non-root
- [ ] CORS rejects unauthorized origins
- [ ] CORS allows authorized origin
- [ ] Admin endpoints return 403 for non-admin users
- [ ] No secrets in ConfigMaps
- [ ] Unauthenticated requests return 403
- [ ] Invalid tokens return 403

### Resilience

- [ ] API pod recovers after kill, health check passes
- [ ] PostgreSQL recovers after kill, data intact
- [ ] Keycloak recovers after kill, OIDC works
- [ ] Helm upgrade: rolling update completes, data preserved
- [ ] Full pod restart: all pods recover, data intact

---

## 13. Troubleshooting

### Quick diagnostic script

```bash
echo "=== Pods ==="
kubectl get pods -n infragate -o wide

echo -e "\n=== Services ==="
kubectl get svc -n infragate

echo -e "\n=== Ingress ==="
kubectl get ingress -n infragate

echo -e "\n=== PVCs ==="
kubectl get pvc -n infragate

echo -e "\n=== Recent Events ==="
kubectl get events -n infragate --sort-by=.lastTimestamp | tail -20

echo -e "\n=== API Health ==="
kubectl exec -n infragate deploy/infragate-api -- curl -s http://localhost:8000/health 2>/dev/null || echo "API not reachable"

echo -e "\n=== API Logs (last 10 lines) ==="
kubectl logs -n infragate -l app.kubernetes.io/component=api --tail=10 2>/dev/null || echo "No API logs"
```

### Pod stuck in CrashLoopBackOff

```bash
kubectl logs -n infragate <pod-name> --previous
kubectl describe pod -n infragate <pod-name>
```

| Pod | Common cause | Fix |
|---|---|---|
| API | Database not ready, wrong OCI credentials, missing OCI key secret | Wait for PostgreSQL, verify values-local.yaml, check secret |
| Keycloak | DB connection refused, OOM, slow JVM startup | Wait for PostgreSQL, increase memory, check startup probe (210s) |
| Frontend | ConfigMap not created, nginx config error | Check `kubectl get cm -n infragate`, verify ConfigMap exists |
| PostgreSQL | PVC not bound, storage class unavailable | Check `kubectl get pvc -n infragate`, verify storage class |

### Terraform operations fail in the portal

```bash
kubectl logs -n infragate -l app.kubernetes.io/component=api -f --tail=50
```

| Error | Cause | Fix |
|---|---|---|
| `401 NotAuthenticated` | OCI credentials wrong | Verify fingerprint, user OCID, private key match |
| `404 NotAuthorizedOrNotFound` | Missing IAM policies | Add cluster/network/instance-family manage policies |
| `NoSuchBucket` on a bucket that exists | `any-user where request.user.id = ...` pattern silently fails for S3-compat on Identity Domain tenancies | Convert policy to group-based: add `infragate-svc` to `InfragateSvcGroup`, grant `object-family` via the group (see Step 4) |
| `409 Conflict` | CIDR or resource name overlap | Use a different CIDR range |
| `500 LimitExceeded` | OCI service limit hit | Request increase: Governance > Limits |
| State lock error | Previous Terraform crashed | Clear lock in Object Storage |
| `terraform init` fails | No outbound from pod | Verify pod can reach `registry.terraform.io` and OCI APIs |
| `No valid credential sources` | OCI key not mounted | Check `kubectl get secret infragate-oci-key -n infragate` |

### Keycloak won't start

```bash
kubectl logs -n infragate -l app.kubernetes.io/component=keycloak --tail=50

# If OOM killed:
kubectl get pod -n infragate -l app.kubernetes.io/component=keycloak \
  -o jsonpath='{.items[0].status.containerStatuses[0].lastState}'

# Increase memory if needed:
helm upgrade infragate deploy/helm/ -n infragate --reuse-values \
  --set keycloak.resources.limits.memory=2Gi
```

### Images not found (ErrImagePull / ImagePullBackOff)

```bash
kubectl describe pod -n infragate <pod-name> | grep -A5 "Events"

# Verify images are pullable
# Path A (k3s):
sudo k3s crictl pull ghcr.io/solvialab/infragate-api:dev-latest
# Path B (OKE) — pull happens automatically, check node connectivity
```

Common causes:
- GHCR repository is private — make it public or configure `imagePullSecrets`
- CI pipeline hasn't run — check GitHub Actions for latest build
- VM/nodes have no outbound internet — check security list / NSG rules

### Ingress not routing (502, 404, connection refused)

```bash
kubectl get ingress -n infragate -o wide

# Path A (k3s) — bundled traefik in kube-system
kubectl get pods -n kube-system | grep traefik
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=20

# Path B (OKE) — ingress-nginx in its own namespace
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=20

# Both — verify endpoints are populated
kubectl get endpoints -n infragate
# All services should show IP:PORT (not <none>)
```

Common causes:
- `className` mismatch: k3s values must use `traefik`, OKE values must use `nginx`
- **(A)** OS firewall blocking port 80 — check `firewall-cmd --list-all`
- **(A)** `firewalld` blocking inter-pod traffic — traefik logs show `no route to host`. Fix: `sudo firewall-cmd --zone=trusted --add-interface=cni0 --permanent && sudo firewall-cmd --zone=trusted --add-interface=flannel.1 --permanent && sudo firewall-cmd --reload`
- **(B)** OCI LB stuck provisioning — check OCI Console > Networking > Load Balancers
- **(B)** ingress-nginx not installed
- Backend service not ready — endpoints show `<none>`
- DNS not resolving — verify domain points to VM IP (A) or LB IP (B)

### Database connection issues

```bash
kubectl get pods -n infragate -l app.kubernetes.io/component=postgresql
kubectl logs -n infragate -l app.kubernetes.io/component=postgresql --tail=20

# Test from API pod
kubectl exec -n infragate deploy/infragate-api -- python3 -c "
from sqlalchemy import create_engine
import os
engine = create_engine(os.environ.get('DATABASE_URL', ''))
with engine.connect() as conn:
    print('DB connection: OK')
"

# Check DATABASE_URL
kubectl exec -n infragate deploy/infragate-api -- env | grep DATABASE_URL
# Should show: postgresql://infragate:<password>@infragate-postgresql:5432/infragate
```

---

*Last updated: 2026-04-24*
*Maintainer: Solvia Lab s.r.o.*
