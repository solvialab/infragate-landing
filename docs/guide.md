# Infragate — Deployment Guide

This guide covers three deployment paths:

- **[OCI Marketplace (recommended)](#marketplace-deployment)** — one-click "Launch Stack" from the OCI Console. Ideal for OCI customers who want Infragate running in minutes.
- **[Existing OKE cluster](#existing-oke-cluster-deployment)** — deploy Infragate into an existing Oracle Kubernetes Engine cluster using Helm. Best for teams that already manage their own OKE infrastructure.
- **[Single-node k3s](#single-node-k3s-deployment)** — step-by-step guide for deploying on a single OCI VM with k3s. Ideal for dev/test or Always Free tier.

---

## Marketplace Deployment

### What the customer gets

When an OCI customer clicks **Launch Stack** on the Marketplace listing, the OCI Resource Manager presents a guided form and provisions the entire Infragate stack automatically:

| Component | What gets created |
|---|---|
| **OKE Cluster** | New managed Kubernetes cluster (or deploys into an existing one) |
| **Networking** | VCN, subnets (API, node, LB), internet gateway, route tables, security lists (or uses existing) |
| **IAM** | Dynamic group for OKE nodes + IAM policies for image pulling |
| **Ingress** | NGINX ingress controller with OCI flexible load balancer |
| **Infragate** | Full Helm release — API, frontend, PostgreSQL, Keycloak (optional) |
| **Secrets** | OCI API key mounted as Kubernetes secret, auto-generated DB and Keycloak passwords |

### Customer prerequisites

Before clicking "Launch Stack", the customer needs:

| Requirement | Where to create it | Why |
|---|---|---|
| **OCI API Key** | Identity > Users > API Keys > Add API Key | Infragate uses this to provision OKE clusters in the customer's tenancy |
| **API Key Fingerprint** | Shown when the API key is uploaded | Required for OCI API authentication |
| **Object Storage Bucket** | Object Storage > Create Bucket (`infragate-tfstate`) | Stores Terraform state for each provisioned cluster |
| **S3 Compatibility Credentials** | Identity > Users > Customer Secret Keys | Used by Terraform to read/write state via S3-compatible API |
| **Domain Name** | Any DNS provider | A record pointing to the load balancer IP (configured after deploy) |
| **Compartment** | Identity > Compartments | Target compartment for all Infragate resources |

### Customer deployment flow

```
Step 1: Find "Infragate" on OCI Marketplace
         ↓
Step 2: Click "Launch Stack"
         ↓
Step 3: Fill in the guided form:
         - Compartment + Region
         - Domain name (e.g. infragate.company.com)
         - New OKE cluster OR select existing
         - New VCN OR select existing networking
         - OCI API key fingerprint + private key PEM
         - S3 credentials for Terraform state
         - Bundled Keycloak OR external OIDC provider
         ↓
Step 4: Click "Plan" → Review resources → Click "Apply"
         ↓
Step 5: Wait ~15-20 minutes for provisioning
         ↓
Step 6: Note the outputs:
         - Infragate Portal URL
         - Keycloak Admin Console URL
         - Load Balancer IP
         - Post-deployment instructions
         ↓
Step 7: Create DNS A record:
         infragate.company.com → <Load Balancer IP>
         ↓
Step 8: Configure Keycloak (if using bundled):
         - Log into Keycloak admin console
         - Create realm "infragate"
         - Create client "infragate-portal" (public, PKCE)
         - Create users and assign admin role
         ↓
Step 9: Access the portal and start provisioning clusters
```

### Post-deployment: What customers need to configure

The Resource Manager Stack handles all infrastructure. Customers only need to:

1. **DNS** — Point their domain at the load balancer IP (shown in stack outputs)
2. **Keycloak realm** — Create the `infragate` realm and `infragate-portal` client (if using bundled Keycloak)
3. **Users** — Create users in Keycloak and assign the `admin` realm role to platform administrators
4. **Platform config** — Log into Infragate admin panel and configure:
   - CIDR pool (available /24 ranges for cluster allocation)
   - Allowed VM shapes
   - Allowed Kubernetes versions
   - Node images (or leave empty for auto-select)
   - Resource limits per user

### Upgrading

Customers upgrade by re-running the Resource Manager Stack with updated container image tags:

1. OCI Console > Resource Manager > Stacks > Select Infragate stack
2. Edit variables > Update `api_image_tag` and `frontend_image_tag` to new version
3. Click "Apply" — Helm performs a rolling update

### Marketplace customer support flow

| Issue | Resolution |
|---|---|
| Stack Plan fails | Check IAM policies — service account needs `manage cluster-family`, `manage virtual-network-family` in the target compartment |
| Pods not starting | Check image pull — ensure images are accessible from OKE nodes. For private registries, set `global.imagePullSecrets` |
| Keycloak slow to start | Normal — first boot takes 60-90s. Startup probe allows up to 210s |
| OIDC errors | Verify issuer URL matches the JWT `iss` claim exactly. For bundled Keycloak: `https://<domain>/auth/realms/infragate` |
| Terraform operations fail | Check OCI credentials — fingerprint, private key, user OCID must all match the same API key |

---

## Existing OKE Cluster Deployment

Deploy Infragate into an existing OKE cluster. This path is for teams that already have a Kubernetes cluster and want Infragate running in a dedicated namespace.

### Prerequisites

| Requirement | Details |
|---|---|
| **OKE cluster** | Running cluster with `kubectl` access |
| **Helm 3.x** | Installed on your workstation |
| **ingress-nginx** | Installed in the cluster (see below) |
| **OCI API Key** | For Infragate to provision managed clusters |
| **Object Storage bucket** | `infragate-tfstate` for Terraform state |
| **S3 compatibility credentials** | Customer Secret Keys for Terraform state |
| **Domain name** | DNS A record pointing to the OCI load balancer IP |

### Step 1 — Install ingress-nginx (if not present)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape"=flexible \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape-flex-min"=10 \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape-flex-max"=100
```

### Step 2 — Create namespace and secrets

```bash
kubectl create namespace infragate

kubectl create secret generic infragate-oci-key \
  --from-file=oci_api_key.pem=/path/to/your/oci_api_key.pem \
  -n infragate
```

### Step 3 — Deploy with Helm

```bash
helm upgrade --install infragate deploy/helm/ \
  -n infragate \
  -f deploy/helm/values-oke.yaml \
  --set global.domain=infragate.example.com \
  --set postgresql.auth.password=YOUR_DB_PASSWORD \
  --set postgresql.auth.postgresPassword=YOUR_DB_PASSWORD \
  --set keycloak.admin.password=YOUR_KC_PASSWORD \
  --set api.oci.tenancyOcid=ocid1.tenancy.oc1..xxx \
  --set api.oci.userOcid=ocid1.user.oc1..xxx \
  --set api.oci.fingerprint=xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx \
  --set api.oci.namespace=YOUR_OBJ_STORAGE_NAMESPACE \
  --set api.oci.parentCompartmentOcid=ocid1.compartment.oc1..xxx \
  --set api.terraform.s3AccessKey=YOUR_S3_KEY \
  --set api.terraform.s3SecretKey=YOUR_S3_SECRET \
  --wait --timeout 10m
```

The `values-oke.yaml` file is pre-configured for OKE with:
- GHCR image references (`ghcr.io/solvialab/infragate-api`, `ghcr.io/solvialab/infragate-frontend`)
- OCI Block Volume storage class (`oci-bv`) for PostgreSQL persistence
- nginx ingress class with SSE proxy timeouts
- No control-plane tolerations (OKE worker nodes accept all pods)

### Step 4 — Verify pods are running

```bash
kubectl get pods -n infragate
# All pods should show Running and 1/1 Ready
```

### Step 5 — Configure DNS

Get the load balancer IP:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Create a DNS A record pointing your domain to this IP.

### Step 6 — Configure Keycloak and Infragate

Follow the same Keycloak setup and Infragate configuration steps as the k3s deployment (Sections 9-11 below).

### Upgrading on OKE

```bash
# Pull latest chart changes
git pull

# Upgrade — Helm performs a rolling update
helm upgrade infragate deploy/helm/ -n infragate \
  -f deploy/helm/values-oke.yaml \
  --reuse-values \
  --wait --timeout 5m
```

To update to a specific image version:

```bash
helm upgrade infragate deploy/helm/ -n infragate \
  -f deploy/helm/values-oke.yaml \
  --reuse-values \
  --set api.image.tag=<NEW_TAG> \
  --set frontend.image.tag=<NEW_TAG> \
  --wait --timeout 5m
```

---

## Single-node k3s Deployment

Deploy Infragate on a single OCI VM running k3s at `dev.infragate.cloud`. This guide goes from bare VM to a working platform.

---

## Setup checklist

Use this to track your progress. Check off each step as you complete it.

- [ ] **1. Prerequisites** — OCI account, domain, SSH key, repo cloned
- [ ] **2. Provision the OCI VM** — VM created, VCN ports opened (80+443 ingress, all-protocol egress), DNS configured
- [ ] **3. Install k3s** — k3s running, kubectl working, nginx ingress with hostNetwork, CoreDNS fixed
- [ ] **4. Install tooling** — Helm installed (Docker optional, only for local builds)
- [ ] **5. OCI service account** — IAM user, policies, API key, state bucket, S3 keys
- [ ] **6. Container images** — CI pushes to GHCR automatically, verify images are available
- [ ] **7. Create secrets** — OCI key secret created in `infragate` namespace
- [ ] **8. Deploy with Helm** — All pods Running (postgresql, keycloak if deployed, api, frontend)
- [ ] **9. Configure Keycloak** — **Option A** (deploy your own) OR **Option B** (bring your own)
- [ ] **10. Enable HTTPS** — cert-manager installed, TLS certificate Ready
- [ ] **11. Configure Infragate** — CIDRs, shapes, node images, limits, K8s versions, cluster templates set via Admin UI
- [ ] **12. Verify** — Landing page loads, sign-in works, admin panel accessible

---

## Table of contents

1. [Prerequisites](#1-prerequisites)
2. [Provision the OCI VM](#2-provision-the-oci-vm)
3. [Install k3s](#3-install-k3s)
4. [Install tooling](#4-install-tooling)
5. [OCI service account setup](#5-oci-service-account-setup)
6. [Container images](#6-container-images)
7. [Create secrets](#7-create-secrets)
8. [Deploy with Helm](#8-deploy-with-helm)
9. [Configure Keycloak](#9-configure-keycloak) — Option A (deploy your own) / Option B (bring your own)
10. [Enable HTTPS](#10-enable-https)
11. [Configure Infragate](#11-configure-infragate)
12. [Verify](#12-verify)
13. [Maintenance](#13-maintenance)

---

## 1. Prerequisites

| Item | Details |
|---|---|
| OCI account | Active tenancy with OKE enabled |
| Domain | `dev.infragate.cloud` — DNS A record pointed at the VM's public IP |
| SSH key pair | For VM access |
| Infragate repo | Cloned locally or on the VM |

---

## 2. Provision the OCI VM

Create a compute instance in OCI. The Always Free Ampere A1 shape works well.

**Recommended spec:**

| Setting | Value |
|---|---|
| Shape | VM.Standard.A1.Flex (ARM64) |
| OCPUs | 2–4 |
| Memory | 12–24 GB |
| OS | Oracle Linux 8 or Ubuntu 22.04 |
| Boot volume | 100 GB |
| VCN | New or existing |

**Open these ports in the VCN security list:**

Ingress rules (stateful) — Source Port Range: All, Destination Port as listed:

| Direction | Destination Port | Protocol | Source CIDR | Purpose |
|---|---|---|---|---|
| Ingress | 22 | TCP | `<YOUR_PUBLIC_IP>/32` | SSH access |
| Ingress | 80 | TCP | 0.0.0.0/0 | HTTP (redirects to HTTPS) |
| Ingress | 443 | TCP | 0.0.0.0/0 | HTTPS — all web traffic |
| Ingress | 6443 | TCP | `<YOUR_PUBLIC_IP>/32` | Kubernetes API (optional, for remote kubectl) |

> `<YOUR_PUBLIC_IP>` = the public IP of your workstation or office network (the machine you SSH from). This restricts SSH and K8s API access to your location only. Find it with `curl ifconfig.me`. Use `0.0.0.0/0` instead if you need access from anywhere (less secure).

Egress rules (stateful) — Source Port Range: All, Destination Port: All:

| Direction | Destination Port | Protocol | Destination CIDR | Purpose |
|---|---|---|---|---|
| Egress | All | All | 0.0.0.0/0 | Allow all outbound (Terraform needs to reach OCI APIs, Helm repos, container registries) |

> OCI security lists are **stateful** by default — response traffic for allowed ingress rules is automatically permitted without an explicit egress rule. The "allow all outbound" egress rule is needed for the VM to initiate connections (apt/dnf updates, Terraform API calls, image pulls).
>
> **Important:** The egress rule must use **All Protocols**, not just TCP. Kubernetes pods need UDP/53 for DNS resolution. If you set egress to TCP only, pod DNS will fail and cert-manager, image pulls, and Terraform will break.

**Open the same ports in the OS firewall:**

Oracle Linux:

```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=6443/tcp

# Allow k3s pod and service network traffic through the firewall
# Without this, pods cannot resolve DNS or reach external services
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
sudo firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16

sudo firewall-cmd --reload
```

Ubuntu:

```bash
sudo iptables -I INPUT 1 -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 1 -p tcp --dport 443 -j ACCEPT
sudo iptables -I INPUT 1 -p tcp --dport 6443 -j ACCEPT

# Allow k3s pod and service network traffic
sudo iptables -I INPUT 1 -s 10.42.0.0/16 -j ACCEPT
sudo iptables -I INPUT 1 -s 10.43.0.0/16 -j ACCEPT
sudo iptables -I FORWARD 1 -s 10.42.0.0/16 -j ACCEPT
sudo iptables -I FORWARD 1 -s 10.43.0.0/16 -j ACCEPT

sudo netfilter-persistent save
```

**Point your domain** at the VM's public IP:

```
dev.infragate.cloud → <VM_PUBLIC_IP>  (A record)
```

---

## 3. Install k3s

> **Checklist:**
> - [ ] SELinux packages installed (Oracle Linux only)
> - [ ] k3s installed with Traefik disabled, service running
> - [ ] `kubectl get nodes` shows Ready
> - [ ] kubectl config copied to `~/.kube/config`, `KUBECONFIG` exported
> - [ ] nginx ingress controller installed with `hostNetwork: true`
> - [ ] `sudo ss -tlnp | grep ':80'` shows nginx listening
> - [ ] CoreDNS configured with `forward . 8.8.8.8 8.8.4.4` and domain hairpin
> - [ ] Pod DNS resolution verified

SSH into the VM:

```bash
ssh opc@<VM_PUBLIC_IP>
```

### 3.1 Prerequisites (Oracle Linux only)

Oracle Linux requires SELinux support packages before k3s can install. Skip this on Ubuntu.

```bash
sudo dnf install -y container-selinux selinux-policy-base
sudo rpm -i https://rpm.rancher.io/k3s/stable/common/centos/8/noarch/k3s-selinux-1.6-1.el8.noarch.rpm
```

### 3.2 Install k3s without Traefik

Traefik is disabled in favour of nginx-ingress, which has better SSE support for the Terraform log streaming.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
```

Verify the install succeeded and the service is running:

```bash
sudo systemctl status k3s
```

You should see `Active: active (running)`. If it failed, check the logs:

```bash
sudo journalctl -xeu k3s
```

### 3.3 Verify

```bash
sudo /usr/local/bin/k3s kubectl get nodes
```

One node with status `Ready`.

> On Oracle Linux, `sudo` uses a restricted `secure_path` that does not include `/usr/local/bin`. If `sudo k3s` gives "command not found" even though k3s is running, use the full path `/usr/local/bin/k3s` or add it to the secure path:
> ```bash
> sudo visudo
> # Find the line: Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin
> # Change to:     Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin
> ```

### 3.4 Set up kubectl for your user

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config
```

Make it permanent so every new shell uses it:

```bash
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
```

Verify:

```bash
kubectl get nodes
kubectl get pods -A
```

### 3.5 Install nginx ingress controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml
```

Wait for it to be ready:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Patch the controller to use host networking so it binds directly to the host's ports 80/443 (single-node, no cloud LB). The `hostPort` approach does not work reliably with flannel CNI — `hostNetwork` is required:

```bash
kubectl patch deployment ingress-nginx-controller \
  -n ingress-nginx \
  --type=json \
  -p='[{"op": "add", "path": "/spec/template/spec/hostNetwork", "value": true}]'
```

Wait for the pod to restart and verify it's listening:

```bash
kubectl rollout status deployment ingress-nginx-controller -n ingress-nginx
sudo ss -tlnp | grep ':80'
```

You should see nginx listening on `0.0.0.0:80` and `0.0.0.0:443`.

If the ingress-nginx service stays `<pending>` (no cloud LB), switch to NodePort:

```bash
kubectl patch svc ingress-nginx-controller \
  -n ingress-nginx \
  -p '{"spec": {"type": "NodePort"}}'
```

### 3.6 Fix CoreDNS for external DNS resolution

By default k3s CoreDNS may not be able to resolve external domains from pods (needed by cert-manager, Terraform, image pulls). Configure it to use Google DNS and add a hairpin entry so pods can reach the ingress by domain name:

```bash
# Get the node internal IP
NODE_IP=$(kubectl get node -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "Node IP: $NODE_IP"

# Update CoreDNS: use Google DNS for forwarding + add domain hairpin
kubectl patch configmap coredns -n kube-system --type=merge \
  -p "{\"data\":{\"NodeHosts\":\"$NODE_IP $(hostname)\n$NODE_IP dev.infragate.cloud\"}}"
```

Now edit the Corefile to set the DNS forwarder:

```bash
kubectl edit configmap coredns -n kube-system
```

Find the `forward` line and make sure it reads:

```
forward . 8.8.8.8 8.8.4.4
```

Restart CoreDNS:

```bash
kubectl rollout restart deploy/coredns -n kube-system
```

Verify pods can resolve external DNS:

```bash
kubectl run dns-test --rm -it --image=busybox --restart=Never -- nslookup acme-v02.api.letsencrypt.org 8.8.8.8
```

> If DNS resolution fails (SERVFAIL / "no route to host"), check:
> 1. **VCN egress rule** must use **All Protocols** (not just TCP) — pods need UDP/53 for DNS
> 2. **OS firewall** must trust the pod/service CIDRs (see Section 2)

---

## 4. Install tooling

### 4.1 Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 4.2 Docker

Needed to build images on the VM. Skip if building elsewhere.

Oracle Linux 8:

```bash
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

---

## 5. OCI service account setup

> **Checklist:**
> - [ ] IAM user `infragate-svc` created
> - [ ] IAM group `infragate-svc-group` created with policies
> - [ ] API key generated and `.pem` downloaded
> - [ ] Tenancy OCID, user OCID, fingerprint, region noted
> - [ ] Terraform state bucket `infragate-tfstate` created (versioned)
> - [ ] S3-compatible Customer Secret Key generated (access key + secret key)
> - [ ] Parent compartment OCID noted

Infragate provisions OKE clusters via Terraform using an OCI IAM service account.

### 5.1 Create IAM user and group

In the OCI Console:

1. **Identity & Security > Users > Create User** — name it `infragate-svc`
2. **Identity & Security > Groups > Create Group** — name it `infragate-svc-group`
3. Add `infragate-svc` to `infragate-svc-group`

### 5.2 Create IAM policies

Create a policy in the compartment where clusters will be provisioned:

```
Allow group infragate-svc-group to manage cluster-family in compartment <your-compartment>
Allow group infragate-svc-group to manage virtual-network-family in compartment <your-compartment>
Allow group infragate-svc-group to manage instance-family in compartment <your-compartment>
Allow group infragate-svc-group to manage compartments in compartment <your-compartment>
Allow group infragate-svc-group to read tenancies in tenancy
Allow group infragate-svc-group to read objectstorage-namespaces in tenancy
```

### 5.3 Generate API key

1. Go to the `infragate-svc` user > **API Keys > Add API Key > Generate API Key Pair**
2. Download the private key (`.pem` file)
3. Note from the **Configuration File Preview**:
   - `tenancy` — tenancy OCID
   - `user` — user OCID
   - `fingerprint` — key fingerprint
   - `region` — e.g. `eu-frankfurt-1`

Copy the key to the VM:

```bash
scp oci_api_key.pem opc@<VM_PUBLIC_IP>:~/.oci/oci_api_key.pem
```

Create the directory on the VM first if it doesn't exist:

```bash
ssh opc@<VM_PUBLIC_IP> "mkdir -p ~/.oci && chmod 700 ~/.oci"
```

### 5.4 Create Terraform state bucket

1. **Storage > Object Storage > Create Bucket**
   - Name: `infragate-tfstate`
   - Enable versioning
2. Note the **Object Storage Namespace** (under **Tenancy Details**)

### 5.5 Generate S3-compatible Customer Secret Key

1. Go to `infragate-svc` user > **Customer Secret Keys > Generate Secret Key**
2. Save both the **Access Key** and the **Secret Key** — the secret is shown only once

### 5.6 Note the parent compartment OCID

This is the compartment where Infragate will create child compartments for each cluster.

---

## 6. Container images

### How images are built and published

Infragate ships with CI/CD pipelines for both GitHub and GitLab. Both use the same tagging scheme:

| Branch | Tags | GitHub example | GitLab example |
|---|---|---|---|
| `DEV` | `dev-latest`, `dev-<sha>` | `ghcr.io/solvialab/infragate-api:dev-latest` | `registry.gitlab.com/your-group/infragate/api:dev-latest` |
| `main` | `latest`, `<sha>` | `ghcr.io/solvialab/infragate-api:latest` | `registry.gitlab.com/your-group/infragate/api:latest` |

| Platform | Pipeline file | Registry |
|---|---|---|
| GitHub | `.github/workflows/ci.yml` | GHCR (`ghcr.io`) |
| GitLab | `.gitlab-ci.yml` | GitLab Container Registry (`registry.gitlab.com`) |

**No local builds or image imports needed.** Your cluster pulls images directly from the registry.

### Verify images are available

```bash
# GitHub (GHCR)
docker pull ghcr.io/solvialab/infragate-api:dev-latest
docker pull ghcr.io/solvialab/infragate-frontend:dev-latest

# GitLab
docker pull registry.gitlab.com/your-group/infragate/api:dev-latest

# Or verify from k3s on the VM
sudo k3s crictl pull ghcr.io/solvialab/infragate-api:dev-latest
```

### Private registry authentication

Private registries (GitLab CR, OCIR, private GHCR) require an image pull secret:

```bash
kubectl create namespace infragate

# GitHub (private repo)
kubectl create secret docker-registry regcred \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USER \
  --docker-password=YOUR_GITHUB_PAT \
  -n infragate

# GitLab (deploy token recommended)
kubectl create secret docker-registry regcred \
  --docker-server=registry.gitlab.com \
  --docker-username=deploy-token-username \
  --docker-password=YOUR_DEPLOY_TOKEN \
  -n infragate

# OCI Container Registry (OCIR)
kubectl create secret docker-registry regcred \
  --docker-server=<region>.ocir.io \
  --docker-username='<namespace>/oracleidentitycloudservice/<email>' \
  --docker-password=YOUR_AUTH_TOKEN \
  -n infragate
```

Then pass the secret to Helm:

```bash
helm upgrade --install infragate deploy/helm/ -n infragate \
  -f deploy/helm/values-dev.yaml \
  --set 'global.imagePullSecrets[0].name=regcred' \
  ...
```

Public GHCR packages don't need this step.

### Fallback: Build locally on VM

If you need to test images that haven't been pushed yet (e.g. local branch):

```bash
cd ~/infragate
docker build -t infragate-api:local -f Dockerfile .
docker build -t infragate-frontend:local -f Dockerfile.frontend .

docker save infragate-api:local -o /tmp/api.tar
sudo /usr/local/bin/k3s ctr images import /tmp/api.tar && rm /tmp/api.tar

docker save infragate-frontend:local -o /tmp/frontend.tar
sudo /usr/local/bin/k3s ctr images import /tmp/frontend.tar && rm /tmp/frontend.tar
```

Use `pullPolicy: Never` and `tag: local` in your values file for local images.

---

## 7. Create secrets

Make sure the OCI API key from Section 5.3 is on the VM. If you haven't copied it yet:

```bash
# Run from your local machine (not the VM)
ssh opc@<VM_PUBLIC_IP> "mkdir -p ~/.oci && chmod 700 ~/.oci"
scp /path/to/oci_api_key.pem opc@<VM_PUBLIC_IP>:~/.oci/oci_api_key.pem
```

Verify the key is there, then create the secret. The namespace will be created by Helm in the next step — we use `--create-namespace` here just in case you run this before Helm:

```bash
ls -la ~/.oci/oci_api_key.pem

kubectl create namespace infragate || true

kubectl create secret generic infragate-oci-key \
  --from-file=oci_api_key.pem=$HOME/.oci/oci_api_key.pem \
  -n infragate
```

Label the namespace so Helm can manage it:

```bash
kubectl label namespace infragate app.kubernetes.io/managed-by=Helm
kubectl annotate namespace infragate meta.helm.sh/release-name=infragate meta.helm.sh/release-namespace=infragate
```

---

## 8. Deploy with Helm

> **Checklist:**
> - [ ] `values-dev.yaml` created with your domain and settings
> - [ ] `helm upgrade --install` completed without errors
> - [ ] All 4 pods Running: postgresql, keycloak, api, frontend
> - [ ] `curl http://dev.infragate.cloud/health` returns `{"status":"ok"}`

### 8.1 Create your values file

The Helm chart ships with three values files (see [STACK.md — Helm configuration](./STACK.md#8-helm-configuration) for details):

| File | Purpose |
|---|---|
| `values.yaml` | Chart defaults — all fields documented, never edit directly |
| `values-oci.yaml` | OCI deployment template — copy this to get started |
| `values-dev.yaml` | Your environment config — domain, TLS, and deployment-specific settings |

Create your values file by copying the OCI template:

```bash
cp deploy/helm/values-oci.yaml deploy/helm/values-dev.yaml
```

Edit `deploy/helm/values-dev.yaml`:

```yaml
global:
  domain: dev.infragate.cloud
  # imagePullSecrets: [{ name: regcred }]  # uncomment for private registries

api:
  image:
    # Change repository to match your CI platform:
    #   GitHub:  ghcr.io/solvialab/infragate-api
    #   GitLab:  registry.gitlab.com/your-group/infragate/api
    repository: ghcr.io/solvialab/infragate-api
    tag: dev-latest                # or pin to a commit: dev-0e45d7a
    pullPolicy: Always             # Always for dev-latest, IfNotPresent for pinned tags
  env:
    APP_ENV: production
    CORS_ORIGINS: "https://dev.infragate.cloud"
  oidc:
    issuer: https://dev.infragate.cloud/auth/realms/infragate
    clientId: infragate-portal
  oci:
    region: eu-frankfurt-1

frontend:
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
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
  tls:
    enabled: false                 # enabled later with cert-manager

tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
  - key: node-role.kubernetes.io/master
    operator: Exists
    effect: NoSchedule
```

### 8.2 Install

Replace the placeholder values below. All `api.oci.*` credentials come from the **infragate-svc** service account created in Section 5:

| Placeholder | Where to find it |
|---|---|
| `<DB_PASSWORD>` | Choose a password for PostgreSQL |
| `<KC_ADMIN_PASSWORD>` | Choose a password for Keycloak admin console |
| `tenancyOcid` | Section 5.3 — Configuration File Preview → `tenancy` |
| `userOcid` | Section 5.3 — Configuration File Preview → `user` (the **infragate-svc** user OCID, not your personal account) |
| `fingerprint` | Section 5.3 — Configuration File Preview → `fingerprint` (of the **infragate-svc** API key) |
| `namespace` | Section 5.4 — Object Storage Namespace (under Tenancy Details) |
| `parentCompartmentOcid` | Section 5.6 — Parent compartment OCID |
| `s3AccessKey` | Section 5.5 — Customer Secret Key access key |
| `s3SecretKey` | Section 5.5 — Customer Secret Key secret key |

```bash
helm upgrade --install infragate ./deploy/helm \
  -n infragate \
  -f deploy/helm/values-dev.yaml \
  --set postgresql.auth.password=<DB_PASSWORD> \
  --set keycloak.admin.password=<KC_ADMIN_PASSWORD> \
  --set api.oci.tenancyOcid=ocid1.tenancy.oc1..aaa... \
  --set api.oci.userOcid=ocid1.user.oc1..aaa... \
  --set api.oci.fingerprint=aa:bb:cc:dd:ee:ff:00:11:... \
  --set api.oci.namespace=<OCI_NAMESPACE> \
  --set api.oci.parentCompartmentOcid=ocid1.compartment.oc1..aaa... \
  --set api.terraform.stateBucket=infragate-tfstate \
  --set api.terraform.s3AccessKey=<S3_ACCESS_KEY> \
  --set api.terraform.s3SecretKey=<S3_SECRET_KEY> \
  --wait --timeout 10m
```

### 8.3 Check status

```bash
kubectl get pods -n infragate
```

Expected output:

```
NAME                                    READY   STATUS    RESTARTS   AGE
infragate-postgresql-0                  1/1     Running   0          2m
infragate-keycloak-xxxxxxxxxx-xxxxx     1/1     Running   0          2m
infragate-api-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
infragate-frontend-xxxxxxxxxx-xxxxx     1/1     Running   0          2m
```

If a pod is stuck:

```bash
kubectl logs -n infragate deploy/infragate-api
kubectl logs -n infragate deploy/infragate-keycloak
kubectl logs -n infragate infragate-postgresql-0
kubectl describe pod -n infragate <pod-name>
```

---

## 9. Configure Keycloak

Choose **one** of the two options below:

- **Option A** — Deploy your own Keycloak (included in Helm chart, good for dev/testing)
- **Option B** — Bring your own Keycloak (enterprise environments with an existing IdP)

---

### Option A: Deploy your own Keycloak

> **Checklist:**
> - [ ] Keycloak admin console accessible via port-forward
> - [ ] `infragate` realm created
> - [ ] `infragate-portal` client created with correct redirect URIs
> - [ ] `admin` realm role created
> - [ ] Admin user created with password and `admin` role assigned
> - [ ] (Optional) Regular test user created
> - [ ] Keycloak exposed through the ingress at `/auth`

#### 9A.1 Access Keycloak admin console

Port-forward to access it:

```bash
kubectl port-forward svc/infragate-keycloak 8080:8080 -n infragate
```

If working from a remote machine, SSH tunnel first:

```bash
ssh -L 8080:localhost:8080 opc@<VM_PUBLIC_IP>
```

Open `http://localhost:8080` and log in with the admin credentials from Section 8.2.

#### 9A.2 Create realm

1. Realm dropdown (top-left, says "master") > **Create Realm**
2. Realm name: `infragate`
3. **Create**

#### 9A.3 Create client

1. **Clients > Create client**
2. Client ID: `infragate-portal`, Client type: **OpenID Connect**, Client authentication: **Off**
3. **Next** > Standard flow: **On**, Direct access grants: **On** > **Next**
4. Access settings:
   - Root URL: `https://dev.infragate.cloud`
   - Home URL: `https://dev.infragate.cloud`
   - Valid redirect URIs: `https://dev.infragate.cloud/*`
   - Valid post logout redirect URIs: `https://dev.infragate.cloud/*`
   - Web origins: `https://dev.infragate.cloud`
5. **Save**

#### 9A.4 Create admin role

1. **Realm roles > Create role**
2. Role name: `admin`
3. **Save**

#### 9A.5 Create your first user (admin)

1. **Users > Create user**
2. Fill in username, email, first name, last name
3. **Create**
4. **Credentials** tab > **Set password** > set a password, toggle off "Temporary"
5. **Role mapping** tab > **Assign role** > filter by realm roles > select `admin`

This user will see admin navigation links in the Infragate UI after login.

#### 9A.6 Create regular users

Same as above, but skip step 5 (no admin role). These users can deploy clusters within their limits but cannot access the admin panel.

#### 9A.7 Expose Keycloak through the ingress

The OIDC issuer URL must be reachable from both the browser and the API pod. The Helm chart already includes a `/auth/` location block in the nginx ConfigMap (`deploy/helm/templates/configmap-nginx.yaml`) that proxies to the in-cluster Keycloak.

Verify it looks like this (no `/auth/` suffix on `proxy_pass` — Keycloak 24+ serves at the root):

```nginx
# Keycloak — reverse proxy for OIDC (login, token exchange, discovery)
location /auth/ {
    proxy_pass http://infragate-keycloak:8080/;
    ...
}
```

> `location /auth/` strips the `/auth` prefix before forwarding. A request to `/auth/realms/infragate` becomes `/realms/infragate` on Keycloak, which is correct for Keycloak 24+. Do **not** use `proxy_pass .../auth/;` — that would produce `/auth/realms/infragate` on Keycloak, which no longer exists.

This makes the issuer URL `https://dev.infragate.cloud/auth/realms/infragate`, which matches the `api.oidc.issuer` in the values file. No changes needed if you deployed using the Helm chart as-is.

---

### Option B: Bring your own Keycloak

> **Checklist:**
> - [ ] Existing Keycloak instance reachable from both the browser and the k3s cluster
> - [ ] `infragate` realm created (or existing realm chosen)
> - [ ] `infragate-portal` public OIDC client created with PKCE
> - [ ] `admin` realm role created and assigned to at least one user
> - [ ] Helm values updated to disable in-cluster Keycloak and point at external issuer
> - [ ] JWKS endpoint reachable from the API pod (tested with curl)

Use this option if your organization already runs a Keycloak instance (or any OIDC-compatible IdP that supports PKCE). Infragate only needs:

1. A **public OIDC client** (no client secret — the frontend uses Authorization Code + PKCE)
2. A **realm role** named `admin` to distinguish platform admins from regular users
3. The JWT `sub` claim as the stable user identifier
4. Realm roles exposed in the token under `realm_access.roles`

#### 9B.1 Disable in-cluster Keycloak

In your `values-dev.yaml`, disable the bundled Keycloak:

```yaml
keycloak:
  enabled: false
```

#### 9B.2 Create the client in your existing Keycloak

Log into your Keycloak admin console and navigate to the realm you want to use (or create a new `infragate` realm).

1. **Clients > Create client**
2. Client ID: `infragate-portal`
3. Client type: **OpenID Connect**
4. Client authentication: **Off** (public client — PKCE is used instead of a client secret)
5. **Next** > Standard flow: **On**, Direct access grants: **On** > **Next**
6. Access settings:
   - Root URL: `https://dev.infragate.cloud`
   - Home URL: `https://dev.infragate.cloud`
   - Valid redirect URIs: `https://dev.infragate.cloud/*`
   - Valid post logout redirect URIs: `https://dev.infragate.cloud/*`
   - Web origins: `https://dev.infragate.cloud`
7. **Save**

#### 9B.3 Create the admin role

1. **Realm roles > Create role**
2. Role name: `admin`
3. **Save**
4. Assign this role to users who should have platform admin access

Infragate checks for `admin` in the `realm_access.roles` array of the JWT. Users without this role are regular users — they can deploy clusters but cannot access the admin panel.

#### 9B.4 Verify token structure

Infragate expects these claims in the JWT access token:

| Claim | Used for |
|---|---|
| `sub` | Unique user identifier (Keycloak UUID) — stored as `keycloak_id` in DB |
| `preferred_username` | Display name in the UI |
| `email` | Stored on first login |
| `realm_access.roles` | Array of strings — checked for `"admin"` role |

Most Keycloak installations include these by default. If you use custom mappers or a different IdP, verify the token contains these claims.

#### 9B.5 Configure Helm values

Update `values-dev.yaml` to point at your external Keycloak:

```yaml
keycloak:
  enabled: false

api:
  oidc:
    # The public issuer URL — must match the `iss` claim in the JWT
    # and be reachable from the user's browser (for OIDC discovery + login redirect)
    issuer: https://keycloak.yourcompany.com/realms/infragate
    # Internal URL — used by the API pod to fetch JWKS keys
    # Set this if the public issuer URL is not reachable from inside the cluster
    # (e.g., split-horizon DNS, or Keycloak is behind a different ingress)
    # Leave empty if the public URL works from inside the cluster too
    internalUrl: ""
    clientId: infragate-portal
```

If your Keycloak is on a different domain than `dev.infragate.cloud`, the API pod needs to reach the JWKS endpoint directly. Two common scenarios:

**Keycloak reachable from the cluster at the public URL** (most common):

```yaml
api:
  oidc:
    issuer: https://keycloak.yourcompany.com/realms/infragate
    internalUrl: ""   # API will use the issuer URL directly
```

**Keycloak not reachable at the public URL from inside the cluster** (split DNS, internal network):

```yaml
api:
  oidc:
    issuer: https://keycloak.yourcompany.com/realms/infragate
    internalUrl: http://keycloak.internal.yourcompany.com:8080/realms/infragate
```

> The `issuer` must match the `iss` claim in the JWT exactly. The `internalUrl` is only used by the backend to fetch JWKS signing keys — it does not appear in tokens.

#### 9B.6 Remove the /auth proxy block

Since Keycloak is external, you do not need the `/auth/` proxy in the nginx ConfigMap. If you previously had it, remove the `location /auth/ { ... }` block from `deploy/helm/templates/configmap-nginx.yaml`.

#### 9B.7 Deploy and verify connectivity

```bash
helm upgrade infragate ./deploy/helm \
  -n infragate \
  -f deploy/helm/values-dev.yaml \
  --reuse-values
```

Verify the API pod can reach the JWKS endpoint:

```bash
# Get the OIDC discovery URL the API will use
kubectl exec -n infragate deploy/infragate-api -- \
  curl -sf "$KEYCLOAK_INTERNAL_URL/protocol/openid-connect/certs" | head -c 200
```

If this returns a JSON object with `"keys": [...]`, the connection is working. If it fails, check:

- Network policies or firewalls between the cluster and your Keycloak
- DNS resolution from inside the pod (`kubectl exec ... -- nslookup keycloak.yourcompany.com`)
- The `internalUrl` value if you set one

#### 9B.8 Create users

Users are auto-provisioned in the Infragate database on their first login (`GET /api/v1/users/me`). You do not need to create them manually in Infragate — just ensure they exist in your Keycloak realm and assign the `admin` role to platform administrators.

---

## 10. Enable HTTPS

### 10.1 Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.1/cert-manager.yaml

kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=120s
```

### 10.2 Create ClusterIssuer

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@infragate.cloud
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

### 10.3 Enable TLS in Helm values

Update `deploy/helm/values-dev.yaml`:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
  tls:
    enabled: true
    secretName: infragate-tls
```

Upgrade:

```bash
helm upgrade infragate ./deploy/helm \
  -n infragate \
  -f deploy/helm/values-dev.yaml \
  --reuse-values
```

Check certificate status:

```bash
kubectl get certificate -n infragate
kubectl describe certificate infragate-tls -n infragate
```

The certificate should become `Ready` within a couple of minutes.

**Troubleshooting certificate issues:**

If the certificate stays `False`, inspect the chain:

```bash
kubectl get certificate,certificaterequest,order,challenge -n infragate
kubectl describe challenge -n infragate
```

Common issues:

| Symptom | Cause | Fix |
|---|---|---|
| Challenge says "Timeout during connect" | Port 80 not reachable from internet | Check VCN ingress rule for port 80/TCP from 0.0.0.0/0 exists |
| Challenge says "no route to host" on pod IP | cert-manager self-check can't reach ingress | Verify CoreDNS hairpin entry from Section 3.6 points `dev.infragate.cloud` at the node IP |
| Order not created, CertificateRequest stuck | cert-manager can't reach Let's Encrypt (DNS failure) | Verify pod DNS works (Section 3.6), check VCN egress allows All Protocols |

To retry after fixing, delete the failed resources and cert-manager will recreate them:

```bash
kubectl delete challenge,order,certificaterequest -n infragate --all
```

---

## 11. Configure Infragate

> **Checklist:**
> - [ ] Logged in as admin — admin navigation links visible
> - [ ] CIDR pool configured (at least one /24 range)
> - [ ] OCI Settings filled in (compartment, region, cluster tier, bucket, namespace)
> - [ ] VM shapes added (at least one)
> - [ ] *(Optional)* Node images configured (leave empty for auto-select)
> - [ ] Resource limits set and saved
> - [ ] K8s versions added
> - [ ] *(Optional)* Cluster templates created

All platform configuration is done through the **Admin UI** — no curl commands needed.

### 11.1 Log in as admin

1. Open `https://dev.infragate.cloud`
2. Click **Log In** — you are redirected to Keycloak
3. Log in with the admin user created in Section 9.5

Because this user has the `admin` realm role in Keycloak, the navigation bar shows admin links (All clusters, Users & limits, Configuration, Audit log) instead of the regular user navigation.

### 11.2 Navigate the Admin panel

The admin navigation links are shown directly in the top navigation bar:

- **All clusters** — overview of every cluster across all users, stats (including estimated monthly spend), cost per cluster, deploy/scale/destroy actions
- **Users & limits** — user list, cluster limits, per-user overrides
- **Configuration** — platform settings (this is what we need now)
- **Cluster templates** — pre-configured cluster profiles for the deploy form, with estimated cost per template
- **Audit log** — append-only record of every operation

### 11.3 Configuration tab

Click **Configuration**. This page lets you set up everything the platform needs.

**CIDR Pool**

Add /24 CIDR ranges that Infragate can allocate to clusters. Type a CIDR (e.g. `10.120.1.0/24`) into the input field and click **Add CIDR**. Add as many as you need — each cluster consumes one.

**OCI Settings**

| Field | Value |
|---|---|
| Parent Compartment OCID | OCID of the compartment from Section 5.6 |
| OCI Region | e.g. `eu-frankfurt-1` |
| Cluster Tier | **Basic** (free) or **Enhanced** (~$0.10/hr, full scaling) |
| Terraform State Bucket | `infragate-tfstate` |
| Object Storage Namespace | Your OCI namespace from Section 5.4 |

**Available VM Shapes**

Add the VM shapes users can choose when deploying. Type the OCI shape name (e.g. `VM.Standard.A1.Flex`) and an optional display label, then click **Add shape**.

Common shapes:

| Shape | Description |
|---|---|
| `VM.Standard.A1.Flex` | Ampere ARM64, flexible OCPU/RAM |
| `VM.Standard.E4.Flex` | AMD EPYC, flexible OCPU/RAM |
| `VM.Standard.E2.1.Micro` | Always Free micro instance (1 OCPU, 1 GB) |

**Node Images**

*(Optional)* Add OCI compute images for worker node pools. If no images are configured, OKE auto-selects the latest compatible image. To pin a specific image, paste the image OCID (e.g. `ocid1.image.oc1.eu-frankfurt-1.aaaa...`) and add a display label (e.g. "Oracle Linux 8 — OKE 1.32").

Find available images in the OCI Console under **Compute > Custom Images** or via CLI:
```bash
oci compute image list --compartment-id <tenancy-ocid> --operating-system "Oracle Linux" --operating-system-version "8"
```

**Global Resource Limits**

Set the default limits that apply to all users:

| Setting | Recommended starting value |
|---|---|
| Clusters per user | 1 |
| Max node pools per cluster | 3 |
| Max nodes per pool | 3 |
| Max OCPU per node | 1 |
| Max RAM per node (GB) | 12 |
| Max storage per node (GB) | 50 |

Click **Save configuration** to apply.

### 11.4 Cluster templates

Navigate to **Cluster templates** in the admin nav. This page lets you create pre-configured cluster profiles that appear as selectable cards on the deploy form.

Click **+ Add template** to create a new template:

| Field | Description |
|---|---|
| Template name | Display name shown on the deploy form card |
| Description | Short description shown below the name |
| K8s version | Pre-selected version (optional — user selects if left blank) |
| VM shape | Pre-selected shape (optional) |
| Node image | Pre-selected OCI compute image (optional — auto-selects latest OKE image if left blank) |
| Node pools | Pre-defined pool layout — click "Add pool" to define pools with name, nodes, OCPU, RAM, storage |
| Default tier | Suggested cluster tier — Basic or Enhanced (optional) |
| TTL | Time-to-live in hours — clusters expire after this duration (optional) |
| Destroy protection | When enabled, clusters created from this template cannot be destroyed without admin approval |
| Required role | Keycloak realm role — only users with this role see this template on the deploy form. Leave empty to make the template available to all users |
| Sort order | Controls card position in the deploy form (lower numbers appear first) |

The templates table shows estimated monthly and hourly cost for each template. When adding or editing a template, a live cost preview updates as you change pools, shape, or tier.

Templates can be edited at any time. **Disable** removes a template from the deploy form without deleting it — existing clusters that were created from the template retain their reference. **Delete** permanently removes the template from the admin panel.

When a user selects a template on the deploy form, the pool configuration, K8s version, shape, and node image are pre-filled and locked. Destroy protection and TTL policies are applied server-side at deploy time and cannot be overridden by the user.

#### Environment-tier gating with roles

Use `required_role` to control which teams can provision which cluster types. A typical enterprise setup:

| Template | Required role | TTL | Destroy protection | Intended audience |
|---|---|---|---|---|
| DEV — Small | *(none)* | 72h | No | All developers — lightweight clusters for local testing |
| TEST — Medium | `testing` | 168h | No | QA engineers — integration and regression testing |
| UAT — Large | `uat` | *(none)* | Yes | Release managers — pre-production validation |
| PROD — HA | `production` | *(none)* | Yes | Production team — full HA with destroy protection |

**Setup steps:**

1. **Create roles in Keycloak** — go to **Realm roles > Create role** and create `testing`, `uat`, and `production` roles (the DEV template has no role requirement, so all users can see it)
2. **Assign roles to users** — go to **Users > [user] > Role mapping > Assign role** and grant the appropriate roles. A user can have multiple roles (e.g. a senior engineer might have both `testing` and `uat`)
3. **Set `required_role` on each template** — in the Infragate admin panel under **Cluster templates**, edit each template and set the role field

Users only see the templates they have access to on the deploy form. Restricted templates don't appear at all — no error messages, no greyed-out cards. The API enforces the same check server-side, so role bypassing via direct API calls is not possible.

**Available K8s Versions**

Add the Kubernetes versions users can select. Type a version (e.g. `v1.35`) and click **Add version**. Check which versions are available in your OCI region under **Developer Services > Kubernetes Clusters > Create Cluster**.


### 11.5 Per-user overrides

Go to **Users & limits**. Click **Edit Limits** on a user to set per-user limit overrides. When an override is set, it takes precedence over the global default for that user. Leave a field blank to inherit the global value.

---

## 12. Verify

### 12.1 End-to-end check

1. Open `https://dev.infragate.cloud` — landing page loads
2. **Log In** — redirects to Keycloak, log in
3. As admin, the **All clusters** page loads (regular users see **My Clusters**)
4. Click **Deploy** — form shows configured shapes, images, K8s versions, and limits
5. As admin, the navigation bar shows admin links (All clusters, Users & limits, etc.)

### 12.2 API health

```bash
curl https://dev.infragate.cloud/health
# {"status":"ok","env":"production"}
```

### 12.3 Pods

```bash
kubectl get pods -n infragate -o wide
```

### 12.4 Test deploy

Deploy a test cluster through the UI to confirm Terraform can reach OCI and provision resources. Watch the live log stream — it should show `terraform init`, `plan`, and `apply` output in real time.

---

## 13. Maintenance

### Upgrade Infragate

CI pushes new images to GHCR on every push to `DEV` or `main`. To upgrade the running deployment:

**Code-only changes (no Helm chart changes):**

No `git pull` needed. Just tell Helm to use the new image tag:

```bash
# Use dev-latest to always get the newest DEV image
helm upgrade infragate ./deploy/helm \
  -n infragate \
  --reuse-values \
  --set api.image.tag=dev-latest \
  --set frontend.image.tag=dev-latest \
  --set api.image.pullPolicy=Always \
  --set frontend.image.pullPolicy=Always

# Or pin to a specific commit SHA
helm upgrade infragate ./deploy/helm \
  -n infragate \
  --reuse-values \
  --set api.image.tag=dev-abc1234 \
  --set frontend.image.tag=dev-abc1234
```

**Helm chart or Terraform changes:**

If the chart templates, values, or Terraform files changed, pull first:

```bash
cd ~/infragate
git pull origin DEV

helm upgrade infragate ./deploy/helm \
  -n infragate \
  -f deploy/helm/values-dev.yaml \
  --set api.image.tag=dev-latest \
  --set frontend.image.tag=dev-latest \
  --reuse-values
```

**Verify the upgrade:**

```bash
kubectl rollout status deployment/infragate-api -n infragate
kubectl rollout status deployment/infragate-frontend -n infragate
```

### View logs

```bash
kubectl logs -n infragate deploy/infragate-api -f
kubectl logs -n infragate deploy/infragate-keycloak -f
kubectl logs -n infragate infragate-postgresql-0 -f
```

### Database backup

```bash
kubectl exec -n infragate infragate-postgresql-0 -- \
  pg_dump -U infragate infragate > infragate-backup-$(date +%Y%m%d).sql
```

### Restore database

```bash
cat infragate-backup.sql | kubectl exec -i -n infragate infragate-postgresql-0 -- \
  psql -U infragate infragate
```

### Keycloak production mode

The Helm chart runs Keycloak in **production mode** (`start` instead of `start-dev`). This enables theme caching, token caching, and JVM optimizations — login/logout redirects are significantly faster.

Key differences from dev mode:
- First boot takes ~60-90s (one-time config build), subsequent startups are faster
- JVM heap set to 70% of container memory (1536Mi limit)
- DB connection pool pre-warmed (5 initial, 20 max)
- OIDC well-known and JWKS endpoints cached at the nginx layer (1h TTL)

If you need to access the Keycloak admin console for debugging, the admin credentials are the same as configured during install (`--set keycloak.admin.password`).

> **Docker Compose** (`backend/docker-compose.yml`) still uses `start-dev` for local development — this is intentional for faster iteration.

### Restart a component

```bash
kubectl rollout restart deploy/infragate-api -n infragate
kubectl rollout restart deploy/infragate-keycloak -n infragate
kubectl rollout restart deploy/infragate-frontend -n infragate
```

### Upgrade k3s

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
sudo systemctl restart k3s
kubectl get nodes
```

### Uninstall

```bash
helm uninstall infragate -n infragate
kubectl delete pvc -n infragate --all
kubectl delete namespace infragate
```

---

For identity provider integration, API reference, and resource limits see [INTEGRATION.md](./INTEGRATION.md).
For architecture and technical stack see [STACK.md](./STACK.md).
For testing procedures see [TESTING.md](./TESTING.md).
For marketplace listing strategy see [MARKETPLACE.md](./MARKETPLACE.md).
