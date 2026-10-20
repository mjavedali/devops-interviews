Create **one realistic, end-to-end, production-grade Staff/Principal-level platform engineering project** focused specifically on:

> **Zero-Downtime In-Place Kubernetes Upgrades vs. Blue-Green Cluster Migration for Mission-Critical Production Workloads, with Base Tagging, Standardized Node Pools, and OS Image Hardening across Multi-Cloud Kubernetes Clusters.**

The goal is to build **deep practical and architectural expertise**, not just learn definitions.

Use a realistic enterprise scenario with **EKS, AKS, and GKE**, production workloads, multiple environments, strict availability requirements, security/compliance requirements, and a platform team responsible for the Kubernetes fleet.

---

## 1. Project Objective

Build a platform that can safely:

- upgrade an existing production Kubernetes cluster **in-place with zero customer-visible downtime**
- alternatively provision a **new Green cluster** and migrate production workloads from Blue → Green
- decide intelligently between the two strategies
- standardize cluster infrastructure across AWS, Azure, and GCP
- enforce mandatory resource tags/labels
- standardize node pools
- build and consume hardened OS images
- detect configuration drift and compliance violations
- progressively roll out changes with measurable safety gates
- stop and roll back when production health deteriorates

The project should demonstrate **how a Staff Engineer thinks about reliability, blast radius, operational risk, standardization, and fleet-scale automation**.

---

# 2. The Production Scenario

Assume:

```text
Enterprise Platform
│
├── AWS EKS
├── Azure AKS
└── Google GKE
```

The organization operates:

- dozens/hundreds of Kubernetes clusters
- customer-facing APIs
- highly available stateless services
- stateful services
- ingress
- service mesh
- persistent storage
- autoscaling
- observability
- security agents
- GitOps
- infrastructure-as-code

Business requirement:

```text
No planned customer-visible downtime
during cluster upgrades or migrations.
```

Define realistic SLOs and use them throughout the project.

---

# 3. Start With the Core Problem

Explain the fundamental question:

> **When a production Kubernetes cluster needs a version/OS/node upgrade, should we upgrade the existing cluster in-place, or build a new cluster and migrate workloads to it?**

Explain what actually changes during:

- control-plane upgrades
- worker-node upgrades
- node replacement
- OS image replacement
- Kubernetes version changes
- CNI/CSI upgrades
- add-on upgrades

Then explain why:

> Kubernetes saying “upgrade succeeded” does **not** necessarily mean the application experienced zero downtime.

Use concrete examples.

---

# 4. Strategy A — In-Place Upgrade

Implement a realistic production in-place upgrade.

Show the complete lifecycle:

```text
Preflight
   ↓
Compatibility Validation
   ↓
Capacity Validation
   ↓
Control Plane Upgrade
   ↓
Canary Node Upgrade
   ↓
Observe
   ↓
Progressive Node Rollout
   ↓
Application Validation
   ↓
Complete
```

The project must demonstrate:

- cordon
- drain
- pod eviction
- PDB
- readiness probes
- graceful shutdown
- topology spread
- node replacement
- surge capacity
- autoscaling interaction

Explain what happens to real traffic during node replacement.

Demonstrate how to protect:

- HTTP APIs
- long-lived connections
- gRPC
- background workers
- stateful workloads

---

# 5. Strategy B — Blue-Green Cluster Migration

Build a completely new Green cluster rather than mutating Blue.

Architecture:

```text
                 Production Traffic
                        |
                  Traffic Layer
                        |
               +--------+--------+
               |                 |
             BLUE              GREEN
          Existing           New Cluster
```

Implement:

```text
Provision Green
      ↓
Bootstrap platform components
      ↓
Deploy applications
      ↓
Validate Green
      ↓
Canary traffic
      ↓
Progressive traffic migration
      ↓
100% Green
      ↓
Observe
      ↓
Retire Blue
```

Show how workload migration actually works.

Cover:

- application manifests
- configuration
- secrets
- service accounts
- IAM/workload identity
- ingress
- certificates
- persistent storage
- stateful applications
- external dependencies

Do not hand-wave data migration.

For stateful workloads, explain:

- replication
- write ownership
- synchronization
- consistency
- rollback limitations
- split-brain risks

---

# 6. Build the Decision Framework

Create a practical **Staff Engineer decision framework**.

Compare:

```text
In-Place Upgrade
vs.
Blue-Green Migration
```

Evaluate:

- availability risk
- blast radius
- rollback complexity
- infrastructure cost
- operational complexity
- speed
- testing capability
- stateful workloads
- compliance
- customer impact
- team maturity

Then create rules such as:

```text
Use In-Place when...
Use Blue-Green when...
Use a Hybrid Strategy when...
```

The answer must be based on **engineering trade-offs**, not “Blue-Green is always safer.”

---

# 7. Zero-Downtime Traffic Migration

Implement progressive traffic shifting:

```text
Blue 100% / Green 0%

Blue 95% / Green 5%

Blue 75% / Green 25%

Blue 50% / Green 50%

Blue 10% / Green 90%

Blue 0% / Green 100%
```

Use realistic traffic-management mechanisms.

Explain how:

- DNS
- load balancers
- ingress
- Gateway API
- service mesh

can participate in migration.

For every traffic step define:

- success criteria
- observation window
- rollback criteria
- metrics

---

# 8. SLO-Gated Migration

Make the migration **observable and automated**.

Use:

```text
Traffic Shift
      ↓
Metrics
      ↓
SLO Evaluation
      ↓
Healthy?
   /      \
 YES       NO
 |          |
Continue   Rollback
```

Use signals such as:

- 5xx rate
- p95/p99 latency
- pod failures
- readiness failures
- resource saturation
- synthetic transactions
- critical business transactions

Explain why a single metric is insufficient.

---

# 9. Failure Injection

Intentionally break the migration.

Include realistic failures such as:

### Failure 1

Pods cannot be evicted because of a PDB.

### Failure 2

New nodes have insufficient capacity.

### Failure 3

Green cluster shows higher latency.

### Failure 4

Green has image-pull failures.

### Failure 5

A CNI problem appears after upgrade.

### Failure 6

A stateful workload is not synchronized.

### Failure 7

Traffic rollback occurs after writes have already reached Green.

For each failure show:

```text
Symptom
Detection
Root Cause
Immediate Action
Rollback
Permanent Fix
```

This is important: **show the debugging process**, not just the final solution.

---

# 10. Base Tagging Standard

Now add the second major responsibility:

> Standardize cluster/resource metadata across multi-cloud infrastructure.

Define a standard such as:

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

Separate:

```text
Cloud Tags
Kubernetes Labels
Kubernetes Annotations
Terraform Metadata
```

Explain why each exists.

Show how inconsistent tagging creates problems for:

- cost allocation
- ownership
- security
- compliance
- incident response
- automation

---

# 11. Tagging Enforcement

Implement **policy-as-code**.

The flow should look like:

```text
Developer Commit
      ↓
CI Validation
      ↓
Terraform Validation
      ↓
Policy Check
      ↓
Deployment
      ↓
Runtime Compliance Check
```

Demonstrate:

- valid resource
- invalid resource
- policy failure
- remediation

Use realistic policy tooling such as:

- OPA
- Gatekeeper
- Kyverno
- cloud-native policy controls

Do not just show policy syntax; explain **where enforcement belongs and why**.

---

# 12. Standard Node Pool Architecture

Define standard node-pool types:

```text
system
general
compute
memory
specialized
spot/preemptible
```

For every pool define:

- labels
- taints
- tolerations
- instance profile
- CPU/memory characteristics
- scaling boundaries
- workload placement
- upgrade strategy

Show how an application gets scheduled onto the intended pool.

Example:

```text
Workload
   ↓
Node Selector / Affinity
   ↓
Taints / Tolerations
   ↓
Correct Node Pool
```

Then explain **why standard node pools reduce operational complexity during upgrades**.

---

# 13. Standard OS Image Strategy

Build a golden-image lifecycle.

```text
Base OS
   ↓
Hardening
   ↓
Security Configuration
   ↓
Required Agents
   ↓
Vulnerability Scan
   ↓
Validation
   ↓
Approved Golden Image
   ↓
Node Pool
```

Cover:

- OS patching
- kernel configuration
- CIS/security hardening
- SSH configuration
- audit logging
- security agents
- vulnerability scanning
- image versioning
- image promotion
- rollback

Show how the image is consumed consistently across node pools.

---

# 14. Immutable Node Lifecycle

Adopt:

```text
Old Node
   ↓
Cordon
   ↓
Drain
   ↓
Replace
   ↓
Hardened New Node
```

instead of manual server modification.

Explain:

- configuration drift
- immutable infrastructure
- reproducibility
- security posture
- upgrade safety

---

# 15. Multi-Cloud Design

Create one repository that supports:

```text
AWS EKS
Azure AKS
Google GKE
```

but does **not** incorrectly pretend the three platforms are identical.

Show:

- shared standards
- cloud-specific modules
- common tagging model
- common node-pool model
- cloud-specific implementation differences

The project should explicitly identify:

```text
What should be standardized?
What should remain cloud-specific?
```

This is a key Staff-level discussion.

---

# 16. Infrastructure Repository

Create a realistic repository such as:

```text
platform/
├── terraform/
│   ├── modules/
│   ├── aws/
│   ├── azure/
│   └── gcp/
│
├── kubernetes/
│   ├── base/
│   ├── policies/
│   └── workloads/
│
├── images/
├── compliance/
├── upgrade/
├── observability/
├── tests/
└── runbooks/
```

Provide meaningful implementation for the important pieces.

Do not create hundreds of files just for the sake of size.

---

# 17. Automation

Build a small **upgrade/migration orchestrator**.

Use a state machine such as:

```text
DISCOVER
   ↓
PREFLIGHT
   ↓
VALIDATE
   ↓
CANARY
   ↓
OBSERVE
   ↓
PROGRESS
   ↓
COMPLETE
```

Failure:

```text
OBSERVE
   ↓
SLO FAILURE
   ↓
PAUSE
   ↓
ROLLBACK
```

Show how the automation handles:

- retries
- idempotency
- checkpoints
- concurrency
- approval gates
- auditability

---

# 18. Fleet-Level Extension

Once the single-cluster implementation works, extend it to a fleet.

Example:

```text
Ring 0 → Dev
Ring 1 → Test
Ring 2 → Staging
Ring 3 → Small Production
Ring 4 → Large Production
Ring 5 → Mission Critical
```

Explain:

- rollout waves
- regional sequencing
- concurrency limits
- maintenance windows
- freeze windows
- failure propagation
- emergency pause

The project should show how a Staff Engineer moves from:

```text
“I can upgrade a cluster”
```

to:

```text
“I can safely operate upgrades across hundreds of clusters.”
```

---

# 19. Observability

Create an upgrade/migration dashboard showing:

### Cluster Health

- node readiness
- pending pods
- evictions
- pod restarts

### Upgrade Health

- upgrade progress
- node replacement rate
- drain duration
- failures

### Application Health

- request rate
- errors
- latency

### Migration Health

- Blue traffic %
- Green traffic %
- SLO status
- rollback status

Explain which metrics are **diagnostic** versus **decision-making** metrics.

---

# 20. Staff-Level Troubleshooting

Include realistic production situations.

For example:

> “The node upgrade is progressing, but customer latency is increasing.”

Walk through the exact investigation:

```text
What do you check first?
What hypotheses do you form?
Which metrics/logs do you inspect?
What do you stop?
When do you rollback?
```

Also include:

> “Green is healthy but rollback is no longer safe because writes have occurred.”

Explain the architectural implications.

These scenarios should train **engineering judgment**, not command memorization.

---

# 21. Final Architecture Review

At the end, present the final system as a Staff-level architecture.

Explain:

```text
Infrastructure as Code
        +
Standardized Images
        +
Standard Node Pools
        +
Policy as Code
        +
Progressive Upgrades
        +
Blue-Green Migration
        +
SLO Gates
        +
Observability
        +
Automated Rollback
        +
Fleet Rollouts
```

Then answer:

> **Why is zero downtime an architectural property rather than simply an upgrade technique?**

---

# 22. Final Deliverables

The project should finish with:

```text
1. Architecture diagram

2. Repository structure

3. Terraform implementation

4. Kubernetes manifests

5. Node-pool standards

6. OS image lifecycle

7. Tagging/compliance policies

8. In-place upgrade workflow

9. Blue-Green workflow

10. Traffic migration workflow

11. SLO/rollback logic

12. Failure-injection scenarios

13. Upgrade runbook

14. Migration runbook

15. Staff-level design decisions

16. Interview/design-review questions
```

Keep the project **focused on this specific problem**.

Do not turn it into a generic Kubernetes course.

The depth should come from **real implementation + production failure modes + architecture decisions + Staff-level trade-offs**.

For every major component, use the pattern:

```text
Concept
→ Why it matters
→ Production design
→ Implementation
→ Failure scenario
→ Debugging
→ Rollback
→ Staff-level trade-off
```

The final result should feel like a project that a **Staff Platform/SRE Engineer could discuss in a production architecture review or Staff-level interview**, not a beginner Kubernetes lab.