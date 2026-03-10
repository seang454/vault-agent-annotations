vault.hashicorp.com/agent-inject: "true"
vault.hashicorp.com/role: "{{ .Chart.Name }}"
vault.hashicorp.com/agent-pre-populate-only: "false"
```

| Annotation | Value | Meaning |
|---|---|---|
| `agent-inject` | `"true"` | Tells Vault **mutating webhook** to inject a sidecar container into this pod |
| `role` | `order-service` | The **Vault role** this pod will authenticate as (maps to a Vault policy) |
| `agent-pre-populate-only` | `"false"` | Sidecar **keeps running** after startup to refresh secrets continuously |

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