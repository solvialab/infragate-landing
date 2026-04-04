# Infragate — Stack & Architecture

Technical reference for the Infragate stack — frontend, backend, infrastructure, and the full provisioning flow.

---

## Table of contents

1. [Stack overview](#1-stack-overview)
2. [Frontend](#2-frontend)
3. [Backend](#3-backend)
4. [Database schema](#4-database-schema)
5. [Terraform layer](#5-terraform-layer)
6. [Provisioning flow](#6-provisioning-flow)
7. [Project structure](#7-project-structure)
8. [Helm configuration](#8-helm-configuration)
9. [Local development](#9-local-development)

---

## 1. Stack overview

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML / CSS / JS — single file, no build step |
| Backend | FastAPI (Python 3.12), async, SSE log streaming |
| Database | PostgreSQL 16 — clusters, jobs, users, config, templates, audit log |
| IaC | Terraform 1.7 — OCI provider, per-job execution |
| State backend | OCI Object Storage (S3-compatible API) |
| Auth | OIDC PKCE + Bearer JWT validation (JWKS) |
| Container | Docker — multi-stage builds, Docker Compose for local dev |
| DNS / CDN | Cloudflare |
| Domain | infragate.cloud (Namecheap → Cloudflare NS) |

---

## 2. Frontend

Single `index.html` with external `css/` and `js/` assets. No framework, no build step, no runtime dependencies — served from any static host.

**Pages:**
- Landing — product overview, sign in
- Deploy — cluster provisioning form (Standard + Advanced tabs), live Terraform log stream, plan preview
- Dashboard (My Clusters) — cluster cards with status, pool visualisation, tier pill, estimated cost, actions
- Detail — full cluster info, node pools, cost breakdown (per-pool + control plane + total), kubeconfig + SSH key download
- Admin — All clusters (with cost per cluster + total spend), Users & limits, Configuration, Cluster templates (with cost per template), Audit log

**Auth flow:** OIDC Authorization Code + PKCE. On login the frontend exchanges the code for a JWT, stores it in memory, and attaches it as `Authorization: Bearer` on every API call.

**Limit enforcement:** On load, `GET /api/v1/users/me` returns the user's resolved effective limits. The deploy form uses these to constrain pool counts, node counts, and compute values. The cluster limit wall is shown if the user is at their limit.

**Config:** `js/config.js` — `API_BASE`, `OIDC_ISSUER`, `OIDC_CLIENT_ID`.

---

## 3. Backend

FastAPI application. All endpoints require a valid JWT except health check.

### Auth (`app/core/auth.py`)

- `get_current_user()` — validates JWT against IdP JWKS, returns decoded claims
- `require_admin()` — additionally asserts `admin` role in the configured roles claim
- JWKS fetched on startup, cached, auto-refreshed on key rotation

### Routers

**`app/routers/users.py`**
- `GET /api/v1/users/me` — auto-provisions user on first login, returns user context + resolved effective limits via `_resolve_limits()`
- `GET /api/v1/users/deploy-options` — deploy form options: CIDRs, shapes, K8s versions, node images, active templates (filtered by user's Keycloak roles via `required_role`), pricing

**`app/routers/clusters.py`**
- `GET /api/v1/clusters` — user's active clusters (includes `estimated_monthly_cost` per cluster)
- `POST /api/v1/clusters` — deploy: validates limits, resolves template policies (destroy protection, TTL), allocates CIDR, creates cluster + job records, starts Terraform runner
- `GET /api/v1/clusters/:id` — cluster detail + `ssh_key_available` + `estimated_monthly_cost` + `cost_breakdown` (per-pool + control plane)
- `GET /api/v1/clusters/:id/kubeconfig` — returns stored kubeconfig YAML
- `GET /api/v1/clusters/:id/sshkey` — returns SSH private key (PEM)
- `POST /api/v1/clusters/:id/scale` — scale node pools (nodes, OCPU, RAM, storage), add new pools, or remove existing pools (both Basic and Enhanced)
- `DELETE /api/v1/clusters/:id` — destroy (enforces destroy protection — admin + `?force=true` required for protected clusters)
- `GET /api/v1/clusters/admin/:id/kubeconfig` — admin: kubeconfig for any cluster
- `GET /api/v1/clusters/admin/:id/sshkey` — admin: SSH key for any cluster

**`app/routers/jobs.py`**
- `GET /api/v1/jobs/:id/logs` — SSE stream of Terraform stdout, line by line

**`app/routers/admin.py`**
- `GET/PUT /api/v1/admin/config` — platform-wide config
- `GET/PUT /api/v1/admin/config/cluster-type` — tier toggle
- `GET/POST /api/v1/admin/config/cidrs` — CIDR pool management
- `DELETE /api/v1/admin/config/cidrs/:cidr` — remove CIDR from pool
- `PATCH /api/v1/admin/config/cidrs/:cidr` — toggle CIDR enabled state
- `GET/POST /api/v1/admin/config/shapes` — allowed VM shapes
- `DELETE /api/v1/admin/config/shapes/:name` — remove VM shape
- `PATCH /api/v1/admin/config/shapes/:name` — toggle shape enabled state
- `GET/POST /api/v1/admin/config/k8s-versions` — allowed Kubernetes versions
- `DELETE /api/v1/admin/config/k8s-versions/:version` — remove K8s version
- `PATCH /api/v1/admin/config/k8s-versions/:version` — toggle K8s version enabled state
- `GET/POST /api/v1/admin/config/images` — allowed node images
- `DELETE /api/v1/admin/config/images/:image_id` — remove node image
- `PATCH /api/v1/admin/config/images/:image_id` — toggle node image enabled state
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

## 4. Database schema

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
| `user_id` | UUID FK → users | |
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
| `template_id` | UUID FK → cluster_templates | Template used for provisioning (nullable) |
| `destroy_protection` | Boolean | Inherited from template — prevents user destroy |
| `ttl_hours` | Integer nullable | Time-to-live from template |
| `ttl_expires_at` | Timestamp nullable | Computed at deploy time from TTL |
| `created_at` | Timestamp | |
| `updated_at` | Timestamp | |

### `node_pools`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `cluster_id` | UUID FK → clusters | |
| `name` | String | |
| `nodes` | Integer | |
| `cpu` | Integer | OCPU |
| `ram` | Integer | GB |
| `storage` | Integer | GB |
| `status` | String | |
| `ocid` | String | Node pool OCID |

### `jobs`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `cluster_id` | UUID FK → clusters | |
| `user_id` | UUID FK → users | |
| `operation` | String | `deploy` \| `scale` \| `destroy` |
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
| `required_role` | String nullable | Keycloak realm role required to see/use this template |
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

## 5. Terraform layer

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
- `oci_containerengine_node_pool` — one per pool (1–3 pools), SSH public key injected

**Data sources:**
- `oci_identity_availability_domains` — region AD discovery
- `oci_containerengine_node_pool_option` — latest OL8 image lookup (fallback when no image_id configured)
- `oci_containerengine_cluster_kube_config` — kubeconfig YAML (uses OCI CLI token helper)

**Key variables:**
- `cluster_type` — `BASIC_CLUSTER` or `ENHANCED_CLUSTER`, set from platform config
- `image_id` — OCI node image OCID; empty string = auto-select latest from OKE options
- `pool_count`, `node_count`, `node_shape`, `ocpu_count`, `memory_in_gbs` per pool
- `vcn_ocid`, `compartment_ocid`, `subnet_ocid` — optional overrides for existing infrastructure

When override OCIDs are provided, the corresponding resources are skipped and the module wires up to the existing infrastructure instead.

### Runner template (`terraform/runner-template/`)

A thin wrapper module copied per provisioning job. Backend copies this template to `terraform/runner/{job_id}/`, writes `job.tfvars`, then executes `terraform init && terraform plan && terraform apply`.

**State path:** `{user_id}/{cluster_id}/terraform.tfstate` in the `infragate-tfstate` OCI Object Storage bucket — enables safe concurrent operations and clean destroy.

### Terraform service (`backend/app/services/terraform.py`)

FastAPI ↔ Terraform bridge:
- Copies runner template to `terraform/runner/{job_id}/`
- Writes `job.tfvars` with cluster parameters + OCI credentials
- Runs `terraform init → plan → apply` as an async subprocess
- Streams stdout line-by-line via SSE to the frontend log viewer
- Parses `terraform output -json` — stores OCIDs, API endpoint, kubeconfig, SSH private key in the database
- Updates cluster status: `provisioning` → `running` (or `error`)
- On destroy: releases CIDR back to pool, cleans up runner directory
- On scale: updates pool records in DB, rewrites tfvars, re-applies

---

## 6. Provisioning flow

```
User (browser)
    │
    │  OIDC login (PKCE) — via your existing IdP
    ▼
Your Identity Provider  ─────────────────► JWT issued
    │
    │  POST /api/v1/clusters  (Bearer JWT)
    ▼
Infragate API (FastAPI)
    │  1. Validate JWT against IdP JWKS
    │  2. Resolve effective limits (per-user override or global config)
    │  3. Check cluster limit
    │  4. Allocate CIDR from pool
    │  5. Write cluster record (status: provisioning)
    │  6. Write job record (operation: deploy)
    │  7. Copy runner-template → terraform/runner/{job_id}/
    │  8. Write job.tfvars (cluster params + cluster_type)
    │  9. terraform init + plan + apply (subprocess)
    │  10. Stream stdout line-by-line via SSE → browser log viewer
    │  11. Parse outputs → store OCIDs, endpoint, kubeconfig, SSH key in DB
    │  12. Update cluster status: running
    ▼
OCI
    ├── oci_identity_compartment
    ├── oci_core_vcn
    ├── oci_core_subnet
    ├── oci_core_internet_gateway
    ├── oci_core_route_table
    ├── oci_core_security_list
    ├── oci_containerengine_cluster   (BASIC or ENHANCED)
    └── oci_containerengine_node_pool (per pool, 1–3)
```

---

## 7. Project structure

```
infragate/
├── index.html                      — portal UI (single file)
├── css/
│   ├── variables.css
│   ├── base.css
│   ├── components.css
│   ├── landing.css
│   ├── deploy.css
│   ├── dashboard.css
│   ├── detail.css
│   ├── admin.css
│   └── responsive.css
├── js/
│   ├── config.js                   — API_BASE, OIDC config, global limits sync
│   ├── state.js                    — shared state: POOLS, LIMITS, GLOBAL_LIMITS, SCALE_CURRENT, SCALE_POOL_LIST
│   ├── api.js                      — auth (OIDC PKCE), API helpers, user context
│   ├── utils.js                    — clipboard copy, toast notification helpers
│   ├── stepper.js                  — numeric stepper component
│   ├── errors.js                   — shared error/success card builder for deploy/scale/destroy
│   ├── modals.js                   — overlay/modal helpers
│   ├── preview.js                  — architecture preview sync, override field listeners
│   ├── pools.js                    — pool tab management
│   ├── router.js                   — hash-based page routing, nav toggle
│   ├── deploy.js                   — deploy form, plan preview, download helpers
│   ├── destroy.js                  — destroy flow, protection enforcement, cleanup
│   ├── scale.js                    — scale preview plan and apply logic
│   ├── admin.js                    — admin actions (edit user, config, shapes, CIDRs)
│   ├── cost.js                     — client-side cost estimation (OCI PAYG defaults)
│   ├── templates.js                — cluster template selector + admin CRUD
│   ├── theme.js                    — dark/light theme toggle with localStorage
│   ├── env.js                      — runtime config (OIDC issuer/client), overridden by Helm ConfigMap
│   └── init.js                     — boot sequence, auth init, initial page load
├── terraform/
│   ├── module/
│   │   ├── main.tf                 — all OCI resources + cluster_type variable
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── runner-template/
│   │   └── main.tf                 — per-job wrapper, copied by backend
├── Dockerfile                      — production API (multi-stage, non-root, SHA256-verified Terraform)
├── Dockerfile.frontend             — production frontend (nginx:alpine, non-root, port 8080)
├── .dockerignore                   — excludes tests, docs, CI from image builds
├── .editorconfig                   — consistent formatting across editors (indent, charset, EOL)
├── .gitlab-ci.yml                  — GitLab CI/CD pipeline (test, lint, build+push to GitLab CR)
├── deploy/
│   ├── deploy.sh                   — automated deployment script with config validation
│   ├── marketplace/                — OCI Marketplace Resource Manager Stack
│   │   ├── schema.yaml            — OCI Console form definition (variable groups, dynamic types)
│   │   ├── variables.tf           — all stack variables
│   │   ├── provider.tf            — OCI, Kubernetes, Helm providers
│   │   ├── locals.tf              — computed values, password generation
│   │   ├── datasources.tf         — OCI data sources
│   │   ├── network.tf             — VCN, subnets, gateways, security lists (conditional)
│   │   ├── oke.tf                 — OKE cluster + node pool (conditional)
│   │   ├── iam.tf                 — dynamic group + IAM policies
│   │   ├── helm.tf                — ingress-nginx + Infragate Helm release
│   │   ├── outputs.tf             — post-deployment URLs and instructions
│   │   └── build-stack.sh         — builds zip for marketplace upload
│   └── helm/
│       ├── Chart.yaml              — chart metadata (kubeVersion, keywords, icon, maintainers)
│       ├── values.yaml             — chart defaults (all fields, placeholders, comments)
│       ├── values-oci.yaml         — single-node k3s template (copy to values-dev.yaml)
│       ├── values-oke.yaml         — existing OKE cluster values (oci-bv storage, nginx ingress)
│       ├── values-dev.yaml         — your deployment values (copy of values-oci.yaml, customised)
│       └── templates/
│           ├── _helpers.tpl        — template helpers (fullname, labels, URLs)
│           ├── api.yaml            — API deployment + service (securityContext, probes, RBAC)
│           ├── frontend.yaml       — Frontend deployment + service (non-root nginx, readOnly FS)
│           ├── postgresql.yaml     — PostgreSQL StatefulSet (non-root, startup probe)
│           ├── keycloak.yaml       — Keycloak deployment (non-root, startup probe)
│           ├── serviceaccount.yaml — dedicated ServiceAccounts (automountToken: false)
│           ├── configmap-nginx.yaml     — nginx config (security headers, SSE proxy, rate limits)
│           ├── configmap-frontend.yaml  — runtime JS config (OIDC issuer/client from Helm values)
│           ├── configmap-initdb.yaml    — PostgreSQL init script (Keycloak schema)
│           ├── secrets.yaml        — API, PostgreSQL, Keycloak secrets
│           ├── ingress.yaml        — Ingress with TLS support
│           └── namespace.yaml      — Namespace creation
└── backend/
    ├── app/
    │   ├── main.py                 — FastAPI entry, CORS, router registration
    │   ├── core/
    │   │   ├── config.py           — pydantic-settings, reads .env
    │   │   ├── database.py         — SQLAlchemy engine + get_db()
    │   │   └── auth.py             — JWKS validation, get_current_user(), require_admin()
    │   ├── models/
    │   │   └── models.py           — User, Cluster, NodePool, Job, Config, ClusterTemplate, AuditLog
    │   ├── schemas/
    │   │   └── schemas.py          — Pydantic request/response shapes
    │   ├── services/
    │   │   └── cost.py             — cost estimation engine (OCI PAYG defaults, admin overrides)
    │   └── routers/
    │       ├── users.py            — /users/me, _resolve_limits(), deploy-options (incl. pricing)
    │       ├── clusters.py         — CRUD, scale, destroy, kubeconfig, sshkey, cost injection
    │       ├── jobs.py             — SSE log streaming
    │       └── admin.py            — config, users, stats (incl. spend), templates, audit
    ├── docker-compose.yml          — FastAPI + PostgreSQL + Keycloak (local dev)
    ├── Dockerfile.dev              — development Dockerfile (hot-reload, used by docker-compose)
    ├── requirements.txt            — runtime + test dependencies
    ├── pytest.ini                  — test runner configuration
    ├── tests/
    │   ├── conftest.py             — fixtures: in-memory SQLite, mock auth, seed data
    │   ├── test_health.py          — health + root endpoints
    │   ├── test_users.py           — /users/me, deploy-options, limit resolution
    │   ├── test_cluster_access.py  — kubeconfig + SSH key download, admin access control
    │   ├── test_admin_config.py    — config CRUD, CIDR/shape/k8s-version management
    │   ├── test_admin_templates.py — template CRUD, enable/disable, hard delete
    │   ├── test_admin_users.py     — user listing, per-user overrides, search
    │   ├── test_admin_clusters.py  — admin stats, cluster listing, search
    │   └── test_cost.py            — cost estimation engine (pure unit tests)
    ├── init.sql                    — CREATE SCHEMA IF NOT EXISTS keycloak
    └── .env                        — secrets (gitignored)
```

---

## 8. Helm configuration

Infragate ships four Helm values files, each serving a distinct role in the deployment pipeline:

| File | Role | Committed to repo | When to use |
|---|---|---|---|
| `values.yaml` | Chart defaults | ✅ Yes | Never used directly — Helm reads it automatically as the base layer |
| `values-oci.yaml` | Single-node k3s template | ✅ Yes | Copy this to create your own `values-dev.yaml` for k3s deployments |
| `values-oke.yaml` | Existing OKE cluster | ✅ Yes | Pass to `helm upgrade` with `-f` when deploying to an existing OKE cluster |
| `values-dev.yaml` | Your deployment values | ✅ Yes | Pass to `helm upgrade` with `-f` for your environment |

### `values.yaml` — Chart defaults

The canonical reference for every configurable field. Contains all parameters with placeholder values, inline documentation, and sensible defaults. Helm merges this automatically before any `-f` overrides. Operators should read this file to understand the full set of available options, but never edit it directly for a specific deployment.

### `values-oci.yaml` — OCI deployment template

A production-ready starting point for single-node OCI deployments. Pre-configured with:

- GHCR image references (`ghcr.io/solvialab/infragate-api`, `ghcr.io/solvialab/infragate-frontend`)
- SSE-optimised nginx proxy timeouts for Terraform log streaming
- Control-plane tolerations for single-node scheduling
- TLS disabled by default (enabled after cert-manager setup)

This file is committed to the repository and kept up to date as a template. To deploy on k3s:

```bash
cp deploy/helm/values-oci.yaml deploy/helm/values-dev.yaml
# Edit values-dev.yaml with your domain, TLS settings, etc.
```

### `values-oke.yaml` — Existing OKE cluster

A production-ready values file for deploying Infragate into an existing OKE cluster. Key differences from `values-oci.yaml`:

- OCI Block Volume storage class (`oci-bv`) for PostgreSQL persistence
- `pullPolicy: IfNotPresent` (OKE nodes pull from GHCR directly)
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
  └── values-dev.yaml (-f override)
        └── --set flags (highest priority)
```

This separation ensures the chart ships with complete, documented defaults while each deployment environment maintains its own configuration without modifying committed files.

---

## 9. Testing

### Test suite

87 automated tests covering business logic, validation rules, API contracts, access control, and cost estimation. Tests run against an in-memory SQLite database with mocked authentication — no external services required.

```bash
cd backend
pip install -r requirements.txt
pytest tests/ -v --tb=short --cov=app --cov-report=term-missing
```

| Test file | Tests | Coverage area |
|---|---|---|
| `test_health.py` | 2 | Health and root endpoints |
| `test_users.py` | 13 | User auto-provisioning, effective limits, deploy options, template filtering, pricing |
| `test_cluster_access.py` | 10 | Kubeconfig + SSH key download (user + admin), access control |
| `test_admin_config.py` | 29 | Config CRUD, cluster type, CIDR/shape/K8s version management |
| `test_admin_templates.py` | 12 | Template CRUD, enable/disable, duplicate rejection, storage validation |
| `test_admin_users.py` | 8 | User listing, per-user overrides, search, storage validation |
| `test_admin_clusters.py` | 6 | Admin stats with cost estimation, cluster listing, search |
| `test_cost.py` | 7 | Cost engine — basic/enhanced tiers, multi-pool, custom pricing, shape overrides |

### Test infrastructure (`conftest.py`)

- **Database:** In-memory SQLite with `StaticPool` for thread safety (FastAPI TestClient runs requests in a separate thread). PostgreSQL UUID type shimmed to `VARCHAR(36)` for SQLite compatibility.
- **Auth:** Keycloak completely mocked — hardcoded user and admin token payloads injected via FastAPI dependency overrides.
- **Fixtures:** `db_session`, `client` (regular user), `admin_client` (admin), `seed_config`, `seed_user`, `seed_admin_user`.

### CI pipeline

Infragate ships with two CI/CD pipeline configurations — use whichever matches your Git platform:

**GitHub Actions** (`.github/workflows/ci.yml`) — runs on every push to `main`/`DEV` and on PRs to `main`:

| Job | What it does |
|---|---|
| `test` | Python unit tests (87 tests) with coverage |
| `helm-lint` | Helm lint + template render (default values + k3s values + OKE values) |
| `docker-build-push` | Builds both Docker images and pushes to GHCR (`dev-latest` / `latest` + commit SHA) |
| `terraform-validate` | `terraform validate` on core module, runner template, and marketplace stack |

**GitLab CI** (`.gitlab-ci.yml`) — runs on every push to `main`/`DEV` and on merge requests:

| Job | What it does |
|---|---|
| `test` | Python unit tests (87 tests) with coverage |
| `helm-lint` | Helm lint + template render (default values + k3s values + OKE values) |
| `build-api` / `build-frontend` | Builds and pushes images to GitLab Container Registry (`dev-latest` / `latest` + commit SHA) |
| `terraform-validate` | `terraform validate` on core module, runner template, and marketplace stack |

Both pipelines use the same tagging scheme: `latest` + SHA for `main`, `dev-latest` + `dev-SHA` for `DEV`.

---

## 10. Local development

```bash
cd backend
cp /path/to/secrets/.env .env
docker compose up --build
```

| Service | URL | Credentials |
|---|---|---|
| API + docs | http://localhost:8000/docs | — |
| Keycloak admin | http://localhost:8080 | admin / admin |
| PostgreSQL | localhost:5432 | infragate / infragate_dev |

The included `docker-compose.yml` bundles Keycloak as a convenience for local development. In production, point `OIDC_ISSUER` at your existing identity provider.

**Normal stop/start** (data persists):
```bash
docker compose down
docker compose up
```

**Full reset** (wipes database):
```bash
docker compose down -v
docker compose up --build
```

---

For user and admin workflow see [README.md](./README.md).
For integration and deployment see [INTEGRATION.md](./INTEGRATION.md).
