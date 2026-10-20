# Atlas: Zero-Downtime Kubernetes Fleet Evolution

## Zero-Downtime In-Place Kubernetes Upgrades vs. Blue-Green Cluster Migration for Mission-Critical Production Workloads

A production-grade Staff/Principal-level platform engineering project focused on safely evolving Kubernetes fleets across **AWS EKS, Azure AKS, and Google GKE**.

The core design principle is simple:

> **A Kubernetes upgrade is successful only when customer-facing application health remains within defined SLOs.**

This project treats Kubernetes upgrades and cluster migrations as controlled, observable, reversible state transitions rather than one-off infrastructure operations.

---

# 1. Project Objective

Build a platform that can safely:

- upgrade an existing production Kubernetes cluster **in place with zero customer-visible downtime**
- provision a new **Green cluster** and migrate workloads from Blue to Green
- decide intelligently between in-place and blue-green strategies
- standardize cluster infrastructure across AWS, Azure, and GCP
- enforce mandatory resource metadata
- standardize node pools
- build and consume hardened OS/node-image generations
- detect configuration drift and compliance violations
- progressively roll out changes with measurable safety gates
- stop or roll back when production health deteriorates
- scale the process from one cluster to hundreds of clusters

The project is designed to demonstrate how a Staff/Principal engineer reasons about:

- reliability
- blast radius
- state transitions
- reversibility
- operational risk
- standardization
- automation
- fleet-scale rollout safety
- provider differences

---

# 2. Enterprise Production Scenario

Assume a global SaaS organization named **Atlas Commerce**.

```text
                        Atlas Commerce

                          Global Edge
                              |
                +-------------+-------------+
                |                           |
             Americas                     Europe/APAC
                |                           |
        +-------+-------+           +-------+-------+
        |       |       |           |       |       |
       EKS     AKS     GKE         EKS     AKS     GKE
```

The organization operates approximately:

```text
~180 Kubernetes clusters

Development       40
Integration       25
Staging           20
Small Production  45
Large Production  35
Mission Critical  15
```

Production workloads include:

- public REST APIs
- gRPC APIs
- payment services
- order-processing workers
- Kafka consumers
- WebSocket services
- ingress gateways
- service mesh
- observability agents
- security agents
- CSI drivers
- CNI components
- GitOps controllers
- PostgreSQL operators
- Redis
- persistent volumes
- autoscaling workloads
- external managed databases

The platform team owns:

```text
Terraform
+
GitOps
+
Policy as Code
+
Immutable Node Lifecycle
+
Cluster Upgrade Automation
+
Migration Automation
+
Fleet Governance
```

Business requirement:

```text
No planned customer-visible downtime
during cluster upgrades or migrations.
```

---

# 3. Production SLOs

## Customer SLO

Tier-0 APIs use:

```text
Monthly availability             >= 99.99%

Successful request rate          >= 99.95%

p95 latency                      < 250 ms

p99 latency                      < 600 ms

Critical transaction success     >= 99.90%
```

## Change Safety Gates

Infrastructure changes use tighter short-window thresholds.

```text
5xx rate
  absolute       < 0.50%
  AND
  delta baseline < +0.20 percentage points

p95 latency
  < 300 ms
  AND
  < baseline × 1.15

p99 latency
  < 750 ms
  AND
  < baseline × 1.20

critical synthetic transactions
  success > 99.5%

ready endpoints
  >= expected replicas - allowed disruption

pending critical pods
  0 for > 5 min

CrashLoopBackOff critical pods
  0

database replication lag
  < service-specific threshold

queue lag
  < 2 × normal baseline
```

A cluster operation may report:

```text
Cluster Upgrade: SUCCESS
```

while customers are experiencing:

```text
increased p99 latency
HTTP 5xx
gRPC resets
queue lag
failed checkout transactions
```

Therefore infrastructure completion is not the success criterion.

---

# 4. Core Problem

The fundamental engineering question is:

> **When a production Kubernetes cluster needs a version, node, OS, networking, or add-on upgrade, should we mutate the existing cluster or build a new one and migrate traffic?**

Two strategies exist.

## In-Place

```text
Cluster N
   |
control plane N -> N+1
   |
nodes N -> N+1
   |
old image -> new image
```

The existing cluster identity remains mostly constant.

## Blue-Green

```text
BLUE
cluster N
old networking
old nodes
old configuration

        ↓ migration

GREEN
cluster N+1
new networking
new nodes
new configuration
```

The real Staff-level question is not:

```text
Which strategy is safer?
```

It is:

```text
Which combination of risks are we willing to accept
for this workload and this specific change?
```

---

# 5. What Actually Changes During an Upgrade?

## Control Plane

Typical changes include:

```text
kube-apiserver
scheduler
controller-manager
managed control-plane internals
API versions
admission behavior
feature gates
```

Application failures can still happen because:

- controllers stop understanding removed APIs
- admission webhooks become incompatible
- operators fail
- CRDs use deprecated API versions
- clients rely on removed behavior
- reconcilers stop working

## Worker Nodes

Worker upgrades typically mean:

```text
create or reimage new node
        ↓
register kubelet
        ↓
node Ready
        ↓
cordon old node
        ↓
evict workloads
        ↓
schedule replacements
        ↓
terminate old node
```

## OS Image Replacement

An OS image replacement may change:

- kernel
- container runtime
- security agent
- sysctl values
- filesystem behavior
- cgroup implementation
- bootstrap scripts
- TLS libraries
- package versions
- networking behavior

## Kubernetes Version

May change:

- kubelet behavior
- API compatibility
- admission behavior
- feature gates
- scheduling behavior
- deprecations
- runtime integration

## CNI / CSI

May change:

- pod networking
- IP allocation
- routes
- MTU
- network policy
- eBPF behavior
- storage mounting
- volume attachment
- topology handling

## Add-ons

Examples:

```text
CoreDNS
metrics server
service mesh
ingress controller
GitOps controller
security agents
observability agents
```

---

# 6. Why Upgrade Success Does Not Mean Zero Downtime

Assume:

```text
Deployment replicas = 3

Node A -> pod-1
Node B -> pod-2
Node C -> pod-3
```

Node A is drained.

The replacement becomes:

```text
pod-4 Pending
```

Reason:

```text
Insufficient CPU
```

Now only two replicas are serving.

If traffic remains constant:

```text
p99:
380 ms -> 1.8 sec
```

Clients begin timing out.

Kubernetes behaved correctly.

The platform operation failed from the customer's perspective.

---

# 7. Final Architecture

```text
                                ┌─────────────────────────┐
                                │       Git / PR          │
                                └────────────┬────────────┘
                                             │
                                             ▼
                              ┌───────────────────────────┐
                              │ CI / Policy Validation    │
                              │                           │
                              │ Terraform validate        │
                              │ OPA / Conftest            │
                              │ Kubernetes policy tests   │
                              │ compatibility tests       │
                              └────────────┬──────────────┘
                                           │
                                           ▼
              ┌─────────────────────────────────────────────────────┐
              │                Fleet Control Plane                  │
              │                                                     │
              │ Cluster inventory                                  │
              │ Ring controller                                    │
              │ Upgrade/migration state machine                     │
              │ Approval service                                   │
              │ Checkpoint store                                   │
              │ Audit log                                          │
              └───────────────┬─────────────────────────────────────┘
                              │
                 ┌────────────┼────────────┐
                 │            │            │
                 ▼            ▼            ▼
              AWS/EKS      Azure/AKS     GCP/GKE
                 │            │            │
            cloud module  cloud module  cloud module
                 │            │            │
                 └────────────┼────────────┘
                              │
                              ▼
                ┌────────────────────────────┐
                │ Kubernetes Platform Layer  │
                │                            │
                │ GitOps                     │
                │ Gateway/Ingress            │
                │ Service mesh               │
                │ Observability              │
                │ Security agents            │
                │ Policies                   │
                │ CSI/CNI                    │
                └────────────────────────────┘
                              │
                              ▼
                ┌────────────────────────────┐
                │ SLO Evaluation Engine      │
                │                            │
                │ Prometheus                 │
                │ OpenTelemetry              │
                │ Synthetic tests            │
                │ Business KPIs              │
                └─────────────┬──────────────┘
                              │
                       healthy│unhealthy
                         ┌────┴─────┐
                         ▼          ▼
                     PROGRESS     PAUSE
                                    │
                                    ▼
                                ROLLBACK
```

---

# 8. Repository Structure

```text
platform/
├── terraform/
│   ├── modules/
│   │   ├── cluster-contract/
│   │   ├── node-pool-contract/
│   │   ├── tagging/
│   │   └── observability/
│   │
│   ├── aws/
│   │   ├── eks-cluster/
│   │   └── eks-nodepool/
│   │
│   ├── azure/
│   │   ├── aks-cluster/
│   │   └── aks-nodepool/
│   │
│   └── gcp/
│       ├── gke-cluster/
│       └── gke-nodepool/
│
├── kubernetes/
│   ├── base/
│   │   ├── ingress/
│   │   ├── observability/
│   │   ├── security/
│   │   └── gitops/
│   ├── policies/
│   └── workloads/
│
├── images/
│   ├── aws/
│   ├── azure/
│   ├── gcp/
│   ├── hardening/
│   └── validation/
│
├── compliance/
│   ├── opa/
│   ├── kyverno/
│   └── tests/
│
├── upgrade/
│   ├── orchestrator/
│   ├── strategies/
│   ├── gates/
│   └── inventory/
│
├── observability/
│   ├── dashboards/
│   ├── alerts/
│   └── slo/
│
├── tests/
│   ├── integration/
│   ├── compatibility/
│   ├── chaos/
│   └── migration/
│
└── runbooks/
    ├── in-place-upgrade.md
    ├── blue-green-migration.md
    ├── failed-drain.md
    └── stateful-cutover.md
```

---

# 9. Shared Platform Contract

Do not build a giant Terraform abstraction like:

```hcl
if cloud == "aws" ...
else if cloud == "azure" ...
else if cloud == "gcp" ...
```

Standardize intent instead.

```hcl
cluster = {
  environment      = "production"
  service_tier     = "tier0"
  kubernetes_track = "stable"

  node_pools = {
    system = {
      min          = 3
      max          = 6
      capacity     = "ondemand"
      architecture = "amd64"
    }

    general = {
      min      = 9
      max      = 30
      capacity = "ondemand"
    }

    compute = {
      min      = 0
      max      = 50
      capacity = "ondemand"
    }
  }
}
```

Cloud-specific implementations turn this contract into:

```text
EKS Managed Node Groups
AKS Agent Pools
GKE Node Pools
```

---

# 10. Base Tagging Standard

Mandatory metadata:

```text
environment
application
service
team
owner
business_unit
cost_center
region
cluster
managed_by
data_classification
compliance_scope
platform_version
```

Terraform:

```hcl
locals {
  mandatory_tags = {
    environment         = var.environment
    application         = var.application
    service             = var.service
    team                = var.team
    owner               = var.owner
    business_unit       = var.business_unit
    cost_center         = var.cost_center
    region              = var.region
    cluster             = var.cluster_name
    managed_by          = "terraform"
    data_classification = var.data_classification
    compliance_scope    = var.compliance_scope
    platform_version    = var.platform_version
  }
}
```

---

# 11. Metadata Types

## Cloud Tags

Used for:

- billing
- cloud IAM
- inventory
- security tooling
- asset discovery
- incident ownership
- compliance

## Kubernetes Labels

Used for machine-readable Kubernetes relationships.

```yaml
metadata:
  labels:
    app.kubernetes.io/name: orders-api
    platform.example.com/team: checkout
    platform.example.com/tier: tier0
```

## Kubernetes Annotations

Used for richer metadata.

```yaml
metadata:
  annotations:
    platform.example.com/owner-email: checkout-oncall@example.com
    platform.example.com/runbook: "runbooks/orders-api"
```

## Terraform Metadata

Defines desired infrastructure ownership and lifecycle.

```hcl
module "cluster" {
  source = "../../modules/cluster"

  metadata = local.mandatory_tags
}
```

---

# 12. Tagging Enforcement

```text
Developer PR
     |
     v
terraform fmt/validate
     |
     v
terraform plan
     |
     v
OPA / Conftest
     |
     v
security/compliance checks
     |
     v
Merge
     |
     v
Terraform apply
     |
     v
Cloud-native policy
     |
     v
Kubernetes admission policy
     |
     v
Continuous drift scan
```

## Terraform Validation

```hcl
variable "tags" {
  type = map(string)

  validation {
    condition = alltrue([
      for key in [
        "environment",
        "application",
        "service",
        "team",
        "owner",
        "cost_center",
        "managed_by",
        "data_classification",
        "compliance_scope",
        "platform_version"
      ] : contains(keys(var.tags), key)
    ])

    error_message = "Mandatory platform tags are missing."
  }
}
```

## OPA Policy

```rego
package terraform.tags

required_tags := {
  "environment",
  "application",
  "service",
  "team",
  "owner",
  "cost_center",
  "managed_by",
  "data_classification",
  "compliance_scope",
  "platform_version",
}

deny[msg] {
  rc := input.resource_changes[_]

  rc.type in {
    "aws_eks_cluster",
    "aws_eks_node_group",
    "azurerm_kubernetes_cluster",
    "google_container_cluster",
  }

  tags := object.get(rc.change.after, "tags", {})

  required := required_tags[_]
  not tags[required]

  msg := sprintf(
    "%s.%s missing mandatory tag %s",
    [rc.type, rc.name, required],
  )
}
```

CI:

```bash
terraform plan -out plan.bin
terraform show -json plan.bin > plan.json

conftest test plan.json \
  --policy compliance/opa/
```

Failure:

```text
FAIL - aws_eks_cluster.prod missing mandatory tag cost_center
```

---

# 13. Kubernetes Policy

Kyverno:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-platform-labels
spec:
  validationFailureAction: Enforce

  rules:
    - name: require-service-metadata

      match:
        any:
          - resources:
              kinds:
                - Deployment
                - StatefulSet

      validate:
        message: "Mandatory platform labels are missing."

        pattern:
          metadata:
            labels:
              app.kubernetes.io/name: "?*"
              platform.example.com/team: "?*"
              platform.example.com/environment: "?*"
              platform.example.com/data-classification: "?*"
```

Invalid workload:

```yaml
metadata:
  labels:
    app: orders
```

Result:

```text
DENIED

mandatory platform labels are missing
```

The fix belongs in Git.

---

# 14. Standard Node Pool Architecture

| Pool | Purpose | Capacity | Typical Workloads |
|---|---|---|---|
| system | platform components | on-demand | DNS, ingress, GitOps |
| general | default applications | on-demand | APIs, web services |
| compute | CPU-heavy | on-demand | encoding, batch |
| memory | memory-heavy | on-demand | caches, JVM workloads |
| specialized | GPU/local SSD/etc. | workload dependent | ML/HPC |
| spot | interruptible | spot/preemptible | fault-tolerant batch |

Standard labels:

```text
platform.example.com/pool=system
platform.example.com/capacity=on-demand
platform.example.com/architecture=amd64
platform.example.com/image-generation=2026.08.3
platform.example.com/platform-version=v42
```

Taints:

```text
platform.example.com/pool=system:NoSchedule

platform.example.com/pool=gpu:NoSchedule

platform.example.com/capacity=spot:NoSchedule
```

---

# 15. Scheduling Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation-engine
spec:
  replicas: 6

  template:
    metadata:
      labels:
        app: recommendation-engine

    spec:
      nodeSelector:
        platform.example.com/pool: compute

      tolerations:
        - key: platform.example.com/pool
          operator: Equal
          value: compute
          effect: NoSchedule
```

Flow:

```text
recommendation-engine
        ↓
nodeSelector
        ↓
compute pool
        ↓
toleration
        ↓
scheduler
        ↓
compute node
```

---

# 16. Upgrade Settings by Pool

Tier-0 pools:

```text
surge:            >= 20%
max unavailable:  1
zones:            >= 3
minimum nodes:     >= 3
```

General:

```text
surge: 20–33%
```

Spot:

```text
max unavailable: 20%
```

System pools are conservative because failures can affect:

```text
CoreDNS
CNI
CSI
ingress
GitOps
security components
```

---

# 17. Why Standard Pools Matter

Without standards:

```text
cluster A: c6i.large
cluster B: m5.2xlarge
cluster C: custom labels
cluster D: random taints
cluster E: custom bootstrap
```

With standards:

```text
system
general
compute
memory
specialized
spot
```

Fleet automation can reason about known classes.

> Standardization reduces the number of infrastructure states that automation must safely reason about.

---

# 18. Golden OS / Node Image Lifecycle

```text
Upstream OS/Image
       ↓
Platform security profile
       ↓
Kernel / sysctl policy
       ↓
Runtime requirements
       ↓
Security agents
       ↓
Vulnerability scan
       ↓
Boot test
       ↓
Kubernetes registration test
       ↓
Workload test
       ↓
Approved generation
       ↓
Dev
       ↓
Staging
       ↓
Production
```

Use explicit generations:

```text
2026.08.1
2026.08.2
2026.08.3
```

Never:

```text
latest
```

---

# 19. Hardening Contract

Every supported node must meet requirements such as:

```text
containerd enabled
unsupported services disabled
host firewall validated
SSH disabled or restricted
root login prohibited
IMDS protection where applicable
kernel parameters validated
audit logging enabled
disk encryption enabled
secure boot where supported
security agent installed where appropriate
CIS baseline evaluated
FIPS variant where required
```

---

# 20. Multi-Cloud Image Strategy

Do not force one identical binary image across EKS, AKS, and GKE.

Standardize the **security and validation contract**.

## EKS

```text
Packer
 ↓
EKS-compatible AMI
 ↓
security scan
 ↓
AMI ID
 ↓
launch-template version
 ↓
managed node group
```

## AKS

Prefer supported platform images with supported hardening controls:

```text
Azure Linux / supported Ubuntu
+
Trusted Launch
+
FIPS where required
+
supported OS configuration
+
security validation
```

## GKE

Prefer supported GKE node-image types:

```text
COS_CONTAINERD
```

or Ubuntu when required.

Common abstraction:

```text
Golden Node Security Profile
```

Not:

```text
One identical custom VM image
```

---

# 21. Immutable Node Lifecycle

Do not patch production nodes manually.

Bad:

```bash
apt upgrade
yum update
vi /etc/sysctl.conf
```

Preferred:

```text
node generation 41
      ↓
new generation 42 approved
      ↓
create replacement
      ↓
cordon old node
      ↓
drain
      ↓
remove
```

Benefits:

```text
reproducibility
drift reduction
security evidence
rollback to known generation
predictable upgrades
```

---

# 22. Tier-0 Workload Resilience Contract

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api

spec:
  replicas: 6

  strategy:
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 2

  template:
    metadata:
      labels:
        app: orders-api

    spec:
      terminationGracePeriodSeconds: 60

      containers:
        - name: api
          image: example/orders:v42

          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - sleep 10

          readinessProbe:
            httpGet:
              path: /ready
              port: 8080

            periodSeconds: 5
            failureThreshold: 2

          resources:
            requests:
              cpu: "500m"
              memory: 512Mi

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule

          labelSelector:
            matchLabels:
              app: orders-api
```

---

# 23. Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: orders-api

spec:
  minAvailable: 5

  selector:
    matchLabels:
      app: orders-api
```

With six replicas:

```text
one voluntary disruption allowed
```

A PDB is not an HA guarantee.

It does not provide:

- spare CPU
- spare IP capacity
- correct readiness
- healthy replacement nodes
- multi-zone placement
- connection draining

---

# 24. Strategy A: In-Place Upgrade

Lifecycle:

```text
PREFLIGHT
   ↓
COMPATIBILITY
   ↓
CAPACITY
   ↓
CONTROL_PLANE
   ↓
CANARY_NODE_POOL
   ↓
OBSERVE
   ↓
POOL_ROLLOUT
   ↓
APPLICATION_VALIDATE
   ↓
COMPLETE
```

---

# 25. Preflight

Collect:

```text
current Kubernetes version
target Kubernetes version
provider
region
node pools
node versions
image generations
CNI
CSI
CoreDNS
admission webhooks
CRDs/operators
PDBs
available capacity
cloud quota
zone distribution
pending pods
current SLO health
active incidents
```

Commands:

```bash
kubectl get nodes -L \
  topology.kubernetes.io/zone,\
  platform.example.com/pool,\
  platform.example.com/image-generation

kubectl get pdb -A

kubectl get pods -A \
  --field-selector=status.phase=Pending
```

---

# 26. Compatibility Validation

Validate:

```text
removed Kubernetes APIs
admission webhook compatibility
CRD versions
controller compatibility
service mesh
CNI
CSI
metrics stack
GitOps controller
security agents
```

Safe sequence:

```text
application/API compatibility
        ↓
control plane
        ↓
critical add-ons
        ↓
worker nodes
```

---

# 27. Capacity Validation

Example:

```text
general pool:
18 nodes

average CPU:    64%
average memory: 58%
```

Upgrade:

```text
max unavailable = 1
surge nodes      = 3
```

Validate more than aggregate CPU.

Check:

```text
per-zone cloud quota
VM availability
IP/subnet capacity
pod density
storage attachment limits
daemonset overhead
special instance availability
```

---

# 28. Control Plane Upgrade

Proceed only when:

```text
SLO healthy
compatibility passed
capacity passed
change window open
freeze absent
approval granted
```

Provider abstraction:

```python
provider.upgrade_control_plane(
    cluster=cluster,
    target_version=target_version,
)
```

Then observe:

```text
API server errors
controller reconciliation
admission errors
GitOps reconciliation
operator health
CNI
application SLO
```

Example soak:

```text
20 minutes
```

---

# 29. Canary Node Upgrade

Example:

```text
general-canary

nodes = 2
target Kubernetes = new
target OS image    = new
```

Label:

```text
platform.example.com/upgrade-ring=canary
```

Canary workloads receive:

```text
1–5% production traffic
```

This catches:

```text
kernel regressions
CNI differences
DNS timing
security agent overhead
storage driver issues
MTU issues
image pull issues
```

---

# 30. Cordon

```bash
kubectl cordon node-42
```

Result:

```text
SchedulingDisabled
```

Existing pods continue running.

---

# 31. Drain

```bash
kubectl drain node-42 \
  --ignore-daemonsets \
  --delete-emptydir-data=false \
  --timeout=20m
```

Conceptually:

```text
request eviction
     ↓
PDB consulted
     ↓
pod starts graceful shutdown
     ↓
readiness becomes false
     ↓
endpoint removed
     ↓
replacement scheduled
     ↓
replacement Ready
```

---

# 32. Real Traffic During HTTP Pod Replacement

Correct lifecycle:

```text
SIGTERM pod B
        ↓
application enters draining state
        ↓
readiness false
        ↓
endpoint removed
        ↓
new requests -> A/C
        ↓
existing requests finish
        ↓
process exits
```

Bad lifecycle:

```text
SIGTERM
 ↓
process exits immediately
 ↓
connections reset
 ↓
HTTP 502
```

---

# 33. Long-Lived Connections

For WebSockets and long-lived HTTP:

```text
load-balancer deregistration
application draining
preStop delay
connection close signaling
sufficient termination grace
```

Clients should support reconnect behavior.

---

# 34. gRPC

Because many RPCs may share one HTTP/2 connection:

```text
mark instance unready
        ↓
send GOAWAY when appropriate
        ↓
stop accepting new RPCs
        ↓
complete existing RPCs
        ↓
terminate
```

Clients need retries and reconnection support.

---

# 35. Background Workers

Correct shutdown:

```text
SIGTERM
 ↓
stop polling
 ↓
finish/checkpoint current job
 ↓
ack only completed work
 ↓
exit
```

Jobs should be idempotent because retries and duplicate delivery can happen.

---

# 36. Stateful Workload Protection

Example:

```text
3 PostgreSQL members

zone A: primary
zone B: replica
zone C: replica
```

Before drain:

```text
replication healthy?
quorum healthy?
replacement attach possible?
storage topology compatible?
PDB correct?
```

Use slower rollout settings.

Example:

```text
maxUnavailable=1
surge=1
soak=30m
```

---

# 37. Autoscaler Interaction

Danger:

```text
upgrade adds surge nodes
        ↓
workloads redistribute
        ↓
autoscaler sees excess capacity
        ↓
scale-down competes with upgrade
```

Mitigations:

```text
disable aggressive scale-down
maintain upgrade headroom
pre-provision surge capacity
avoid unrelated burst workloads consuming surge nodes
```

---

# 38. Progressive Worker Rollout

Example 18-node pool:

```text
Step 0: 2 canary nodes

Step 1: 2 nodes
observe 10m

Step 2: 3 nodes
observe 15m

Step 3: 4 nodes
observe 20m

Step 4: remaining nodes
observe 30m
```

Every transition requires an SLO gate.

---

# 39. In-Place Rollback

## Application

```text
v42 -> v41
```

## Nodes

Usually:

```text
create previous image/version nodes
cordon new nodes
drain
replace
```

Not:

```text
downgrade existing nodes in place
```

## Control Plane

Control-plane downgrade may be unsupported or difficult.

Therefore compatibility testing before mutation is critical.

---

# 40. Strategy B: Blue-Green Cluster Migration

Architecture:

```text
                         Global Traffic
                               |
                       Traffic Controller
                               |
                      +--------+--------+
                      |                 |
                  BLUE 95%           GREEN 5%
                      |                 |
               Existing cluster      New cluster
```

Green has its own:

```text
networking
cluster
node pools
identity
ingress
CNI
CSI
security agents
GitOps
observability
applications
```

---

# 41. Blue-Green Lifecycle

```text
PROVISION_GREEN
      ↓
BOOTSTRAP_PLATFORM
      ↓
VALIDATE_FOUNDATION
      ↓
DEPLOY_APPLICATIONS
      ↓
SYNC_STATE
      ↓
SHADOW/SYNTHETIC_TEST
      ↓
CANARY_TRAFFIC
      ↓
PROGRESS_TRAFFIC
      ↓
CUTOVER
      ↓
SOAK
      ↓
RETIRE_BLUE
```

---

# 42. Provision Green

```hcl
module "green_cluster" {
  source = "../../terraform/aws/eks-cluster"

  cluster_name = "prod-orders-use1-green"

  kubernetes_version = var.target_version

  platform_version = "v42"

  node_pools = local.standard_node_pools

  metadata = merge(
    local.metadata,
    {
      lifecycle = "green"
    }
  )
}
```

---

# 43. Bootstrap Order

```text
1 cloud networking
2 cluster control plane
3 system node pool
4 CNI
5 CSI
6 DNS
7 ingress/Gateway
8 workload identity
9 certificate controller
10 secret provider
11 observability
12 security agents
13 policy engine
14 GitOps controller
15 application namespaces
16 workloads
```

---

# 44. Application Configuration

Use GitOps:

```text
apps/orders/base
apps/orders/overlays/prod
```

Avoid embedding cluster-specific provider details directly inside application manifests unless truly required.

---

# 45. Secrets

Preferred architecture:

```text
AWS Secrets Manager
Azure Key Vault
GCP Secret Manager
Enterprise Vault
```

Then:

```text
External Secrets controller
        ↓
Kubernetes Secret
```

Do not copy plaintext Secrets from Blue to Green.

---

# 46. Workload Identity

Blue and Green both need equivalent authorization.

Validate:

```text
Can Green application:

read secrets?
publish Kafka?
access object storage?
call database?
decrypt KMS data?
```

Running pods are not sufficient evidence.

---

# 47. Certificates

Prefer:

```text
cert-manager
+
central PKI / ACME
```

Validate:

```text
SANs
expiry
trust chains
mTLS identities
mesh trust
```

---

# 48. Persistent Storage Classification

Classify state into:

```text
A. external managed database

B. replicated application-level datastore

C. regional/distributed storage

D. single-cluster PVC/block volume
```

Each requires a different migration plan.

---

# 49. External Managed Database

Architecture:

```text
Blue API ----+
             |
             +----> external DB
             |
Green API ---+
```

Both application versions must be schema-compatible.

Use expand-contract migrations:

```text
1 add new schema elements
2 deploy compatible applications
3 migrate traffic
4 remove old schema later
```

---

# 50. Stateful In-Cluster Database

Example:

```text
BLUE PRIMARY
      |
      | replication
      v
GREEN REPLICA
```

Procedure:

```text
1 create Green replicas
2 seed base backup
3 begin WAL/binlog replication
4 monitor lag
5 verify data
6 prevent Green writes
7 optional read-only traffic
8 reach cutover checkpoint
9 fence Blue writer
10 wait for replication lag to reach zero
11 promote Green
12 update writer endpoint
13 allow Green writes
```

---

# 51. Write Fencing

```text
BLUE writer
    |
    X fenced

replication lag -> zero

GREEN replica
    |
promote
    |
GREEN writer
```

Possible mechanisms:

```text
database permission revoke
network policy
application feature flag
lease/fencing token
database leader election
```

---

# 52. Split-Brain Risk

Worst case:

```text
Blue accepts order 100

Green accepts order 101
```

Now the histories diverge.

Rollback is no longer:

```text
traffic -> Blue
```

It becomes a data-reconciliation problem.

---

# 53. PVC Snapshots

Snapshots are good for:

```text
immutable data
cold migration
read-only service
planned write freeze
```

They are not live replication.

```text
snapshot T0
       ↓
restore Green
       ↓
Blue keeps accepting writes
```

Green becomes stale immediately.

---

# 54. Traffic Layer

## DNS

Useful for:

```text
cluster discovery
regional failover
coarse migration
```

Not ideal for precise 5% traffic canaries because of caching and TTL behavior.

## Global L7 Load Balancer

```text
Global LB
   |
   +-- weight 95 -> Blue
   |
   +-- weight 5 -> Green
```

## Gateway API

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orders

spec:
  rules:
    - backendRefs:
        - name: orders-blue
          port: 80
          weight: 95

        - name: orders-green
          port: 80
          weight: 5
```

## Service Mesh

Useful for:

```text
request-level weighting
header-based routing
user cohorts
regional cohorts
synthetic-only traffic
```

Cross-cluster meshes add complexity in:

```text
identity
trust
routing
service discovery
connectivity
```

---

# 55. Traffic Migration Plan

```text
Blue 100 / Green 0
       ↓
Blue 95 / Green 5
       ↓
Blue 75 / Green 25
       ↓
Blue 50 / Green 50
       ↓
Blue 10 / Green 90
       ↓
Blue 0 / Green 100
```

| Step | Minimum Observe | Progress When | Rollback When |
|---|---:|---|---|
| 0/100 | 15m synthetic | Green passes validation | any Tier-0 failure |
| 5% | 15m | error/latency within limits | >0.2pp error delta |
| 25% | 20m | SLO and saturation healthy | p99 >1.2x baseline |
| 50% | 30m | DB/queues healthy | business KPI degradation |
| 90% | 45m | full production profile healthy | saturation |
| 100% | 60m+ | sustained health | rollback if still safe |

---

# 56. SLO Evaluation

```yaml
service: orders-api

gates:
  - name: server-errors
    query: |
      sum(rate(http_requests_total{status=~"5.."}[5m]))
      /
      sum(rate(http_requests_total[5m]))

    operator: "<"
    threshold: 0.005

  - name: latency-p99
    query: |
      histogram_quantile(
        0.99,
        sum by (le)(
          rate(http_request_duration_seconds_bucket[5m])
        )
      )

    operator: "<"
    threshold: 0.750

  - name: checkout-success
    source: business_metric
    operator: ">="
    threshold: 0.995
```

---

# 57. Why One Metric Is Insufficient

Example:

```text
5xx = 0.01%
```

Looks healthy.

But:

```text
p99 = 8 seconds
```

Clients time out before responses arrive.

Another example:

```text
HTTP 200
```

but the downstream payment fails.

Therefore gates combine:

```text
request health
+
latency
+
saturation
+
Kubernetes health
+
dependency health
+
synthetics
+
business metrics
```

---

# 58. Diagnostic vs Decision Metrics

## Decision Metrics

Used to determine whether rollout proceeds:

```text
5xx
p99
critical transaction success
replication lag
queue lag
SLO burn
```

## Diagnostic Metrics

Used to understand failures:

```text
CNI errors
node CPU
conntrack usage
DNS latency
pod restarts
TCP retransmits
Envoy resets
image pull failures
scheduler failures
```

---

# 59. SLO Gate Logic

```python
def evaluate(service, window):
    results = metrics.evaluate(service, window)

    if results.error_rate > service.max_error_rate:
        return Failure("error-rate")

    if results.error_delta > service.max_error_delta:
        return Failure("error-regression")

    if results.p99 > service.max_p99:
        return Failure("latency-p99")

    if results.synthetic_success < service.min_synthetic:
        return Failure("synthetic")

    if results.business_success < service.min_business_success:
        return Failure("business-transaction")

    return Success()
```

---

# 60. Upgrade / Migration Orchestrator

State machine:

```text
DISCOVER
   ↓
PREFLIGHT
   ↓
VALIDATE
   ↓
APPROVAL
   ↓
CANARY
   ↓
OBSERVE
   │
   ├── healthy -> PROGRESS
   │
   └── unhealthy
           ↓
         PAUSE
           ↓
       ROLLBACK
```

Implementation:

```python
class UpgradeRun:

    def run(self):
        while True:
            state = self.store.current_state()

            if state == "DISCOVER":
                self.discover()
                self.transition("PREFLIGHT")

            elif state == "PREFLIGHT":
                self.preflight()
                self.transition("VALIDATE")

            elif state == "VALIDATE":
                self.validate()
                self.transition("CANARY")

            elif state == "CANARY":
                self.canary()
                self.transition("OBSERVE")

            elif state == "OBSERVE":
                result = self.slo_gate.evaluate()

                if result.healthy:
                    self.transition("PROGRESS")
                else:
                    self.transition("PAUSE")

            elif state == "PROGRESS":
                if self.rollout.complete():
                    self.transition("COMPLETE")
                else:
                    self.rollout.next_batch()
                    self.transition("OBSERVE")

            elif state == "PAUSE":
                self.stop_mutations()
                self.transition("ROLLBACK")

            elif state == "ROLLBACK":
                self.rollback()
                return

            elif state == "COMPLETE":
                self.finalize()
                return
```

---

# 61. Idempotency

Every action has an idempotency key:

```text
run:
upgrade-prod-use1-20260821-001

operation:
upgrade-prod-use1-20260821-001/general-pool/batch-03
```

After controller restart:

```text
read checkpoint
 ↓
detect existing operation
 ↓
continue validation
```

Never create duplicate batches.

---

# 62. Checkpointing

```json
{
  "run_id": "prod-use1-20260821-001",
  "strategy": "IN_PLACE",
  "cluster": "prod-use1-orders",
  "state": "OBSERVE",
  "target_version": "N+1",
  "pool": "general",
  "batch": 3,
  "nodes_completed": 7,
  "last_gate": "healthy",
  "approval_id": "CHG-55123"
}
```

Store this in durable storage.

---

# 63. Concurrency Controls

```text
cluster lock      = 1 active mutation
node-pool lock    = 1 active rollout
region limit      = 2 prod clusters
mission critical  = 1 cluster globally
```

Example fleet concurrency:

```text
Dev                20
Test               10
Staging             5
Small prod          3
Large prod          2
Mission critical    1
```

---

# 64. Decision Framework

| Attribute | In-Place | Blue-Green |
|---|---|---|
| Extra infrastructure | Low/moderate | High |
| Blast radius | Existing cluster | New environment + cutover |
| Control-plane rollback | Weak | Strong before cutover |
| Node rollback | Replacement | Traffic rollback often easy |
| Testing capability | Partial | Strong |
| Migration complexity | Lower | Higher |
| Network/IAM duplication | Low | High |
| Stateful migration | Easier | Harder |
| Drift cleanup | Limited | Excellent |
| Major architecture changes | Riskier | Strong |
| Cost | Lower | Higher |
| Operational maturity | Moderate | High |

---

# 65. Use In-Place When

```text
normal supported Kubernetes minor upgrade

cluster is healthy

applications are disruption tolerant

node pools are standardized

surge capacity exists

network architecture is unchanged

stateful workloads are difficult to move

provider-supported upgrade path is mature
```

---

# 66. Use Blue-Green When

```text
major network redesign

CNI replacement

security-boundary change

large Terraform redesign

cluster has significant drift

large platform generation jump

regulatory validation requires clean environment

migration must be validated with real production traffic
```

---

# 67. Use Hybrid When

Example:

```text
control-plane minor upgrade
        =
in-place

new hardened OS generation
        =
new node pool

major CNI architecture change
        =
new Green cluster
```

Strategy is selected per risk domain.

---

# 68. Staff-Level Risk Model

```text
Expected Change Risk

≈

Probability of Failure
×
Blast Radius
×
Recovery Time
×
State Irreversibility
```

Blue-green often reduces:

```text
blast radius before cutover
recovery time before state changes
```

but increases:

```text
system complexity
migration failure modes
identity/network errors
state synchronization risk
cost
```

Therefore blue-green is not automatically safer.

---

# 69. Failure Injection 1: PDB Blocks Drain

## Symptom

```text
Cannot evict pod as it would violate the pod's disruption budget
```

## Detection

```bash
kubectl get pdb -A
kubectl describe pdb orders-api
kubectl get deploy orders-api
```

Example:

```text
replicas = 2
PDB minAvailable = 2
```

Allowed disruptions:

```text
0
```

## Immediate Action

Pause the upgrade.

Do not force drain Tier-0 workloads.

## Rollback

```text
uncordon node
```

## Permanent Fix

```text
replicas: 3
minAvailable: 2
```

## Staff-Level Trade-Off

PDB feasibility belongs in preflight, not as a surprise halfway through fleet execution.

---

# 70. Failure Injection 2: Insufficient Capacity

## Symptom

```text
Pending pods

0/18 nodes available:
Insufficient cpu
```

## Detection

```bash
kubectl get pods -A --field-selector=status.phase=Pending
kubectl describe pod <pod>
kubectl top nodes
```

Also inspect:

```text
cloud quota
subnet IPs
zone capacity
instance availability
```

## Immediate Action

Pause drains.

## Rollback

Uncordon known-good nodes.

## Permanent Fix

```text
20–30% headroom
quota prevalidation
capacity reservations
alternative instance families
IP capacity monitoring
```

## Staff-Level Trade-Off

Average CPU is not a sufficient capacity model.

---

# 71. Failure Injection 3: Green Latency Regression

## Symptom

```text
Blue p99  420 ms
Green p99 880 ms
```

## Detection

Compare:

```text
Envoy upstream latency
DNS latency
CPU throttling
network RTT
DB RTT
TLS handshake
CNI
node type
security agent overhead
```

Example root cause:

```text
Green DB RTT = 12 ms
Blue DB RTT  = 2 ms
```

Green traffic crosses an unexpected centralized firewall route.

## Immediate Action

```text
95/5 -> 100/0
```

## Permanent Fix

Correct routing/private endpoint architecture.

## Staff-Level Trade-Off

Green being Ready does not prove production equivalence.

---

# 72. Failure Injection 4: Image Pull Failure

## Symptom

```text
ImagePullBackOff
```

## Detection

```bash
kubectl describe pod
```

Possible causes:

```text
registry permission
private DNS
node IAM
workload identity
regional image missing
architecture mismatch
```

## Immediate Action

Stop migration.

## Permanent Fix

Add a Green preflight check that pulls representative production images.

---

# 73. Failure Injection 5: CNI Regression

## Symptom

```text
new nodes Ready

pods Running

intermittent service timeout
```

## Detection

Correlate failures with:

```text
node generation
CNI version
```

Inspect:

```text
IP allocation
routes
MTU
iptables/eBPF
conntrack
network policy
```

Example root cause:

```text
MTU mismatch
```

## Immediate Action

Stop node rollout.

Cordon new nodes.

## Rollback

Replace with prior CNI/image generation.

## Permanent Fix

Canary networking tests:

```text
pod-to-pod
cross-zone
pod-to-service
pod-to-external
large-packet
network-policy
```

---

# 74. Failure Injection 6: Stateful Sync Incomplete

## Symptom

```text
Green replication lag = 42 minutes
```

## Detection

Inspect:

```text
WAL/binlog position
replication throughput
disk IOPS
network throughput
replay errors
```

## Immediate Action

Do not advance traffic or writer cutover.

## Permanent Fix

Explicit state gate:

```text
replication_lag < 5 seconds
for 30 minutes
```

---

# 75. Failure Injection 7: Rollback After Green Writes

Assume:

```text
Blue 10%
Green 90%
```

Green has already become the primary writer.

Naive response:

```text
traffic -> Blue
```

may cause data loss.

## Correct Reasoning

### Shared External DB

Traffic rollback may still be safe if app versions remain schema-compatible.

### Reverse Replication

If:

```text
Green primary
      |
      v
Blue replica
```

Blue may become a viable failback target.

### No Reverse Synchronization

Rollback is no longer safe.

Recovery becomes:

```text
fix Green forward
```

or perform controlled data reconciliation.

> Traffic rollback and data rollback are different operations.

---

# 76. Rollback Boundaries

```text
PHASE                       ROLLBACK

Green infrastructure       trivial
Green apps deployed        trivial
5% read traffic            easy
50% read traffic           easy
Green write shadowing      moderate
Green becomes writer       conditional
Blue replication stopped   difficult
Blue destroyed             impossible
```

Track:

```text
rollback_class
```

in orchestration state.

---

# 77. Fleet Rollout

```text
Ring 0   disposable dev
   ↓
Ring 1   integration
   ↓
Ring 2   staging
   ↓
Ring 3   small production
   ↓
Ring 4   large production
   ↓
Ring 5   mission critical
```

Promotion requires:

```text
ring success rate
healthy SLOs
no unresolved Sev-1/2
no common failure signature
minimum soak period
```

---

# 78. Regional Sequencing

Example:

```text
Wave 1:
dev/us-east

Wave 2:
staging/us-east

Wave 3:
small prod/us-west

Wave 4:
small prod/europe

Wave 5:
large prod/us-east

Wave 6:
mission critical passive region

Wave 7:
mission critical primary region
```

Avoid changing all replicas of a global service at once.

---

# 79. Freeze Windows

Evaluate:

```text
maintenance window
business blackout
holiday freeze
product launch
ongoing incident
regional outage
cloud provider incident
error-budget status
```

Example:

```text
Black Friday:
Ring 4 and Ring 5 blocked
```

For critical security remediation:

```text
security override
+
reduced concurrency
```

---

# 80. Emergency Pause

Global control:

```text
FLEET_UPGRADES_PAUSED=true
```

Stops:

```text
new control-plane mutations
new node drains
new traffic shifts
Blue retirement
```

Existing actions must reach a safe checkpoint before stopping.

---

# 81. Drift Detection

## Terraform Drift

```text
desired cloud state
vs
actual cloud state
```

Example:

```bash
terraform plan -detailed-exitcode
```

## GitOps Drift

```text
Git
vs
live Kubernetes
```

Detect:

```text
manual kubectl changes
deleted resources
configuration changes
replica differences
```

## Compliance Drift

Example:

```text
node missing:
platform.example.com/image-generation
```

Result:

```text
NON_COMPLIANT
```

This can block the next rollout wave.

---

# 82. Observability Dashboard

```text
┌─────────────────────────────────────────────────────────┐
│ Cluster prod-orders-use1                               │
├─────────────────────────────────────────────────────────┤
│ Kubernetes        N -> N+1                             │
│ Strategy          IN_PLACE                             │
│ State             OBSERVE                              │
│ Batch             3 / 6                                │
│ SLO               HEALTHY                              │
└─────────────────────────────────────────────────────────┘

Cluster
-----------------------------------------------
Nodes Ready             21 / 21
Pending Pods             0
Pod Restarts             2
PDB blocked evictions    0

Upgrade
-----------------------------------------------
Old nodes                11
New nodes                10
Current image            image-2026.08.3
Median drain             94 sec
p95 drain                212 sec

Application
-----------------------------------------------
Requests/s               23,100
5xx                      0.08%
p95                      191 ms
p99                      443 ms

Migration
-----------------------------------------------
Blue                     75%
Green                    25%
Synthetic                PASS
Business SLI             PASS
Rollback                 AVAILABLE
```

---

# 83. Staff-Level Troubleshooting Scenario

Scenario:

> The node upgrade is progressing, but customer latency is increasing.

First action:

```text
STOP further mutations.
```

Do not immediately keep draining nodes while investigating.

---

# 84. Investigation Order

Start with correlation.

```text
When did latency increase?

Did it align with:

node replacement?
application deployment?
traffic spike?
dependency failure?
```

Then segment by:

```text
node generation
availability zone
pod
cluster
service version
```

Key question:

> Is the new infrastructure correlated with the problem?

---

# 85. Hypothesis: Capacity Loss

Check:

```text
ready replicas
CPU
CPU throttling
pending pods
HPA
node allocatable
```

Example:

```text
normal:
24 replicas × 35% CPU

upgrade:
18 replicas × 82% CPU
```

Likely capacity issue.

---

# 86. Hypothesis: New Node Networking

Compare:

```text
old node p99 = 410ms
new node p99 = 1.3s
```

Inspect:

```text
CNI
DNS
MTU
packet loss
TCP retransmits
conntrack
routing
```

---

# 87. Hypothesis: Connection Redistribution

Node drain can create:

```text
pods terminate
 ↓
clients reconnect
 ↓
TLS handshakes increase
 ↓
LB redistributes traffic
 ↓
DB connection pools refill
 ↓
latency increases
```

Inspect:

```text
new TCP connections/sec
TLS handshakes
DB connection count
Envoy downstream connections
```

The fix may be smaller rollout batches.

---

# 88. Hypothesis: Autoscaler Lag

```text
pods evicted
 ↓
replacement pods pending
 ↓
autoscaler creates nodes
 ↓
bootstrap takes minutes
```

Remaining pods become overloaded.

Fix:

```text
pre-provision upgrade capacity
```

---

# 89. Pause vs Rollback

Immediate rollback:

```text
critical transaction < 99%
OR
5xx > 1%
OR
p99 > 2× baseline for 5m
OR
data integrity concern
```

Pause and investigate:

```text
p99 > 1.2× baseline
but SLO still healthy

moderate saturation increase

isolated restart increase
```

Important distinction:

```text
stop making it worse
```

is not always the same as:

```text
reverse everything immediately
```

---

# 90. Green Healthy but Rollback Unsafe

Suppose:

```text
100% traffic on Green
Green DB primary
Blue DB stale
```

Then Green networking fails.

This means the architecture crossed its safe rollback boundary.

Future architecture must include one of:

```text
shared external database
```

or:

```text
reverse replication
```

or:

```text
multi-cluster active/standby database architecture
```

Recovery properties must be designed before migration.

---

# 91. EKS Example

```hcl
resource "aws_launch_template" "general" {
  name_prefix = "${var.cluster_name}-general-"

  image_id      = var.approved_ami_id
  instance_type = var.instance_types[0]

  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
  }

  tag_specifications {
    resource_type = "instance"
    tags          = var.tags
  }
}

resource "aws_eks_node_group" "general" {
  cluster_name    = var.cluster_name
  node_group_name = "general"

  node_role_arn = var.node_role_arn
  subnet_ids    = var.private_subnets

  scaling_config {
    min_size     = 6
    desired_size = 9
    max_size     = 30
  }

  update_config {
    max_unavailable = 1
  }

  launch_template {
    id      = aws_launch_template.general.id
    version = aws_launch_template.general.latest_version
  }

  labels = {
    "platform.example.com/pool" = "general"
  }

  tags = var.tags
}
```

---

# 92. AKS Example

```hcl
resource "azurerm_kubernetes_cluster_node_pool" "general" {
  name                  = "general"
  kubernetes_cluster_id = var.cluster_id

  vm_size = var.vm_size

  auto_scaling_enabled = true
  min_count            = 6
  max_count            = 30

  max_pods = 50

  node_labels = {
    "platform.example.com/pool" = "general"
  }

  upgrade_settings {
    max_surge = "33%"
  }

  tags = var.tags
}
```

---

# 93. GKE Example

```hcl
resource "google_container_node_pool" "general" {
  name    = "general"
  cluster = var.cluster_id

  autoscaling {
    min_node_count = 2
    max_node_count = 10
  }

  node_config {
    machine_type = var.machine_type
    image_type   = "COS_CONTAINERD"

    labels = {
      "platform.example.com/pool" = "general"
    }

    resource_labels = var.labels
  }

  upgrade_settings {
    max_surge       = 1
    max_unavailable = 0
  }
}
```

---

# 94. What Should Be Standardized?

Across EKS, AKS, and GKE:

```text
cluster naming
metadata schema
Kubernetes labels
node-pool workload classes
availability tiers
PDB requirements
readiness requirements
topology requirements
SLO gates
GitOps model
policy model
security objectives
image-generation metadata
upgrade rings
observability
rollback semantics
audit format
```

---

# 95. What Should Stay Cloud-Specific?

Do not hide:

```text
AWS VPC CNI
Azure CNI
GKE Dataplane

AWS IAM
Azure Workload Identity
GCP Workload Identity

EBS
Azure Disk
Persistent Disk

AWS launch templates
AKS image channels
GKE image types

provider maintenance controls
cloud load balancers
quota models
node bootstrap
control-plane upgrade behavior
```

Principle:

> Standardize the operational contract, not the provider implementation.

---

# 96. Upgrade Runbook

## T-7 Days

```text
verify target version support
scan deprecated APIs
validate add-ons
validate node images
reserve quota/capacity
run integration tests
```

## T-1 Day

```text
verify SLO/error budget
confirm freeze window
confirm no major incident
verify PDBs
verify backups
verify rollback assets
verify previous image generation
```

## T-30 Minutes

```text
record baseline metrics
disable conflicting changes
confirm autoscaling headroom
run synthetics
```

## Execution

```text
control-plane upgrade
observe
canary node upgrade
observe
progressive batches
SLO gate every batch
```

## Completion

```text
validate node versions
validate image generation
verify workloads
verify SLO
restore automation
archive audit record
```

---

# 97. Migration Runbook

## Prepare

```text
select strategy
classify every state store
identify rollback boundary
design traffic control
define Green capacity
```

## Build

```text
provision Green
install platform components
validate identities
validate secrets
validate ingress/certs
deploy applications
```

## State

```text
synchronize databases
restore/replicate storage
verify consistency
verify replication lag
```

## Validate

```text
synthetic transactions
performance tests
dependency tests
chaos tests
```

## Traffic

```text
100/0
95/5
75/25
50/50
10/90
0/100
```

## Stateful Cutover

```text
fence Blue writes
catch up replication
promote Green
establish reverse replication when required
```

## Soak

Observe for the defined production window.

## Retire

```text
disable Blue writes
retain recovery infrastructure
archive evidence
destroy Blue only after rollback TTL
```

---

# 98. Failure Injection Suite

```text
experiment/pdb-block

experiment/node-capacity-exhaustion

experiment/green-network-latency

experiment/registry-auth-failure

experiment/cni-mtu-mismatch

experiment/database-replication-lag

experiment/post-write-green-failure
```

Success means:

```text
orchestrator detected risk
stopped safely
preserved customer SLO
created useful diagnostics
preserved rollback where possible
```

---

# 99. Auditability

Every run emits an audit record.

```json
{
  "run_id": "run-55192",
  "actor": "platform-upgrader",
  "strategy": "BLUE_GREEN",
  "cluster": "prod-use1-orders",
  "source_version": "N",
  "target_version": "N+1",
  "source_image": "2026.07.4",
  "target_image": "2026.08.3",
  "approval": "CHG-55123",
  "traffic_steps": [
    "0%",
    "5%",
    "25%"
  ],
  "result": "ROLLED_BACK",
  "rollback_reason": "p99 latency regression"
}
```

Supports:

```text
security audits
incident review
compliance evidence
fleet analytics
```

---

# 100. Recommended Build Sequence

```text
Phase 1
Single EKS cluster
standard pools + resilient workload

Phase 2
In-place node upgrade
manual SLO gates

Phase 3
Automated upgrade orchestrator

Phase 4
Blue-green EKS clusters
traffic shift

Phase 5
Stateful migration

Phase 6
Failure-injection exercises

Phase 7
AKS implementation

Phase 8
GKE implementation

Phase 9
Metadata/policy enforcement

Phase 10
Fleet rings

Phase 11
Automated drift/compliance

Phase 12
Production architecture review
```

---

# 101. Staff-Level Design Decisions

## Decision 1: Success Is Application SLO

```text
Node Ready != application healthy

Upgrade complete != customer healthy
```

## Decision 2: Immutable Node Replacement

Avoid:

```text
manual host drift
unknown rollback
irreproducible fixes
```

## Decision 3: Standardize Workload Classes, Not VM SKUs

Portable:

```text
general
compute
memory
```

Cloud-specific:

```text
c7i.2xlarge
Standard_D8ds_v5
n2-standard-8
```

## Decision 4: Prefer Provider-Supported Image Models Where Appropriate

Over-customization can reduce:

```text
supportability
upgrade compatibility
security update reliability
```

## Decision 5: Application Readiness Is a Platform Contract

Tier-0 workloads require:

```text
multiple replicas
PDB
readiness
graceful termination
topology spreading
resource requests
SLO instrumentation
```

## Decision 6: Rollback Is State-Aware

Track separately:

```text
traffic reversible?
data reversible?
infrastructure reversible?
```

## Decision 7: Cloud Differences Remain Visible

Meaningful provider constraints should not be hidden behind fake abstractions.

---

# 102. Unsafe Design Patterns

Avoid:

```text
force-draining PDB violations

DNS TTL as the only migration controller

Node Ready as Green validation

upgrading CNI + Kubernetes + OS + service mesh together

building Green without a state strategy

allowing Blue and Green to write independently

progressing only on HTTP 5xx

using latest image tags

manual production host patching

one rollout concurrency for all workload tiers

destroying Blue immediately after cutover

forcing identical OS-image implementation across clouds
```

---

# 103. Final System

```text
                 Infrastructure as Code
                         +
                  Metadata Standards
                         +
                  Policy as Code
                         +
               Standard Node Pools
                         +
              Hardened Node Baseline
                         +
                Immutable Lifecycle
                         +
        ┌────────────────┴────────────────┐
        │                                 │
 In-Place Upgrade                  Blue-Green Cluster
        │                                 │
        └────────────────┬────────────────┘
                         │
                  Progressive Change
                         │
                         ▼
                    SLO Gates
                         │
                  ┌──────┴──────┐
                  │             │
               Healthy       Unhealthy
                  │             │
               Continue       Pause
                                │
                             Rollback
                                │
                                ▼
                      Incident Evidence
                                │
                                ▼
                       Fleet Learning
                                │
                                ▼
                    Next Rollout Ring
```

---

# 104. Why Zero Downtime Is an Architectural Property

Zero downtime does not come from:

```bash
kubectl drain
```

or:

```text
maxSurge=1
```

or:

```text
Blue-Green
```

It comes from:

```text
application replication
+
correct readiness
+
graceful termination
+
connection draining
+
sufficient capacity
+
failure-domain spreading
+
load balancing
+
idempotent workers
+
database topology
+
state replication
+
compatible schemas
+
network redundancy
+
observability
+
SLO-based decisions
+
rollback design
+
progressive blast radius
```

Examples:

```text
Perfect Kubernetes upgrade
+
single application replica
=
downtime
```

```text
Perfect Blue-Green infrastructure
+
unsynchronized database
=
data loss
```

```text
six replicas
+
all replicas in one failure domain
=
availability failure
```

Therefore:

> **Zero downtime is an emergent property of application architecture, infrastructure architecture, traffic management, state management, and operational control working together.**

---

# 105. Staff / Principal Design Review Questions

You should be able to defend answers to:

```text
Why didn't you use Blue-Green everywhere?

Where exactly is your rollback boundary?

How do you know Green is healthy?

What happens when a PDB blocks indefinitely?

How much excess capacity does an upgrade require?

What happens to an active gRPC stream during drain?

Why can't PVC snapshots solve your database migration?

How do you prevent split-brain?

What changes after Green accepts its first write?

Why isn't DNS weighting sufficient for precise canaries?

How do you roll out a CNI change independently?

How do you prove an OS image is compliant?

Why isn't the same golden image used for EKS, AKS, and GKE?

What abstraction exists across clouds?

What deliberately remains cloud-specific?

How do you prevent fleet-wide correlated failure?

What is the maximum production concurrency?

What automatically stops a rollout?

What happens if the orchestrator crashes halfway through?

How does the next process know where to resume?

Which metrics make decisions?

Which metrics are diagnostic only?

When does latency trigger pause rather than rollback?

How long do you retain Blue?

What prevents Blue from being destroyed too early?

Who owns recovery after data ownership moves to Green?
```

---

# 106. Final Deliverables

The completed Atlas project contains:

```text
1. Architecture diagram

2. Repository structure

3. Terraform implementation

4. Kubernetes manifests

5. Node-pool standards

6. OS/node-image lifecycle

7. Tagging/compliance policies

8. In-place upgrade workflow

9. Blue-Green migration workflow

10. Traffic migration workflow

11. SLO/rollback logic

12. Failure-injection scenarios

13. Upgrade runbook

14. Migration runbook

15. Staff-level design decisions

16. Interview/design-review questions
```

---

# 107. Resume / Interview Summary

A concise Staff-level description:

> **Designed a multi-cloud Kubernetes fleet-evolution platform that treats upgrades and cluster migrations as progressive, SLO-gated state transitions, standardizes workload and security contracts across EKS, AKS, and GKE without hiding provider differences, and explicitly models blast radius, data ownership, reversibility, compliance, and fleet-level failure containment.**
