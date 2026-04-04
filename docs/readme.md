# Infragate

**Internal Developer Platform for Oracle Cloud — automated OKE lifecycle management for teams.**

Infragate is an internal developer platform that lets engineers spin up, scale, and tear down managed Kubernetes clusters on OCI without needing cloud console access, IAM knowledge, or Terraform expertise. A cluster is ready in minutes — no tickets, no waiting.

---

## Table of contents

1. [Quick start](#1-quick-start)
2. [User guide](#2-user-guide)
3. [Admin guide](#3-admin-guide)
4. [Cluster tiers](#4-cluster-tiers)
5. [Resource limits](#5-resource-limits)
6. [Access — kubeconfig and SSH](#6-access--kubeconfig-and-ssh)
7. [Networking](#7-networking)
8. [Testing & CI](#8-testing--ci)
9. [Deployment phases](#9-deployment-phases)
10. [License](#10-license)

---

## 1. Quick start

### OCI Marketplace (recommended)

1. Find **Infragate** on the [OCI Marketplace](https://cloud.oracle.com/marketplace)
2. Click **Launch Stack** — a guided form walks you through:
   - Compartment, region, domain name
   - New OKE cluster or deploy into existing
   - New VCN or use existing networking
   - Bundled Keycloak or connect your own OIDC provider
3. Click **Apply** — Resource Manager provisions everything automatically
4. Point your DNS to the load balancer IP shown in the outputs
5. Log in and start provisioning clusters

### Existing OKE cluster

For teams that already have an OKE cluster:

```bash
helm upgrade --install infragate deploy/helm/ -n infragate \
  -f deploy/helm/values-oke.yaml \
  --set global.domain=infragate.example.com \
  --set postgresql.auth.password=YOUR_DB_PASSWORD \
  --set keycloak.admin.password=YOUR_KC_PASSWORD \
  --set api.oci.tenancyOcid=ocid1.tenancy... \
  --set api.oci.userOcid=ocid1.user... \
  --set api.oci.fingerprint=xx:xx:xx... \
  --set api.oci.namespace=YOUR_NAMESPACE \
  --set api.oci.parentCompartmentOcid=ocid1.compartment...
```

> Helm chart access is available to licensed customers. Contact [hello@infragate.cloud](mailto:hello@infragate.cloud) for details.

### Single-node k3s

For dev/test or Always Free tier VMs:

```bash
helm upgrade --install infragate deploy/helm/ -n infragate \
  -f deploy/helm/values-dev.yaml \
  --set postgresql.auth.password=YOUR_DB_PASSWORD \
  --set keycloak.admin.password=YOUR_KC_PASSWORD
```

> Helm chart access is available to licensed customers. Contact [hello@infragate.cloud](mailto:hello@infragate.cloud) for details.

See [GUIDE.md](./GUIDE.md) for the full step-by-step deployment guide for all three paths.

---

## 2. User guide

### Signing in

Infragate uses your organisation's existing identity provider (Keycloak, Azure AD, Okta, Google Workspace, or any OIDC-compliant IdP). Click **Log In** — you are redirected to your IdP's login page and returned to the portal after authentication. No separate Infragate account is needed.

### Deploying a cluster

1. Navigate to **Deploy** from the top nav
2. *(Optional)* Select a **Cluster template** — pre-configured profiles created by your admin that pre-fill and lock resource fields (pools, nodes, CPU, RAM, storage). Choose "Custom" to configure everything manually with your account's limits enforced
3. Enter a **Cluster name** — this becomes the base name for all OCI resources (`{name}-cluster`, `{name}-vcn`, etc.)
4. Choose a **CIDR range** from the available pool — each cluster gets its own /24
5. Select a **Kubernetes version**, **VM shape**, and **Node image** (locked if the template defines them; otherwise user-selectable — image defaults to auto-select if none configured)
6. If using "Custom" mode, configure one or more **Node pools** — each pool has a name, node count, OCPU, RAM, and storage, constrained by your account's effective limits
7. *(Optional)* Use the **Advanced tab** to bring existing OCI infrastructure — your own VCN, compartment, or subnet
8. Review the **Deployment Summary** — shows all resources that will be created, plus live estimated monthly and hourly cost
9. Click **Deploy**

Infragate runs `terraform apply` and streams the live output in the portal. The cluster appears in **My Clusters** once provisioning completes.

> The deploy form enforces your account's effective limits — maximum pools, nodes, OCPU, RAM, and storage. If a limit prevents you from deploying, contact your admin to request a higher limit.

### Scaling a cluster

1. In **My Clusters**, click **Scale** on a running cluster
2. Adjust resources per pool — nodes, OCPU, RAM, and storage
3. Add new node pools or remove existing ones as needed
4. Review the change preview and click **Apply**

Scaling behaviour depends on the cluster tier configured by your admin — see [Cluster tiers](#4-cluster-tiers).

### Destroying a cluster

1. In **My Clusters**, click **Destroy**
2. Infragate runs `terraform plan -destroy` and shows the exact resources that will be removed
3. Confirm — `terraform destroy` runs and all OCI resources are cleaned up
4. The CIDR range is returned to the pool and available for reuse

Destroy is permanent and cannot be undone.

> **Destroy protection:** Clusters created from a template with destroy protection enabled cannot be destroyed by regular users. An admin must override the protection with the `?force=true` query parameter.

### Cluster detail

Click **Details** on any cluster card to see:

- **Cluster information** — name, status, K8s version, region, CIDR, compartment, VM shape, cluster tier, estimated monthly cost, OCID
- **Node pools** — pool name, node count, OCPU, RAM, storage, and status per pool
- **Cost breakdown** — per-pool monthly cost, control plane cost (Enhanced tier), and total with hourly rate
- **Access** — kubeconfig download and SSH key download (see [Section 5](#5-access--kubeconfig-and-ssh))

---

## 3. Admin guide

Admin access is granted via your identity provider through role assignment — see [INTEGRATION.md](./INTEGRATION.md). Admins have a separate panel accessible from the top nav.

### All clusters

A full view of every cluster across all users — status, owner, CIDR, K8s version, tier, resource counts, estimated cost (monthly + hourly), and age. The stats bar shows total estimated monthly spend across all active clusters. From this view admins can:

- Click **Details** on any cluster to open its full detail view including kubeconfig and SSH key download
- Click **Destroy** to destroy any cluster regardless of owner
- Click **New Cluster** to provision a cluster outside of normal user quotas

### Users & limits

Lists every user who has signed into Infragate. For each user, admins can:

- View current cluster count and active limit
- View resolved effective limits (per-user override if set, otherwise global default)
- Click **Edit Limits** to set per-user overrides for any combination of: cluster limit, pool max, node max, OCPU, RAM, storage, and cluster tier
- Reset all overrides at once to revert the user to global platform defaults

Limit changes take effect on the user's next page load or login — no restart required.

### Configuration

Platform-wide settings manageable at runtime — no redeployment needed:

| Setting | Description |
|---|---|
| Cluster tier | Basic (free) or Enhanced (~$0.10/hr) — applies to all new clusters |
| Cluster limit | Default max active clusters per user |
| Node pool limit | Max pools per cluster |
| Nodes per pool | Max nodes per pool |
| OCPU / RAM / Storage | Max compute per node |
| CIDR pool | Add, remove, and enable/disable /24 ranges available for cluster allocation |
| VM shapes | Add, remove, and enable/disable shapes in the deploy form |
| K8s versions | Add, remove, and enable/disable Kubernetes versions |
| Node images | Add, remove, and enable/disable OCI compute images for worker node pools |

### Cluster templates

Dedicated admin page for creating and managing cluster templates — pre-configured profiles that appear as selectable cards on the deploy form. Templates encode:

| Setting | Description |
|---|---|
| K8s version | Pre-selected Kubernetes version (optional — user selects if not set) |
| VM shape | Pre-selected node shape (optional) |
| Node image | Pre-selected OCI compute image (optional — auto-selects latest OKE image if not set) |
| Node pools | Pre-defined pool layout — name, node count, OCPU, RAM, storage per pool |
| Tier default | Suggested cluster tier — Basic or Enhanced (suggestion only, not enforced) |
| TTL | Optional time-to-live in hours — clusters expire after this duration |
| Destroy protection | When enabled, users cannot destroy clusters created from this template without admin approval |
| Required role | Keycloak realm role — only users with this role can see and use the template (leave empty for all users) |
| Sort order | Controls display position in the deploy form (lower numbers appear first) |

The templates table shows estimated monthly and hourly cost per template. The add/edit modal includes a live cost preview that updates as you change pools, shape, or tier.

**Environment-tier gating** — use `required_role` to control which teams can deploy which cluster sizes. For example, create a `DEV — Small` template visible to everyone, a `TEST — Medium` template requiring a `testing` role, a `UAT — Large` template requiring a `uat` role, and a `PROD — HA` template requiring a `production` role. Users only see the templates they have access to on the deploy form. Create the roles in Keycloak under **Realm roles** and assign them to the relevant users.

Templates can be created, edited, enabled/disabled, and permanently deleted. Disabled templates no longer appear in the deploy form but remain referenced by existing clusters. Permanently deleting a template removes it from the admin panel entirely.

### Audit Log

Every provisioning, scaling, and destroy operation is recorded with user identity, operation type, cluster name, outcome, and duration. The log is append-only and filterable by user, operation, and date range.

---

## 4. Cluster tiers

The admin configures the cluster tier for the entire platform. Individual users do not choose — they see the current tier as an informational indicator on the deploy form and on their cluster cards.

| | Basic | Enhanced |
|---|---|---|
| Cost | Free | ~$0.10/hr per cluster |
| Node scaling | ⚠️ Full — but existing nodes require manual cycling | ✅ Full in-place |
| Cluster Autoscaler | ❌ | ✅ |
| Best for | Dev/test, cost-sensitive teams | Production, automated scaling |

**Basic cluster scaling:** Both Basic and Enhanced clusters support full scaling — nodes, OCPU, RAM, and storage. When a Basic cluster is scaled, OKE updates the desired configuration and new nodes are provisioned automatically. However, existing nodes are not replaced in-place. To apply configuration changes to running nodes, manually cycle them via the OCI Console:

**OKE → Cluster → Node Pool → Nodes → Cordon & drain → Terminate**

The portal shows a clear warning when scaling a Basic cluster.

**Switching tiers:** Admins can switch the platform tier at any time in Configuration. The change applies to all new clusters — existing clusters retain the tier they were provisioned with.

**Per-user tier override:** Admins can give a specific user Enhanced access while keeping the global default as Basic — useful for individual teams that need production-grade scaling without upgrading the entire platform.

---

## 5. Resource limits

Limits cascade: global config sets the baseline for all users, per-user overrides take precedence when set.

| Limit | Global default | Per-user override |
|---|---|---|
| Clusters per user | Inherited from config | ✅ |
| Node pools per cluster | 3 | ✅ |
| Nodes per pool | 3 | ✅ |
| OCPU per node | 1 | ✅ |
| RAM per node | 12 GB | ✅ |
| Storage per node | 50 GB | ✅ |
| Cluster tier | BASIC_CLUSTER | ✅ |

The deploy form always reflects the user's current effective limits. If a user hits a limit, the form shows a clear message with a link to contact support. Limits can be raised at any time by an admin without restarting the platform.

---

## 6. Access — kubeconfig and SSH

### Kubeconfig

Available on the cluster detail page once the cluster is running. The kubeconfig uses the OCI CLI exec plugin (`oci ce cluster generate-token`) — users need the [OCI CLI](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm) installed locally.

```bash
export KUBECONFIG=~/oke-myname-cluster-kubeconfig.yaml
kubectl get nodes
```

Admins can download the kubeconfig for any cluster from **All Clusters → Details**.

### SSH key

The SSH private key for cluster nodes is generated by Terraform at provisioning time and made available on the cluster detail page.

> ⚠️ **Save the SSH key when you first see it.** Store it in a safe location — for security it may not be retrievable after the first download.

```bash
ssh -i ~/oke-myname-cluster-key.pem opc@<node-private-ip>
```

Admins can download SSH keys for any cluster from **All Clusters → Details**.

---

## 7. Networking

### Default setup

By default, Infragate creates a full network stack per cluster:

- A **VCN** with a /16 range derived from the cluster's /24 CIDR
- A **private worker subnet** (/24)
- An **internet gateway** for egress
- A **route table** routing `0.0.0.0/0` to the IGW
- A **security list** with OKE-required ingress/egress rules — worker node ports, API server access, node-to-node communication

### Bringing your own network

Users can supply existing OCIDs via the **Advanced tab** in the deploy form. Infragate skips creating whichever resources are already provided:

| Override | Effect |
|---|---|
| Existing VCN OCID | Uses your VCN — skips VCN, IGW, and route table creation |
| Existing Compartment OCID | Places resources in your compartment — skips compartment creation |
| Existing Subnet OCID | Places nodes in your subnet — skips subnet and security list creation |

Any combination is supported.

### Custom security rules

Infragate's default security list covers the ports required by OKE. For custom ingress/egress rules — exposing specific node ports, restricting egress to corporate CIDRs, or integrating with existing NSGs — the recommended approach is to **bring your own subnet** via the Advanced tab with your security configuration already applied. Infragate uses your subnet and skips creating its own security list entirely, giving your network team full control without adding UI complexity.

---

## 8. Testing & CI

### Running tests

```bash
cd backend
pip install -r requirements.txt
pytest tests/ -v --tb=short --cov=app --cov-report=term-missing
```

87 automated tests covering user provisioning, limit resolution, admin config CRUD, cluster templates, cost estimation, access control (kubeconfig + SSH key), and API contracts. Tests run against an in-memory SQLite database with mocked authentication — no external services required.

### CI pipeline

Infragate ships with CI/CD pipelines for both GitHub and GitLab:

**GitHub Actions** (`.github/workflows/ci.yml`):

| Job | What it validates |
|---|---|
| `test` | 87 Python unit tests with coverage |
| `helm-lint` | Helm lint + template rendering (default + k3s + OKE values) |
| `docker-build-push` | Build + push images to GHCR (`dev-latest` on DEV, `latest` on main) |
| `terraform-validate` | Core module, runner template, and marketplace stack |

**GitLab CI** (`.gitlab-ci.yml`):

| Job | What it validates |
|---|---|
| `test` | 87 Python unit tests with coverage |
| `helm-lint` | Helm lint + template rendering (default + k3s + OKE values) |
| `build-api` / `build-frontend` | Build + push images to GitLab Container Registry |
| `terraform-validate` | Core module, runner template, and marketplace stack |

Both use the same tagging scheme (`latest` / `dev-latest` + commit SHA). The Helm chart's `imagePullSecrets` support makes it work with any private container registry.

---

## 9. Deployment phases

| Phase | Status | Description |
|---|---|---|
| 1 — OCI Foundation | ✅ Complete | IAM, compartment, Object Storage, service account |
| 2 — Terraform Module | ✅ Complete | Module validated with real `terraform plan` against OCI |
| 3 — Container Host | ✅ Complete | k3s on OCI ARM instance in eu-frankfurt-1 |
| 4 — Identity Integration | ✅ Complete | OIDC config, realm/client setup |
| 5 — FastAPI | ✅ Complete | Full backend with Terraform runner integration |
| 6 — Wire frontend | ✅ Complete | Frontend connected to live API |
| 7 — Cluster templates | ✅ Complete | Admin-managed templates, deploy form selector, destroy protection, TTL |
| 8 — Cost visibility | ✅ Complete | Live cost estimation across deploy form, dashboard, detail page, admin panels, and templates |

---

## 10. License

Infragate is licensed under the [Business Source License 1.1](./LICENSE).

| Parameter | Value |
|---|---|
| Licensor | Solvia Lab s.r.o. |
| Licensed Work | Infragate |
| Additional Use Grant | Production use permitted, except offering as a commercial managed service |
| Change Date | 2030-03-19 |
| Change License | Apache License, Version 2.0 |

**What this means:**

- You may use, modify, and redistribute Infragate freely for internal and non-production purposes
- Production use is permitted, provided you do not offer Infragate as a hosted service to third parties
- On the Change Date (or 4 years after any given release), that version automatically converts to Apache 2.0
- For commercial managed-service use or alternative licensing, contact [hello@infragate.cloud](mailto:hello@infragate.cloud)

---

For a complete feature overview see [FEATURES.md](./FEATURES.md).
For integration and deployment details see [INTEGRATION.md](./INTEGRATION.md).
For the full technical stack and backend architecture see [STACK.md](./STACK.md).

Built by [Solvia Lab s.r.o.](https://infragate.cloud) · [solvialab.tech](https://solvialab.tech)
