# Infragate — Integration Guide

This guide is for platform, identity, and infrastructure teams responsible for deploying Infragate within an existing enterprise environment. Infragate is designed to integrate with your current tooling rather than replace it.

---

## Table of contents

1. [Prerequisites](#1-prerequisites)
2. [Identity provider integration](#2-identity-provider-integration)
3. [Existing OCI infrastructure](#3-existing-oci-infrastructure)
4. [Resource limits and user overrides](#4-resource-limits-and-user-overrides)
5. [Cluster tiers](#5-cluster-tiers)
6. [API integration](#6-api-integration)
7. [Deployment](#7-deployment)
8. [Configuration reference](#8-configuration-reference)
9. [Security considerations](#9-security-considerations)

---

## 1. Prerequisites

| Requirement | Details |
|---|---|
| OCI tenancy | Active tenancy with OKE enabled in at least one region |
| OCI service account | IAM user with API key — see Section 3 |
| OIDC-compliant IdP | Any provider supporting OIDC Authorization Code flow with PKCE |
| Kubernetes cluster | OKE, k3s, or any K8s cluster with Helm 3.x and an ingress controller |
| PostgreSQL 14+ | Bundled via Helm chart, or bring your own managed/self-hosted instance |

---

## 2. Identity provider integration

Infragate authenticates users via standard **OpenID Connect (PKCE)** and validates **Bearer JWTs** on every API request. It does not ship its own user directory and does not require a specific identity provider.

### 2.1 How it works

1. The frontend redirects the user to your IdP's authorization endpoint
2. After login, the IdP issues a JWT access token
3. The frontend sends that token as `Authorization: Bearer <token>` on every API call
4. Infragate validates the token against your IdP's JWKS endpoint
5. On first login, Infragate auto-provisions the user in its database using the `sub` claim as the unique identifier

No user migration or directory sync is required.

### 2.2 Required JWT claims

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

### 2.3 IdP client configuration

| Setting | Value |
|---|---|
| Client type | Public (no client secret — PKCE only) |
| Grant type | Authorization Code |
| PKCE method | S256 |
| Redirect URI | `https://infragate.your-domain.com` |
| Logout redirect URI | `https://infragate.your-domain.com` |
| Access token format | JWT |

### 2.4 Provider-specific notes

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
- Add a `Groups` claim to the access token under Authorization Server → Claims
- Set `OIDC_ROLES_CLAIM=groups`
- Set `OIDC_ISSUER=https://your-org.okta.com/oauth2/default`

**Google Workspace**
- Create an OAuth 2.0 client ID (Web application) in Google Cloud Console
- Add the authorized redirect URI
- Google does not support custom roles in the access token — use the `ADMIN_EMAILS` fallback (see Section 8)
- Set `OIDC_ISSUER=https://accounts.google.com`

---

## 3. Existing OCI infrastructure

### 3.1 Required OCI service account

**IAM user:** `infragate-svc` (or any name you prefer)

**API key:** Generate an RSA key pair. Store the private key securely — see Section 9.

**Group and policy:**

```
Allow group infragate-svc-group to manage cluster-family in compartment <your-compartment>
Allow group infragate-svc-group to manage virtual-network-family in compartment <your-compartment>
Allow group infragate-svc-group to manage instance-family in compartment <your-compartment>
Allow group infragate-svc-group to manage compartments in compartment <your-compartment>
Allow group infragate-svc-group to read tenancies in tenancy
Allow group infragate-svc-group to read objectstorage-namespaces in tenancy
```

**Object Storage bucket:** Create a bucket for Terraform state. Enable versioning. Note the bucket name and your Object Storage namespace (found under Tenancy details in the OCI Console).

**Customer Secret Key:** Generate an S3-compatible key on the `infragate-svc` user for Terraform state access.

### 3.2 Using existing VCNs, compartments, or subnets

Users can supply existing OCIDs via the **Advanced** tab in the deploy form:

| Field | Behaviour when set |
|---|---|
| Existing VCN OCID | Infragate uses this VCN, skips creating VCN + IGW + route table |
| Existing Compartment OCID | Cluster resources placed in this compartment, skips compartment creation |
| Existing Subnet OCID | Worker nodes placed in this subnet, skips subnet + security list creation |

Any combination is supported. When using an existing subnet, verify CIDR ranges do not overlap with existing VCN allocations.

### 3.3 CIDR pool management

Infragate manages a pool of /24 CIDR ranges allocated on cluster creation and released on destroy. Pre-populate the pool to reflect your allocations:

```bash
curl -X POST https://infragate.your-domain.com/api/v1/admin/config/cidrs \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "cidr": "10.120.10.0/24" }'
```

---

## 4. Resource limits and user overrides

Infragate has a two-tier limit system. Global config sets the baseline for all users. Per-user overrides take precedence when set.

### 4.1 Global limits

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

### 4.2 Per-user overrides

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

### 4.3 Effective limits

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

## 5. Cluster tiers

| | Basic (`BASIC_CLUSTER`) | Enhanced (`ENHANCED_CLUSTER`) |
|---|---|---|
| Cost | Free | ~$0.10/hr per cluster |
| Node scaling via API | ⚠️ Partial | ✅ Full in-place scaling |
| Autoscaler support | ❌ | ✅ |

**Basic cluster scaling:** Infragate updates the desired node count in OKE. New nodes are provisioned automatically. Existing nodes must be manually cycled via the OCI Console:

**OKE → Cluster → Node Pool → Nodes → Cordon & drain → Terminate**

The UI shows a warning when scaling a Basic cluster. The API returns `400` if a scale operation is attempted on a Basic cluster programmatically.

**Switching tiers:** `PUT /api/v1/admin/config/cluster-type` or the admin panel toggle. Applies to all new clusters — existing clusters retain the tier they were provisioned with.

**Per-user override:** `override_cluster_type` on `PATCH /api/v1/admin/users/:id` — gives a specific user a different tier than the platform default.

---

## 6. API integration

Every action available in the UI is also available via REST.

### 6.1 Authentication

```bash
curl https://infragate.your-domain.com/api/v1/clusters \
  -H "Authorization: Bearer $TOKEN"
```

### 6.2 Deploying a cluster from CI/CD

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

### 6.3 Endpoint reference

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/v1/users/me` | User | User context + effective limits |
| GET | `/api/v1/users/deploy-options` | User | Deploy form options: CIDRs, shapes, K8s versions, node images, active templates (filtered by user's `realm_access.roles` vs template `required_role`), pricing |
| GET | `/api/v1/clusters` | User | List user's clusters (includes `estimated_monthly_cost`) |
| POST | `/api/v1/clusters` | User | Deploy cluster — optional `template_id` for template-based deploy (server validates resource values match template definition) — returns job_id + SSE stream URL |
| GET | `/api/v1/clusters/:id` | User | Cluster detail, OCIDs, API endpoint, tier, ssh_key_available, `estimated_monthly_cost`, `cost_breakdown` |
| GET | `/api/v1/clusters/:id/kubeconfig` | User | Kubeconfig YAML download |
| GET | `/api/v1/clusters/:id/sshkey` | User | SSH private key download (.pem) |
| POST | `/api/v1/clusters/:id/scale` | User | Scale node pools (nodes, OCPU, RAM, storage), add or remove pools (both Basic and Enhanced) |
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
| DELETE | `/api/v1/admin/config/shapes/:name` | Admin | Remove VM shape |
| PATCH | `/api/v1/admin/config/shapes/:name` | Admin | Toggle shape enabled state |
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

## 7. Deployment

Infragate supports three deployment paths. Choose the one that fits your environment:

| Path | Best for | Guide |
|---|---|---|
| **OCI Marketplace** | OCI customers who want one-click deployment via Resource Manager | [GUIDE.md — Marketplace](./GUIDE.md#marketplace-deployment) |
| **Existing OKE cluster** | Teams with an existing Kubernetes cluster on OCI | [GUIDE.md — OKE](./GUIDE.md#existing-oke-cluster-deployment) |
| **Single-node k3s** | Dev/test, Always Free tier, single-VM setups | [GUIDE.md — k3s](./GUIDE.md#single-node-k3s-deployment) |

All paths use the same Helm chart with different values files:

| Values file | Target |
|---|---|
| `values-oke.yaml` | Existing OKE cluster — OCI Block Volume storage, nginx ingress |
| `values-oci.yaml` | Single-node k3s — control-plane tolerations, local storage |
| `values.yaml` | Chart defaults — all fields documented, used as base layer |

Container images are built by CI and pushed to your container registry on every commit (`dev-latest` for DEV, `latest` for main). CI pipelines are included for both GitHub Actions (GHCR) and GitLab CI (GitLab CR). The Helm chart supports `imagePullSecrets` for private registries. No local builds needed.

### 7.1 Environment variables

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

### 7.2 OCI private key

Mount the `infragate-svc` private key into the container at `OCI_PRIVATE_KEY_PATH`. On Kubernetes:

```bash
kubectl create secret generic infragate-oci-key \
  --from-file=oci_api_key.pem=./infragate-svc.pem \
  --namespace infragate
```

### 7.3 Frontend

The frontend is a single `index.html` with accompanying `css/` and `js/` assets. Served by nginx in the `infragate-frontend` container. Runtime OIDC configuration is injected via a Helm ConfigMap — no manual `config.js` edits needed when deploying with Helm.

For custom deployments outside Helm, configure the backend connection in `js/config.js`:

```javascript
const API_BASE       = 'https://infragate-api.your-domain.com';
const OIDC_ISSUER    = 'https://your-idp.example.com/realms/your-realm';
const OIDC_CLIENT_ID = 'infragate-portal';
```

---

## 8. Configuration reference

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

**Admin access for Google Workspace:** Set `ADMIN_EMAILS` to a comma-separated list of email addresses that should have admin access, since Google access tokens do not carry role claims.

---

## 9. Security considerations

**OCI private key** — store in a secrets manager (HashiCorp Vault, OCI Vault, AWS Secrets Manager, Azure Key Vault). Never commit to version control or bake into a container image.

**SSH private keys** — generated by Terraform at provisioning time and stored in the database. Treat with the same care as the OCI private key. Consider clearing the stored key after first user download and logging the access event in the audit log.

**Customer Secret Key** — treat with the same care as the OCI private key. Rotate via the OCI Console if compromised — update `OCI_S3_SECRET_KEY` and restart the container.

**JWT validation** — Infragate fetches your IdP's JWKS on startup and caches public keys. Keys are re-fetched automatically on rotation. No tokens are stored server-side. The nginx proxy caches the JWKS and OIDC well-known endpoints (1-hour TTL) to reduce load on the IdP.

**Admin role** — grants access to all clusters, all users, all overrides, kubeconfig and SSH keys across the tenancy, and the full audit log. Assign only to platform administrators.

**Per-user overrides** — only admins can set or reset user overrides. Users cannot modify their own limits. Resolved effective limits are returned to users read-only via `/api/v1/users/me`.

**Network exposure** — the Infragate API should not be exposed to the public internet unless required. Deploy behind your organisation's internal network, VPN, or API gateway. The frontend can be served publicly while the API remains internal.

**Audit log** — every provisioning, scaling, and destroy operation is recorded with user identity, operation, cluster name, outcome, and duration. The log is append-only and available to admins at `GET /api/v1/admin/audit`.

**Least privilege** — the IAM policy in Section 3.1 grants only the permissions Infragate needs. Do not use a tenancy-admin user as the service account.

---

For deployment assistance or enterprise support, contact [support@infragate.cloud](mailto:support@infragate.cloud).
