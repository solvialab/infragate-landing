# Infragate — Integration & Stack Reference

Technical reference for platform, identity, and infrastructure teams deploying Infragate as a self-hosted OCI-native Internal Developer Platform (IDP). Covers the full stack — frontend, backend, database, Terraform layer, provisioning flow — plus IdP integration, API surface, Helm configuration, deployment paths, and security.

Infragate is designed to integrate with your current tooling rather than replace it.

---

## Table of contents

1. [Prerequisites](#1-prerequisites)
2. [Stack overview](#2-stack-overview)
3. [Frontend](#3-frontend)
4. [Backend](#4-backend)
5. [Database schema](#5-database-schema)
6. [Terraform layer](#6-terraform-layer)
7. [Provisioning flow](#7-provisioning-flow)
8. [Identity provider integration](#8-identity-provider-integration)
9. [Existing OCI infrastructure](#9-existing-oci-infrastructure)
10. [Resource limits and user overrides](#10-resource-limits-and-user-overrides)
11. [Cluster tiers](#11-cluster-tiers)
12. [API reference](#12-api-reference)
13. [Helm configuration](#13-helm-configuration)
14. [Deployment](#14-deployment)
15. [Configuration reference](#15-configuration-reference)
16. [Security considerations](#16-security-considerations)

---

## 1. Prerequisites

| Requirement | Details |
|---|---|
| OCI tenancy | Active tenancy with OKE enabled in at least one region |
| OCI service account | IAM user with API key — see Section 9 |
| OIDC-compliant IdP | Any provider supporting OIDC Authorization Code flow with PKCE |
| Kubernetes cluster | OKE, k3s, or any K8s cluster with Helm 3.x and an ingress controller |
| PostgreSQL 14+ | Bundled via Helm chart, or bring your own managed/self-hosted instance |
| Container registry | GitHub Container Registry (GHCR), GitLab Container Registry, or any OCI-compliant registry. Prebuilt images are published by the reference CI pipelines — see Section 14 |

---

## 2. Stack overview

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML / CSS / JS — single file, no build step |
| Backend | FastAPI (Python 3.12), async, SSE log streaming |
| Database | PostgreSQL 16 — clusters, jobs, users, config, templates, audit log |
| IaC | Terraform 1.7 — OCI provider, per-job execution |
| State backend | OCI Object Storage (S3-compatible API) |
| Auth | OIDC PKCE + Bearer JWT validation (JWKS) |
| Container images | Prebuilt multi-stage images published to GHCR and GitLab Container Registry by the reference CI pipelines |
| CI/CD | GitHub Actions and GitLab CI — pipelines ship with the repository |
| DNS / CDN | Cloudflare (reference setup) |
| Domain | infragate.cloud (Namecheap â†’ Cloudflare NS, reference setup) |

---

## 3. Frontend

Single `index.html` with external `css/` and `js/` assets. No framework, no build step, no runtime dependencies — served from any static host.

**Pages:**
- Landing — product overview, sign in
- Deploy — cluster provisioning form (Standard + Advanced tabs), live Terraform log stream, plan preview
- Dashboard (My Clusters) — cluster cards with status, pool visualisation, tier pill, estimated cost, actions, and status cards (online, provisioning, upgrading k8s, destroying, failed, total)
- Detail — full cluster info, node pools, cost breakdown (per-pool + control plane + total), kubeconfig + SSH key download
- Admin — All clusters (with cost per cluster), status + capacity stats (online, provisioning, upgrading k8s, destroying, failed, total, CIDRs used, monthly spend), Users & limits, Configuration, Cluster templates (with cost per template), Audit log

**Auth flow:** OIDC Authorization Code + PKCE. On login the frontend exchanges the code for a JWT, stores it in memory, and attaches it as `Authorization: Bearer` on every API call.

**Limit enforcement:** On load, `GET /api/v1/users/me` returns the user's resolved effective limits. The deploy form uses these to constrain pool counts, node counts, and compute values. The cluster limit wall is shown if the user is at their limit. Node counts may be set to zero on deploy and scale — useful for pausing compute charges on Basic clusters.

**Config:** `js/config.js` — `API_BASE`, `OIDC_ISSUER`, `OIDC_CLIENT_ID`. Injected at runtime via a Helm ConfigMap when deployed through the chart, so values do not need to be edited for each environment.

---

## 4. Backend

FastAPI application. All endpoints require a valid JWT except the health check.

### Auth (`app/core/auth.py`)

- `get_current_user()` — validates JWT against IdP JWKS, returns decoded claims
- `require_admin()` — additionally asserts `admin` role in the configured roles claim
- JWKS fetched on startup, cached, auto-refreshed on key rotation

### Routers

**`app/routers/users.py`**
- `GET /api/v1/users/me` - auto-provisions user on first login, returns user context + resolved effective limits via `_resolve_limits()`
- `GET /api/v1/users/deploy-options` - deploy form options: CIDRs, shapes, K8s versions, node images, active templates (filtered by user's IdP roles via `required_role`), pricing
- `GET /api/v1/users/deploy-options/k8s-compatible?shape=<shape>` - shape-aware K8s list for deploy/template UI (intersection of admin-enabled versions and versions that have OKE node-image compatibility for the shape in current region)
- `GET /api/v1/users/overrides/compartments` - OCI-backed compartment autocomplete source for Advanced override fields
- `GET /api/v1/users/overrides/vcns?compartment_ocid=<ocid>` - OCI-backed VCN autocomplete source (optionally scoped by compartment)
- `GET /api/v1/users/overrides/subnets?vcn_ocid=<ocid>&compartment_ocid=<ocid>` - OCI-backed subnet autocomplete source (scoped by VCN/compartment when provided)

**`app/routers/clusters.py`**
- `GET /api/v1/clusters` — user's active clusters (includes `estimated_monthly_cost`)
- `POST /api/v1/clusters` — deploy: validates limits, resolves template policies (destroy protection, TTL), allocates CIDR, creates cluster + job records, starts Terraform runner
- `GET /api/v1/clusters/:id` — cluster detail + `ssh_key_available` + `estimated_monthly_cost` + `cost_breakdown`
- `GET /api/v1/clusters/:id/kubeconfig` — stored kubeconfig YAML
- `GET /api/v1/clusters/:id/sshkey` — SSH private key (PEM)
- `POST /api/v1/clusters/:id/scale` — scale node pools (nodes, OCPU, RAM, storage), add or remove pools (both Basic and Enhanced)
- `POST /api/v1/clusters/:id/upgrade` — upgrade cluster Kubernetes version (control plane upgrade; worker node rotation handled separately)
- `DELETE /api/v1/clusters/:id` — destroy (enforces destroy protection — admin + `?force=true` required for protected clusters)
- `GET /api/v1/clusters/admin/:id/kubeconfig` — admin: kubeconfig for any cluster
- `GET /api/v1/clusters/admin/:id/sshkey` — admin: SSH key for any cluster

**`app/routers/jobs.py`**
- `GET /api/v1/jobs/:id/logs` — SSE stream of Terraform stdout, line by line

**`app/routers/admin.py`**
- `GET/PUT /api/v1/admin/config` — platform-wide config
- `GET/PUT /api/v1/admin/config/cluster-type` — tier toggle
- `GET/POST /api/v1/admin/config/cidrs` + `DELETE` + `PATCH` — CIDR pool management
- `GET/POST /api/v1/admin/config/shapes` + `DELETE` + `PATCH` — allowed VM shapes
- `GET/POST /api/v1/admin/config/k8s-versions` + `DELETE` + `PATCH` — allowed Kubernetes versions
- `GET/POST /api/v1/admin/config/images` + `DELETE` + `PATCH` — allowed node images
- `GET /api/v1/admin/clusters` — all clusters across all users (includes `estimated_monthly_cost`)
- `GET /api/v1/admin/users` — all users with overrides + effective limits
- `PATCH /api/v1/admin/users/:id` — set/reset per-user limit overrides
- `GET /api/v1/admin/stats` — platform stats (includes `estimated_monthly_spend`)
- `GET /api/v1/admin/templates` — list all cluster templates (including inactive)
- `POST /api/v1/admin/templates` — create template
- `PATCH /api/v1/admin/templates/:id` — update template
- `DELETE /api/v1/admin/templates/:id` — permanently delete template
- `GET /api/v1/admin/audit` — audit log with filters

### Limit resolution (`app/routers/users.py`)

`_resolve_limits(user, config)` — single source of truth for effective limits. Returns per-user override if set (not `NULL`), otherwise falls back to global config value. Used by both `/users/me` and `/admin/users`.

---

## 5. Database schema

### `users`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `keycloak_id` | String | IdP `sub` claim — unique index |
| `username` | String | |
| `email` | String | |
| `cluster_limit` | Integer | Inherited from global config on creation |
| `is_active` | Boolean | |
| `override_pool_max` | Integer nullable | NULL = inherit global |
| `override_node_max` | Integer nullable | |
| `override_cpu_max` | Integer nullable | |
| `override_ram_max` | Integer nullable | |
| `override_storage_max` | Integer nullable | |
| `override_cluster_type` | String nullable | NULL = inherit global |
| `created_at` | Timestamp | |

### `clusters`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `user_id` | UUID FK â†’ users | |
| `name` | String | e.g. `analytics-cluster` |
| `status` | String | `provisioning` \| `running` \| `error` \| `destroying` \| `destroyed` |
| `cidr` | String | Allocated /24 |
| `vcn_cidr` | String | |
| `k8s_version` | String | |
| `region` | String | |
| `compartment` | String | |
| `compartment_ocid` | String | |
| `cluster_type` | String | `BASIC_CLUSTER` \| `ENHANCED_CLUSTER` |
| `node_shape` | String | |
| `image_id` | String nullable | OCI node image OCID — NULL = auto-select |
| `ocid` | String | OKE cluster OCID |
| `vcn_ocid` | String | |
| `subnet_ocid` | String | |
| `api_endpoint` | String | |
| `kubeconfig` | Text | Stored after provisioning |
| `ssh_private_key` | Text | PEM — stored after provisioning |
| `template_id` | UUID FK â†’ cluster_templates | Template used for provisioning (nullable) |
| `destroy_protection` | Boolean | Inherited from template — prevents user destroy |
| `ttl_hours` | Integer nullable | Time-to-live from template |
| `ttl_expires_at` | Timestamp nullable | Computed at deploy time from TTL |
| `created_at` | Timestamp | |
| `updated_at` | Timestamp | |

### `node_pools`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `cluster_id` | UUID FK â†’ clusters | |
| `name` | String | |
| `nodes` | Integer | `0` valid — pauses compute, preserves pool config |
| `cpu` | Integer | OCPU |
| `ram` | Integer | GB |
| `storage` | Integer | GB |
| `status` | String | |
| `ocid` | String | Node pool OCID |

### `jobs`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `cluster_id` | UUID FK â†’ clusters | |
| `user_id` | UUID FK â†’ users | |
| `operation` | String | `deploy` \| `scale` \| `upgrade` \| `destroy` |
| `status` | String | `running` \| `success` \| `error` |
| `logs` | Text | Accumulated Terraform output |
| `started_at` | Timestamp | |
| `finished_at` | Timestamp | |
| `duration_s` | Float | |

### `cluster_templates`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `name` | String unique | Template display name |
| `description` | Text nullable | Short description shown on deploy form |
| `k8s_version` | String nullable | Pre-selected K8s version |
| `shape` | String nullable | Pre-selected VM shape |
| `image` | String nullable | Pre-selected OCI node image OCID |
| `pools` | JSON | `[{name, nodes, cpu, ram, storage}]` |
| `tier_default` | String nullable | `BASIC_CLUSTER` \| `ENHANCED_CLUSTER` \| null |
| `ttl_hours` | Integer nullable | Time-to-live for clusters created from this template |
| `destroy_protection` | Boolean | Inherited by clusters — prevents user destroy |
| `required_role` | String nullable | IdP role required to see/use this template |
| `is_active` | Boolean | `false` = soft-deleted, hidden from deploy form |
| `sort_order` | Integer | Controls display position (lower = first) |
| `created_at` | Timestamp | |
| `updated_at` | Timestamp | |

### `config` (single row)
Global platform configuration — region, compartment, limits, CIDR pool, allowed shapes/versions/images, cluster tier, pricing overrides.

The `pricing` JSON column stores optional rate overrides for cost estimation. When empty, OCI PAYG defaults are used (OCPU: $0.025/hr, RAM: $0.0015/GB/hr, Storage: $0.0255/GB/mo, Enhanced CP: $0.10/hr). Admins can override rates for custom contracts.

### `audit_log`
Append-only record of every operation — user, operation, cluster name, status, duration.

---

## 6. Terraform layer

### Module (`terraform/module/`)

Reusable OKE module. Called once per cluster provisioning job.

**Resources created:**
- `tls_private_key` — RSA 4096-bit SSH key for worker node access
- `oci_identity_compartment` — isolated per cluster
- `oci_core_vcn`
- `oci_core_subnet`
- `oci_core_internet_gateway`
- `oci_core_route_table`
- `oci_core_security_list` — OKE-required ports
- `oci_containerengine_cluster` — `type = var.cluster_type` (`BASIC_CLUSTER` or `ENHANCED_CLUSTER`)
- `oci_containerengine_node_pool` — one per pool (1–3 pools by default), SSH public key injected

**Data sources:**
- `oci_identity_availability_domains` — region AD discovery
- `oci_containerengine_node_pool_option` — latest OL8 image lookup (fallback when no `image_id` configured)
- `oci_containerengine_cluster_kube_config` — kubeconfig YAML (uses OCI CLI token helper)

**Architecture-aware image selection:** When `image_id` is empty, the module filters available OKE images by the requested shape's architecture — ARM shapes (`VM.Standard.A*`) map to `aarch64` images, GPU shapes map to `Gen2-GPU` variants, and all other shapes map to plain x86_64. This prevents mismatches that caused nodes to fail to launch.

**Key variables:**
- `cluster_type` — `BASIC_CLUSTER` or `ENHANCED_CLUSTER`, set from platform config
- `image_id` — OCI node image OCID; empty string = auto-select latest matching the shape architecture
- `pool_count`, `node_count`, `node_shape`, `ocpu_count`, `memory_in_gbs` per pool
- `vcn_ocid`, `compartment_ocid`, `subnet_ocid` — optional overrides for existing infrastructure

When override OCIDs are provided, the corresponding resources are skipped and the module wires up to the existing infrastructure instead.

### Runner template (`terraform/runner-template/`)

A thin wrapper module copied per provisioning job. The backend copies this template to `terraform/runner/{job_id}/`, writes `job.tfvars`, then executes `terraform init && terraform plan && terraform apply`.

**State path:** `{user_id}/{cluster_id}/terraform.tfstate` in the `infragate-tfstate` OCI Object Storage bucket — enables safe concurrent operations and clean destroy.

### Terraform service (`backend/app/services/terraform.py`)

FastAPI â†” Terraform bridge:
- Copies runner template to `terraform/runner/{job_id}/`
- Writes `job.tfvars` with cluster parameters + OCI credentials
- Runs `terraform init â†’ plan â†’ apply` as an async subprocess
- Streams stdout line-by-line via SSE to the frontend log viewer
- Parses `terraform output -json` — stores OCIDs, API endpoint, kubeconfig, SSH private key in the database
- Updates cluster status: `provisioning` â†’ `running` (or `error`)
- On destroy: releases CIDR back to pool, cleans up runner directory
- On scale: updates pool records in DB, rewrites tfvars, re-applies

---

## 7. Provisioning flow

```
User (browser)
    â”‚
    â”‚  OIDC login (PKCE) — via your existing IdP
    â–¼
Your Identity Provider  â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â–º JWT issued
    â”‚
    â”‚  POST /api/v1/clusters  (Bearer JWT)
    â–¼
Infragate API (FastAPI)
    â”‚  1. Validate JWT against IdP JWKS
    â”‚  2. Resolve effective limits (per-user override or global config)
    â”‚  3. Check cluster limit
    â”‚  4. Allocate CIDR from pool
    â”‚  5. Write cluster record (status: provisioning)
    â”‚  6. Write job record (operation: deploy)
    â”‚  7. Copy runner-template â†’ terraform/runner/{job_id}/
    â”‚  8. Write job.tfvars (cluster params + cluster_type)
    â”‚  9. terraform init + plan + apply (subprocess)
    â”‚  10. Stream stdout line-by-line via SSE â†’ browser log viewer
    â”‚  11. Parse outputs â†’ store OCIDs, endpoint, kubeconfig, SSH key in DB
    â”‚  12. Update cluster status: running
    â–¼
OCI
    â”œâ”€â”€ oci_identity_compartment
    â”œâ”€â”€ oci_core_vcn
    â”œâ”€â”€ oci_core_subnet
    â”œâ”€â”€ oci_core_internet_gateway
    â”œâ”€â”€ oci_core_route_table
    â”œâ”€â”€ oci_core_security_list
    â”œâ”€â”€ oci_containerengine_cluster   (BASIC or ENHANCED)
    â””â”€â”€ oci_containerengine_node_pool (per pool)
```

---

## 8. Identity provider integration

Infragate authenticates users via standard **OpenID Connect (PKCE)** and validates **Bearer JWTs** on every API request. It does not ship its own user directory and does not require a specific identity provider.

### 8.1 How it works

1. The frontend redirects the user to your IdP's authorization endpoint
2. After login, the IdP issues a JWT access token
3. The frontend sends that token as `Authorization: Bearer <token>` on every API call
4. Infragate validates the token against your IdP's JWKS endpoint
5. On first login, Infragate auto-provisions the user in its database using the `sub` claim as the unique identifier

No user migration or directory sync is required.

### 8.2 Required JWT claims

| Claim | Type | Description |
|---|---|---|
| `sub` | string | Unique user identifier — used as the primary key |
| `preferred_username` | string | Display name shown in the UI |
| `email` | string | User email address |
| `realm_access.roles` | string[] | Role list — `admin` grants admin panel access; custom roles gate cluster template visibility (see below) |

> **Note on role claim path:** `realm_access.roles` is the Keycloak default. If your IdP uses a different claim path (e.g. `roles`, `groups`, or a custom claim), set `OIDC_ROLES_CLAIM` in the environment configuration to match.

#### Roles used by Infragate

| Role | Purpose | Required |
|---|---|---|
| `admin` | Grants access to the admin panel (all clusters, users, config, templates, audit log) | At least one user |
| Custom roles (e.g. `testing`, `uat`, `production`) | Gate which cluster templates a user can see and deploy. Admins set `required_role` on each template; only users with that role see the template on the deploy form | Optional |

A typical enterprise setup creates roles matching environment tiers — `testing`, `uat`, `production` — so admins can offer DEV/TEST/UAT/PROD cluster templates with progressively restricted access. Users without any custom role can still deploy using templates that have no `required_role` set, or use the "Custom" option if enabled. See [GUIDE.md — Environment-tier gating](./GUIDE.md#environment-tier-gating-with-roles) for a full walkthrough.

### 8.3 IdP client configuration

| Setting | Value |
|---|---|
| Client type | Public (no client secret — PKCE only) |
| Grant type | Authorization Code |
| PKCE method | S256 |
| Redirect URI | `https://infragate.your-domain.com` |
| Logout redirect URI | `https://infragate.your-domain.com` |
| Access token format | JWT |

### 8.4 Provider-specific notes

**Keycloak**
- Create a realm (e.g. `infragate`) or use an existing one
- Create a client with Client Authentication set to **Off** (public client)
- Assign users the `admin` realm role to grant admin panel access
- Set `OIDC_ISSUER=https://keycloak.your-domain.com/realms/{realm-name}`

**Azure Active Directory**
- Register an application in Azure AD with a public client redirect URI
- Enable the `openid`, `profile`, and `email` scopes
- Add app roles: `admin` and `user` — assign via Azure AD group membership
- Set `OIDC_ROLES_CLAIM=roles`
- Set `OIDC_ISSUER=https://login.microsoftonline.com/{tenant-id}/v2.0`

**Okta**
- Create an OIDC Web Application (set to public / PKCE)
- Add a `Groups` claim to the access token under Authorization Server â†’ Claims
- Set `OIDC_ROLES_CLAIM=groups`
- Set `OIDC_ISSUER=https://your-org.okta.com/oauth2/default`

**Google Workspace**
- Create an OAuth 2.0 client ID (Web application) in Google Cloud Console
- Add the authorized redirect URI
- Google does not support custom roles in the access token — use the `ADMIN_EMAILS` fallback (see Section 15)
- Set `OIDC_ISSUER=https://accounts.google.com`

---

## 9. Existing OCI infrastructure

### 9.1 Required OCI service account

**IAM user:** `infragate-svc` (or any name you prefer)

**API key:** Generate an RSA key pair. Store the private key securely — see Section 16.

**Group and policy:**

```
Allow group <infragate-group-name> to manage cluster-family in compartment <your-compartment>
Allow group <infragate-group-name> to manage virtual-network-family in compartment <your-compartment>
Allow group <infragate-group-name> to manage instance-family in compartment <your-compartment>
Allow group <infragate-group-name> to manage compartments in compartment <your-compartment>
Allow group <infragate-group-name> to manage object-family in compartment <your-compartment>
Allow group <infragate-group-name> to read all-resources in tenancy
Allow group <infragate-group-name> to read tenancies in tenancy
Allow group <infragate-group-name> to inspect compartments in tenancy
Allow group <infragate-group-name> to use tag-namespaces in tenancy
```

**Required root-level OKE service policy (separate policy):**

```
Allow service OKE to manage all-resources in tenancy
```

Without this OKE service policy, API calls such as node pool options lookups can fail with `RelatedResourceNotAuthorizedOrNotFound` / "Unable to retrieve required tenancy details", even when the `infragate-svc` user policy is otherwise correct.

If your tenancy prefers principal-bound policies over group policies, you can use equivalent `any-user where request.user.id = '<infragate-svc-user-ocid>'` statements.

**Object Storage bucket:** Create a bucket for Terraform state. Enable versioning. Note the bucket name and your Object Storage namespace (found under Tenancy details in the OCI Console).

**Customer Secret Key:** Generate an S3-compatible key on the `infragate-svc` user for Terraform state access.

### 9.2 Using existing VCNs, compartments, or subnets

Users can supply existing OCIDs via the **Advanced** tab in the deploy form:

| Field | Behaviour when set |
|---|---|
| Existing VCN OCID | Infragate uses this VCN, skips creating VCN + IGW + route table |
| Existing Compartment OCID | Cluster resources placed in this compartment, skips compartment creation |
| Existing Subnet OCID | Worker nodes placed in this subnet, skips subnet + security list creation |

Any combination is supported. When using an existing subnet, verify CIDR ranges do not overlap with existing VCN allocations.

Important behavior: setting only `Existing Compartment OCID` does not trigger VCN/subnet auto-discovery. If `Existing VCN OCID` and `Existing Subnet OCID` are blank, Terraform creates a new VCN/subnets/network stack in that compartment.

> Shared compartment guidance: for multiple clusters in one compartment, provide a dedicated BYO `existing_compartment_ocid` for all of them. Avoid reusing a compartment that was auto-created for a different cluster.

> Note: If you deploy with existing VCN/subnet overrides, Infragate does not manage route tables for those existing resources. The existing network must already provide equivalent private-subnet egress required for OKE worker node registration: route `0.0.0.0/0` to a NAT Gateway (or equivalent corporate egress path) and route `all-<region>-services-in-oracle-services-network` to a Service Gateway. This is a routing requirement, not an "open ingress" security rule.

> âš ï¸ **Terraform drift.** Resources **created by Infragate** (VCN, IGW, route table, subnet, security list) are managed by the cluster's Terraform state. Manual changes in the OCI Console — extra ingress/egress rules, additional route entries, CIDR changes, added subnets — will be **overwritten on the next apply** (scale, template change, destroy). If you need external control or a custom security posture, **bring your own VCN or subnet** via the Advanced tab instead — Infragate references those resources as data sources and **never modifies them**.

### 9.3 CIDR pool management

Infragate manages a pool of /24 CIDR ranges allocated on cluster creation and released on destroy. Pre-populate the pool to reflect your allocations:

```bash
curl -X POST https://infragate.your-domain.com/api/v1/admin/config/cidrs \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "cidr": "10.120.10.0/24" }'
```

---

## 10. Resource limits and user overrides

Infragate has a two-tier limit system. Global config sets the baseline for all users. Per-user overrides take precedence when set.

### 10.1 Global limits

Set via the admin panel or `PUT /api/v1/admin/config`:

| Limit | Default | Description |
|---|---|---|
| `cluster_limit` | 1 | Max active clusters per user |
| `pool_max` | 3 | Max node pools per cluster |
| `node_max` | 3 | Max nodes per pool |
| `cpu_max` | 1 | Max OCPU per node |
| `ram_max` | 12 | Max RAM per node (GB) |
| `storage_max` | 50 | Max boot volume per node (GB) — minimum 50 (OCI hard floor) |
| `cluster_type` | BASIC_CLUSTER | Cluster tier for all new clusters |

Node counts accept `0` on both deploy and scale — lets users keep a cluster around without running compute. Basic control planes stay free, so a zero-node Basic cluster has no OCI compute charges.

### 10.2 Per-user overrides

Admins can override any limit for a specific user via the admin panel or API. When an override is set it takes precedence over the global value. `null` means the user inherits the global default.

```bash
# Give a specific user more resources
PATCH /api/v1/admin/users/:id
{
  "cluster_limit": 5,
  "override_pool_max": 5,
  "override_node_max": 10,
  "override_ram_max": 64,
  "override_cluster_type": "ENHANCED_CLUSTER"
}

# Upgrade a single user to Enhanced while keeping everyone else on Basic
PATCH /api/v1/admin/users/:id
{ "override_cluster_type": "ENHANCED_CLUSTER" }

# Reset all overrides — user reverts to global defaults
PATCH /api/v1/admin/users/:id
{ "reset_overrides": true }
```

### 10.3 Effective limits

`GET /api/v1/users/me` returns the resolved effective limits — the values actually enforced for that user. The frontend uses these to set deploy form constraints. Changes take effect on next page load or login.

```json
{
  "cluster_limit": 3,
  "effective_pool_max": 5,
  "effective_node_max": 10,
  "effective_cpu_max": 1,
  "effective_ram_max": 64,
  "effective_storage_max": 50,
  "effective_cluster_type": "ENHANCED_CLUSTER"
}
```

---

## 11. Cluster tiers

| | Basic (`BASIC_CLUSTER`) | Enhanced (`ENHANCED_CLUSTER`) |
|---|---|---|
| Cost | Free | ~$0.10/hr per cluster |
| Node scaling via API | âš ï¸ Partial (node/pool count changes are automatic; resource-profile changes need manual cycling) | âœ… Full in-place scaling |
| Autoscaler support | âŒ | âœ… |

**Basic cluster scaling:** Infragate applies node/pool count changes automatically in OKE (no manual rotation required). Manual node cycling is required only for per-node resource changes (shape, OCPU, RAM, storage):

**OKE â†’ Cluster â†’ Node Pool â†’ Nodes â†’ Cordon & drain â†’ Terminate**

The UI shows a warning when scaling a Basic cluster. The API returns `400` if an unsupported scale operation is attempted on a Basic cluster programmatically.

**Kubernetes version upgrades:** `POST /api/v1/clusters/:id/upgrade` upgrades cluster control plane version and updates node-pool target Kubernetes/image configuration. Existing workers are not force-rotated by this action:
- Direct upgrades are limited to the same or next minor version (for example `1.33.x -> 1.34.x`; not `1.33.x -> 1.35.x`).
- Basic: perform rolling worker refresh with scaling (for each pool, scale `N -> 2N -> N` after upgrade so new nodes come up on the new target version/image; Upgrade modal shows the exact live values).
- Enhanced: perform rolling node replacement to move workers to the new version with minimal downtime.

**Switching tiers:** `PUT /api/v1/admin/config/cluster-type` or the admin panel toggle. Applies to all new clusters — existing clusters retain the tier they were provisioned with.

**Per-user override:** `override_cluster_type` on `PATCH /api/v1/admin/users/:id` — gives a specific user a different tier than the platform default.

---

## 12. API reference

Every action available in the UI is also available via REST.

### 12.1 Authentication

```bash
curl https://infragate.your-domain.com/api/v1/clusters \
  -H "Authorization: Bearer $TOKEN"
```

### 12.2 Deploying a cluster from CI/CD

```bash
RESPONSE=$(curl -s -X POST https://infragate.your-domain.com/api/v1/clusters \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "staging",
    "cidr": "10.120.10.0/24",
    "k8s_version": "1.32",
    "shape": "VM.Standard.E4.Flex",
    "pools": [
      { "name": "staging-pool-1", "nodes": 2, "cpu": 1, "ram": 12, "storage": 50 }
    ]
  }')

JOB_ID=$(echo $RESPONSE | jq -r .job_id)
STREAM_URL=$(echo $RESPONSE | jq -r .stream_url)

# Stream provisioning logs
curl -N -H "Authorization: Bearer $TOKEN" \
  https://infragate.your-domain.com$STREAM_URL
```

### 12.3 Endpoint reference

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/v1/users/me` | User | User context + effective limits |
| GET | `/api/v1/users/deploy-options` | User | Deploy form options: CIDRs, shapes, K8s versions, node images, active templates (filtered by user's IdP roles vs template `required_role`), pricing |
| GET | `/api/v1/users/deploy-options/k8s-compatible?shape=<shape>` | User | Deploy/template K8s versions compatible with selected shape in current region and enabled by admin (node-image-backed compatibility) |
| GET | `/api/v1/users/overrides/compartments` | User | Advanced-tab compartment autocomplete options from OCI |
| GET | `/api/v1/users/overrides/vcns?compartment_ocid=<ocid>` | User | Advanced-tab VCN autocomplete options from OCI (optionally scoped by compartment) |
| GET | `/api/v1/users/overrides/subnets?vcn_ocid=<ocid>&compartment_ocid=<ocid>` | User | Advanced-tab subnet autocomplete options from OCI (optionally scoped by VCN/compartment) |
| GET | `/api/v1/clusters` | User | List user's clusters (includes `estimated_monthly_cost`) |
| POST | `/api/v1/clusters` | User | Deploy cluster — optional `template_id` for template-based deploy (server validates resource values match template definition) — returns job_id + SSE stream URL |
| GET | `/api/v1/clusters/:id` | User | Cluster detail, OCIDs, API endpoint, tier, ssh_key_available, `estimated_monthly_cost`, `cost_breakdown` |
| GET | `/api/v1/clusters/:id/kubeconfig` | User | Kubeconfig YAML download |
| GET | `/api/v1/clusters/:id/sshkey` | User | SSH private key download (.pem) |
| POST | `/api/v1/clusters/:id/scale` | User | Scale node pools (nodes, OCPU, RAM, storage), add or remove pools (both Basic and Enhanced) |
| POST | `/api/v1/clusters/:id/upgrade` | User | Upgrade Kubernetes version for a running cluster |
| DELETE | `/api/v1/clusters/:id` | User | Destroy cluster (`?force=true` for protected clusters, admin only) |
| GET | `/api/v1/jobs/:id/logs` | User | SSE log stream |
| GET | `/api/v1/admin/config` | Admin | Platform configuration |
| PUT | `/api/v1/admin/config` | Admin | Update configuration |
| GET | `/api/v1/admin/config/cluster-type` | Admin | Current cluster tier |
| PUT | `/api/v1/admin/config/cluster-type` | Admin | Set cluster tier |
| GET | `/api/v1/admin/config/cidrs` | Admin | CIDR pool |
| POST | `/api/v1/admin/config/cidrs` | Admin | Add CIDR to pool |
| DELETE | `/api/v1/admin/config/cidrs/:cidr` | Admin | Remove CIDR from pool |
| PATCH | `/api/v1/admin/config/cidrs/:cidr` | Admin | Toggle CIDR enabled state |
| GET | `/api/v1/admin/config/shapes` | Admin | Allowed VM shapes |
| POST | `/api/v1/admin/config/shapes` | Admin | Add VM shape |
| POST | `/api/v1/admin/config/shapes/sync` | Admin | Sync allowed VM shapes from live OCI OKE options (merge, preserve labels/toggles) |
| DELETE | `/api/v1/admin/config/shapes/:name` | Admin | Remove VM shape |
| PATCH | `/api/v1/admin/config/shapes/:name` | Admin | Toggle shape enabled state |
| GET | `/api/v1/admin/oci/node-shapes` | Admin | Live OKE-compatible node shapes from OCI |
| GET | `/api/v1/admin/config/k8s-versions` | Admin | Allowed Kubernetes versions |
| POST | `/api/v1/admin/config/k8s-versions` | Admin | Add K8s version |
| DELETE | `/api/v1/admin/config/k8s-versions/:version` | Admin | Remove K8s version |
| PATCH | `/api/v1/admin/config/k8s-versions/:version` | Admin | Toggle K8s version enabled state |
| GET | `/api/v1/admin/config/images` | Admin | Allowed node images |
| POST | `/api/v1/admin/config/images` | Admin | Add node image |
| DELETE | `/api/v1/admin/config/images/:image_id` | Admin | Remove node image |
| PATCH | `/api/v1/admin/config/images/:image_id` | Admin | Toggle node image enabled state |
| GET | `/api/v1/admin/clusters` | Admin | All clusters across all users (includes `estimated_monthly_cost`) |
| GET | `/api/v1/clusters/admin/:id/kubeconfig` | Admin | Kubeconfig for any cluster |
| GET | `/api/v1/clusters/admin/:id/sshkey` | Admin | SSH key for any cluster |
| GET | `/api/v1/admin/users` | Admin | All users with overrides + effective limits |
| PATCH | `/api/v1/admin/users/:id` | Admin | Update user limits and overrides |
| GET | `/api/v1/admin/stats` | Admin | Platform stats (includes `estimated_monthly_spend`) |
| GET | `/api/v1/admin/templates` | Admin | List all cluster templates (including inactive) |
| POST | `/api/v1/admin/templates` | Admin | Create cluster template |
| PATCH | `/api/v1/admin/templates/:id` | Admin | Update cluster template |
| DELETE | `/api/v1/admin/templates/:id` | Admin | Permanently delete cluster template |
| GET | `/api/v1/admin/audit` | Admin | Audit log with filters |

Full OpenAPI documentation is available at `/docs` on any running Infragate instance.

---

## 13. Helm configuration

Infragate ships four Helm values files, each serving a distinct role in the deployment pipeline:

| File | Role | Committed to repo | When to use |
|---|---|---|---|
| `values.yaml` | Chart defaults | âœ… Yes | Never used directly — Helm reads it automatically as the base layer |
| `values-oci.yaml` | Single-node k3s template | âœ… Yes | Copy this to create your own `values-dev.yaml` for k3s deployments |
| `values-oke.yaml` | Existing OKE cluster | âœ… Yes | Pass to `helm upgrade` with `-f` when deploying to an existing OKE cluster |
| `values-dev.yaml` | Your deployment values | âœ… Yes | Pass to `helm upgrade` with `-f` for your environment |

### `values.yaml` — Chart defaults

The canonical reference for every configurable field. Contains all parameters with placeholder values, inline documentation, and sensible defaults. Helm merges this automatically before any `-f` overrides. Operators should read this file to understand the full set of available options, but never edit it directly for a specific deployment.

### `values-oci.yaml` — k3s deployment template

A production-ready starting point for single-node OCI k3s deployments. Pre-configured with:

- GHCR image references (`ghcr.io/solvialab/infragate-api`, `ghcr.io/solvialab/infragate-frontend`) — swap to GitLab Container Registry by changing `image.repository` if you build through GitLab CI instead
- SSE-optimised nginx proxy timeouts for Terraform log streaming
- Control-plane tolerations for single-node scheduling
- TLS disabled by default (enabled after cert-manager setup)

To deploy on k3s:

```bash
cp deploy/helm/values-oci.yaml deploy/helm/values-dev.yaml
# Edit values-dev.yaml with your domain, TLS settings, etc.
```

### `values-oke.yaml` — Existing OKE cluster

A production-ready values file for deploying Infragate into an existing OKE cluster. Key differences from `values-oci.yaml`:

- OCI Block Volume storage class (`oci-bv`) for PostgreSQL persistence
- `pullPolicy: IfNotPresent` (OKE nodes pull from GHCR or GitLab CR directly)
- No control-plane tolerations (OKE worker nodes accept all pods)
- nginx ingress class with OCI load balancer annotations

Use this directly — no copy needed:

```bash
helm upgrade --install infragate deploy/helm/ -n infragate \
  -f deploy/helm/values-oke.yaml \
  --set global.domain=infragate.example.com \
  --set postgresql.auth.password=YOUR_DB_PASSWORD \
  ...
```

### `values-dev.yaml` — Your deployment values

Your environment-specific configuration. Created by copying `values-oci.yaml` and customising it with your domain, TLS settings, and any environment-specific overrides.

Sensitive values (database passwords, OCI credentials, S3 keys) are never stored in values files. Pass them at deploy time via `--set`:

```bash
helm upgrade --install infragate ./deploy/helm \
  -n infragate \
  -f deploy/helm/values-dev.yaml \
  --set postgresql.auth.password=<DB_PASSWORD> \
  --set api.oci.tenancyOcid=ocid1.tenancy... \
  ...
```

### Layering order

Helm merges values files in order, with later files overriding earlier ones:

```
values.yaml (chart defaults)
  â””â”€â”€ values-dev.yaml (-f override)
        â””â”€â”€ --set flags (highest priority)
```

This separation ensures the chart ships with complete, documented defaults while each deployment environment maintains its own configuration without modifying committed files.

### Private registry pull secrets

For private GHCR or GitLab Container Registry images, create a pull secret in the target namespace and reference it through `imagePullSecrets` in your values override:

```bash
kubectl create secret docker-registry infragate-pull \
  --docker-server=ghcr.io \
  --docker-username=<github-user> \
  --docker-password=<github-pat> \
  --namespace infragate
```

```yaml
imagePullSecrets:
  - name: infragate-pull
```

Substitute `registry.gitlab.com`, your GitLab username, and a deploy token when pulling from GitLab Container Registry.

---

## 14. Deployment

Infragate supports three deployment paths. Choose the one that fits your environment:

| Path | Best for | Guide |
|---|---|---|
| **Existing OKE cluster** | Teams with an existing Kubernetes cluster on OCI (recommended for production) | [GUIDE.md — OKE](./GUIDE.md#existing-oke-cluster-deployment) |
| **Single-node k3s** | Dev/test, Always Free tier, single-VM setups | [GUIDE.md — k3s](./GUIDE.md#single-node-k3s-deployment) |
| **OCI Marketplace (planned)** | One-click deployment via Resource Manager — coming in a future release | — |

### 14.1 OKE vs k3s — full differences

Both paths deploy the same application from the same Helm chart. The values files differ only where the runtime platform genuinely differs:

| Concern | OKE (existing cluster) | k3s (single-node VM) |
|---|---|---|
| Values file | `deploy/helm/values-oke.yaml` | `deploy/helm/values-oci.yaml` |
| Ingress controller | **ingress-nginx** installed separately with OCI flexible Load Balancer annotations | **Traefik** bundled with k3s (ships enabled, no extra install) |
| Chart `ingress.className` | `nginx` | `traefik` |
| Load balancer | OCI Flexible LB provisioned by the `ingress-nginx` Service (type `LoadBalancer`) | k3s `klipper-lb` binds directly to the VM's ports 80/443 |
| TLS | cert-manager + ClusterIssuer (Let's Encrypt or corporate CA) | cert-manager on a single-node VM, or terminate TLS at Cloudflare |
| Storage class | `oci-bv` (OCI Block Volume) | `local-path` (k3s default, node-local disk) |
| PostgreSQL volume floor | 50 GB (OCI BV minimum) | 20 GB typical |
| Image pull policy | `IfNotPresent` (nodes pull once, re-use across restarts) | `Always` (single VM, easier to roll latest tag) |
| Scheduling | No tolerations — OKE has dedicated worker nodes | Control-plane tolerations — pods must tolerate the single-node taint |
| OS firewall | Managed by OCI (security lists / NSGs on the cluster subnets) | Managed on the VM itself (`firewalld` / `ufw`) — must open 80, 443, 6443, plus trust the CNI interfaces (`cni0`, `flannel.1`) |
| Horizontal scale | OKE can scale worker pool via the OCI Console or autoscaler | Single node — scale up = bigger VM shape |
| Testing playbook | [TESTING.md — Path B](./TESTING.md#3-path-b-existing-oke-cluster) | [TESTING.md — Path A](./TESTING.md#2-path-a-k3s-on-oci-vm) |

> **Important:** Do not cross the ingress choice. ingress-nginx on k3s means disabling bundled Traefik and patching the controller for host networking; Traefik on OKE means writing `IngressRoute` CRDs and losing the nginx-based SSE proxy config shipped with the chart. Keep each path on its native controller.

### 14.2 Values files

| Values file | Target | Committed to repo |
|---|---|---|
| `values.yaml` | Chart defaults — every field documented, used as base layer | âœ… |
| `values-oke.yaml` | Existing OKE cluster — OCI Block Volume storage, nginx ingress, no tolerations | âœ… |
| `values-oci.yaml` | Single-node k3s — Traefik ingress, local-path storage, control-plane tolerations | âœ… |
| `values-dev.yaml` | Your environment-specific copy — customise domain, TLS, image tags | âœ… (template) |

### 14.3 Container images

Container images are built by CI and pushed to your container registry on every commit (`dev-latest` / `dev-<sha>` for DEV, `latest` / `<sha>` for main). CI pipelines are included for both **GitHub Actions (GHCR)** at `.github/workflows/ci.yml` and **GitLab CI (GitLab Container Registry)** at `.gitlab-ci.yml`. The Helm chart supports `imagePullSecrets` for private registries. No local builds are required for operators — pull the published images directly.

### 14.4 Environment variables

When deploying via Helm, all environment variables below are set automatically through the `values-*.yaml` files and `--set` flags. This reference is for teams integrating Infragate into custom deployment pipelines or running outside of Helm.

```env
# OIDC — point at your existing IdP
OIDC_ISSUER=https://your-idp.example.com/realms/your-realm
OIDC_CLIENT_ID=infragate-portal
OIDC_ROLES_CLAIM=realm_access.roles    # adjust for your IdP

# Database
DATABASE_URL=postgresql://user:password@host:5432/infragate

# OCI
OCI_TENANCY_OCID=ocid1.tenancy.oc1..
OCI_USER_OCID=ocid1.user.oc1..
OCI_FINGERPRINT=xx:xx:xx:..
OCI_REGION=eu-frankfurt-1
OCI_PRIVATE_KEY_PATH=/etc/infragate/oci_api_key.pem
OCI_NAMESPACE=your-object-storage-namespace
OCI_PARENT_COMPARTMENT_OCID=ocid1.compartment.oc1..

# Terraform state (OCI Object Storage S3-compatible)
TF_STATE_BUCKET=infragate-tfstate
TF_STATE_ENDPOINT=https://{namespace}.compat.objectstorage.{region}.oraclecloud.com
OCI_S3_ACCESS_KEY=your-customer-secret-access-key
OCI_S3_SECRET_KEY=your-customer-secret-secret-key

# App
APP_ENV=production

# Optional — Google Workspace admin fallback
ADMIN_EMAILS=you@yourcompany.com,colleague@yourcompany.com
```

### 14.5 OCI private key

Mount the `infragate-svc` private key into the container at `OCI_PRIVATE_KEY_PATH`. On Kubernetes:

```bash
kubectl create secret generic infragate-oci-key \
  --from-file=oci_api_key.pem=./infragate-svc.pem \
  --namespace infragate
```

### 14.6 Frontend

The frontend is a single `index.html` with accompanying `css/` and `js/` assets. Served by nginx in the `infragate-frontend` container. OIDC discovery is resolved at runtime from the proxied Keycloak well-known endpoint (`/auth/realms/infragate/.well-known/openid-configuration`), so no manual frontend config file edits are required for standard deployments.

For custom deployments outside Helm, configure the backend connection in `js/config.js`:

```javascript
const API_BASE       = 'https://infragate-api.your-domain.com';
const OIDC_ISSUER    = 'https://your-idp.example.com/realms/your-realm';
const OIDC_CLIENT_ID = 'infragate-portal';
```

---

## 15. Configuration reference

All settings are manageable at runtime via the admin panel or API — no redeployment required.

| Setting | Description | Default |
|---|---|---|
| `region` | OCI region for provisioned clusters | — |
| `compartment_ocid` | Parent compartment for cluster compartments | — |
| `cluster_type` | Cluster tier: `BASIC_CLUSTER` or `ENHANCED_CLUSTER` | `BASIC_CLUSTER` |
| `cluster_limit` | Default max clusters per user | 1 |
| `pool_max` | Max node pools per cluster | 3 |
| `node_max` | Max nodes per pool | 3 |
| `cpu_max` | Max OCPU per node | 1 |
| `ram_max` | Max RAM per node (GB) | 12 |
| `storage_max` | Max boot volume per node (GB) | 50 |
| `allowed_shapes` | VM shapes shown in the deploy form | — |
| `allowed_k8s_versions` | Kubernetes versions shown in the deploy form | — |
| `allowed_images` | OCI compute images for worker node pools — `[{image_id, label, enabled}]` | `[]` (auto-select) |
| `cidr_pool` | Available /24 ranges for cluster allocation | — |
| `pricing` | Cost estimation rate overrides (JSON) — OCPU/hr, RAM/GB/hr, storage/GB/mo, Enhanced CP/hr, shape-specific overrides | OCI PAYG defaults |

> Regional availability note: `allowed_shapes` and `allowed_images` should be curated from live OKE options in the customer's target region/tenancy. In Admin Configuration, use **Sync from OCI** for VM shapes first, then curate labels/toggles. CLI cross-check:
>
> `oci ce node-pool-options get --node-pool-option-id all --profile <PROFILE>`
>
> Use `data.shapes` and `data.sources` from that response as the source of truth for what to enable.
>
> If OCI sync is unavailable, `/api/v1/admin/config/shapes/sync` keeps existing shapes unchanged and returns a warning so admins can fall back to manual curation. The sync call authenticates with API pod OCI credentials (service account user/key), not end-user credentials.
>
> Deploy/template form behavior: Kubernetes versions are filtered by selected shape and region. A version appears only when it is both OCI-compatible for that shape (node-image-backed) and enabled in admin config. If compatibility lookup fails, UI remains fail-closed and blocks incompatible submission paths.

**Admin access for Google Workspace:** Set `ADMIN_EMAILS` to a comma-separated list of email addresses that should have admin access, since Google access tokens do not carry role claims.

---

## 16. Security considerations

**OCI private key** — store in a secrets manager (HashiCorp Vault, OCI Vault, AWS Secrets Manager, Azure Key Vault). Never commit to version control or bake into a container image.

**SSH private keys** — generated by Terraform at provisioning time and stored in the database. Treat with the same care as the OCI private key. Consider clearing the stored key after first user download and logging the access event in the audit log.

**Customer Secret Key** — treat with the same care as the OCI private key. Rotate via the OCI Console if compromised — update `OCI_S3_SECRET_KEY` and restart the container.

**JWT validation** — Infragate fetches your IdP's JWKS on startup and caches public keys. Keys are re-fetched automatically on rotation. No tokens are stored server-side. The nginx proxy caches the JWKS and OIDC well-known endpoints (1-hour TTL) to reduce load on the IdP.

**Admin role** — grants access to all clusters, all users, all overrides, kubeconfig and SSH keys across the tenancy, and the full audit log. Assign only to platform administrators.

**Per-user overrides** — only admins can set or reset user overrides. Users cannot modify their own limits. Resolved effective limits are returned to users read-only via `/api/v1/users/me`.

**Network exposure** — the Infragate API should not be exposed to the public internet unless required. Deploy behind your organisation's internal network, VPN, or API gateway. The frontend can be served publicly while the API remains internal.

**Audit log** — every provisioning, scaling, upgrade, and destroy operation is recorded with user identity, operation, cluster name, outcome, and duration. The log is append-only and available to admins at `GET /api/v1/admin/audit`.

**Least privilege** — the IAM policy in Section 9.1 grants only the permissions Infragate needs. Do not use a tenancy-admin user as the service account.

**Supply chain** — images are built and signed by the reference GitHub Actions and GitLab CI pipelines. For highly regulated environments, mirror published images into your own registry (OCIR, Harbor, Artifactory) and pin by SHA256 digest rather than tag.

---

For user and admin workflow see [README.md](./README.md).
For full testing procedures and validation matrix see [TESTING.md](./TESTING.md).
For deployment assistance or enterprise support, contact [support@infragate.cloud](mailto:support@infragate.cloud).

