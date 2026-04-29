# Infragate

**OCI-native Internal Developer Platform (IDP) for OKE on Oracle Cloud — automated lifecycle management and governance for platform teams.**

Infragate is a self-hosted internal developer platform (IDP) that lets engineers spin up, scale, and tear down managed Kubernetes clusters on OCI without needing cloud console access, IAM knowledge, or Terraform expertise. A cluster is ready in minutes — no tickets, no waiting.

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

Pick the deployment path that matches your environment — both use the same Helm chart with different values files, tuned for the ingress controller, storage class, and networking model of each target.

| | **Existing OKE cluster** *(recommended for production)* | **Single-node k3s on an OCI VM** |
|---|---|---|
| Best for | Enterprise teams already running OKE | Dev/test, demos, Always Free tier |
| Ingress controller | ingress-nginx + OCI flexible Load Balancer (installed separately) | Traefik (bundled with k3s, zero setup) |
| Storage class | `oci-bv` (OCI Block Volume) | `local-path` (k3s default) |
| Image pull policy | `IfNotPresent` (OKE nodes pull from GHCR) | `Always` (single VM refresh-on-restart) |
| Scheduling | No tolerations (OKE has dedicated workers) | Control-plane tolerations (single-node scheduling) |
| Values file | `deploy/helm/values-oke.yaml` | `deploy/helm/values-k3s.yaml` |
| Chart values `ingress.className` | `nginx` | `traefik` |
| Setup time | ~30 min (cluster exists) | ~15 min |
| Guide | Available during evaluation/POC | Available during evaluation/POC |
| Testing playbook | Available during evaluation/POC | Available during evaluation/POC |

### Existing OKE cluster (recommended)

For teams that already have an OKE cluster — deploy Infragate via Helm in minutes:

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

> Contact [hello@infragate.cloud](mailto:hello@infragate.cloud) for Helm chart access and onboarding support.

### Single-node k3s

For dev/test, demos, or OCI Always Free tier VMs. Run this on an OCI VM in the same region as your target tenancy for low-latency access to the OCI API:

```bash
helm upgrade --install infragate deploy/helm/ -n infragate \
  -f deploy/helm/values-k3s.yaml \
  --set global.domain=infragate.example.com \
  --set postgresql.auth.password=YOUR_DB_PASSWORD \
  --set keycloak.admin.password=YOUR_KC_PASSWORD \
  --set api.oci.tenancyOcid=ocid1.tenancy... \
  --set api.oci.userOcid=ocid1.user... \
  --set api.oci.fingerprint=xx:xx:xx... \
  --set api.oci.namespace=YOUR_NAMESPACE \
  --set api.oci.parentCompartmentOcid=ocid1.compartment...
```

OCI credentials are still required — Infragate calls the OCI API to provision OKE clusters regardless of where Infragate itself runs. See customer onboarding runbooks (gated) for OCI service-account setup and state-backend provisioning steps.

> Contact [hello@infragate.cloud](mailto:hello@infragate.cloud) for Helm chart access and onboarding support.

### OCI Marketplace (planned)

An OCI Marketplace listing with one-click "Launch Stack" deployment is planned for a future release. In the meantime, the Helm-based deployment paths above provide the same functionality with full flexibility.

Full deployment runbook is available during evaluation/POC.

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
Kubernetes version changes are handled via a separate **Upgrade** action.

### Destroying a cluster

1. In **My Clusters**, click **Destroy**
2. Infragate runs `terraform plan -destroy` and shows the exact resources that will be removed
3. Confirm — `terraform destroy` runs and cluster-scoped OCI resources are cleaned up
4. The CIDR range is returned to the pool and available for reuse

Destroy is permanent and cannot be undone.

> **Requests workflow:** The admin **Requests** area handles both protected-cluster destroy approvals and per-user limit-increase requests. Users submit a reason instead of destroying directly or when they need higher limits; admins approve, adjust, or deny with a note. Destroy approval still opens the Terraform destroy plan before force-destroy. Limit approval writes granted values into the user's per-user overrides. Results appear in the user's Activity inbox.
>
> **Destroy behavior note:** Cluster compartments are intentionally **not auto-deleted** (to avoid breaking shared/multi-cluster setups). In Object Storage, only the destroyed cluster's `.tfstate` object is removed; the user-level prefix/folder is kept for other clusters.

### Activity

The user nav includes an **Activity** inbox with an unread badge. It persists important events across refreshes and sessions; today it records destroy-request approvals/denials, limit request submissions and reviews, TTL expiry warnings at 24h, 4h, and 1h remaining, deploy/scale/upgrade/destroy start and completion events, and admin-driven account limit changes.

### Cluster detail

Click **Details** on any cluster card to see:

- **Cluster information** — name, status, K8s version, region, CIDR, compartment, VM shape, cluster tier, estimated monthly cost, OCID
- **Node pools** — pool name, node count, OCPU, RAM, storage, and status per pool
- **Cost breakdown** — per-pool monthly cost, control plane cost (Enhanced tier), and total with hourly rate
- **Access** — kubeconfig download and SSH key download (see [Section 5](#5-access--kubeconfig-and-ssh))

---

## 3. Admin guide

Admin access is granted via your identity provider through role assignment — see customer integration docs (gated). Admins have a separate panel accessible from the top nav.

### All clusters

A full view of every cluster across all users — status, owner, CIDR, K8s version, tier, resource counts, estimated cost (monthly + hourly), and age. The stats cards show online, provisioning, upgrading k8s, destroying, failed, total, CIDRs used, and monthly spend. From this view admins can:

- Click **Details** on any cluster to open its full detail view including kubeconfig and SSH key download
- Click **Scale** on Active/Error clusters to adjust pools, nodes, and per-node resources
- Click **Upgrade k8s** on Active/Error clusters to upgrade Kubernetes version
- Click **Destroy** to destroy any cluster regardless of owner
- Click **View logs** while an operation is running or after a failure
- Click **New Cluster** to provision a cluster outside of normal user quotas

### Users & limits

Lists every user who has signed into Infragate. For each user, admins can:

- View current cluster count and active limit
- View resolved effective limits (per-user override if set, otherwise global default)
- Click **Edit Limits** to set per-user overrides for any combination of: cluster limit, pool max, node max, OCPU, RAM, storage, and cluster tier
- Reset a user's overrides back to global defaults; direct admin limit edits and resets create user Activity notifications
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
| VM shapes | Sync from OCI, plus add/remove/enable/disable shapes in the deploy form |
| K8s versions | Add, remove, and enable/disable Kubernetes versions |
| Node images | Add, remove, and enable/disable OCI compute images for worker node pools |

> Note: Shape and image availability is region- and tenancy-specific. In **Admin → Configuration**, use **Sync from OCI** in the VM Shapes section first, then curate/label as needed. For CLI verification, fetch current OKE node pool options in the target region:
>
> `oci ce node-pool-options get --node-pool-option-id all --profile <PROFILE>`
>
> Use the returned `shapes` and `sources` lists as the source of truth for VM shapes and node images.
>
> If **Sync from OCI** is unavailable, Infragate keeps existing shape config unchanged and you can continue with manual curation from the CLI output above. Sync uses the API pod OCI service account credentials/policies (not the browser/user profile).
>
> K8s version freshness: OCI can publish new OKE patch versions after Infragate is deployed. In **Admin -> Configuration -> K8s versions**, use **Refresh from OCI** periodically (recommended monthly, and before template reviews or upgrade waves), then enable the versions users may deploy. Upgrade recommendation pills use the latest enabled version, so stale config can hide available upgrades.
>
> Deploy form behavior: Kubernetes versions are filtered by the currently selected VM shape and region. Users only see versions that are both OCI-compatible for that shape (based on OKE node image availability) and enabled in Admin Configuration. If compatibility lookup is unavailable, the K8s list is not shown (fail-closed) and deploy is blocked until validation succeeds. If a version is missing, either enable it in Admin Configuration or choose a shape that supports it.

### Cluster templates

Dedicated admin page for creating and managing cluster templates — pre-configured profiles that appear as selectable cards on the deploy form. Templates encode:

| Setting | Description |
|---|---|
| K8s version | Pre-selected Kubernetes version (optional — user selects if not set) |
| VM shape | Pre-selected node shape (optional) |
| Node image | Pre-selected OCI compute image (optional — auto-selects latest OKE image if not set) |
| Node pools | Pre-defined pool layout — name, node count, OCPU, RAM, storage per pool |
| Tier default | Suggested cluster tier — Basic or Enhanced (suggestion only, not enforced) |
| TTL | Optional time-to-live in hours — when reached, Infragate automatically starts cluster destroy/cleanup |
| Destroy protection | When enabled, the "Destroy" button on the user's cluster opens a "Request destroy" modal instead. Requests land in Admin → Requests (with a live count badge in the nav) where an admin approves, reviews the destroy plan, then confirms force-destroy, or denies with a note |
| Required role | Keycloak realm role — only users with this role can see and use the template (leave empty for all users) |
| Sort order | Controls display position in the deploy form (lower numbers appear first) |

The templates table shows estimated monthly and hourly cost per template. The add/edit modal includes a live cost preview that updates as you change pools, shape, or tier.

Template modal behavior: when a VM shape is selected, the K8s dropdown is filtered to versions OCI reports as compatible for that shape in the current region (node-image-backed compatibility). If compatibility lookup is unavailable, no fallback list is shown (fail-closed). Template save/update re-validates shape+K8s compatibility and blocks incompatible combinations.

**Environment-tier gating** — use `required_role` to control which teams can deploy which cluster sizes. For example, create a `DEV — Small` template visible to everyone, a `TEST — Medium` template requiring a `testing` role, a `UAT — Large` template requiring a `uat` role, and a `PROD — HA` template requiring a `production` role. Users only see the templates they have access to on the deploy form. Create the roles in Keycloak under **Realm roles** and assign them to the relevant users.

Templates can be created, edited, enabled/disabled, and permanently deleted. Disabled templates no longer appear in the deploy form but remain referenced by existing clusters. Permanently deleting a template removes it from the admin panel entirely.

### Requests

Approval queue for user-submitted destroy requests on protection-enabled clusters and limit-increase requests for per-user overrides. The admin nav shows live request count badges that refresh every 5 seconds (hidden when zero pending). Filter by Pending / Approved / Denied / All. Each pending row has a **Review** action.

For destroy requests, the review modal lets the admin:
- Add an optional admin note
- Click **Approve & review plan** — the normal destroy plan opens. After the admin confirms the plan, the request is marked `approved` and `force=true` destroy starts. Audit-logged as `destroy-request:approve`.
- Click **Deny** — the note is surfaced on the user's cluster card as a red "Destroy denied" pill (with the note in the tooltip). Audit-logged as `destroy-request:deny`.

For limit requests, the review modal lets the admin grant the requested values or adjust them before approval. Approved values are written into the user's per-user overrides and the user receives an Activity notification. Denials can include an admin note and are audit-logged as `limit-request:deny`.

Approval is row-locked to prevent two admins double-approving. Bypass path: admins can still force-destroy any protected cluster directly via `?force=true` from the All Clusters table — useful for incident response.

### Audit Log

Every provisioning, scaling, upgrade, and destroy operation is recorded with user identity, operation type, cluster name, outcome, and duration — including `destroy-request:*` and `limit-request:*` events from the Requests queue. The log is append-only and filterable by user, operation, and date range.

---

## 4. Cluster tiers

The admin configures the cluster tier for the entire platform. Individual users do not choose — they see the current tier as an informational indicator on the deploy form and on their cluster cards.

| | Basic | Enhanced |
|---|---|---|
| Cost | Free | ~$0.10/hr per cluster |
| Node scaling | ⚠️ Full — add/remove nodes or pools is automatic; resource-profile changes require manual cycling | ✅ Full in-place |
| Cluster Autoscaler | ❌ | ✅ |
| Best for | Dev/test, cost-sensitive teams | Production, automated scaling |

**Basic cluster scaling:** Both Basic and Enhanced clusters support full scaling — nodes, OCPU, RAM, and storage. In Basic clusters, adding/removing nodes or node pools is automatic and does not require manual rotation. Manual cycling is required when changing per-node resources (shape, OCPU, RAM, storage), and is also recommended after a Kubernetes upgrade so workers pick up the new target image/version.

**OKE → Cluster → Node Pool → Nodes → Cordon & drain → Terminate**

Recommended rolling cycle for each affected Basic node pool (current size = `N`):
1. Scale from `N` to `2N`
2. Wait until the additional `N` nodes are `Ready/Active`
3. Scale back from `2N` to `N` (OKE removes extra nodes automatically)
4. Verify workloads and node health before moving to the next pool

The portal shows tier-specific warnings/messages for this flow.

**Kubernetes version upgrades:** Use **Upgrade k8s** on a running cluster. This updates the control plane and node-pool target Kubernetes/image configuration:
- Direct upgrades are limited to the same or next minor version (for example `1.33.x -> 1.34.x`; not `1.33.x -> 1.35.x`).
- Basic: perform rolling worker refresh per pool (`N -> 2N -> N`) after nodes become `Ready/Active`; the Upgrade modal shows live per-pool steps based on current node counts.
- Enhanced: rollout is fully automated (control plane plus worker refresh), no manual cycling required.

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

Available on the cluster detail page once the cluster is running. The default download is a universal kubeconfig with an embedded per-user ServiceAccount token, so users only need `kubectl` — no OCI CLI, local OCI config, or plugins. The explicit `/kubeconfig-oci` endpoint remains available for power users who prefer OCI CLI exec auth.

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

Important behavior: `Existing Compartment OCID` does **not** auto-discover an existing VCN or subnet from that compartment. If VCN/subnet fields are left empty, Infragate creates a new VCN/subnets/network stack inside the provided compartment.

Advanced override UX guardrails: OCID fields validate type prefixes (`ocid1.vcn.`, `ocid1.compartment.`, `ocid1.subnet.`) and provide OCI-backed autocomplete lists for compartments, VCNs, and subnets.

> Shared compartment guidance: if you want multiple clusters in one compartment, use a dedicated BYO compartment via `Existing Compartment OCID` for all those clusters. Do not reuse a compartment that was auto-created for another cluster as a shared target.

> Note: If you deploy with existing VCN/subnet overrides, Infragate does not manage route tables for those existing resources. Your existing network must already provide equivalent private-subnet egress required for OKE worker node registration: route `0.0.0.0/0` to a NAT Gateway (or equivalent corporate egress path) and route `all-<region>-services-in-oracle-services-network` to a Service Gateway. This is a routing requirement, not an "open ingress" security rule.

> ⚠️ **Drift behaviour.** Infragate-created VCN, subnet, IGW, route table, and security list resources are fully managed by the cluster's Terraform state — manual edits in the OCI Console will be overwritten on the next apply. BYO resources (existing VCN / subnet / compartment) are referenced read-only and never modified.

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

133 automated tests covering user provisioning, limit resolution and limit requests, admin config CRUD, cluster templates, cost estimation, access control (kubeconfig + SSH key), lifecycle notifications, and API contracts. Tests run against an in-memory SQLite database with mocked authentication — no external services required.

### CI pipeline (maintainer-owned)

Infragate includes reference CI/CD pipelines for maintainers on GitHub and GitLab. These pipelines validate the repo and publish release images. Customer operators typically do not run these pipelines — they deploy versioned images from GHCR, OCIR, or an internal mirror.

**GitHub Actions** (`.github/workflows/ci.yml`):

| Job | What it validates |
|---|---|
| `test` | 133 Python unit tests with coverage |
| `helm-lint` | Helm lint + template rendering (default + k3s + OKE values) |
| `docker-build-push` | Build + push images to GHCR (`dev-latest` on DEV, `latest` on main) |
| `terraform-validate` | Core module and runner template |

**GitLab CI** (`.gitlab-ci.yml`):

| Job | What it validates |
|---|---|
| `test` | 133 Python unit tests with coverage |
| `helm-lint` | Helm lint + template rendering (default + k3s + OKE values) |
| `build-api` / `build-frontend` | Build + push images to GitLab Container Registry |
| `terraform-validate` | Core module and runner template |

Both use the same tagging scheme (`latest` / `dev-latest` + commit SHA) for maintainer release flow. For customer environments, prefer pinned release tags or digests from your approved registry. The Helm chart's `imagePullSecrets` support makes this work with any private container registry.

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
| 9 — Lifecycle validation (k3s) | ✅ Complete | End-to-end verified on single-node k3s VM: deploy, scale, upgrade k8s, destroy, and TTL auto-cleanup |

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
For integration, deployment, stack architecture, and API reference see customer integration docs (gated).
For end-to-end validation procedures and testing matrix, see docs available during evaluation/POC.

Built by [Solvia Lab s.r.o.](https://solvialab.tech)
