# Infragate — Features

A complete overview of what Infragate can do. For deployment instructions see [GUIDE.md](./GUIDE.md), for technical architecture see [STACK.md](./STACK.md).

---

## Self-service cluster provisioning

- **One-click deploy** — engineers provision fully managed OKE clusters through a clean web portal, no CLI, Terraform, or OCI Console knowledge required
- **Live Terraform streaming** — `terraform init`, `plan`, and `apply` output streams in real time to the browser during deploy, scale, and destroy operations
- **Template-based or custom** — choose from admin-defined cluster templates or configure everything manually
- **Automatic resource creation** — each cluster gets its own compartment, VCN, subnet, internet gateway, route table, security list, and node pools
- **Bring your own infrastructure** — optionally supply existing VCN, compartment, or subnet OCIDs via the Advanced tab to skip resource creation and wire into existing networks
- **CIDR pool management** — admins pre-populate a pool of /24 ranges; each cluster allocates one on deploy and releases it on destroy
- **Multi-pool support** — configure 1–N node pools per cluster, each with independent node count, OCPU, RAM, and storage sizing
- **Node image selection** — admins configure allowed OCI compute images; users select an image on the deploy form or leave it as auto-select (latest OKE-compatible image). Templates can lock a specific image

---

## Cluster templates

- **Pre-configured profiles** — admins create templates that encode K8s version, VM shape, node image, pool layout, tier, TTL, and destroy protection
- **Deploy form pre-fill + lock** — selecting a template pre-fills and locks resource fields (pools, nodes, CPU, RAM, storage, pool names, and add/remove pool controls). Users can still set cluster name, CIDR, compartment, and advanced overrides. Select "Custom" to unlock all fields and configure manually with limit enforcement
- **Template values can exceed user limits** — templates represent admin-pre-approved configurations, so template-defined values are not clamped to the user's personal limits. The lock prevents users from editing these values
- **Destroy protection** — template policy that prevents users from destroying clusters without admin override (`?force=true`)
- **Time-to-live (TTL)** — optional expiry in hours, enforced at deploy time; also available on custom deploys without a template
- **Live cost preview** — add/edit modal shows estimated monthly and hourly cost that updates as you change pools, shape, or tier
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
| Dashboard cluster cards | Estimated monthly cost per cluster |
| Cluster detail page | Monthly cost + full breakdown (per-pool cost, control plane cost, total with hourly rate) |
| Admin — All Clusters table | Monthly + hourly cost per cluster |
| Admin — Stats bar | Total estimated monthly spend across all active clusters |
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
- **Pool add/remove** — add new node pools or remove existing ones directly from the scale modal, no redeployment needed
- **Per-pool control** — scale each pool independently
- **Change preview** — review all changes before applying (current vs. new values, new pools highlighted, removed pools shown)
- **Enhanced tier** — full in-place scaling via OKE API, no manual node cycling
- **Basic tier** — node count scaling is automatic; existing node config changes require manual OCI Console cycling (portal warns users)

---

## Cluster lifecycle

- **Status tracking** — real-time status across all views: provisioning, running, scaling, destroying, error, destroyed
- **TTL visibility** — dashboard cards show color-coded countdown badges (green >24h, orange <24h, red <4h) for clusters with TTL. Detail page shows full expiry timestamp and remaining time
- **Destroy protection** — protected clusters show a red "Protected" badge on dashboard cards and detail page. Non-admin users cannot destroy protected clusters (button disabled + 403 from API). Admins see a "Force Destroy" option that overrides protection via `?force=true`
- **Destroy with cleanup** — `terraform destroy` removes all OCI resources; CIDR returned to pool for reuse
- **Error recovery** — failed deployments show troubleshooting tips and a "Clean up" button to remove partial resources
- **Kubeconfig download** — available on the detail page once the cluster is running; uses OCI CLI exec plugin
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

Five dedicated admin pages accessible to users with the `admin` role:

### All Clusters
- Every cluster across all users with status, owner, CIDR, K8s version, tier, resources, cost, and age
- Stats bar: total clusters, online, provisioning, CIDRs used, estimated monthly spend
- Actions: detail view, scale, destroy, new cluster (bypasses user quotas)

### Users & Limits
- All users with cluster count, limit, and per-user override badges
- Edit Limits modal: set per-user overrides for any combination of limits and tier
- Reset to global defaults with one click

### Configuration
- Platform-wide settings: region, compartment, cluster tier, state bucket, namespace
- CIDR pool: add/remove /24 ranges with allocation status
- VM shapes: add shapes with optional display labels
- K8s versions: manage available versions
- Node images: add OCI compute images with display labels, enable/disable, auto-select fallback when no images configured
- Global resource limits: cluster limit, pool max, node max, OCPU, RAM, storage
- All changes take effect immediately — no restart or redeployment needed

### Cluster Templates
- Template table with name, pools, shape, image, K8s version, cost, TTL, protection status, required role, active state
- Add/edit modal with live cost preview
- Enable/disable toggle and permanent delete

### Audit Log
- Append-only record of every deploy, scale, and destroy operation
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

- **87 automated tests** — business logic, validation rules, API contracts, access control, and cost estimation
- **Zero external dependencies** — in-memory SQLite with mocked auth; no database server, IdP, or OCI access needed to run tests
- **Coverage areas** — user provisioning, limit resolution, admin config CRUD, cluster templates, cost engine (basic/enhanced tiers, multi-pool, custom pricing)
- **CI pipeline** — GitHub Actions and GitLab CI run the full suite with coverage on every push to `main`/`DEV` and on PRs/MRs to `main`

---

## Deployment options

- **OCI Marketplace — Resource Manager Stack** — one-click "Launch Stack" deployment from OCI Console with a guided form. Provisions OKE cluster, networking, ingress, and Infragate automatically. Supports creating a new cluster or deploying into an existing one
- **OCI Marketplace — Helm Chart** — for advanced users who want to install on their own Kubernetes cluster using Helm
- **Existing OKE cluster** — deploy into an existing Oracle Kubernetes Engine cluster with `values-oke.yaml`. Pre-configured for OCI Block Volume storage, nginx ingress with OCI load balancer, and GHCR images
- **CI/CD with any container registry** — GitHub Actions (GHCR) and GitLab CI (GitLab CR) pipelines included out of the box. Images tagged `dev-latest` for DEV branch, `latest` for main. Helm chart supports `imagePullSecrets` for private registries (GitLab CR, OCIR, Docker Hub, etc.)
- **k3s on OCI** — single VM deployment pulling images from GHCR, with control-plane tolerations for single-node scheduling
- **Any Kubernetes** — Helm chart works on any K8s cluster with ingress
- **Docker Compose** — included for local development with bundled Keycloak + PostgreSQL

---

## Marketplace features

- **Guided deployment form** — OCI Resource Manager schema with dynamic dropdowns for compartment, VCN, subnet, shapes, images, and existing clusters
- **Conditional OKE creation** — create a new OKE cluster with VCN, subnets, and security lists, or deploy into an existing cluster
- **Bring your own identity** — deploy bundled Keycloak or connect to an existing OIDC provider (Keycloak, Azure AD, Okta, Google Workspace)
- **OCI credential validation** — schema validates API key fingerprint format, requires private key PEM, and auto-injects tenancy/user OCID from Resource Manager context
- **Auto-generated passwords** — database and Keycloak passwords auto-generated when not provided, with retrieval instructions in stack outputs
- **Ingress-NGINX with OCI LB** — automatically deploys ingress controller with OCI flexible load balancer annotations
- **Post-deployment guide** — stack outputs include step-by-step instructions for DNS setup, Keycloak config, and first login

---

Built by [Solvia Lab s.r.o.](https://solvialab.tech) · [![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?logo=linkedin&logoColor=white)](https://www.linkedin.com/company/infragate-cloud/)
