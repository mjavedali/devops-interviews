## How ArgoCD works- interview ready

Think of **Argo CD** as:

> **A GitOps controller for Kubernetes that continuously makes the cluster match what is declared in Git.**

The interview version is not “Argo CD deploys YAML.”

It is:

```text
Git = desired state
Kubernetes = actual state
Argo CD = reconciliation engine
```

Keep this architecture in your head:

```text
Developer
   |
   | git push
   v
Git Repository
   |
   | Kubernetes manifests / Helm / Kustomize
   v
+----------------------+
|       Argo CD        |
|                      |
| watches Git          |
| watches Kubernetes   |
| compares both        |
| reconciles drift     |
+----------+-----------+
           |
           v
    Kubernetes Cluster
```

---

# 1. Start without Argo CD

Imagine you have this deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: orders
          image: mycompany/orders:v1
```

Normally you might deploy it manually:

```bash
kubectl apply -f deployment.yaml
```

Then later:

```bash
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
```

The problem is:

```text
Who deployed what?
What version should production have?
Did someone manually change production?
Does Git actually represent production?
```

Someone might run:

```bash
kubectl scale deployment orders --replicas=10
```

Now:

```text
Git says:        replicas = 3
Cluster says:    replicas = 10
```

Which one is correct?

With GitOps:

> Git is the source of truth.

So Argo CD sees:

```text
Desired state: 3
Actual state: 10
```

and can report:

```text
OutOfSync
```

or automatically restore:

```text
10 → 3
```

if automated self-healing is configured.

That reconciliation loop is the heart of Argo CD.

---

# 2. Basic Git repository

Suppose your repository is:

```text
shop-gitops/
│
├── frontend/
│   ├── deployment.yaml
│   └── service.yaml
│
└── orders/
    ├── deployment.yaml
    └── service.yaml
```

Inside:

```yaml
containers:
  - name: orders
    image: mycompany/orders:v1
```

Argo CD is configured to watch:

```text
repo:
github.com/company/shop-gitops

path:
orders/

destination:
production Kubernetes cluster

namespace:
shop
```

So:

```text
Git
 |
 | orders/deployment.yaml
 v
Argo CD
 |
 v
Kubernetes production cluster
```

---

# 3. The most important Argo CD object: Application

In Argo CD, an `Application` basically says:

> Take configuration from **this Git location** and synchronize it into **this Kubernetes destination**.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: orders
  namespace: argocd

spec:

  project: default

  source:
    repoURL: https://github.com/company/shop-gitops.git
    targetRevision: main
    path: orders

  destination:
    server: https://kubernetes.default.svc
    namespace: shop

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Read this section by section.

---

# 4. `source`

```yaml
source:
  repoURL: https://github.com/company/shop-gitops.git
  targetRevision: main
  path: orders
```

Meaning:

```text
Repository:
shop-gitops

Branch:
main

Folder:
orders/
```

So Argo CD reads:

```text
orders/deployment.yaml
orders/service.yaml
...
```

---

# 5. `destination`

```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: shop
```

Meaning:

> Deploy this application into the `shop` namespace of this Kubernetes cluster.

Conceptually:

```text
Git: orders/
      |
      v
   Argo CD
      |
      v
Cluster: production
Namespace: shop
```

---

# 6. Sync

Suppose Git contains:

```yaml
replicas: 3
```

but Kubernetes has:

```yaml
replicas: 2
```

Argo CD compares:

```text
Git desired state

      VS

Kubernetes live state
```

It reports:

```text
OutOfSync
```

If you manually click:

```text
SYNC
```

or run:

```bash
argocd app sync orders
```

Argo CD changes Kubernetes to:

```yaml
replicas: 3
```

Now:

```text
Git     = 3
Cluster = 3

Status:
Synced
```

---

# 7. Automated sync

You can configure:

```yaml
syncPolicy:
  automated: {}
```

Now Argo CD can automatically synchronize Git changes.

Typical workflow:

```text
Developer
    |
    | changes image
    v
Git

orders:v1
   ↓
orders:v2

    |
    | git push
    v

Argo CD detects change
    |
    v
Kubernetes Deployment updated
    |
    v
orders:v2 running
```

Notice something very important:

The developer doesn't need to do:

```bash
kubectl apply
```

Production is changed by changing Git.

That is **GitOps**.

---

# 8. Pull model vs Push model

This is a common interview question.

Traditional CI/CD often looks like:

```text
CI pipeline
    |
    | kubectl apply
    |
    v
Production cluster
```

That is effectively:

> CI pushes deployment changes into Kubernetes.

Argo CD is different.

```text
CI
 |
 | builds image
 v
Container Registry

Git updated
 |
 v
Argo CD
 |
 | pulls/reconciles desired state
 v
Kubernetes
```

So usually:

```text
CI = build/test/package

Argo CD = continuous delivery/deployment
```

This separation is important.

For example:

```text
GitHub Actions
      |
      | docker build
      | docker push
      v
orders:v42
      |
      | update Git manifest
      v
image: orders:v42
      |
      v
Argo CD
      |
      v
Kubernetes
```

---

# 9. What does Argo CD actually monitor?

Two worlds:

```text
                     Argo CD

             +----------+----------+
             |                     |
             v                     v

       Desired state           Live state

           Git                Kubernetes
```

It continuously asks:

```text
What should exist?

vs

What actually exists?
```

This is called **reconciliation**.

Same general controller philosophy Kubernetes itself uses.

---

# 10. Sync status

You should know these for interviews.

## Synced

```text
Git == Kubernetes
```

Example:

```text
Git:
replicas: 3

Cluster:
replicas: 3
```

Status:

```text
Synced
```

---

## OutOfSync

```text
Git != Kubernetes
```

Example:

```text
Git:
image: orders:v5

Cluster:
image: orders:v4
```

Result:

```text
OutOfSync
```

---

# 11. Health status

This is different from Sync status.

This distinction gets asked.

Imagine:

```text
Git:
replicas: 3

Cluster:
replicas: 3
```

Everything matches Git.

Therefore:

```text
Sync Status = Synced
```

But suppose all pods are crashing:

```text
orders-123   CrashLoopBackOff
orders-456   CrashLoopBackOff
orders-789   CrashLoopBackOff
```

Then application health can be:

```text
Degraded
```

So:

```text
Sync Status
=
Does Kubernetes configuration match Git?


Health Status
=
Are Kubernetes resources actually healthy?
```

You can have:

```text
Synced + Healthy
```

or:

```text
Synced + Degraded
```

Very good interview point.

---

# 12. `selfHeal`

Suppose Git says:

```yaml
replicas: 3
```

An engineer runs:

```bash
kubectl scale deployment orders --replicas=10
```

Now:

```text
Git    = 3
Cluster = 10
```

Without self-healing:

```text
Argo CD:
"OutOfSync"
```

With:

```yaml
automated:
  selfHeal: true
```

Argo CD says:

```text
Git is the source of truth.

10 → 3
```

So:

```text
Engineer changes cluster
       |
       v
Drift detected
       |
       v
Argo CD
       |
       v
Restore Git state
```

This is **drift correction**.

---

# 13. `prune`

Another important interview concept.

Suppose Git originally contains:

```text
deployment.yaml
service.yaml
configmap.yaml
```

Argo CD creates all three.

Later someone deletes:

```text
configmap.yaml
```

from Git.

Without pruning, Kubernetes might still contain the old ConfigMap.

With:

```yaml
automated:
  prune: true
```

Argo CD understands:

```text
This resource no longer exists in desired state.
```

and deletes it from Kubernetes.

So:

```text
selfHeal
=
Fix resources that someone changed.


prune
=
Delete resources removed from Git.
```

Memorize that distinction.

---

# 14. Helm with Argo CD

Very common real-world setup.

Instead of raw YAML:

```text
orders/
├── Chart.yaml
├── values.yaml
└── templates/
```

Argo CD can render Helm charts.

Example:

```yaml
source:

  repoURL: https://github.com/company/shop-gitops.git

  path: charts/orders

  targetRevision: main

  helm:
    valueFiles:
      - values-prod.yaml
```

Then:

```text
Git
 |
 | Helm chart
 | values-prod.yaml
 v
Argo CD
 |
 | helm template-like rendering
 v
Kubernetes YAML
 |
 v
Kubernetes
```

A useful interview detail:

> Argo CD typically uses Helm primarily as a **templating mechanism**; Argo CD itself manages the application's lifecycle and reconciliation.

---

# 15. Kustomize

Another common configuration:

```text
orders/
├── base/
│   ├── deployment.yaml
│   └── service.yaml
│
└── overlays/
    ├── dev/
    ├── staging/
    └── production/
```

Production could say:

```yaml
replicas: 10
```

while dev says:

```yaml
replicas: 1
```

Then Argo CD applications point at different paths:

```text
orders-dev
   ↓
overlays/dev


orders-prod
   ↓
overlays/production
```

So Argo CD works nicely with:

```text
Raw YAML
Helm
Kustomize
```

---

# 16. Argo CD architecture

For interviews, know these components:

```text
                  +--------------------+
                  |   Argo CD Server   |
                  | UI / API / CLI     |
                  +---------+----------+
                            |
             +--------------+--------------+
             |                             |
             v                             v

+-----------------------+       +----------------------+
| Application           |       | Repository Server    |
| Controller            |       |                      |
|                       |       | reads Git            |
| compares desired/live |       | renders Helm         |
| reconciles cluster    |       | renders Kustomize    |
+-----------+-----------+       +----------------------+
            |
            v
       Kubernetes
```

The main three to remember:

```text
argocd-server
repo-server
application-controller
```

---

# 17. `argocd-server`

Think:

```text
Argo CD frontend
```

Provides:

```text
Web UI
API
CLI connectivity
Authentication
```

When you open:

```text
Argo CD UI
```

you're talking to the API server component.

Don't think of it as the component performing every deployment.

---

# 18. `repo-server`

Think:

> Git/config rendering engine.

Suppose you have:

```text
Helm chart
```

Argo needs to turn:

```text
templates + values
```

into:

```text
Kubernetes manifests
```

The repo server is heavily involved in retrieving and generating the desired manifests.

Conceptually:

```text
Git Repository
       |
       v
Repo Server
       |
       | Helm/Kustomize/render
       v
Desired Kubernetes YAML
```

---

# 19. Application Controller

This is the most important component.

Think:

> Reconciliation brain.

It compares:

```text
desired state
```

against:

```text
live state
```

and controls synchronization.

Conceptually:

```text
Git desired manifests
          |
          v
+----------------------+
| Application          |
| Controller           |
|                      |
| COMPARE              |
+----------+-----------+
           |
      differences?
        /      \
      no        yes
      |          |
    Synced    OutOfSync
                 |
                 v
               Sync
```

If you're asked:

> Which Argo CD component performs reconciliation?

Think:

**Application Controller.**

---

# 20. Full deployment example

Suppose your app currently runs:

```text
orders:v17
```

Your Deployment Git manifest:

```yaml
containers:
  - name: orders
    image: registry.example.com/orders:v17
```

A developer commits code.

CI:

```text
1. Runs tests
2. Builds Docker image
3. Pushes orders:v18
```

Then something updates Git:

```diff
- image: registry.example.com/orders:v17
+ image: registry.example.com/orders:v18
```

Flow:

```text
Developer
   |
   v
Application Git Repo
   |
   v
CI
   |
   | build
   v
Container Registry
orders:v18


GitOps Repo
   |
   | image changed v17 → v18
   v
Argo CD
   |
   v
Kubernetes
   |
   v
RollingUpdate
   |
   v
orders:v18
```

This distinction is important:

```text
Docker image lives:
Container Registry

Desired deployment version lives:
Git
```

---

# 21. GitOps repo vs application repo

Real companies often separate them.

For example:

```text
orders-source-code/
├── src/
├── Dockerfile
└── tests/
```

and:

```text
production-gitops/
└── orders/
    ├── deployment.yaml
    ├── service.yaml
    └── values.yaml
```

Pipeline:

```text
Application repo
      |
      v
CI builds:
orders:1.4.7
      |
      v
Container Registry
      |
      v
GitOps repo updated:

image:
  tag: 1.4.7

      |
      v
Argo CD
      |
      v
Production
```

This is a very common architecture.

---

# 22. AppProject

You saw:

```yaml
project: default
```

inside an Application.

An Argo CD `AppProject` is mainly used to put boundaries around groups of applications.

Think:

> What is this team/application allowed to deploy, and where?

For example:

```text
Team A:

Allowed Git repos:
team-a/*

Allowed destination:
namespace team-a

Not allowed:
kube-system
```

Conceptually:

```text
AppProject
   |
   +-- allowed Git repositories
   |
   +-- allowed clusters
   |
   +-- allowed namespaces
   |
   +-- permitted resource kinds
```

So:

```text
Application
=
what to deploy


AppProject
=
security/organizational boundary around applications
```

---

# 23. Multi-cluster

Argo CD does not need to manage only the cluster where it runs.

You can have:

```text
                   Argo CD
                      |
          +-----------+-----------+
          |           |           |
          v           v           v

       dev K8s    staging K8s   prod K8s
```

Applications can have destinations pointing at different registered clusters.

So one Argo CD control plane can manage many Kubernetes clusters, depending on your architecture and access model.

---

# 24. ApplicationSet

This is very interview-relevant.

Suppose you have:

```text
100 Kubernetes clusters
```

and want the same monitoring agent deployed everywhere.

Creating 100 Application YAML files is annoying.

Instead use:

```text
ApplicationSet
```

Think:

> A generator for Argo CD Applications.

For example:

```text
ApplicationSet
     |
     +--> app-us-east
     |
     +--> app-us-west
     |
     +--> app-eu
     |
     +--> app-india
```

One template creates multiple Applications.

This is commonly used for:

```text
multi-cluster deployments
multi-environment deployments
many microservices
```

Easy distinction:

```text
Application
=
deploy one logical application


ApplicationSet
=
generate many Applications
```

---

# 25. App of Apps pattern

Another common interview topic.

Suppose your platform contains:

```text
frontend
orders
payments
redis
monitoring
ingress
```

Instead of manually creating six Argo Applications, create one root Application.

```text
root-app
   |
   +--> frontend Application
   +--> orders Application
   +--> payments Application
   +--> monitoring Application
```

Git might look like:

```text
apps/
├── frontend.yaml
├── orders.yaml
├── payments.yaml
└── monitoring.yaml
```

Argo CD manages:

```text
root Application

         ↓

child Applications

         ↓

actual workloads
```

This is the **App of Apps** pattern.

ApplicationSet is often another way to solve the “many Applications” problem, depending on what you're trying to accomplish.

---

# 26. Rollback

Suppose production is:

```text
Git commit A
orders:v17
```

You merge:

```text
Git commit B
orders:v18
```

Production breaks.

The GitOps-friendly approach is typically:

```text
Revert commit B
```

Git becomes:

```text
orders:v17
```

Then:

```text
Argo CD sees Git change

v18 → v17

and reconciles Kubernetes.
```

So Git gives you:

```text
history
auditing
PR review
rollback path
```

This is one reason GitOps is powerful.

---

# 27. Sync waves and hooks

Suppose an application requires:

```text
1. Database migration
2. Backend deployment
3. Frontend deployment
```

Sometimes deployment order matters.

Argo CD supports deployment ordering mechanisms such as **sync waves** and **resource hooks**.

Conceptually:

```text
Wave -1
Database migration

     ↓

Wave 0
Backend

     ↓

Wave 1
Frontend
```

Hooks can represent lifecycle actions such as:

```text
PreSync
Sync
PostSync
SyncFail
```

Interview answer:

> Sync waves control ordering between resources; hooks allow lifecycle-specific jobs/actions around synchronization.

---

# 28. Argo CD vs Argo Rollouts

People confuse these constantly.

## Argo CD

Answers:

> What Kubernetes state should be deployed from Git?

Example:

```text
orders:v2 should run.
```

## Argo Rollouts

Answers:

> How should I progressively release that new version?

Example:

```text
v1 90%
v2 10%

wait

v1 50%
v2 50%

wait

v2 100%
```

Therefore:

```text
Argo CD
=
GitOps continuous delivery


Argo Rollouts
=
advanced progressive delivery
```

They work very well together.

And from your Istio discussion:

```text
Argo Rollouts
       |
       | controls traffic percentages
       v
Istio
       |
       +-- 90% → v1
       +-- 10% → v2
```

That's a strong interview connection.

---

# 29. Argo CD + Istio together

Imagine Git contains:

```text
Deployment
Service
VirtualService
DestinationRule
AuthorizationPolicy
```

Argo CD does not particularly care that some YAML belongs to Istio.

It sees:

```text
Kubernetes resources
```

and reconciles them.

So:

```text
Git
 |
 +-- Deployment
 +-- Service
 +-- VirtualService
 +-- DestinationRule
 +-- AuthorizationPolicy
 |
 v
Argo CD
 |
 v
Kubernetes API
 |
 +---- Kubernetes controller handles Deployment
 |
 +---- Istio handles VirtualService etc.
```

Important distinction:

```text
Argo CD
=
gets desired configuration into Kubernetes


Istio
=
controls runtime service networking
```

For your systems framing:

```text
Git/Argo CD
      |
      | desired state
      v
Kubernetes + Istio
      |
      | runtime behavior
      v
Pods / Envoy / traffic
```

---

# 30. Very common interview question:
# “What happens after I commit code?”

A strong answer:

> The CI pipeline tests the application, builds a container image, and pushes it to the container registry. The deployment manifest or Helm values in the GitOps repository is then updated with the new image version. Argo CD monitors the GitOps repository, detects that the desired state differs from the live Kubernetes state, marks the Application OutOfSync, and, if automated sync is enabled, reconciles Kubernetes to the new desired state. Kubernetes then performs the rollout.

Flow:

```text
git commit
    ↓
CI
    ↓
test
    ↓
docker build
    ↓
registry/orders:v42
    ↓
update GitOps repo
    ↓
Argo CD detects change
    ↓
OutOfSync
    ↓
Sync
    ↓
Kubernetes Deployment updated
    ↓
new pods
```

That answer alone covers a lot.

---

# 31. “What if someone changes production manually?”

Strong interview answer:

> Argo CD detects configuration drift because the live Kubernetes state no longer matches Git. The Application becomes OutOfSync. With automated self-healing enabled, Argo CD can revert the manual modification and restore the Git-defined state.

```text
Git:
replicas=3

Engineer:
kubectl scale → 10

Argo:
3 != 10

      ↓

OutOfSync

      ↓

selfHeal

      ↓

replicas=3
```

---

# 32. “What if I delete something from Git?”

Strong answer:

> With pruning enabled, Argo CD removes resources that were previously managed by the Application but no longer exist in the desired Git state.

```text
Git before:
Deployment
Service
ConfigMap

Git after:
Deployment
Service

        ↓

prune=true

        ↓

ConfigMap deleted from Kubernetes
```

---

# 33. “Why Argo CD instead of Jenkins doing kubectl apply?”

Good answer:

Jenkins approach:

```text
Jenkins
   |
   | credentials
   |
   | kubectl apply
   v
Production
```

Potential issues:

```text
CI needs cluster credentials
deployment state lives partly in pipeline
manual drift is harder to notice
Git may not reflect the live cluster
```

GitOps approach:

```text
CI → builds artifact

Git → desired deployment state

Argo CD → reconciles cluster
```

Advantages:

```text
Git is source of truth
declarative configuration
drift detection
self-healing
audit history
PR-driven changes
easy rollback/revert
separation of CI and CD
```

---

# 34. One subtle interview question:
# “Does Argo CD continuously poll Git?”

The important answer isn't the implementation detail.

Say:

> Argo CD continuously reconciles the desired Git state with the Kubernetes live state. Repository changes can be detected periodically and can also be accelerated through Git webhooks.

The important concept is:

```text
event/change
     +
reconciliation loop
```

not:

```text
"Argo only reacts when CI tells it to deploy."
```

---

# 35. What Argo CD does NOT do

Useful in interviews.

Argo CD isn't primarily responsible for:

```text
compiling Java
running unit tests
building Docker images
pushing images to registry
```

That's CI territory.

For example:

```text
GitHub Actions / Jenkins
          |
          | CI
          v
Build + Test + Docker Push
          |
          v
GitOps repo
          |
          | CD
          v
       Argo CD
          |
          v
      Kubernetes
```

---

# 36. Interview-ready architecture

Memorize this:

```text
                    APPLICATION CODE

Developer
    |
    v
Application Git
    |
    v
+------------------+
| CI               |
|                  |
| test             |
| build            |
| docker push      |
+--------+---------+
         |
         v
Container Registry
orders:v52


                     GITOPS

GitOps Repository
image: orders:v52
         |
         |
         v
+---------------------------+
|          Argo CD          |
|                           |
| Repo Server               |
|      ↓                    |
| Desired Manifests         |
|      ↓                    |
| Application Controller    |
|      ↓                    |
| Compare desired vs live   |
+------------+--------------+
             |
             v
      Kubernetes API
             |
             v
       Deployment
             |
             v
         Pods:v52
```

---

# 37. Interview cheat sheet

| Term | Remember it as |
|---|---|
| **Application** | Git source → Kubernetes destination |
| **Application Controller** | Reconciliation brain |
| **Repo Server** | Fetch/render Git, Helm, Kustomize |
| **Argo CD Server** | UI/API/CLI |
| **Synced** | Git matches cluster |
| **OutOfSync** | Git differs from cluster |
| **Healthy** | Workloads are functioning |
| **Degraded** | Resources have health problems |
| **Self Heal** | Undo cluster drift |
| **Prune** | Delete resources removed from Git |
| **AppProject** | Security/boundary for Applications |
| **ApplicationSet** | Generate many Applications |
| **App of Apps** | Root Application manages child Applications |
| **Sync Wave** | Deployment ordering |
| **Hook** | Run something before/during/after sync |

---

# 38. The 30-second interview answer

If an interviewer asks:

> **“Explain how Argo CD works.”**

Say:

> Argo CD is a Kubernetes-native GitOps continuous delivery tool. Git acts as the source of truth for the application's desired Kubernetes state. An Argo CD Application defines the Git repository, revision and path, as well as the target Kubernetes cluster and namespace. The application controller continuously compares the desired state generated from Git with the live state in Kubernetes. If they differ, the application becomes OutOfSync and Argo CD can either synchronize manually or automatically. With self-healing enabled it can correct manual configuration drift, and with pruning enabled it can remove resources that are no longer declared in Git. Argo CD also supports raw manifests, Helm and Kustomize, and can manage multiple clusters and applications.

That is a solid interview answer.

---

# 39. The one picture to remember

```text
                        Git
                  DESIRED STATE
                        |
                        v
                 +-------------+
                 |   Argo CD   |
                 |             |
                 |   COMPARE   |
                 +------+------+
                        |
                desired == live?
                  /          \
                yes           no
                |              |
             Synced        OutOfSync
                               |
                               v
                         RECONCILE
                               |
                               v
                        Kubernetes
                         LIVE STATE
```

And the sentence I would remember above everything else:

> **Argo CD does not fundamentally “run deployments.” It continuously reconciles Kubernetes' live state toward the desired state declared in Git.**

That phrasing will help you answer most follow-up questions correctly.