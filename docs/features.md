# Infragate by Solvia Lab — Features

A complete overview of Infragate by Solvia Lab as an OCI-native Internal Developer Platform (IDP) for governed OKE lifecycle management: self-service provisioning, BYON networking, access delivery, approvals, Activity history, and FinOps visibility. Detailed deployment and integration runbooks are available during evaluation/POC.

---

## Self-service cluster provisioning

- **One-click deploy** — engineers provision fully managed OKE clusters through a clean web portal, no CLI, Terraform, or OCI Console knowledge required
- **Live Terraform streaming** — `terraform init`, `plan`, and `apply` output streams in real time to the browser during deploy, scale, upgrade, and destroy operations
- **Template-based or custom** — choose from admin-defined cluster templates or configure everything manually
- **Automatic resource creation** — each cluster gets its own compartment, VCN, subnet, internet gateway, route table, security list, and node pools
- **VPN-first Kubernetes API access** — admins can enable public OKE API endpoints restricted to runner/VPN/corporate CIDRs. Infragate creates a dedicated public API endpoint subnet and TCP/6443 allowlist rules, avoiding DRG cost and LPG scaling limits while keeping access unavailable from the open internet.
- **Bring your own infrastructure** — optionally supply existing VCN, compartment, or subnet OCIDs via the Advanced tab to skip resource creation and wire into existing networks. Supplied BYO resources are referenced read-only and never edited by Terraform; if only an existing VCN is supplied, Infragate may still create cluster subnets/security lists inside it. Infragate-created VCN/subnet/security list resources are fully managed and manual OCI Console edits to them will be overwritten on the next apply. When using existing VCN/subnet overrides, route tables are not managed by Infragate, so equivalent private-subnet egress must already exist for OKE workers: route `0.0.0.0/0` to NAT Gateway (or equivalent corporate egress path) and route `all-<region>-services-in-oracle-services-network` to Service Gateway. This is routing, not an open ingress security rule.
- **BYON scope behavior** - `Existing Compartment OCID` alone reuses only the compartment. Infragate does not auto-discover existing VCN/subnet objects inside that compartment; leave VCN/subnet blank only when you want Infragate to create a fresh network stack there.
- **Advanced override guardrails** - Advanced OCID fields validate resource-type prefixes and provide OCI-backed autocomplete for compartments, VCNs, and subnets to reduce typo/copy-paste errors before deploy.
- **Shared compartment pattern** — for multiple clusters in one compartment, use a dedicated BYO compartment OCID in Advanced for every cluster in that shared domain; avoid reusing an auto-created per-cluster compartment as a shared target.
- **CIDR pool management** — admins pre-populate a pool of /24 ranges; each cluster allocates one on deploy and releases it on destroy
- **Multi-pool support** — configure 1–N node pools per cluster, each with independent node count, OCPU, RAM, and storage sizing
- **Shape sync from OCI** - Admin Configuration includes **Sync from OCI** for VM shapes, pulling OKE-compatible shapes for the current region/tenancy and merging them into `allowed_shapes` while preserving existing labels/toggles
- **K8s version sync from OCI** - Admin Configuration can refresh available OKE versions for the configured region so deploy options and upgrade recommendation pills stay aligned with newly published patch versions
- **Sync fallback behavior** - if OCI shape sync is unavailable (credentials/policy/network), existing `allowed_shapes` remain unchanged and admins can continue with manual shape curation from `oci ce node-pool-options get --node-pool-option-id all`
- **Shape-aware K8s picker** - deploy form Kubernetes options are filtered by the selected VM shape and region, and only include admin-enabled versions with OKE node-image compatibility for that shape. If compatibility lookup fails, the picker is fail-closed (no permissive fallback list), preventing invalid shape/version combinations before apply
- **Node image selection** - admins configure allowed OCI compute images; users select an image on the deploy form or leave it as auto-select (latest OKE-compatible image). Templates can lock a specific image. Recommended onboarding flow: sync shapes from OCI first, then curate images from `oci ce node-pool-options get --node-pool-option-id all`
- **Architecture-aware auto-select** — when no image is set, Terraform filters OKE image options by the requested shape's architecture: ARM shapes (`VM.Standard.A*`) resolve to `aarch64` images, GPU shapes to `Gen2-GPU` variants, and all others to plain x86_64 — preventing shape/image mismatches that would otherwise block node launches

---

## Cluster templates

- **Pre-configured profiles** — admins create templates that encode K8s version, VM shape, node image, pool layout, tier, TTL, and destroy protection
- **Deploy form pre-fill + lock** — selecting a template pre-fills and locks resource fields (pools, nodes, CPU, RAM, storage, pool names, and add/remove pool controls). Users can still set cluster name, CIDR, compartment, and advanced overrides. Select "Custom" to unlock all fields and configure manually with limit enforcement
- **Template values can exceed user limits** — templates represent admin-pre-approved configurations, so template-defined values are not clamped to the user's personal limits. The lock prevents users from editing these values
- **Requests workflow** — protected-cluster destroy approvals and per-user limit-increase requests share a dedicated admin Requests queue. Users submit a reason, admins approve/deny with an optional note, and outcomes are visible in Activity. Destroy approvals still require review of the Terraform destroy plan before force-destroy; limit approvals apply granted overrides to the user's account
- **Time-to-live (TTL)** — optional expiry in hours, enforced at deploy time; when TTL is reached, Infragate automatically triggers destroy/cleanup. Also available on custom deploys without a template
- **Live cost preview** — add/edit modal shows estimated monthly and hourly cost that updates as you change pools, shape, or tier
- **Template shape/K8s guardrail** — template modal uses the same shape-aware compatibility filtering as deploy; save/update re-validates selected shape+K8s and blocks incompatible combinations
- **Role-based access** — restrict templates to users with a specific Keycloak realm role. This enables environment-tier gating across your organisation:

  | Template | Required role | Who sees it |
  |---|---|---|
  | DEV — Small | *(none)* | All users |
  | TEST — Medium | `testing` | QA engineers and testers |
  | UAT — Large | `uat` | Release managers and senior engineers |
  | PROD — HA | `production` | Production team only |

  Create the roles in Keycloak under **Realm roles**, assign them to the relevant users, and set the `required_role` field on each template. Templates without a required role are visible to everyone. Users only see templates they have access to on the deploy form — no error messages, the restricted templates simply don't appear.
- **Sort order** — controls card display position on the deploy form
- **Enable/disable** — disabled templates disappear from the deploy form but remain referenced by existing clusters
- **Permanent delete** — removes a template from the admin panel entirely

---

## Cost visibility (FinOps)

Infragate provides live cost estimation across the entire platform using OCI Pay-As-You-Go rates as defaults, with optional admin overrides for custom contracts.

| Surface | What's shown |
|---|---|
| Deploy form — Deployment Summary | Estimated monthly and hourly cost, updates live as you configure pools |
| Deploy plan confirm modal | Estimated monthly cost in the resource plan |
| eashboard cluster cards | Estimated monthly cost per cluster |
| Cluster detail page | Monthly cost + full breakdown (per-pool cost, control plane cost, total with hourly rate) |
| Admin — All Clusters table | Monthly + hourly cost per cluster |
| Admin — Stats bar | Lifecycle status cards (online, provisioning, upgrading, destroying, failed, total) plus CIDRs used and monthly spend |
| Admin — Cluster Templates table | Monthly + hourly cost per template |
| Admin — Template add/edit modal | Live cost preview that updates on pool/shape/tier changes |

**Pricing model:**
- Compute: `nodes x (OCPU x $0.025/hr + RAM GB x $0.0015/hr)`
- Storage: `nodes x storage GB x $0.0255/mo`
- Enhanced control plane: `$0.10/hr` (Basic is free)
- Shape-specific rate overrides supported for custom OCI contracts
- Both server-side (Python) and client-side (JS) implementations produce identical results

---

## Scaling

- **Full resource scaling** — adjust nodes, OCPU, RAM, and storage per pool from the portal
- **Scale to zero** — node counts accept `0` on deploy and scale, letting users park a cluster configuration without running compute. Useful for pausing charges on idle Basic clusters where the control plane is free
- **Pool add/remove** — add new node pools or remove existing ones directly from the scale modal, no redeployment needed
- **Per-pool control** — scale each pool independently
- **Change preview** — review all changes before applying (current vs. new values, new pools highlighted, removed pools shown)
- **Separate K8s upgrade flow** — Kubernetes version upgrades use a dedicated Upgrade action/modal (not the scale flow)
- **Enhanced tier** — full in-place scaling via OKE API, no manual node cycling
- **Basic tier** — adding/removing nodes or node pools is automatic; changing shape/OCPU/RAM/storage requires rolling node cycling (`N -> 2N -> N` after new nodes are `Ready/Active`)

## Kubernetes upgrades

- **Dedicated action** — available from cluster detail and dashboard actions
- **Shape-aware + allowlist-aware** — upgrade options are filtered to OCI-compatible versions enabled by admin config
- **Sequential minors only** — direct upgrades allow same/next minor only; skip-minor upgrades are blocked with explicit error
- **Control plane + node-pool target update** — upgrade action updates cluster Kubernetes and node-pool target version/image
- **Basic upgrade behavior** — perform rolling worker refresh per pool (`N -> 2N -> N`) after new nodes are `Ready/Active`
- **Enhanced upgrade behavior** — fully automated rollout (control plane + workers), no manual cycling required
- **Tier guidance in UI** — Basic and Enhanced show upgrade-specific notes with rollout guidance
- **Dynamic Basic rollout guidance** — Basic upgrade modal reads live pool/node counts and renders exact per-pool steps (for example `1 -> 2 -> 1` or `3 -> 6 -> 3`) instead of a static example

---

## Cluster lifecycle

- **Status tracking** — real-time status across all views: provisioning, scaling, upgrading, destroying, running, error, destroyed
- **TTL visibility** — dashboard cards show color-coded countdown badges (green >24h, orange <24h, red <4h) for clusters with TTL. Detail page shows full expiry timestamp and remaining time
- **Destroy protection + approval queue** — protected clusters show a red "Protected" badge on dashboard cards, the admin All Clusters table, and the detail page. Non-admin users clicking "Destroy" open a "Request destroy" modal (optional reason) which creates a pending approval ticket. The admin nav shows a live-count "Requests (N)" badge, refreshed every 5s. On the admin Requests page, admins approve, review the destroy plan, then confirm force-destroy, or deny with a note — the user's cluster card then displays a "Destroy pending" (amber) or "Destroy denied" (red, note in tooltip) pill. At most one pending request per cluster. Every submit/approve/deny is audit-logged. Admins can still force-destroy directly via `?force=true`.
- **Activity inbox** — user nav includes a persistent Activity dropdown with unread counts, last events, and mark-read controls. Destroy-request approve/deny events, limit request submit/review events, TTL warnings, deploy/scale/upgrade/destroy lifecycle events, and admin-driven account limit changes emit inbox rows.
- **Destroy with cleanup** — `terraform destroy` removes cluster-scoped OCI resources and returns CIDR to pool; Infragate-managed child compartments may be deleted, external compartment overrides are retained, and only the cluster `.tfstate` object is deleted while the user prefix remains
- **Error recovery** — failed deployments show troubleshooting tips and a "Clean up" button to remove partial resources
- **Kubeconfig download** — universal kubeconfig with embedded per-user ServiceAccount token; no OCI CLI or local OCI config required. Infragate and the user must still reach the OKE API endpoint; the supported no-DRG/no-LPG path is the restricted public API endpoint allowlist. Explicit OCI-exec fallback remains available via `/kubeconfig-oci`.
- **SSH key download** — Terraform-generated private key available on the detail page

---

## Identity and access

- **Any OIDC provider** — works with Keycloak, Azure AD, Okta, Google Workspace, or any OIDC-compliant IdP
- **No user directory** — Infragate auto-provisions users on first login from the JWT `sub` claim
- **PKCE authentication** — Authorization Code + PKCE flow; no client secrets stored in the frontend
- **Role-based access** — `admin` role (from IdP) grants access to the admin panel; custom realm roles can restrict cluster template visibility (e.g. `production`, `staging`); all other users are regular users
- **Session management** — automatic token refresh, silent re-auth, secure logout via IdP end-session endpoint
- **Cached OIDC discovery** — well-known config cached in sessionStorage (1-hour TTL) and at the nginx proxy layer, eliminating network round-trips on page load, sign-in, and sign-out

---

## Resource limits

- **Two-tier limit system** — global platform defaults + per-user overrides
- **Granular control** — limits on clusters per user, pools per cluster, nodes per pool, OCPU, RAM, storage, and cluster tier
- **OCI minimums enforced** — storage per node has a hard floor of 50 GB (OCI compute boot volume minimum), enforced in admin config, per-user overrides, and deploy validation
- **Per-user overrides** — admins can raise or lower any limit for individual users without affecting others
- **Visual limit feedback** — stepper inputs gray out when values reach the configured maximum, giving a clear visual cue that the limit has been reached
- **Live enforcement** — deploy form constraints update on page load; server-side validation on every deploy request
- **Override visibility** — admin Users table shows which users have overrides and their effective resolved limits

---

## Admin panel

Six dedicated admin pages accessible to users with the `admin` role:

### All Clusters
- Every cluster across all users with status, owner, CIDR, K8s version, tier, resources, cost, and age
- Stats bar: online, provisioning (includes scaling), upgrading k8s, destroying, failed, total, CIDRs used, monthly spend
- Actions: details, scale, upgrade k8s, destroy, and state-based view logs; plus new cluster (bypasses user quotas)

### Users & Limits
- All users with cluster count, limit, and per-user override badges
- Edit Limits modal: set per-user overrides for any combination of limits and tier
- Reset to global defaults with one click; direct edits and resets notify the affected user in Activity

### Configuration
- Platform-wide settings: region, compartment, cluster tier, state bucket, namespace
- CIDR pool: add/remove /24 ranges with allocation status
- VM shapes: sync from OCI, then optionally add custom shapes with display labels
- K8s versions: sync from OCI and manage available versions (deploy form shows only versions compatible with the currently selected shape/region; upgrade recommendations use the latest enabled version)
- Node images: add OCI compute images with display labels, enable/disable, auto-select fallback when no images configured
- Global resource limits: cluster limit, pool max, node max, OCPU, RAM, storage
- All changes take effect immediately — no restart or redeployment needed

### Cluster Templates
- Template table with name, pools, shape, image, K8s version, cost, TTL, protection status, required role, active state
- Add/edit modal with live cost preview
- Enable/disable toggle and permanent delete

### Requests
- Approval queue for user-submitted destroy requests on protection-enabled clusters
- Live count badge in the admin nav (`Requests (N)`) refreshed every 5 seconds; hidden when zero pending
- Filters: Pending / Approved / Denied / All
- Columns: requested-at (relative time), cluster, user, reason, status, reviewer, actions
- **Review** action opens a modal: optional admin note, then **Approve** opens the destroy plan for a second confirmation, or **Deny** (note surfaces on user's cluster card as "Destroy denied")
- Limit-request review lets admins grant requested or adjusted values for cluster count, pools, nodes, OCPU, RAM, and storage; approve/deny results notify the user in Activity
- Row-locked approval prevents double-approval when two admins click simultaneously
- Every submit / approve / deny is audit-logged
- Bypass path: admins can still force-destroy directly via `?force=true` for incident response

### Audit Log
- Append-only record of every deploy, scale, upgrade, and destroy operation, plus `destroy-request:*` and `limit-request:*` events
- Columns: timestamp, user, operation, cluster name, status, duration
- Filterable by user, operation type, and status

---

## Architecture

- **No build step** — vanilla HTML/CSS/JS frontend, served from any static host
- **FastAPI backend** — async Python, SSE streaming, JWT validation via JWKS
- **PostgreSQL** — clusters, jobs, users, config, templates, audit log
- **Terraform execution** — per-job runner with isolated state in OCI Object Storage
- **Helm deployment** — single chart with configurable values for any k3s/K8s environment
- **Single-node ready** — runs on a single OCI ARM VM (Always Free tier compatible)

---

## Testing & CI

- **138 automated tests** — business logic, validation rules, API contracts, access control, lifecycle notifications, and cost estimation
- **Zero external dependencies** — in-memory SQLite with mocked auth; no database server, IdP, or OCI access needed to run tests
- **Coverage areas** — user provisioning, limit resolution, admin config CRUD, cluster templates, cost engine (basic/enhanced tiers, multi-pool, custom pricing)
- **Reference CI pipeline (maintainer-owned)** — GitHub Actions and GitLab CI run validation and publish images for maintainer release flow; customer operators consume published image tags/digests

---

## Deployment options

Two first-class deployment paths, each tuned to its target environment. Both use the same Helm chart and the same upstream container images — the difference is the values file, which configures the right ingress controller, storage class, and scheduling for each platform.

| | **Existing OKE cluster** | **Single-node k3s on OCI VM** |
|---|---|---|
| **Best for** | Production, enterprise OKE users | Dev/test, demos, Always Free tier |
| **Values file** | `values-oke.yaml` | `values-k3s.yaml` |
| **Ingress** | ingress-nginx with OCI flexible Load Balancer (installed separately) | Traefik (bundled with k3s, `className: traefik`, no extra install) |
| **Storage class** | `oci-bv` (OCI Block Volume) | `local-path` (k3s default) |
| **PostgreSQL volume floor** | 50 GB (OCI BV minimum) | 20 GB (local disk) |
| **Image pull policy** | `IfNotPresent` | `Always` |
| **Scheduling** | No tolerations (dedicated workers) | Control-plane tolerations for single-node |
| **Setup time** | ~30 min (cluster exists) | ~15 min |
| **End-to-end guide** | Available during evaluation/POC | Available during evaluation/POC |
| **Validation playbook** | Available during evaluation/POC | Available during evaluation/POC |

- **Any Kubernetes** — the Helm chart also works on any K8s cluster that has an ingress controller and a default StorageClass; use `values.yaml` as a starting point and override for your environment
- **Image delivery with any container registry** — release images can be consumed from GHCR or mirrored into your private registry (OCIR, GitLab CR, Harbor, Docker Hub, etc.). Helm chart supports `imagePullSecrets` for private registries.
- **OCI Marketplace (planned)** — one-click "Launch Stack" deployment from OCI Console is planned for a future release
- **In-cluster access agent (planned)** — outbound agent tunnel for private-by-default environments that do not allow public OKE API endpoints, even CIDR-restricted ones. This removes DRG/LPG pressure after bootstrap and gives Rancher-like kubeconfig behavior.

---

## Marketplace features (planned)

> These features are prepared for a future OCI Marketplace listing. The same deployment capabilities are available today via the Helm chart.

- **Guided deployment form** — OCI Resource Manager schema with dynamic dropdowns for compartment, VCN, subnet, shapes, images, and existing clusters
- **Conditional OKE creation** — create a new OKE cluster with VCN, subnets, and security lists, or deploy into an existing cluster
- **Bring your own identity** — deploy bundled Keycloak or connect to an existing OIDC provider (Keycloak, Azure AD, Okta, Google Workspace)
- **OCI credential validation** — schema validates API key fingerprint format, requires private key PEM, and auto-injects tenancy/user OCID from Resource Manager context
- **Auto-generated passwords** — database and Keycloak passwords auto-generated when not provided, with retrieval instructions in stack outputs
- **Ingress-NGINX with OCI LB** — automatically deploys ingress controller with OCI flexible load balancer annotations
- **Post-deployment guide** — stack outputs include step-by-step instructions for eNS setup, Keycloak config, and first login

---

Built by [Solvia Lab s.r.o.](https://solvialab.tech)
