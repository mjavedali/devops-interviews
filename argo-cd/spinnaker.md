```
Enterprise deployment pipelines using ArgoCD (ApplicationSets, progressive rollouts) and
Spinnaker for multi-cluster CD synchronization.
With whatever code examples needed for interview
```
This is the **GitOps + progressive delivery + multi-cluster CD** part of platform engineering.

The clean mental model is:

```text
Developer
   ↓
Git commit
   ↓
CI builds/tests/signs image
   ↓
Git manifest update
   ↓
Argo CD
   ↓
Application / ApplicationSet
   ↓
Kubernetes clusters
   ↓
Progressive Rollout
   ↓
Observe metrics
   ↓
Promote / Abort
```

And Spinnaker can sit in the broader enterprise CD flow when you need **multi-cluster orchestration and synchronization**.

---

# 1. Argo CD

Argo CD is a **GitOps continuous delivery controller for Kubernetes**.

The key principle:

> **Git is the desired state; Argo CD continuously reconciles the cluster toward that state.**

For example Git contains:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  replicas: 3
```

Cluster currently has:

```text
replicas = 2
```

Argo CD detects:

```text
Git desired = 3
Cluster actual = 2
                 ↓
              Argo CD
                 ↓
         update Deployment
                 ↓
Cluster actual = 3
```

So Argo CD is itself a Kubernetes-style reconciliation system.

---

# 2. Pull-based deployment

Traditional CD often looks like:

```text
CI/CD system
     |
     | kubectl apply
     ▼
Kubernetes
```

GitOps looks more like:

```text
Git
 ↓
Argo CD
 ↓
Kubernetes
```

Argo CD runs inside/around the cluster and **pulls desired state from Git**.

Benefits:

```text
auditability
rollback
drift detection
declarative configuration
repeatability
```

---

# 3. Argo CD Application

The basic Argo CD object is an `Application`.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-api
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://git.example.com/platform/apps.git
    targetRevision: main
    path: apps/payment-api/prod

  destination:
    server: https://kubernetes.default.svc
    namespace: payments

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This says:

```text
source:
    Git repository + revision + path

destination:
    Kubernetes cluster + namespace

sync:
    keep cluster aligned with Git
```

---

# 4. Sync vs self-heal

Suppose someone manually does:

```bash
kubectl scale deployment payment-api --replicas=5
```

but Git says:

```text
replicas = 3
```

Argo CD detects drift.

With `selfHeal: true`:

```text
Git: 3
Cluster: 5
    ↓
Argo CD
    ↓
Cluster → 3
```

This is one of the defining GitOps behaviors.

---

# 5. ApplicationSets

`ApplicationSet` solves a very common enterprise problem:

> "I have 50 clusters and want the same application deployed to all or a subset of them."

Instead of manually creating 50 Argo Applications, use one `ApplicationSet`.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: payment-api
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: prod

  template:
    metadata:
      name: 'payment-api-{{name}}'
    spec:
      project: platform

      source:
        repoURL: https://git.example.com/platform/apps.git
        targetRevision: main
        path: apps/payment-api

      destination:
        server: '{{server}}'
        namespace: payments
```

If you have:

```text
cluster-a   env=prod
cluster-b   env=prod
cluster-c   env=dev
cluster-d   env=prod
```

ApplicationSet generates:

```text
payment-api-cluster-a
payment-api-cluster-b
payment-api-cluster-d
```

but not cluster-c.

---

# 6. ApplicationSet generators

You should know the idea of generators more than every field.

Common generators include:

```text
Clusters
Git
List
Matrix
Merge
Pull Request
SCM provider
```

### Cluster generator

Best for:

```text
"Deploy this app to every cluster matching this label."
```

### Git generator

Useful when Git itself defines deployment dimensions.

For example:

```text
environments/
├── dev
├── stage
├── prod
```

ApplicationSet can create Applications based on those directories/files.

### Matrix generator

This is powerful in enterprise environments.

For example:

```text
clusters × environments
```

could create:

```text
cluster-a + dev
cluster-a + prod
cluster-b + dev
cluster-b + prod
```

subject to selectors/templates.

---

# 7. Multi-cluster architecture

A common enterprise setup:

```text
                    Git
                     |
                     v
                Argo CD
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
   Cluster A     Cluster B     Cluster C
       |             |             |
      prod          prod          prod
```

ApplicationSet becomes the templating/multiplication layer:

```text
One ApplicationSet
       ↓
N Argo Applications
       ↓
N clusters
```

This is much easier to scale operationally.

---

# 8. Progressive rollouts

Normal Deployment:

```text
v1 → v2
100% immediately
```

Progressive delivery:

```text
v1
 ↓
5% v2
 ↓
observe metrics
 ↓
25%
 ↓
50%
 ↓
100%
```

The key is:

> **Don't just deploy; continuously evaluate whether the new version is healthy before increasing traffic.**

Argo ecosystem commonly uses **Argo Rollouts** for this.

---

# 9. Argo Rollouts

Instead of:

```yaml
kind: Deployment
```

you can use:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-api
spec:
  replicas: 10

  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause:
            duration: 60
        - setWeight: 25
        - pause:
            duration: 120
        - setWeight: 50
        - pause:
            duration: 300
```

Conceptually:

```text
10 replicas
   ↓
1 replica = new version
9 replicas = old version
   ↓
metrics healthy?
   ↓
increase new version
```

---

# 10. Canary + analysis

The really important enterprise feature is automated health analysis.

Suppose Prometheus exposes:

```text
5xx rate
latency
request success rate
```

A rollout can say:

```text
if:
    error_rate < 1%
    p95_latency < 300ms

then:
    promote

else:
    abort
```

Conceptually:

```text
Deploy 10%
     ↓
Query metrics
     ↓
healthy?
   /     \
 yes      no
 |         |
25%      rollback
 |
50%
 |
100%
```

An `AnalysisTemplate` might look like:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 30s
      count: 5

      successCondition: result[0] >= 0.99
      failureLimit: 2

      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{
              app="payment-api",
              status=~"2.."
            }[2m]))
            /
            sum(rate(http_requests_total{
              app="payment-api"
            }[2m]))
```

This is the kind of code example that demonstrates you understand **progressive delivery**, not just YAML templating.

---

# 11. Blue-green deployment

Another progressive strategy is blue-green:

```text
          Service
             |
       +-----+-----+
       |           |
      Blue        Green
       |           |
      v1          v2
```

Initially:

```text
Service → Blue
```

Deploy Green:

```text
Service → Blue
Green → testing
```

Once Green is healthy:

```text
Service → Green
```

Rollback:

```text
Service → Blue
```

Canary and blue-green solve slightly different operational problems.

---

# 12. Argo CD vs Argo Rollouts

Interviewers may test this.

**Argo CD:**

> "Is the cluster synchronized with Git?"

**Argo Rollouts:**

> "How should traffic move from old version to new version, and when is the new version safe?"

So:

```text
Argo CD
   ↓
deploy desired resources

Argo Rollouts
   ↓
progressively release workload versions
```

---

# 13. Spinnaker

Spinnaker is a broader **continuous delivery platform/orchestration system**, with strong support for multi-cloud and multi-cluster deployment workflows.

A pipeline might look like:

```text
Build complete
     ↓
Deploy
     ↓
Cluster A
     ↓
Wait/verify
     ↓
Cluster B
     ↓
Wait/verify
     ↓
Cluster C
```

You can put gates between stages:

```text
manual approval
automated metrics
security checks
integration tests
```

---

# 14. Spinnaker pipeline concept

A conceptual pipeline:

```text
Trigger
  ↓
Bake / resolve artifact
  ↓
Deploy cluster-A
  ↓
Automated judgment
  ↓
Deploy cluster-B
  ↓
Automated judgment
  ↓
Deploy cluster-C
```

Spinnaker has historically been strong at **multi-stage delivery orchestration**.

An enterprise may use it when the deployment process isn't simply:

```text
Git → Kubernetes
```

but rather:

```text
artifact
  ↓
environment promotion
  ↓
multiple clusters
  ↓
multiple regions
  ↓
approval gates
  ↓
rollback
```

---

# 15. Argo CD vs Spinnaker

This is worth memorizing.

| Argo CD | Spinnaker |
|---|---|
| GitOps-first | Pipeline/orchestration-first |
| Declarative | Workflow-oriented |
| Kubernetes-focused | Multi-cloud/multi-platform |
| Continuous reconciliation | Pipeline stages/gates |
| Excellent Git-to-K8s sync | Excellent complex promotion workflows |

A nuanced enterprise answer is:

> **They don't have to be mutually exclusive.**

For example:

```text
CI
 ↓
Artifact registry
 ↓
Spinnaker
 ↓
promotion/orchestration
 ↓
Git change
 ↓
Argo CD
 ↓
Kubernetes reconciliation
```

Or Argo CD can own Kubernetes delivery while Spinnaker handles broader enterprise orchestration.

---

# 16. Multi-cluster synchronization

Suppose you have:

```text
US-East
US-West
EU-West
APAC
```

and want:

```text
version = 2.14
```

everywhere.

One approach:

```text
Git:
prod/payment-api/version = 2.14
        ↓
ApplicationSet
        ↓
Argo Application per cluster
        ↓
Cluster A/B/C/D
```

You now have a declarative source of truth.

Spinnaker can additionally orchestrate:

```text
US-East
   ↓
health check
   ↓
US-West
   ↓
health check
   ↓
EU
   ↓
health check
   ↓
APAC
```

That's a distinction between **synchronization** and **promotion orchestration**.

---

# 17. Preventing simultaneous bad rollout

An enterprise pipeline should avoid:

```text
version 2.0
 ↓
all 100 clusters
 ↓
2.0 is broken
```

Instead:

```text
Canary cluster
      ↓
regional rollout
      ↓
global rollout
```

For example:

```text
cluster-01 → 2.0
      ↓
metrics
      ↓
cluster-02/03
      ↓
metrics
      ↓
20%
      ↓
50%
      ↓
100%
```

This dramatically reduces blast radius.

---

# 18. Sync waves

Argo CD supports **sync waves**, which let you control deployment ordering.

Example:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

and:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

Then:

```text
Wave 0:
Namespace
CRDs

Wave 1:
Operator

Wave 2:
Application
```

This matters when dependencies must exist first.

---

# 19. Hooks

Argo CD hooks can run actions at lifecycle points.

Conceptually:

```text
PreSync
   ↓
Sync
   ↓
PostSync
```

For example:

```text
PreSync:
database migration

Sync:
application deployment

PostSync:
smoke test
```

Don't use hooks as a replacement for proper declarative resource management, but they're useful when a workflow genuinely requires lifecycle actions.

---

# 20. Drift detection

GitOps becomes powerful when someone manually changes the cluster.

Git:

```text
replicas = 3
```

Cluster:

```text
replicas = 10
```

Argo CD:

```text
OUT OF SYNC
```

Depending on configuration:

```text
manual sync
```

or:

```text
self-heal
```

restores the desired state.

This is one of the strongest interview points:

> **GitOps isn't just deployment automation; it continuously detects and corrects configuration drift.**

---

# 21. Secrets

Don't say:

> "We store Kubernetes Secrets directly in Git."

That's usually a red flag.

Instead, mention mechanisms such as:

```text
External Secrets
Sealed Secrets
Vault
cloud secret manager
SOPS
```

Then Argo CD deploys references/manifests that safely resolve secrets.

The principle:

```text
Git
 ↓
declarative reference
 ↓
secret manager
 ↓
Kubernetes Secret
```

---

# 22. Rollback

GitOps rollback is conceptually simple.

Suppose:

```text
Git:
v2.4.1
```

caused errors.

Revert Git:

```text
v2.4.0
```

Argo CD sees:

```text
desired = v2.4.0
cluster = v2.4.1
```

and reconciles back.

For progressive rollouts, an automated abort can additionally stop promotion and restore traffic.

---

# 23. Important failure cases

Interviewers love these.

### Argo CD says Synced but application is broken

Possible:

```text
manifest is correct
Kubernetes accepted it
application itself is unhealthy
```

So:

```text
Git correctness != application correctness
```

You need health checks and progressive analysis.

### Application is OutOfSync repeatedly

Investigate:

```text
manual mutation
controller mutation
defaulting
ignored fields
non-deterministic manifests
Helm rendering
```

This is often a controller fighting another controller.

### Rollout stuck at 10%

Check:

```text
analysis metrics
traffic routing
service selectors
readiness probes
new ReplicaSet
prometheus query
analysis failure count
```

### New cluster not receiving Application

Check:

```text
cluster registered with Argo CD
labels match ApplicationSet selector
RBAC
ApplicationSet generator
destination
repo access
```

---

# 24. A realistic enterprise architecture

```text
                         Git
                          │
             ┌────────────┴────────────┐
             │                         │
        ApplicationSet            Rollout config
             │                         │
             ▼                         ▼
          Argo CD               Argo Rollouts
             │                         │
     ┌───────┼────────┐                │
     ▼       ▼        ▼                ▼
   Prod-1  Prod-2   Prod-3       Canary traffic
     │       │        │                │
     └───────┼────────┘                ▼
             │                     Prometheus
             │                         │
             ▼                         ▼
       Kubernetes                Analysis / promote
             │
             ▼
        workloads
```

Then Spinnaker can provide an enterprise promotion layer:

```text
                 Artifact
                    │
                    ▼
                Spinnaker
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       Region A  Region B  Region C
          │         │         │
       verify     verify    verify
          │         │         │
          └─────────┼─────────┘
                    ▼
                 Git state
                    │
                    ▼
                  Argo CD
```

---

# 25. Code example: ApplicationSet with environments

A useful interview example is generating one Application per cluster/environment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: payment-api
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:

          - clusters:
              selector:
                matchLabels:
                  team: payments

          - list:
              elements:
                - env: stage
                  path: apps/payment-api/stage

                - env: prod
                  path: apps/payment-api/prod

  template:
    metadata:
      name: 'payment-api-{{name}}-{{env}}'

    spec:
      project: payments

      source:
        repoURL: https://git.example.com/platform/apps.git
        targetRevision: main
        path: '{{path}}'

      destination:
        server: '{{server}}'
        namespace: payments

      syncPolicy:
        automated:
          selfHeal: true
          prune: true
```

This demonstrates:

```text
clusters × environments
```

which is a real enterprise use case.

---

# 26. Progressive deployment YAML

Another excellent code example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-api
spec:
  replicas: 10

  strategy:
    canary:
      canaryService: payment-api-canary
      stableService: payment-api-stable

      steps:
        - setWeight: 10
        - pause:
            duration: 60

        - setWeight: 25
        - pause:
            duration: 120

        - setWeight: 50
        - pause:
            duration: 300

        - setWeight: 100
```

Then add:

```yaml
analysis:
  templates:
    - templateName: success-rate
```

to make promotion metrics-driven.

---

# 27. CI/CD separation

A strong architecture separates **CI** from **CD**.

### CI

```text
source
 ↓
compile
 ↓
unit tests
 ↓
security scan
 ↓
build image
 ↓
sign image
 ↓
push registry
```

### CD

```text
update Git desired state
 ↓
Argo CD
 ↓
deploy
 ↓
progressive rollout
 ↓
metrics
 ↓
promote / abort
```

That leads to:

```text
CI produces an immutable artifact.
CD decides where and when to deploy it.
```

That's a very strong interview statement.

---

# 28. What I'd say in an interview

A polished answer:

> **"I designed enterprise Kubernetes CD around GitOps with Argo CD. Each application was represented declaratively in Git, while ApplicationSets generated Argo Applications across clusters and environments using cluster labels and matrix/list generators. Argo CD continuously reconciled cluster state and detected or corrected drift. For risk-sensitive releases, I used progressive delivery with Argo Rollouts, starting with a small canary percentage and using Prometheus-based analysis of error rate and latency to automatically promote or abort. For broader multi-cluster enterprise workflows, Spinnaker provided staged promotion and orchestration across regions or clusters, with verification gates between stages. I kept CI separate from CD: CI built, scanned and signed immutable images, while CD promoted the desired version through Git and Kubernetes."**

Then, when challenged:

> **"Argo CD answers 'is the cluster synchronized with desired Git state?', Argo Rollouts answers 'is the new version healthy enough to receive more traffic?', and Spinnaker answers 'how do I orchestrate a larger multi-stage promotion across environments or clusters?'"**

That distinction is the most valuable thing to remember.

### Interview cheat sheet

```text
Argo CD
  → GitOps reconciliation
  → drift detection
  → sync/self-heal
  → Applications

ApplicationSet
  → generate many Applications
  → clusters / Git / matrix
  → multi-cluster templating

Argo Rollouts
  → canary
  → blue-green
  → traffic shifting
  → metric-based promotion

Spinnaker
  → multi-stage CD pipelines
  → cross-region / multi-cluster promotion
  → gates / approvals / orchestration

CI
  → build + test + scan + sign

CD
  → promote immutable artifact
  → deploy
  → observe
  → promote / rollback
```

If an interviewer gives you a **real production scenario** such as *“deploy to 100 clusters with zero-downtime, 5% canary, automatic rollback, and regional promotion”*, the architecture above gives you a very clean way to walk through it.