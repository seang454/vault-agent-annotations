```bash
helm-demo-vault/
│
├── 📄 README.md
│   └── Full deployment guide — architecture diagram, step-by-step
│       instructions, secret rotation commands, and Vault vs K8s
│       Secrets comparison table.
│
├── 📄 deploy.sh
│   └── Master deployment script. Runs in order:
│       1) installs Vault via Helm if not present
│       2) creates the K8s namespace
│       3) applies the registry sync CronJob
│       4) deploys each service with the right values file
│       Supports ENV=dev|staging|prod and MODE=umbrella.
│
├── 📁 vault/                          ← HashiCorp Vault config files
│   │
│   ├── 📄 vault-setup.sh
│   │   └── ONE-TIME setup script. Run after Vault is installed.
│   │       - Enables Kubernetes auth method
│   │       - Writes ALL secrets (DB creds, JWT, API keys, registry)
│   │       - Creates policies (who can read what)
│   │       - Binds policies to K8s ServiceAccounts per service
│   │
│   ├── 📄 vault-helm-values.yaml
│   │   └── Config for installing Vault itself via Helm.
│   │       Sets HA mode (3 replicas), enables the Agent Injector
│   │       (the component that auto-injects secrets into pods),
│   │       and configures resource limits.
│   │
│   └── 📄 vault-registry-sync.yaml
│       └── K8s CronJob that runs every 6 hours.
│           Reads Docker Hub credentials from Vault and syncs
│           them into a K8s imagePullSecret so pods can pull
│           private Docker images automatically.
│
├── 📁 charts/                         ← One Helm chart per microservice
│   │
│   ├── 📁 user-service/
│   │   │
│   │   ├── 📄 Chart.yaml
│   │   │   └── Chart metadata — name, version, appVersion,
│   │   │       description, and maintainer info. Helm reads
│   │   │       this to identify the chart.
│   │   │
│   │   ├── 📄 values.yaml             ← DEFAULT config (all envs)
│   │   │   └── Base configuration: replica count, image repo,
│   │   │       ports, resource limits, probe paths, autoscaling
│   │   │       ranges, and the vault: block (secret paths +
│   │   │       registry secret name). NO sensitive values here.
│   │   │
│   │   ├── 📄 values-staging.yaml     ← STAGING overrides only
│   │   │   └── Overrides: 1 replica, RC image tag, autoscaling
│   │   │       disabled, staging hostname. Merged on top of
│   │   │       values.yaml at deploy time.
│   │   │
│   │   ├── 📄 values-prod.yaml        ← PRODUCTION overrides only
│   │   │   └── Overrides: 4 replicas, pinned image tag, higher
│   │   │       CPU/memory limits, larger autoscaling range,
│   │   │       production hostname.
│   │   │
│   │   └── 📁 templates/              ← Helm renders these → K8s YAML
│   │       │
│   │       ├── 📄 deployment.yaml     ⭐ MOST IMPORTANT
│   │       │   └── Defines the pod spec. Contains all Vault Agent
│   │       │       annotations that tell the injector WHICH secrets
│   │       │       to fetch and HOW to render them as files inside
│   │       │       the pod (/vault/secrets/db.env, jwt.env, etc).
│   │       │       App sources these files on startup so secrets
│   │       │       are never in env vars or config files.
│   │       │
│   │       ├── 📄 serviceaccount.yaml ⭐ REQUIRED FOR VAULT
│   │       │   └── Creates a dedicated K8s ServiceAccount for this
│   │       │       service. Vault uses it to verify the pod's
│   │       │       identity via Kubernetes auth before issuing
│   │       │       any secrets. Each service gets its own SA.
│   │       │
│   │       ├── 📄 vault-secret-template.yaml
│   │       │   └── ConfigMap holding Go templates that define the
│   │       │       exact format of each secret file rendered by
│   │       │       the Vault Agent (db.tmpl → db.env, etc).
│   │       │
│   │       ├── 📄 service.yaml
│   │       │   └── K8s Service (ClusterIP). Gives the deployment
│   │       │       a stable internal DNS name so other services
│   │       │       can call it as http://user-service:5000.
│   │       │
│   │       ├── 📄 ingress.yaml
│   │       │   └── Nginx Ingress rule. Routes external HTTP traffic
│   │       │       by hostname + path to this service. Controlled
│   │       │       by ingress.enabled in values so it can be
│   │       │       toggled per environment.
│   │       │
│   │       └── 📄 hpa.yaml
│   │           └── HorizontalPodAutoscaler. Auto-scales pod count
│   │               between minReplicas and maxReplicas based on
│   │               CPU utilization. Skipped when autoscaling
│   │               is disabled in values.
│   │
│   └── 📁 order-service/              ← Identical structure to user-service
│       └── (same files, different secret paths and port 5001)
│           Key difference in deployment.yaml: also has STRIPE_KEY
│           in apikeys.env, and USER_SERVICE_URL env var for
│           calling user-service via K8s internal DNS.
│
├── 📁 umbrella/                       ← Deploys ALL services at once
│   │
│   ├── 📄 Chart.yaml
│   │   └── Declares user-service and order-service as
│   │       sub-chart dependencies via file:// local paths.
│   │
│   ├── 📄 values.yaml
│   │   └── Umbrella-level defaults. Uses user-service: and
│   │       order-service: prefixes to override sub-chart values.
│   │       One file controls both services together.
│   │
│   └── 📄 values-prod.yaml
│       └── Production overrides for both services in one place.
│           Used with: helm upgrade --install myapp ./umbrella
│                        -f umbrella/values-prod.yaml
│
└── 📁 .github/workflows/
    └── 📄 deploy.yaml
        └── GitHub Actions CI/CD pipeline:
            - Job 1 (build): builds Docker images, tags with git SHA,
              pushes to Docker Hub
            - Job 2 (deploy-staging): auto-deploys to staging on
              every push to main
            - Job 3 (deploy-prod): deploys to production via umbrella
              chart, requires manual approval in GitHub environments
              
```