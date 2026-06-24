# GitOps & CD with ArgoCD — Zen Pharma Platform
## 4-Session Curriculum Design Spec

**Audience:** DevOps engineers with basic cloud/K8s knowledge, targeting industry jobs  
**Duration:** 4 sessions × 1.5 hours  
**Format:** Instructor-led live demo; students replicate on own EKS cluster after each session  
**Project:** zen-gitops — GitOps configuration repository for the Zen Pharma platform  
**Anchor service:** auth-service (richest config — secrets, probes, serviceaccount, configmap)

---

## Story Arc

> "You just joined Zen Pharma as a junior DevOps engineer. On Day 1 you are given access to an EKS cluster and told: deploy auth-service. By Session 4, you have built production-grade GitOps across 9 services and 3 environments — and you can defend every decision in an interview."

Each session is a chapter in that story. Every problem is real. Every solution is what the session teaches.

---

## Platform Overview

| Layer | Technology |
|---|---|
| Microservices (9) | auth-service, api-gateway, catalog-service, inventory-service, manufacturing-service, qc-service, supplier-service, notification-service, pharma-ui |
| Language | Spring Boot (Java) + React (pharma-ui) |
| Database | PostgreSQL on AWS RDS |
| Container Registry | AWS ECR |
| Cluster | AWS EKS (3 namespaces: dev, qa, prod) |
| GitOps | ArgoCD watching zen-gitops repo |
| Secrets | AWS Secrets Manager via External Secrets Operator |
| Ingress | NGINX Ingress Controller |
| Monitoring | Prometheus + Grafana |

---

## Session Overview

| Session | Title | Core Skills |
|---|---|---|
| 1 | Day 1 — Understand the platform, deploy manually | K8s components, network flow, raw manifest deploy |
| 2 | Week 1 — kubectl doesn't scale. Enter ArgoCD + Helm | GitOps, Helm chart model, ArgoCD UI |
| 3 | Week 2 — Ship all 9 services. Wire up the full CD pipeline | Multi-env, CI→CD handoff, External Secrets, IRSA |
| 4 | The Interview — Defend what you built | 13 mock interview Q&As with simulations |

---

## Prerequisites

Students should have before Session 1:
- Basic AWS knowledge (IAM, EC2, VPC, networking)
- Basic Git (clone, commit, push, PR)
- Basic Docker concepts (image, container, registry)
- kubectl installed locally
- AWS CLI configured
- EKS cluster created (eksctl or from class infra)

**Cost note:** EKS is not free tier. Students should expect ~$5–10/day for a small cluster. Shut it down after each lab session.

---

# SESSION 1

## "Day 1 — Understand the Platform, Deploy Manually"

**Duration:** 1.5 hours  
**Objective:** Understand every Kubernetes component used in this project. Deploy auth-service to EKS using raw manifests.

---

### Instructor Setup Notes (Before Class)

Before Session 1, ensure:
1. **Docker image in ECR:** Students need an auth-service image in their own ECR repo. Two options:
   - Option A (recommended): Share a pre-built public image (e.g., a Spring Boot hello-world on port 8081) as a stand-in for Session 1. Replace with the real auth-service image in Session 2 when ArgoCD is set up.
   - Option B: Have students build and push the image from `zen-pharma-backend` before class.
2. **RDS dependency:** auth-service requires a PostgreSQL database to pass its readiness probe. In Session 1, students won't have RDS yet. The pod will be `Running` but `0/1 READY`. **This is intentional** — use it to explain what readiness probes do and why the Service gets no endpoints. It sets up the ESO + secrets discussion in Session 3.
3. **kubectl access:** Students should have `kubectl` configured against their EKS cluster before class starts.

---

### Part 1 — Platform Overview (10 min)

**The Story**

You joined Zen Pharma as a junior DevOps engineer. Here's what they're running:

```
zen-pharma-backend       zen-pharma-frontend
(9 Spring Boot services) (React UI)
         │                      │
         ▼                      ▼
     AWS ECR (Docker images)
         │
         ▼
    AWS EKS Cluster
    ├── dev namespace
    ├── qa namespace
    └── prod namespace
         │
         ▼
    AWS RDS (PostgreSQL)
    AWS Secrets Manager (credentials)
```

Your first task from your manager: **"Deploy auth-service to dev."**

No instructions. No automation. Just you, an EKS cluster, and a Docker image sitting in ECR.

**What is auth-service?**
- Spring Boot microservice that handles user authentication and JWT tokens
- Listens on port 8081
- Needs a PostgreSQL database to store users
- Needs a JWT secret key to sign tokens
- Exposes `/actuator/health` (liveness) and `/actuator/health/readiness` (readiness)

**Repository structure we'll use today:**
```
zen-gitops/
├── k8s/
│   ├── namespaces.yaml          ← namespace definitions
│   └── rbac/
│       ├── dev-role.yaml        ← what can be done in dev namespace
│       └── rolebindings.yaml    ← who gets the role
└── envs/
    └── dev/
        └── values-auth-service.yaml  ← all config values for auth-service in dev
```

---

### Part 2 — Kubernetes Components Deep-Dive (25 min)

We'll walk through every Kubernetes object auth-service uses, with the actual YAML from the project.

---

#### 1. Namespace

**What it is:** Logical isolation within a cluster. Like a separate folder per environment.

```yaml
# k8s/namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    env: dev
    managed-by: terraform
```

**In this project:** Three namespaces — dev, qa, prod. A pod in dev cannot access resources in prod (enforced by RBAC + NetworkPolicy).

**Daily DevOps relevance:** When your manager says "deploy to dev," they mean: target the `dev` namespace. When prod is broken, you check the `prod` namespace first.

---

#### 2. Deployment

**What it is:** Manages the desired state of your application. Handles rolling updates, rollbacks, and ensures the right number of pods are running.

**Key fields:**
- `replicas`: how many pods to run (1 in dev, 2 in prod)
- `selector.matchLabels`: how the Deployment finds its own pods
- `template`: the pod definition (what image, what ports, what config)
- `strategy`: RollingUpdate (default) — replaces pods one at a time, zero downtime

**In this project:**
```yaml
# From envs/dev/values-auth-service.yaml
replicaCount: 1
image:
  repository: 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service
  tag: sha-a0017b8
  pullPolicy: Always
```

**Daily DevOps relevance:** When a new version is deployed, K8s creates new pods with the new image and terminates old ones. If new pods fail readiness checks, the rollout pauses — production is protected.

---

#### 3. ReplicaSet

**What it is:** Created automatically by the Deployment. Ensures exactly N copies of the pod are always running.

**You rarely interact with it directly.** The Deployment manages it. But when you do `kubectl get replicasets -n dev`, you'll see one per Deployment revision — that's how rollback works at the K8s level.

---

#### 4. Pod

**What it is:** The smallest deployable unit in Kubernetes. One running instance of auth-service.

**In this project:**
```yaml
# Container spec (from helm-charts/templates/deployment.yaml, rendered with dev values)
containers:
  - name: pharma-service
    image: 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-a0017b8
    ports:
      - containerPort: 8081
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true   # container can't write to its own filesystem
      runAsNonRoot: true
      runAsUser: 1000
    volumeMounts:
      - name: tmp
        mountPath: /tmp              # /tmp must be writable — use emptyDir
volumes:
  - name: tmp
    emptyDir: {}
```

**Why readOnlyRootFilesystem?** Security best practice. Prevents a compromised container from modifying its own binaries. The `tmp` emptyDir is needed because Spring Boot writes temp files to /tmp.

---

#### 5. Service (ClusterIP)

**What it is:** A stable network endpoint for a set of pods. Pods are ephemeral (they restart with new IPs) — Service gives them a permanent address.

**In this project:**
```yaml
service:
  type: ClusterIP
  port: 8081
  targetPort: 8081
```

**DNS inside the cluster:** `auth-service.dev.svc.cluster.local:8081`
**Short form (within same namespace):** `http://auth-service:8081`

**How other services talk to auth-service:**
```
api-gateway pod → http://auth-service:8081/auth/validate → auth-service pod
```

**Types of Services:**
| Type | When to use |
|---|---|
| ClusterIP | Internal service (most microservices) |
| NodePort | Expose on each node's IP (dev/testing only) |
| LoadBalancer | Expose to internet via cloud load balancer |
| Headless | Direct pod DNS, used for StatefulSets |

In this project, all microservices use ClusterIP. External access comes through Ingress.

---

#### 6. ConfigMap

**What it is:** Non-sensitive configuration injected into pods as environment variables or mounted files.

**In this project (auth-service dev):**
```yaml
configmap:
  SPRING_PROFILES_ACTIVE: dev
  DB_HOST: pharma-dev-postgres.cs3c424yurej.us-east-1.rds.amazonaws.com
  LOG_LEVEL: DEBUG
  SERVER_PORT: "8081"
  TOKEN_EXPIRATION_MS: "86400000"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,metrics,prometheus"
```

**Rule of thumb:** If you'd be OK putting it in a GitHub README, put it in ConfigMap. If not, use Secret.

---

#### 7. Secret

**What it is:** Sensitive data (passwords, tokens, keys). Stored base64 encoded in etcd.

**Important:** Base64 is NOT encryption. Secrets are not secure just because they're in a Secret object. Proper secrets management requires encryption at rest (KMS) + never storing them in Git.

**In this project (auth-service needs):**
- `db-credentials`: DB_USERNAME, DB_PASSWORD
- `jwt-secret`: JWT_SECRET

**Injected into the pod via:**
```yaml
envFrom:
  - secretRef:
      name: db-credentials
  - secretRef:
      name: jwt-secret
```

All keys from both Secrets become environment variables in the container.

**In Session 3:** We'll replace manually created Secrets with External Secrets Operator pulling from AWS Secrets Manager.

---

#### 8. ServiceAccount

**What it is:** A Kubernetes identity for pods. Used for RBAC within the cluster and (in AWS) for IRSA — letting pods call AWS APIs without static credentials.

**In this project:**
```yaml
serviceAccount:
  create: true
  name: auth-service
```

**Daily DevOps relevance:** If you see `Error from server (Forbidden)` when a pod tries to call the K8s API, the pod's ServiceAccount doesn't have the right permissions.

---

#### 9. Ingress

**What it is:** Routes external HTTP/HTTPS traffic to internal Services. Think of it as a reverse proxy configured via K8s objects.

**In this project:** NGINX Ingress Controller runs as pods inside the cluster. A cloud LoadBalancer points to the NGINX pods. NGINX reads Ingress rules to know where to forward requests.

```yaml
ingress:
  enabled: false          # disabled in dev (internal only)
  className: nginx
  host: dev.pharma.internal
  path: /auth
  pathType: Prefix
```

**When enabled:** `dev.pharma.internal/auth` → `auth-service:8081`

---

#### 10. HPA (Horizontal Pod Autoscaler)

**What it is:** Automatically scales the number of pods up or down based on CPU or memory usage.

**In this project:**
- dev: disabled (`autoscaling.enabled: false`)
- qa: 1–3 replicas, scales when CPU > 70%
- prod: 2–5 replicas, scales when CPU > 70% or memory > 80%

Today we note it exists. In Session 2 we'll see how Helm activates it per environment by flipping one value.

---

#### 11. RBAC — Role + RoleBinding

**What it is:** Role-Based Access Control. Controls who can do what in which namespace.

**In this project:**
```yaml
# k8s/rbac/dev-role.yaml
kind: Role
metadata:
  name: pharma-deployer
  namespace: dev
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]         # read-only for pods
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

**RoleBinding:** Binds the Role to a subject (user, group, or ServiceAccount).

**ClusterRole vs Role:**
- `Role`: scoped to one namespace
- `ClusterRole`: cluster-wide, can see resources across all namespaces

---

### Part 3 — Network Flow (10 min)

How does a request from outside reach auth-service?

```
External Client (browser / mobile app)
         │
         ▼
AWS Load Balancer  (created automatically by NGINX Ingress Controller)
         │
         ▼
NGINX Ingress Controller Pod  (running in ingress-nginx namespace)
  → reads Ingress rules:
      dev.pharma.internal/auth  →  auth-service:8081
         │
         ▼
auth-service  Service (ClusterIP: 10.96.x.x:8081)
  → kube-proxy routes to one of the running pods via iptables
         │
         ▼
auth-service Pod  (10.0.x.x:8081)
  → Spring Boot handles the request
  → reads DB_HOST from ConfigMap, DB_PASSWORD from Secret
  → connects to RDS PostgreSQL
```

**Key questions to be able to answer:**
1. Why ClusterIP for auth-service and not LoadBalancer?
   - We don't want auth-service exposed directly to the internet. Only Ingress is exposed.
2. If auth-service has 3 pods, which one gets the request?
   - kube-proxy uses round-robin (iptables rules), so load is distributed randomly.
3. How does the api-gateway pod find auth-service?
   - By DNS: `http://auth-service.dev.svc.cluster.local:8081` or just `http://auth-service:8081` within the same namespace.

---

### Part 4 — Live Deploy: auth-service Raw Manifests (35 min)

We deploy auth-service to EKS without any automation. The hard way — so you understand what Helm and ArgoCD do for you in Session 2.

```bash
# ─────────────────────────────────────────
# Step 1: Verify EKS connection
# ─────────────────────────────────────────
kubectl cluster-info
kubectl get nodes

# ─────────────────────────────────────────
# Step 2: Create namespace
# ─────────────────────────────────────────
kubectl apply -f k8s/namespaces.yaml
kubectl get namespaces

# ─────────────────────────────────────────
# Step 3: Apply RBAC
# ─────────────────────────────────────────
kubectl apply -f k8s/rbac/dev-role.yaml
kubectl apply -f k8s/rbac/rolebindings.yaml
kubectl get roles -n dev
kubectl get rolebindings -n dev

# ─────────────────────────────────────────
# Step 4: Create Secrets manually
# (Session 3 will automate this with External Secrets Operator)
# ─────────────────────────────────────────
kubectl create secret generic db-credentials \
  --from-literal=DB_USERNAME=pharma_user \
  --from-literal=DB_PASSWORD=pharmaPass123 \
  -n dev

kubectl create secret generic jwt-secret \
  --from-literal=JWT_SECRET=mysupersecretjwtkey256bitslongkey \
  -n dev

kubectl get secrets -n dev
# Verify — values are base64 encoded
kubectl get secret db-credentials -n dev -o jsonpath='{.data.DB_USERNAME}' | base64 -d && echo

# ─────────────────────────────────────────
# Step 5: Create ConfigMap
# ─────────────────────────────────────────
kubectl create configmap auth-service \
  --from-literal=SPRING_PROFILES_ACTIVE=dev \
  --from-literal=DB_HOST=pharma-dev-postgres.cs3c424yurej.us-east-1.rds.amazonaws.com \
  --from-literal=LOG_LEVEL=DEBUG \
  --from-literal=SERVER_PORT=8081 \
  --from-literal=TOKEN_EXPIRATION_MS=86400000 \
  --from-literal=MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=health,info,metrics,prometheus \
  -n dev

kubectl get configmap auth-service -n dev -o yaml

# ─────────────────────────────────────────
# Step 6: Create ServiceAccount
# ─────────────────────────────────────────
kubectl create serviceaccount auth-service -n dev

# ─────────────────────────────────────────
# Step 7: Write and apply the Deployment manifest
# Write this in class to reinforce understanding
# ─────────────────────────────────────────
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      serviceAccountName: auth-service
      containers:
        - name: auth-service
          image: 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-a0017b8
          imagePullPolicy: Always
          ports:
            - containerPort: 8081
          envFrom:
            - configMapRef:
                name: auth-service
            - secretRef:
                name: db-credentials
            - secretRef:
                name: jwt-secret
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8081
            initialDelaySeconds: 60
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
EOF

# ─────────────────────────────────────────
# Step 8: Expose via Service
# ─────────────────────────────────────────
kubectl expose deployment auth-service \
  --port=8081 --target-port=8081 \
  --type=ClusterIP \
  -n dev

# ─────────────────────────────────────────
# Step 9: Verify everything
# ─────────────────────────────────────────
kubectl get all -n dev
kubectl get pods -n dev -w          # watch pods come up

# Describe pod — understand every field
kubectl describe pod -l app=auth-service -n dev

# Read logs
kubectl logs -l app=auth-service -n dev

# Test health endpoint from inside the pod
kubectl exec -n dev deploy/auth-service -- \
  curl -s http://localhost:8081/actuator/health | python3 -m json.tool

kubectl exec -n dev deploy/auth-service -- \
  curl -s http://localhost:8081/actuator/health/readiness
```

**What to observe:**
- Pod starts in `ContainerCreating` → `Running` → readiness probe passes → `READY: 1/1`
- If readiness fails, Service gets NO endpoints (no traffic routed to the pod)
- `kubectl describe pod` is your best friend — read the Events section every time

---

### Part 5 — What's Wrong With This Approach? (5 min)

| Problem | Impact |
|---|---|
| 9 services × 3 envs = 27 manual deployments | One typo breaks production |
| No Git record of changes | No audit trail — who changed the image tag? |
| Config drift over time | Dev and prod slowly diverge with no one noticing |
| No rollback mechanism | If something breaks, you manually type the old values |
| Manual secret management | Easy to use wrong values per environment |

**Session 2 solves all of this with ArgoCD + Helm.**

**Homework:** Replicate this exact deployment on your own EKS cluster. Make the pod healthy. Hit the `/actuator/health` endpoint. Document every command you ran.

---

# SESSION 2

## "Week 1 — kubectl Doesn't Scale. Enter ArgoCD + Helm"

**Duration:** 1.5 hours  
**Objective:** Understand GitOps principles, Helm chart model, and ArgoCD. Deploy auth-service via ArgoCD.

---

### Part 1 — The Problem (5 min)

**The Story**

Your manager: "Great job with auth-service. Now deploy all 9 services to dev. Then qa. Then prod. And next week there will be a new Docker image for each service. And the week after. And every time there's a code change."

Let's count: 9 services × 3 environments = **27 deployments** just for the initial rollout.
Each deployment: namespace check, secrets, configmap, deployment yaml, service yaml, verify.

Then image updates. Then config changes. Then rollbacks when something breaks.

**There has to be a better way.**

---

### Part 2 — GitOps Principles (10 min)

GitOps is a set of practices where Git is the single source of truth for what should run in your cluster.

**The 4 GitOps principles:**

1. **Declarative** — You describe WHAT you want, not HOW to get there.
   - Before: `kubectl scale deployment auth-service --replicas=3`
   - After: `replicaCount: 3` in Git → ArgoCD applies it

2. **Versioned and immutable** — All desired state is stored in Git. Every change has a commit hash, author, timestamp.

3. **Pulled automatically** — An operator (ArgoCD) continuously pulls from Git and applies changes. You don't push to the cluster.

4. **Continuously reconciled** — If the cluster drifts from Git (someone manually changes something), ArgoCD detects it and can revert automatically.

**What changes in your daily workflow:**
| Before GitOps | After GitOps |
|---|---|
| `kubectl apply -f deployment.yaml` | Edit the YAML file, open a PR |
| `kubectl set image deployment/auth-service ...` | Update `image.tag` in values file, merge the PR |
| Rollback: find old manifest, reapply | `git revert <commit>`, merge PR |
| "Who changed the replica count?" | `git log envs/prod/values-auth-service.yaml` |

---

### Part 3 — Helm Deep-Dive (15 min)

**The problem Helm solves:** 9 services × 3 environments = 27 nearly identical YAML files. Every Deployment has the same structure — only the image, port, and config values differ.

**Solution:** One parameterized chart, with values files per service per environment.

**How this project uses Helm:**

```
helm-charts/values.yaml                     ← defaults for all services
      +
envs/dev/values-auth-service.yaml           ← auth-service dev overrides
      =
Final Kubernetes manifests for auth-service in dev namespace
```

**Walk through the Helm chart structure:**

```
helm-charts/
├── Chart.yaml          ← chart metadata (name, version)
├── values.yaml         ← default values (replicas: 1, resource limits, probe paths)
└── templates/
    ├── _helpers.tpl    ← reusable template functions (fullname, labels)
    ├── deployment.yaml ← Go template — {{ .Values.image.tag }}, {{ .Values.replicaCount }}
    ├── service.yaml    ← uses {{ .Values.service.port }}
    ├── configmap.yaml  ← renders {{ .Values.configmap }} as key-value pairs
    ├── ingress.yaml    ← conditional: {{- if .Values.ingress.enabled }}
    ├── serviceaccount.yaml
    └── hpa.yaml        ← conditional: {{- if .Values.autoscaling.enabled }}
```

**Key Go template syntax to know:**
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
{{- if not .Values.autoscaling.enabled }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

**See what Helm renders:**
```bash
# Preview what manifests Helm would produce
helm template auth-service ./helm-charts \
  -f envs/dev/values-auth-service.yaml \
  -n dev

# Compare with what we wrote manually in Session 1
# They should be identical (or very close)
```

**Why one chart for all 9 services?**
- Add a PodDisruptionBudget template → all 9 services get it automatically
- Change default resource limits → all services pick it up
- Service-specific differences stay in their own values file

---

### Part 4 — ArgoCD Architecture (10 min)

**ArgoCD components:**

```
┌─────────────────────────────────────────────────────┐
│  ArgoCD (installed in argocd namespace)              │
│                                                      │
│  ┌──────────────┐  ┌──────────────────────────────┐ │
│  │  api-server  │  │  repo-server                 │ │
│  │  (UI + CLI)  │  │  - clones Git repo           │ │
│  └──────────────┘  │  - renders Helm templates    │ │
│                    │  - caches rendered manifests  │ │
│  ┌──────────────┐  └──────────────────────────────┘ │
│  │ application  │  ┌──────────────────────────────┐ │
│  │ controller   │  │  dex (SSO / OIDC)            │ │
│  │ (reconciles) │  └──────────────────────────────┘ │
│  └──────────────┘                                   │
└─────────────────────────────────────────────────────┘
         │
         │  watches (polls every 3 min or via webhook)
         ▼
   zen-gitops repo (GitHub)
         │
         │  compares rendered manifests with live cluster state
         ▼
   EKS Cluster (dev / qa / prod namespaces)
```

**The reconciliation loop:**
1. repo-server polls Git every 3 minutes
2. repo-server runs `helm template` with your values file
3. application-controller compares rendered manifests with live cluster state
4. If different → status: **OutOfSync**
5. If automated sync is on → applies the diff automatically
6. If `selfHeal: true` → reverts manual cluster changes back to Git state

**Key ArgoCD concepts:**

| Concept | What it is | In this project |
|---|---|---|
| **AppProject** | Security boundary — which repos, namespaces, resources ArgoCD can touch | `argocd/projects/pharma-project.yaml` |
| **Application** | Links Git source + Helm values → destination namespace | `argocd/apps/dev/auth-service-app.yaml` |
| **Synced** | Cluster matches Git | Healthy state |
| **OutOfSync** | Cluster differs from Git | Needs sync |
| **Degraded** | Resources are not healthy (pods crashing) | Investigate pods |
| **selfHeal** | Auto-revert manual cluster changes | Enabled in dev |
| **prune** | Delete K8s resources removed from Git | Enabled in dev |

---

### Part 5 — Live Demo (25 min)

```bash
# ─────────────────────────────────────────
# Step 1: Install ArgoCD
# ─────────────────────────────────────────
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=300s

kubectl get pods -n argocd

# ─────────────────────────────────────────
# Step 2: Access the ArgoCD UI
# ─────────────────────────────────────────
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open browser: https://localhost:8080

# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo

# ─────────────────────────────────────────
# Step 3: Login via ArgoCD CLI
# ─────────────────────────────────────────
argocd login localhost:8080 \
  --username admin \
  --password <password-from-above> \
  --insecure

# ─────────────────────────────────────────
# Step 4: Create the pharma AppProject
# ─────────────────────────────────────────
# Update the repoURL in pharma-project.yaml with your GitHub username first
kubectl apply -f argocd/projects/pharma-project.yaml

# Verify project created
argocd proj list

# ─────────────────────────────────────────
# Step 5: Create auth-service Application
# ─────────────────────────────────────────
# Update the repoURL in auth-service-app.yaml with your GitHub username
kubectl apply -f argocd/apps/dev/auth-service-app.yaml

# Watch ArgoCD start syncing
argocd app list
argocd app get auth-service-dev

# ─────────────────────────────────────────
# Step 6: Trigger first sync
# ─────────────────────────────────────────
argocd app sync auth-service-dev

# Watch pods come up
kubectl get pods -n dev -w

# Compare with Session 1: ArgoCD did everything we did manually — in seconds
# ─────────────────────────────────────────
# Step 7: Simulate drift
# (Someone panics at 2am and manually changes replicas)
# ─────────────────────────────────────────
kubectl scale deployment auth-service --replicas=3 -n dev

# Watch ArgoCD detect OutOfSync within 3 minutes
argocd app get auth-service-dev
# STATUS: OutOfSync

# Show the diff in UI (desired: 1 replica, live: 3 replicas)
argocd app diff auth-service-dev

# With selfHeal: true, ArgoCD reverts automatically
# Or manually sync to force it back to Git state:
argocd app sync auth-service-dev
kubectl get pods -n dev   # back to 1 replica
```

---

### Part 6 — ArgoCD UI Walkthrough (15 min)

Walk through the ArgoCD UI live while the application is running:

1. **Applications list** — health status (heart icon), sync status, last sync time
2. **Application detail** — the resource tree:
   ```
   Application (auth-service-dev)
   └── Deployment (auth-service)
       └── ReplicaSet (auth-service-7d9f8c)
           └── Pod (auth-service-7d9f8c-xk2pv)  ← click to see logs, events
   └── Service (auth-service)
   └── ConfigMap (auth-service)
   └── ServiceAccount (auth-service)
   ```
3. **Diff view** — what would change if you sync now (shows + and - lines like git diff)
4. **Sync options:**
   - Dry run: show what would happen without applying
   - Force: overwrite even if there are conflicts
   - Prune: delete resources removed from Git
5. **History & rollback** — list of all previous syncs, each with a Git commit hash
6. **App logs** — click any pod → view logs directly in the UI
7. **Projects page** — pharma project: source repos, allowed namespaces, RBAC roles

**RBAC in the pharma AppProject:**
```yaml
# argocd/projects/pharma-project.yaml
roles:
  - name: pharma-admin
    policies:
      - p, proj:pharma:pharma-admin, applications, *, pharma/*, allow
    groups:
      - pharma-devops-team
```

This means: members of `pharma-devops-team` can do anything to any application in the `pharma` project. In Session 3 we'll see how prod gets a stricter policy.

---

### Part 7 — Homework (5 min)

1. Install ArgoCD on your own EKS cluster
2. Fork zen-gitops, update the repoURLs with your GitHub username
3. Create the pharma AppProject and auth-service-dev Application
4. Watch ArgoCD sync auth-service
5. Simulate drift (scale manually) and observe OutOfSync
6. Try adding `selfHeal: true` to the auth-service-app.yaml and repeat drift simulation

---

# SESSION 3

## "Week 2 — Ship All 9 Services. Wire Up the Full CD Pipeline"

**Duration:** 1.5 hours  
**Objective:** Deploy all services across dev/qa/prod. Understand how CI feeds GitOps. Set up External Secrets.

---

### Part 1 — Recap + Problem (5 min)

Auth-service is live via ArgoCD. Now the team needs:
- All 9 services deployed across dev, qa, and prod
- Secrets pulled from AWS (not stored in Git)
- Automatic deployment when CI builds a new Docker image
- QA approval before anything goes to prod

---

### Part 2 — Multi-Environment Architecture (10 min)

Different environments have different deployment strategies — by design.

| Environment | App Structure | Sync Policy | Who Triggers |
|---|---|---|---|
| dev | 9 individual Applications (one per service) | Automated + prune | CI commits image tag → ArgoCD auto-syncs |
| qa | 1 `pharma-qa` app-of-apps | Automated + prune | QA promotion PR merged |
| prod | 1 `pharma-prod` app-of-apps | Manual sync only | Release manager clicks sync in ArgoCD UI |

**App-of-Apps pattern (qa / prod):**

In QA and prod, one ArgoCD Application manages ALL services together:

```
pharma-qa Application (watches envs/qa/ directory)
├── api-gateway      (reads envs/qa/values-api-gateway.yaml)
├── auth-service     (reads envs/qa/values-auth-service.yaml)
├── catalog-service  (reads envs/qa/values-catalog-service.yaml)
└── ... (all 9 services)
```

**Why individual apps in dev but app-of-apps in qa/prod?**
- dev: granular control — you can sync only auth-service without touching catalog-service
- qa/prod: atomic deployments — all services update together in one sync, ensuring compatibility

**Environment differences (from README):**

| Setting | dev | qa | prod |
|---|---|---|---|
| replicaCount | 1 | 1 | 2 |
| Autoscaling | disabled | 1–3 replicas | 2–5 replicas |
| LOG_LEVEL | DEBUG | INFO | WARN |
| CPU request/limit | 100m / 500m | 150m / 500m | 250m / 1000m |
| Memory limit | 512Mi | 512Mi | 1Gi |
| podAntiAffinity | no | no | yes (pods spread across nodes) |

These differences live in the per-environment values files. The Helm chart itself is identical.

---

### Part 3 — How CI Feeds CD (15 min)

The most common interview question: **"How does a code change get to production?"**

```
Developer pushes code to zen-pharma-backend repo
         │
         ▼
CI Pipeline (GitHub Actions)
  1. Build: docker build -t auth-service:sha-a1b2c3d .
  2. Test:  run unit + integration tests
  3. Scan:  Trivy image vulnerability scan
  4. Push:  docker push 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-a1b2c3d
         │
         ▼
  5. Update zen-gitops repo:
     yq e '.image.tag = "sha-a1b2c3d"' -i envs/dev/values-auth-service.yaml
     git add envs/dev/values-auth-service.yaml
     git commit -m "ci(dev): update auth-service -> sha-a1b2c3d"
     git push origin promote/dev/auth-service/sha-a1b2c3d
     # Open PR → auto-merge for dev (no approval required)
         │
         ▼
ArgoCD (watching zen-gitops main branch)
  6. Detects change in envs/dev/values-auth-service.yaml
  7. Renders: helm template + dev values
  8. Applies diff to dev namespace
  9. auth-service rolls out new pod with sha-a1b2c3d image
```

**CI and CD are intentionally decoupled:**
- CI owns: build, test, scan, push image to ECR
- CD owns: update Git, apply to cluster
- The only handoff: CI writes the image tag to a YAML file in zen-gitops

**Promote branch pattern:**
```bash
# Branch names encode: promote/<env>/<service>/<image-sha>
promote/dev/auth-service/sha-a1b2c3d    → auto-merged (no approval)
promote/qa/auth-service/sha-a1b2c3d    → QA team approves PR
promote/prod/auth-service/sha-a1b2c3d  → 2 senior approvals required
```

**Simulate a CI image tag update (do this in class):**
```bash
# What CI does after building a new image
yq e '.image.tag = "sha-demo123"' -i envs/dev/values-auth-service.yaml
git add envs/dev/values-auth-service.yaml
git commit -m "ci(dev): update auth-service -> sha-demo123"
git push

# ArgoCD detects the change and syncs dev within 3 minutes
argocd app get auth-service-dev --watch
# STATUS changes from Synced → OutOfSync → Synced (auto-sync)
kubectl get pods -n dev
# New pod with sha-demo123 image rolling out
```

---

### Part 4 — Live Demo: All 9 Services (25 min)

```bash
# ─────────────────────────────────────────
# Deploy all dev services at once
# ─────────────────────────────────────────
for app in argocd/apps/dev/*.yaml; do
  kubectl apply -f $app
done

# Watch all 9 apps appear in ArgoCD
argocd app list

# Watch all pods come up in dev namespace
kubectl get pods -n dev -w

# ─────────────────────────────────────────
# Deploy QA (app-of-apps)
# ─────────────────────────────────────────
kubectl apply -f argocd/apps/qa/pharma-qa-app.yaml
kubectl get pods -n qa -w

# ─────────────────────────────────────────
# Deploy Prod (app-of-apps, manual sync)
# ─────────────────────────────────────────
kubectl apply -f argocd/apps/prod/pharma-prod-app.yaml

# Note: prod does NOT sync automatically
argocd app get pharma-prod
# STATUS: OutOfSync (waiting for manual trigger)

# Release manager manually triggers sync in UI — or via CLI:
argocd app sync pharma-prod
kubectl get pods -n prod -w

# ─────────────────────────────────────────
# Simulate CI image tag update
# ─────────────────────────────────────────
yq e '.image.tag = "sha-classdemo"' -i envs/dev/values-auth-service.yaml
git add . && git commit -m "demo: update auth-service tag" && git push

# Watch ArgoCD detect and sync automatically
argocd app get auth-service-dev --watch
kubectl get pods -n dev
# Old pod terminating, new pod starting with sha-classdemo
```

---

### Part 5 — External Secrets Operator (15 min)

**Why secrets can't be in Git:**
- Git history is permanent — even if you delete a secret commit, it's recoverable
- Anyone with read access to the repo can read production passwords
- Violates SOC2, PCI-DSS, HIPAA, and basic security hygiene
- In pharma (regulated industry): non-negotiable

**The solution: External Secrets Operator (ESO)**

ESO is a Kubernetes operator that watches ExternalSecret objects and creates K8s Secrets by fetching the actual values from AWS Secrets Manager.

**Full secret flow — 5 layers:**

```
AWS Secrets Manager
  /pharma/dev/db-credentials  →  {"username": "pharma_user", "password": "s3cr3t"}
  /pharma/dev/jwt-secret      →  {"secret": "hs512-key-..."}
         ▲
         │  GetSecretValue  (authenticated via IRSA — no static credentials)
         │
ESO Controller Pod (in kube-system)
  reads ClusterSecretStore  →  knows HOW to connect to AWS
  reads ExternalSecret CRs  →  knows WHAT to fetch and WHERE to put it
  writes K8s Secrets         →  every refreshInterval (1 hour)
         │
         │  creates/updates
         ▼
K8s Secrets (dev namespace)
  db-credentials:  {DB_USERNAME: pharma_user, DB_PASSWORD: s3cr3t}
  jwt-secret:      {JWT_SECRET: hs512-key-...}
         │
         │  envFrom: secretRef
         ▼
auth-service Pod
  env: DB_USERNAME=pharma_user, DB_PASSWORD=s3cr3t, JWT_SECRET=hs512-key-...
```

**IRSA (IAM Roles for Service Accounts):**

ESO never stores AWS credentials. Instead:
1. EKS cluster has an OIDC issuer URL
2. ESO's ServiceAccount is annotated with an IAM role ARN
3. K8s injects a short-lived JWT token into the ESO pod
4. ESO presents the JWT to AWS STS → gets temporary credentials (15-60 min TTL)
5. Uses temp credentials to call Secrets Manager

```yaml
# k8s/external-secrets/cluster-secret-store.yaml
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets      # the SA with the IAM role annotation
            namespace: kube-system
```

**Deploy and verify:**
```bash
# Install ESO
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n kube-system

# Apply ClusterSecretStore (defines connection to AWS)
kubectl apply -f k8s/external-secrets/cluster-secret-store.yaml

# Apply ExternalSecrets for dev namespace
kubectl apply -f k8s/external-secrets/dev-external-secrets.yaml

# Check sync status
kubectl get externalsecret -n dev
# NAME             STATUS        READY
# db-credentials   SecretSynced  True
# jwt-secret       SecretSynced  True

# Verify K8s Secret was created with actual values
kubectl get secret db-credentials -n dev
kubectl get secret db-credentials -n dev \
  -o jsonpath='{.data.DB_USERNAME}' | base64 -d && echo
# pharma_user
```

---

### Part 6 — Dev→QA→Prod Promotion (5 min)

**CODEOWNERS (who must approve changes per path):**
```bash
# .github/CODEOWNERS
/envs/dev/       @pharma-dev-team
/envs/qa/        @pharma-qa-team
/envs/prod/      @pharma-release-managers
/helm-charts/    @pharma-platform-team
```

**Branch protection on main:**
- PRs touching `envs/prod/**` require 2 approvals from `@pharma-release-managers`
- No direct push to main — everything goes through a PR

**ArgoCD RBAC:**
- dev team members: cannot sync prod applications
- release managers only: can trigger prod sync

**The complete promotion flow:**
```
sha-a1b2c3d image built by CI
      │
      ├── promote/dev/auth-service/sha-a1b2c3d  →  auto-merge  →  ArgoCD auto-sync dev
      │
      ├── QA validation passes
      │
      ├── promote/qa/auth-service/sha-a1b2c3d   →  QA approval  →  ArgoCD auto-sync qa
      │
      └── Release sign-off
          promote/prod/auth-service/sha-a1b2c3d →  2 approvals →  ArgoCD manual sync prod
```

---

### Part 7 — Homework (5 min)

1. Deploy all 9 services to your EKS cluster (dev + qa + prod)
2. Simulate a CI image tag update and watch ArgoCD auto-sync dev
3. Install ESO and verify db-credentials syncs from AWS Secrets Manager
4. Practice the promote flow: manually update qa values file, merge PR, observe ArgoCD sync qa

---

# SESSION 4

## "The Interview — Defend What You Built"

**Duration:** 1.5 hours  
**Format:** Mock interview — students answer first (2 minutes each), then model answer + simulation is shown.  
**Objective:** Students can answer any GitOps/K8s interview question by referencing their own project.

---

### Part 1 — How DevOps Interviews Work (10 min)

**What MNC interviewers actually test:**

1. **Understanding, not memorization.** They'll ask "why" after every answer. "We use ArgoCD" → "Why ArgoCD over Flux?" Know the reason behind every decision.

2. **Real project experience.** Generic textbook answers fail. "In our Zen Pharma setup, when ArgoCD detects a change in the zen-gitops repo, it renders the Helm chart with the dev values file and applies only the diff" is 10x better than "ArgoCD watches a Git repo."

3. **Systematic thinking.** For troubleshooting questions, they watch HOW you think. Do you jump to conclusions or narrow down methodically?

4. **Business impact awareness.** "selfHeal: true on prod means compliance — the cluster always matches what was approved in Git" lands better than just "it reverts drift."

**The golden rule for every answer:** Start with the concept, then say "In our project, specifically…" and reference something from zen-gitops.

**Format for troubleshooting answers:**
1. What does this error mean? (one sentence definition)
2. What are the possible causes? (list)
3. How do you narrow down? (exact kubectl commands)
4. What's the fix?

---

### Part 2 — Mock Interview Q&A (65 min)

---

#### Q1: Walk me through your GitOps pipeline end-to-end

**What the interviewer is testing:** Do you understand the full pipeline or just one slice? Can you explain CI vs CD separation clearly?

**Model Answer:**

Our pipeline has two distinct halves — CI (build, test, publish) and CD (deploy via GitOps). They are intentionally decoupled.

```
Developer pushes code to zen-pharma-backend
         │
CI Pipeline
  1. docker build → auth-service:sha-a1b2c3d
  2. Unit + integration tests
  3. Trivy image scan
  4. docker push → AWS ECR
  5. yq updates envs/dev/values-auth-service.yaml: tag: sha-a1b2c3d
  6. git commit + push → PR to zen-gitops
  7. PR auto-merges (for dev)
         │
ArgoCD (watching zen-gitops main branch)
  8. Detects change in values-auth-service.yaml
  9. Runs: helm template pharma-service + dev values
  10. Applies diff to dev namespace
  11. Kubernetes rolling update: new pod up, old pod down
         │
Promotion to QA (after dev validation)
  12. Same tag promoted via PR to envs/qa/
  13. QA team approves PR
  14. ArgoCD auto-syncs qa namespace
         │
Promotion to Prod (after QA sign-off)
  15. PR to envs/prod/ (2 senior approvals + CODEOWNERS)
  16. Merge → ArgoCD detects change
  17. Release manager manually triggers sync in ArgoCD UI
  18. Watches rollout: kubectl rollout status deployment/auth-service -n prod
```

**5 GitOps principles to state:**
1. Git is the single source of truth — no one runs `kubectl apply` manually in prod
2. CI and CD are separated — CI owns build/push; CD is triggered by Git changes
3. Declarative — the repo describes desired state, not imperative steps
4. Auditable — every change has a PR, reviewer, and merge commit
5. Rollback = `git revert` — no special tooling, just change Git

---

#### Q2: How does ArgoCD detect and handle configuration drift?

**What the interviewer is testing:** Understanding of desired state vs live state reconciliation. How selfHeal and prune work.

**Model Answer:**

Drift = the cluster's live state no longer matches what Git says it should be.

**How ArgoCD detects drift:**
- Polls Git every 3 minutes (or via webhook for instant detection)
- repo-server renders Helm templates → produces desired manifests
- application-controller compares desired vs live using a three-way diff
- If they differ → status: OutOfSync

**Scenario: someone runs `kubectl scale deployment auth-service --replicas=3 -n dev`**

With `selfHeal: false` (prod default):
- ArgoCD detects drift within 3 minutes
- Status → OutOfSync, shows the diff in UI
- Cluster stays drifted until an operator manually syncs

With `selfHeal: true` (dev in this project):
- ArgoCD detects drift within 3 minutes
- Immediately reverts: applies the manifest from Git
- Replicas go back to 1 automatically

**Key insight for pharma/regulated environments:**
> "selfHeal on prod is compliance tooling, not just convenience. It guarantees the cluster always reflects what's in Git, which is what your change management system approved. If someone manually patches prod, selfHeal ensures that patch doesn't survive the next reconciliation loop."

---

#### Q3: How do you promote a change from dev → qa → prod?

**Model Answer:**

We use a promote branch pattern — branch names encode the environment, service, and image SHA:

```
promote/dev/auth-service/sha-a1b2c3d   → auto-merge  → ArgoCD auto-sync dev
promote/qa/auth-service/sha-a1b2c3d    → QA approves → ArgoCD auto-sync qa
promote/prod/auth-service/sha-a1b2c3d  → 2 approvals → ArgoCD manual sync
```

Enforcement:
- `.github/CODEOWNERS`: prod paths require release-manager approval
- Branch protection: 2 reviewers required for envs/prod/** PRs
- ArgoCD sync policy: prod = manual sync (human gate before apply)
- ArgoCD RBAC: only release managers can trigger prod sync

**What's different per environment in our values files:**

| Setting | dev | qa | prod |
|---|---|---|---|
| Image tag | sha-latest | sha-tested | sha-signed-off |
| LOG_LEVEL | DEBUG | INFO | WARN |
| replicaCount | 1 | 1 | 2 |
| Sync policy | auto | auto | manual |

---

#### Q4: How do you do a rollback in GitOps?

**Two methods — they are NOT equivalent.**

**Method 1: ArgoCD UI rollback (fast, temporary)**
```bash
argocd app history auth-service-dev
# ID  DATE                REVISION
# 5   2026-05-14 10:00    main (abc123)  ← current (broken)
# 4   2026-05-13 15:00    main (def456)  ← last good

argocd app rollback auth-service-dev 4
# ArgoCD re-applies revision 4 manifests to cluster
# STATUS: OutOfSync (Git still shows broken state)
```

Use when: immediate production fire, need cluster stable in 60 seconds.

Danger: Git still has the broken commit. If someone triggers a sync, broken version comes back. You MUST follow up with a git revert.

**Method 2: Git revert (permanent, compliant)**
```bash
git log --oneline envs/dev/values-auth-service.yaml
# abc123 ci(dev): update auth-service → sha-broken
# def456 ci(dev): update auth-service → sha-good

git revert abc123 --no-edit
git push origin revert/dev-auth-service-rollback
# Open PR, merge
# ArgoCD auto-syncs → cluster back to sha-good
```

Use when: after the immediate fire is out. This is the real rollback — it's what your audit trail references.

**Decision flow:**
```
Production broken right now?
  YES → argocd app rollback <app> <revision>  (60 seconds)
        THEN → git revert → PR → merge  (15 minutes, creates audit trail)
  NO  → git revert → PR → merge  (preferred path, audit-compliant)
```

---

#### Q5: What is IRSA and why not use static AWS credentials?

**Model Answer:**

IRSA (IAM Roles for Service Accounts) lets Kubernetes pods assume AWS IAM roles without storing any AWS credentials in the cluster.

**The problem with static credentials:**
```yaml
# DO NOT DO THIS
apiVersion: v1
kind: Secret
data:
  AWS_ACCESS_KEY_ID: AKIAIOSFODNN7EXAMPLE       # permanent key
  AWS_SECRET_ACCESS_KEY: <base64-encoded-key>   # if leaked, permanent access
```

Problems: long-lived (permanent until manually rotated), no scope (any pod gets same access), stored in etcd (exploitable if cluster is compromised).

**How IRSA works:**
1. EKS cluster has an OIDC issuer URL
2. IAM role has a trust policy: "only allow the `external-secrets` ServiceAccount in `kube-system`"
3. K8s injects a short-lived JWT token into the ESO pod
4. ESO calls AWS STS with the JWT → gets temporary credentials (15-60 min TTL)
5. Uses temp credentials to call Secrets Manager

**Why it's better:**

| | Static Credentials | IRSA |
|---|---|---|
| Credential lifetime | Permanent | 15–60 minutes |
| Scope | Any pod with the secret | Specific namespace + SA name only |
| Storage | In a K8s Secret | Never stored — generated on demand |
| Audit trail | Access key ID | Role + cluster + SA identity in CloudTrail |
| Blast radius | Full AWS account | Only that role's permissions, for max 1h |

---

#### Q6: Walk me through how a secret gets from AWS Secrets Manager into a running pod

**Model Answer — 5 layers:**

```
Layer 1: IRSA authentication
ESO pod presents K8s SA JWT to AWS STS
→ AWS validates JWT against cluster OIDC issuer
→ AWS returns temporary credentials
→ ESO uses temp creds to call Secrets Manager

Layer 2: ClusterSecretStore defines the connection
spec.provider.aws.service: SecretsManager
spec.provider.aws.region: us-east-1
spec.provider.aws.auth.jwt.serviceAccountRef: external-secrets (kube-system)

Layer 3: ExternalSecret defines what to fetch
spec.secretStoreRef: aws-secrets-manager
spec.target.name: db-credentials  ← K8s Secret name to create
spec.data[0].remoteRef.key: /pharma/dev/db-credentials  ← AWS Secrets Manager path
spec.data[0].remoteRef.property: password  ← JSON field within the secret

Layer 4: ESO reconciler creates the K8s Secret
Every refreshInterval (1h):
  GetSecretValue(/pharma/dev/db-credentials) → {"username": "pharma_user", "password": "s3cr3t"}
  Creates/updates K8s Secret: db-credentials in dev namespace

Layer 5: Pod consumes the K8s Secret
envFrom:
  - secretRef:
      name: db-credentials
→ DB_USERNAME=pharma_user and DB_PASSWORD=s3cr3t are env vars in the container
```

---

#### Q7: What are the security implications of `clusterResourceWhitelist: *` in your ArgoCD project?

**In this project:**
```yaml
# argocd/projects/pharma-project.yaml
clusterResourceWhitelist:
  - group: "*"
    kind: "*"
```

**What this means:** Any ArgoCD Application in the `pharma` project can create, modify, or delete ANY Kubernetes resource — including `ClusterRole`, `ClusterRoleBinding`, `ValidatingWebhookConfiguration`, `CustomResourceDefinition`.

**Security risks:**
| Risk | Scenario |
|---|---|
| Privilege escalation | Compromised zen-gitops creates a ClusterRole with admin rights |
| Namespace escape | A dev app creates a ClusterRoleBinding to access prod secrets |
| Persistence | Attacker adds a ValidatingWebhookConfiguration that intercepts all cluster traffic |

**How to harden:**
```yaml
# Restrict to only what's actually needed
clusterResourceWhitelist:
  - group: ""
    kind: Namespace
  - group: "external-secrets.io"
    kind: ClusterSecretStore

# Explicitly exclude: ClusterRole, ClusterRoleBinding, ValidatingWebhookConfiguration
```

---

#### Q8: A pod is stuck in CrashLoopBackOff — walk me through your debugging steps

**Simulate the error:**
```bash
# Delete the secret that auth-service depends on
kubectl delete secret db-credentials -n dev

# Watch auth-service crash
kubectl get pods -n dev -w
# auth-service starts, can't connect to DB, crashes, Kubernetes restarts it
# STATUS: CrashLoopBackOff
```

**Investigate step by step:**
```bash
# Step 1: Get overview
kubectl get pods -n dev
# NAME                          READY   STATUS             RESTARTS   AGE
# auth-service-7d9f8c-xk2pv    0/1     CrashLoopBackOff   5          8m

# Step 2: Describe pod — read Events section
kubectl describe pod auth-service-7d9f8c-xk2pv -n dev
# Last State: Terminated, Exit Code: 1
# Events: Back-off restarting failed container

# Step 3: Read logs from the CRASHED container (--previous is critical)
kubectl logs auth-service-7d9f8c-xk2pv -n dev --previous
# Error: Cannot create DataSource: db-credentials secret not found
# Caused by: secret "db-credentials" not found

# Step 4: Check events at namespace level
kubectl get events -n dev --sort-by='.lastTimestamp' | tail -10

# Step 5: Verify secret exists
kubectl get secret db-credentials -n dev
# Error from server (NotFound): secrets "db-credentials" not found  ← root cause
```

**Exit code cheat sheet:**
| Exit Code | Meaning |
|---|---|
| 0 | Clean exit (unexpected for a server app) |
| 1 | Application error — check logs |
| 137 | OOMKilled — memory limit exceeded |
| 139 | Segfault |

**Fix:**
```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USERNAME=pharma_user \
  --from-literal=DB_PASSWORD=pharmaPass123 \
  -n dev
# Pod auto-restarts and recovers within 30 seconds
```

---

#### Q9: A pod has been in Pending state for 10 minutes — what are the possible causes?

**Simulate the error:**
```bash
# Add a nodeSelector that no node has
kubectl patch deployment auth-service -n dev \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/nodeSelector", "value": {"gpu": "true"}}]'

# New pod will be Pending indefinitely
kubectl get pods -n dev
# STATUS: Pending
```

**Investigate:**
```bash
# Step 1: Describe pod — Events section tells you exactly why
kubectl describe pod auth-service-<new-pod> -n dev
# Events:
# Warning  FailedScheduling  scheduler  0/3 nodes are available:
#   3 node(s) didn't match node selector.

# Verify — no nodes have the gpu=true label
kubectl get nodes --show-labels | grep gpu
# (no output)
```

**All possible causes of Pending:**
1. **Insufficient resources** — `0/3 nodes: 3 Insufficient cpu` → scale up node group
2. **No matching node selector/affinity** — label doesn't exist on any node
3. **Taints not tolerated** — node has a taint the pod doesn't tolerate
4. **PVC not bound** — storage class provisioner issue
5. **Resource quota exceeded** — namespace quota limits hit
6. **Topology spread constraints too strict** — can't spread pods across zones

**Fix for this simulation:**
```bash
kubectl patch deployment auth-service -n dev \
  --type='json' \
  -p='[{"op": "remove", "path": "/spec/template/spec/nodeSelector"}]'
# Pod schedules and starts immediately
```

---

#### Q10: auth-service readiness probe keeps returning 503 — traffic isn't routing to the pod

**Simulate the error:**
```bash
# Change readiness probe path to something that doesn't exist
kubectl patch deployment auth-service -n dev \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/path", "value": "/wrong/path"}]'
```

**Investigate:**
```bash
# Step 1: Pod is Running but not Ready
kubectl get pods -n dev
# READY: 0/1  STATUS: Running

# Step 2: Describe pod — readiness probe failures
kubectl describe pod auth-service-<hash> -n dev
# Readiness probe failed: HTTP probe failed with statuscode: 404

# Step 3: No endpoints — Service has no backends
kubectl get endpoints auth-service -n dev
# ENDPOINTS: <none>  ← traffic won't route to any pod

# Step 4: Manual probe test — confirm the right path works
kubectl exec -n dev deploy/auth-service -- \
  curl -s http://localhost:8081/wrong/path
# 404 Not Found

kubectl exec -n dev deploy/auth-service -- \
  curl -s http://localhost:8081/actuator/health/readiness
# {"status":"UP"}  ← correct path works fine

# Root cause: wrong probe path in the Deployment spec
```

**Fix:** Correct the path back to `/actuator/health/readiness`. ArgoCD syncs the corrected values file → pod becomes ready → Service endpoints populate → traffic flows.

**Key concept:** Readiness probe failure = pod removed from Service endpoints. The pod stays Running but gets zero traffic until the probe passes.

---

#### Q11: You get an OOMKilled event on auth-service every 30 minutes — how do you investigate?

**Simulate the error:**
```bash
# Set memory limit too low for a Spring Boot app
kubectl patch deployment auth-service -n dev \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "64Mi"}]'
# Spring Boot needs at least 256Mi — 64Mi causes OOMKill within minutes
```

**Investigate:**
```bash
# Step 1: Confirm OOMKilled
kubectl describe pod auth-service-<hash> -n dev
# Last State: Terminated
#   Reason:    OOMKilled
#   Exit Code: 137

# Step 2: Check memory usage trend — two patterns tell different stories
kubectl top pod -n dev  # current usage

# In Prometheus/Grafana:
# container_memory_working_set_bytes{pod=~"auth-service-.*"}
#
# Pattern A — Memory leak: gradual increase over 30 min until killed
# Pattern B — Under-provisioning: spikes immediately at startup and crashes
```

**Root cause options:**
| Cause | Pattern | Fix |
|---|---|---|
| Memory limit too low | Pattern B (immediate) | Increase limit to 512Mi |
| JVM heap too high | Pattern B | Set `-Xmx384m -XX:MaxMetaspaceSize=96m` |
| Memory leak (unbounded cache) | Pattern A (gradual) | Find and fix leak — heap dump analysis |

**Fix for this simulation:**
```bash
# Increase memory limit
kubectl patch deployment auth-service -n dev \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "512Mi"}]'

# Better fix: set JVM flags to respect container limits
# In values file configmap:
# JAVA_OPTS: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
# 75% of 512Mi = 384Mi for heap — leaves headroom for metaspace + threads
```

---

#### Q12: NGINX Ingress is returning 502 Bad Gateway for pharma-ui — pods are running and healthy

**Simulate the error:**
```bash
# Change Service targetPort to wrong port
# pharma-ui React app runs on port 3000
kubectl patch service pharma-ui -n dev \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/targetPort", "value": 9999}]'
# NGINX reaches the pod but nothing is listening on 9999 → 502
```

**Investigate (layer by layer):**
```bash
# Step 1: Pods are running AND ready
kubectl get pods -n dev -l app=pharma-ui
# READY: 1/1  ← pods look healthy

# Step 2: Check Service endpoints — do they point to the right port?
kubectl get endpoints pharma-ui -n dev
# ENDPOINTS: 10.0.1.45:9999  ← port is wrong

# Step 3: Manually test from inside NGINX
NGINX_POD=$(kubectl get pods -n ingress-nginx -o name | head -1)
POD_IP=$(kubectl get pod -n dev -l app=pharma-ui -o jsonpath='{.items[0].status.podIP}')
kubectl exec -n ingress-nginx $NGINX_POD -- curl -v http://$POD_IP:9999/
# curl: (7) Failed to connect — connection refused  ← confirms wrong port

# Step 4: Check NGINX logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | grep pharma-ui
# connect() failed (111: Connection refused)

# Root cause: targetPort 9999 but app listens on 3000
```

**502 vs 503 vs 504:**
| Code | Meaning | Common cause |
|---|---|---|
| 502 Bad Gateway | NGINX reached the pod but got bad response | Wrong port, app returning invalid HTTP |
| 503 Service Unavailable | NGINX has no upstream pods | All pods failing readiness, no endpoints |
| 504 Gateway Timeout | NGINX reached pod but it didn't respond in time | App overloaded, slow DB query |

**Fix:**
```bash
kubectl patch service pharma-ui -n dev \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/ports/0/targetPort", "value": 3000}]'
# 502 resolves immediately
```

---

#### Q13: Tell me about a time you convinced your team to adopt GitOps

**STAR format model answer:**

**Situation:** Our team was running Jenkins pipelines with direct `kubectl apply` to all three environments. The setup worked but we had recurring problems: prod deployments nobody could trace to a PR, config drift between environments, and a compliance audit finding — no approval gate for production changes.

**Task:** I was asked to fix the compliance finding. I saw it as the opportunity to move to GitOps.

**Action:**

First, I understood the resistance before proposing anything. I ran a team retro asking "what's painful about deploying right now?" — not "should we change?" The pain points in their own words were:
- "I can never tell what's actually running in prod without sshing into the cluster"
- "Jenkins went down last week and we couldn't deploy for 3 hours"
- "I'm nervous every time I have to do a prod deployment"

Then I proposed GitOps as the solution to THEIR problems, not as a technology migration:
- "Can't tell what's in prod" → `git log envs/prod/values-auth-service.yaml` shows every change ever made
- "Jenkins outage" → ArgoCD lives inside the cluster, not an external dependency
- "Nervous about prod" → every change requires a PR, you see the exact diff before it applies

I ran a 2-week pilot on auth-service in dev only. Showed the ArgoCD UI — the visual diff of what would change before syncing was the moment it clicked for the team.

I kept Jenkins running in parallel for 4 weeks. Didn't ask anyone to trust something unproven.

**Result:** Full migration over 6 weeks. Compliance finding resolved. The team's biggest surprise was rollback time: from "find the Jenkins build, re-trigger" to `git revert` → merge → done in 5 minutes.

**What interviewers listen for:**
- Did you acknowledge the existing solution had value? (Respect for what the team built)
- Did you run a pilot before asking for full commitment?
- Did you solve THEIR problems, not impose your preferences?
- Were there measurable outcomes?

---

### Part 3 — What's Next: Career Advice (10 min)

**To strengthen your DevOps resume beyond this course:**

| Area | What to build | Why |
|---|---|---|
| Infrastructure | Terraform EKS cluster (see zen-infra companion repo) | Completes the full stack — you built what runs ON the cluster, now build the cluster itself |
| CI | GitHub Actions pipelines: build, test, scan, push to ECR | Understand the other half of the pipeline you connected to in Session 3 |
| Observability | Prometheus alerts, Grafana dashboards | Every prod incident involves metrics — know how to read them |
| Security | OPA/Gatekeeper admission policies, Trivy in CI | Regulated industries (pharma, fintech, healthcare) require this |

**Interview readiness checklist:**
- [ ] Can you explain the full GitOps pipeline end-to-end without notes?
- [ ] Can you draw the network flow diagram from memory?
- [ ] Can you debug CrashLoopBackOff in under 5 minutes with kubectl?
- [ ] Do you know when to use ArgoCD rollback vs git revert?
- [ ] Can you explain IRSA without googling?
- [ ] Can you answer "walk me through your project" for 10 minutes straight?

If you can answer all 13 mock interview questions using your OWN deployed cluster as the example — you are ready for a senior DevOps engineer interview.

**The key differentiator in interviews:** Most candidates have watched tutorials. Few have built and broken a real multi-service, multi-environment GitOps setup and debugged it themselves. You now have.

---

### Part 4 — Q&A + Course Wrap-up (5 min)

---

## Suggested Improvements to the Course (Instructor Notes)

The following improvements would strengthen the course but are outside the 4-session scope:

1. **Prerequisites session (Session 0):** 1 hour on EKS cluster setup with eksctl, kubectl config, ECR access, and forking the zen-gitops repo. Many students will struggle with this setup and it burns Session 1 time.

2. **Terraform companion (optional Session 5):** Walk through the `zen-infra` repo — show students how the EKS cluster, RDS, ECR, and IAM roles they've been using were actually created. Connects the full picture.

3. **Monitoring lab:** The repo has `k8s/monitoring/prometheus-values.yaml`. A 30-minute lab deploying Prometheus + Grafana and querying pod metrics would tie directly to the OOMKilled interview question.

4. **CI class (planned):** A dedicated session on GitHub Actions — building images, running tests, pushing to ECR, and triggering the yq+git commit that feeds ArgoCD. Referenced in Session 3 but not taught.

5. **Argo Rollouts (advanced):** Blue-green and canary deployments with traffic splitting — the next level after students understand basic ArgoCD. Covered briefly in the existing interview-questions-part2.md under A7.
