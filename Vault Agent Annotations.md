# Vault Agent Annotations — Detailed Explanation

---

## Overview

These Kubernetes pod annotations configure the **HashiCorp Vault Agent Injector** — a sidecar-based integration that automatically fetches secrets from Vault and writes them as files inside your pod.

**Integration Type:** Vault Agent Injector (Sidecar)  
**Secret Engine:** KV v2  
**Auth Method:** Kubernetes Auth (ServiceAccount token)

---

## Section 1: Core Vault Agent Setup

```yaml
vault.hashicorp.com/agent-inject: "true"
vault.hashicorp.com/role: "{{ .Chart.Name }}"
vault.hashicorp.com/agent-pre-populate-only: "false"
```

| Annotation | Value | Meaning |
|---|---|---|
| `agent-inject` | `"true"` | Tells Vault mutating webhook to inject a sidecar into this pod |
| `role` | `order-service` | The Vault role this pod authenticates as (maps to a Vault policy) |
| `agent-pre-populate-only` | `"false"` | Sidecar keeps running after startup to refresh secrets continuously |

### What happens behind the scenes:

```
Pod created
    ↓
Vault Mutating Webhook intercepts it
    ↓
Injects vault-agent-init (init container) + vault-agent (sidecar)
    ↓
vault-agent authenticates to Vault using the pod's ServiceAccount token
    ↓
Vault checks: "Does role 'order-service' have permission to read these paths?"
    ↓
If yes → writes secret files to /vault/secrets/
```

---

## Section 2: DB Credentials Secret

```yaml
vault.hashicorp.com/agent-inject-secret-db.env: "secret/data/demo/order-service/db"
vault.hashicorp.com/agent-inject-template-db.env: |
  {{- with secret "secret/data/demo/order-service/db" -}}
  export DB_HOST="{{ .Data.data.host }}"
  export DB_PORT="{{ .Data.data.port }}"
  export DB_NAME="{{ .Data.data.name }}"
  export DB_USER="{{ .Data.data.username }}"
  export DB_PASSWORD="{{ .Data.data.password }}"
  export DATABASE_URL="postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@{{ .Data.data.host }}:{{ .Data.data.port }}/{{ .Data.data.name }}"
  {{- end }}
```

**`agent-inject-secret-db.env`** — tells Vault *what* to fetch:
- The part after `agent-inject-secret-` becomes the **output filename**: `/vault/secrets/db.env`
- The value `"secret/data/demo/order-service/db"` is the **path inside Vault KV store**

**`agent-inject-template-db.env`** — tells Vault *how* to format the output.

### Rendered output at `/vault/secrets/db.env`:

```bash
export DB_HOST="prod-db.example.com"
export DB_PORT="5432"
export DB_NAME="orders"
export DB_USER="app_user"
export DB_PASSWORD="s3cr3t!"
export DATABASE_URL="postgresql://app_user:s3cr3t!@prod-db.example.com:5432/orders"
```

### Why `.Data.data.host` has TWO `.data` levels?

KV v2 wraps secrets in an extra metadata layer:

```json
{
  "data": {
    "data": {
      "host": "prod-db.example.com",
      "port": "5432"
    },
    "metadata": { "version": 1 }
  }
}
```

- First `.Data` → Vault API response wrapper
- Second `.data` → your actual secret values

---

## Section 3: JWT Secret

```yaml
vault.hashicorp.com/agent-inject-secret-jwt.env: "secret/data/demo/shared/jwt"
vault.hashicorp.com/agent-inject-template-jwt.env: |
  {{- with secret "secret/data/demo/shared/jwt" -}}
  export JWT_SECRET="{{ .Data.data.secret }}"
  export JWT_EXPIRY="{{ .Data.data.expiry }}"
  export JWT_ISSUER="{{ .Data.data.issuer }}"
  {{- end }}
```

### Rendered output at `/vault/secrets/jwt.env`:

```bash
export JWT_SECRET="eyJhbGc..."
export JWT_EXPIRY="3600"
export JWT_ISSUER="https://auth.example.com"
```

> **Note:** This is stored under `shared/jwt` — meaning **multiple services share this same JWT secret** from one Vault path. Any rotation in Vault automatically propagates to all services.

---

## Section 4: API Keys

```yaml
vault.hashicorp.com/agent-inject-secret-apikeys.env: "secret/data/demo/shared/apikeys"
vault.hashicorp.com/agent-inject-template-apikeys.env: |
  {{- with secret "secret/data/demo/shared/apikeys" -}}
  export STRIPE_KEY="{{ .Data.data.stripe_key }}"
  export SENDGRID_API_KEY="{{ .Data.data.sendgrid_key }}"
  export SENTRY_DSN="{{ .Data.data.sentry_dsn }}"
  {{- end }}
```

### Rendered output at `/vault/secrets/apikeys.env`:

```bash
export STRIPE_KEY="sk_live_abc123..."
export SENDGRID_API_KEY="SG.xyz..."
export SENTRY_DSN="https://abc@sentry.io/123"
```

> **Note:** Also under `shared/` — these 3rd party API keys are shared across multiple services from a single Vault path.

---

## Section 5: Docker Registry Credentials

```yaml
vault.hashicorp.com/agent-inject-secret-registry.env: "secret/data/demo/registry"
vault.hashicorp.com/agent-inject-template-registry.env: |
  {{- with secret "secret/data/demo/registry" -}}
  export REGISTRY_USER="{{ .Data.data.username }}"
  export REGISTRY_PASSWORD="{{ .Data.data.password }}"
  {{- end }}
```

### Rendered output at `/vault/secrets/registry.env`:

```bash
export REGISTRY_USER="myapp-bot"
export REGISTRY_PASSWORD="ghp_token123..."
```

> **Note:** Used by the `vault-registry-sync` CronJob to keep the Kubernetes `imagePullSecret` up to date.

---

## Section 6: Vault Agent Sidecar Resource Limits

```yaml
vault.hashicorp.com/agent-limits-cpu: "100m"
vault.hashicorp.com/agent-limits-mem: "64Mi"
vault.hashicorp.com/agent-requests-cpu: "50m"
vault.hashicorp.com/agent-requests-mem: "32Mi"
```

These control resources for the **vault-agent sidecar container**, not your app container.

| Annotation | Value | Meaning |
|---|---|---|
| `agent-limits-cpu` | `100m` | Max 0.1 CPU core the sidecar can consume |
| `agent-limits-mem` | `64Mi` | Max 64 MB RAM the sidecar can consume |
| `agent-requests-cpu` | `50m` | Guaranteed/reserved 0.05 CPU core |
| `agent-requests-mem` | `32Mi` | Guaranteed/reserved 32 MB RAM |

Setting these prevents the Vault sidecar from consuming too many node resources and helps the Kubernetes scheduler plan capacity accurately.

---

## Secret Path Structure

```
secret/
└── data/
    └── demo/
        ├── order-service/
        │   └── db          ← DB creds (service-specific)
        ├── shared/
        │   ├── jwt         ← JWT secret (shared across services)
        │   └── apikeys     ← 3rd party API keys (shared)
        └── registry        ← Docker registry credentials
```

**Design pattern:**
- `order-service/` → secrets that belong only to this service
- `shared/` → secrets reused by multiple services

---

## Full Pod Startup Flow

```
1. Pod scheduled on node
        ↓
2. Vault webhook injects vault-agent-init + vault-agent sidecar
        ↓
3. vault-agent-init runs FIRST (init container):
   - authenticates to Vault using ServiceAccount token
   - fetches all 4 secrets
   - writes /vault/secrets/db.env
   - writes /vault/secrets/jwt.env
   - writes /vault/secrets/apikeys.env
   - writes /vault/secrets/registry.env
        ↓
4. Your app container starts:
   - runs: source /vault/secrets/db.env
   - runs: source /vault/secrets/jwt.env
   - runs: source /vault/secrets/apikeys.env
   - runs: exec python app.py  ← now has all env vars loaded
        ↓
5. vault-agent sidecar keeps running in background:
   - watches for secret lease expiry
   - auto-refreshes secret files when secrets rotate
   - your app picks up new values on next restart or re-source
```

---

## Security Benefits

| Practice | How it's implemented |
|---|---|
| No secrets in environment variables at deploy time | Secrets injected at runtime by Vault agent |
| No secrets in Helm values or ConfigMaps | All sensitive data lives only in Vault |
| Secrets never written to disk | `emptyDir.medium: Memory` stores files in RAM only |
| Least privilege | Each service uses its own Vault role with scoped policies |
| Automatic rotation support | Sidecar with `pre-populate-only: false` refreshes secrets continuously |