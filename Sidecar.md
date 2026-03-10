# What is a Sidecar?

---

## Simple Analogy 🏍️

> Imagine a **motorcycle with a sidecar attached**.
> - The **motorcycle** = your main app container
> - The **sidecar** = a helper container riding alongside it
> - They travel together, share the same space, but do **different jobs**

---

## In Kubernetes Terms

A **sidecar** is an additional container that runs **inside the same Pod** as your main application container.

```
┌─────────────────────────── POD ───────────────────────────┐
│                                                            │
│   ┌─────────────────────┐    ┌─────────────────────────┐  │
│   │   YOUR APP          │    │   VAULT AGENT SIDECAR   │  │
│   │   (order-service)   │    │   (helper container)    │  │
│   │                     │    │                         │  │
│   │  python app.py      │    │  - talks to Vault       │  │
│   │                     │    │  - fetches secrets      │  │
│   │                     │    │  - writes secret files  │  │
│   └──────────┬──────────┘    └────────────┬────────────┘  │
│              │                            │               │
│              └──────────┬─────────────────┘               │
│                         │                                 │
│              ┌──────────▼──────────┐                      │
│              │  SHARED VOLUME      │                      │
│              │  /vault/secrets/    │                      │
│              │  (shared memory)    │                      │
│              └─────────────────────┘                      │
└────────────────────────────────────────────────────────────┘
```

They **share**:
- Same network (localhost)
- Same storage volumes
- Same lifecycle (start and die together)

---

## How Vault Sidecar Works Step by Step

### Step 1 — Pod is created
```
You deploy your app
        ↓
Kubernetes sees the pod
        ↓
Vault Mutating Webhook intercepts it and ADDS the sidecar automatically
```

### Step 2 — Init container runs first
```
vault-agent-init starts
        ↓
Authenticates to Vault using ServiceAccount token
        ↓
Fetches all secrets
        ↓
Writes files to /vault/secrets/
        ↓
Init container EXITS ✅
```

### Step 3 — Your app starts
```
Your app container starts
        ↓
Runs: source /vault/secrets/db.env
      source /vault/secrets/jwt.env
        ↓
Now has DB_HOST, DB_PASSWORD, JWT_SECRET etc as env vars
        ↓
exec python app.py  ← starts normally with all secrets available
```

### Step 4 — Sidecar keeps running alongside
```
vault-agent sidecar stays alive
        ↓
Watches Vault for secret changes / lease expiry
        ↓
When secret rotates → rewrites the file in /vault/secrets/
        ↓
Your app picks up new values next time it reads the file
```

---

## Visual Timeline

```
TIME ──────────────────────────────────────────────────────────►

         INIT PHASE              RUNNING PHASE
         ──────────              ─────────────────────────────

         vault-agent-init ──►EXIT
                                  vault-agent sidecar ──────────►  (keeps running)
                                  your-app ─────────────────────►  (keeps running)
```

---

## Why Use a Sidecar Instead of Putting Logic in the App?

| Approach | Problem |
|---|---|
| Hard-code secrets in app | ❌ Secrets exposed in code/git |
| Pass secrets as env vars in YAML | ❌ Secrets visible in Kubernetes manifests |
| App calls Vault API directly | ❌ Every app needs Vault SDK + auth logic |
| **Vault sidecar** | ✅ App stays simple, sidecar handles everything |

---

## Real World Sidecar Examples

Sidecars are used for many purposes beyond Vault:

| Sidecar Type | What it Does |
|---|---|
| **Vault Agent** | Fetches and refreshes secrets |
| **Istio / Envoy** | Handles all network traffic (service mesh) |
| **Fluentd / Filebeat** | Collects and ships logs |
| **Prometheus Exporter** | Exposes app metrics |
| **Cloud SQL Proxy** | Manages secure DB connections |

---

## Summary

```
Sidecar = a helper container that runs next to your app
          inside the same pod, sharing the same volumes
          and network, doing supporting work so your
          main app can stay focused on business logic.
```

In your case:
- **Your app** → just reads `/vault/secrets/*.env` files
- **Vault sidecar** → does all the hard work of authenticating, fetching, and refreshing secrets from Vault