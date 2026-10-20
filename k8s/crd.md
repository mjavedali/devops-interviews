
```
Developing custom Kubernetes Controllers/Operators (CRDs, Client-Go, Controller-Runtime) and
managing essential core cluster addons (CoreDNS, Metrics Server, ExternalDNS, Cluster Autoscaler, Karpenter).
```

This is a very common **Kubernetes platform-engineering interview area**. The easiest way to structure it is:

```text
CRD
 ↓
Custom Resource
 ↓
Controller watches resource
 ↓
Reconcile()
 ↓
Create/update dependent Kubernetes resources
 ↓
Observe status
 ↓
Repeat until desired == actual
```

Then layer on the core addons:

```text
CoreDNS          → service discovery / DNS
Metrics Server   → resource metrics
ExternalDNS      → DNS records from K8s resources
Cluster Autoscaler → changes node-group size
Karpenter        → provisions right-sized nodes dynamically
```

---

# 1. What is a Kubernetes Controller?

A controller implements a **control loop**.

You have:

```text
Desired state
    vs
Actual state
    ↓
Controller
    ↓
Take corrective action
```

For example:

```text
Desired:
3 replicas

Actual:
2 replicas

Controller:
create 1 more
```

The important interview phrase is:

> **A controller is level-triggered, not event-triggered. It continuously drives actual state toward desired state.**

Even if an event is missed, the controller can reconcile again and recover.

---

# 2. CRD

A **CRD (CustomResourceDefinition)** extends Kubernetes with your own API type.

Suppose you want:

```yaml
kind: RedisCluster
```

Create a CRD:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: redisclusters.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: redisclusters
    singular: rediscluster
    kind: RedisCluster
    shortNames:
      - rc
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  type: integer
```

Then users can do:

```yaml
apiVersion: example.com/v1
kind: RedisCluster
metadata:
  name: cache
spec:
  replicas: 3
```

Now:

```bash
kubectl get redisclusters
```

works.

---

# 3. CRD vs Operator

Important interview distinction:

**CRD** = defines a new Kubernetes API object.

**Controller** = contains the logic.

**Operator** = controller + domain-specific operational knowledge, often built around one or more CRDs.

Think:

```text
CRD:
"RedisCluster has replicas"

Controller:
"If RedisCluster says replicas=3,
make the cluster actually have 3."
```

---

# 4. Client-Go

`client-go` is the foundational Go client library for Kubernetes.

You can interact with Kubernetes APIs:

```go
pods, err := clientset.CoreV1().
    Pods("default").
    List(ctx, metav1.ListOptions{})
```

A simplified example:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)

func main() {
	config, err := rest.InClusterConfig()
	if err != nil {
		log.Fatal(err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatal(err)
	}

	pods, err := clientset.CoreV1().
		Pods("default").
		List(context.Background(), metav1.ListOptions{})
	if err != nil {
		log.Fatal(err)
	}

	for _, pod := range pods.Items {
		fmt.Fprintln(os.Stdout, pod.Name)
	}
}
```

This is the low-level building block.

---

# 5. Controller-runtime

`controller-runtime` builds higher-level abstractions over client-go.

Instead of manually implementing:

```text
watch
queue
cache
event handling
retries
rate limiting
```

you get framework support for them.

Typical controller:

```go
type RedisClusterReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}
```

and:

```go
func (r *RedisClusterReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,
) (ctrl.Result, error) {

    var redis examplev1.RedisCluster

    err := r.Get(ctx, req.NamespacedName, &redis)
    if err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Desired state logic...

    return ctrl.Result{}, nil
}
```

That's the piece you should know extremely well for interviews.

---

# 6. Basic controller flow

Imagine:

```yaml
apiVersion: example.com/v1
kind: RedisCluster
metadata:
  name: cache
spec:
  replicas: 3
```

The controller gets notified.

```text
RedisCluster created/updated
          ↓
      watch event
          ↓
      work queue
          ↓
      Reconcile()
          ↓
read RedisCluster
          ↓
calculate desired state
          ↓
compare actual state
          ↓
create/update StatefulSet
          ↓
update Status
```

---

# 7. Actual reconcile code

A realistic simplified controller:

```go
func (r *RedisClusterReconciler) Reconcile(
	ctx context.Context,
	req ctrl.Request,
) (ctrl.Result, error) {

	var redis examplev1.RedisCluster

	if err := r.Get(ctx, req.NamespacedName, &redis); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	desiredReplicas := redis.Spec.Replicas

	var sts appsv1.StatefulSet

	err := r.Get(
		ctx,
		types.NamespacedName{
			Name:      redis.Name,
			Namespace: redis.Namespace,
		},
		&sts,
	)

	if apierrors.IsNotFound(err) {
		sts = buildStatefulSet(&redis, desiredReplicas)

		if err := r.Create(ctx, &sts); err != nil {
			return ctrl.Result{}, err
		}
	} else if err != nil {
		return ctrl.Result{}, err
	}

	if sts.Spec.Replicas == nil ||
		*sts.Spec.Replicas != desiredReplicas {

		sts.Spec.Replicas = &desiredReplicas

		if err := r.Update(ctx, &sts); err != nil {
			return ctrl.Result{}, err
		}
	}

	return ctrl.Result{}, nil
}
```

The crucial property is:

> **Reconcile should be idempotent.**

Calling it once or ten times should converge to the same desired state.

---

# 8. OwnerReferences

A controller often creates dependent objects.

For:

```text
RedisCluster
   ↓
StatefulSet
   ↓
Pods
```

you usually establish ownership.

With controller-runtime:

```go
if err := controllerutil.SetControllerReference(
	redis,
	sts,
	r.Scheme,
); err != nil {
	return err
}
```

Now Kubernetes knows:

```text
RedisCluster cache
      ↓ owns
StatefulSet cache
```

This helps with:

- garbage collection
- event relationships
- controller watches
- lifecycle management

Interview phrase:

> **I use OwnerReferences for dependent resources so their lifecycle follows the custom resource.**

---

# 9. Watches

A controller normally watches its CR:

```go
For(&examplev1.RedisCluster{})
```

But often you also need to react when a dependent resource changes:

```text
RedisCluster
    ↓
StatefulSet
    ↓
Pod
```

You might configure ownership:

```go
Owns(&appsv1.StatefulSet{})
```

Then:

```text
StatefulSet changes
       ↓
controller reconciles RedisCluster
```

This is extremely useful for self-healing.

---

# 10. Status vs Spec

This is another interview favorite.

### Spec

What the user **wants**:

```yaml
spec:
  replicas: 3
```

### Status

What the controller **observes**:

```yaml
status:
  readyReplicas: 2
  phase: Degraded
```

So:

```text
Spec   = desired state
Status = observed state
```

A controller should generally not mutate `spec` just to report reality.

---

# 11. Updating status

Example:

```go
redis.Status.ReadyReplicas = ready
redis.Status.Phase = "Ready"

if err := r.Status().Update(ctx, &redis); err != nil {
	return ctrl.Result{}, err
}
```

Often you'll use conditions:

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      reason: StatefulSetReady
      message: "All Redis replicas are ready"
```

Conditions are often better than a single string like `"Ready"`.

---

# 12. Retry behavior

Suppose Kubernetes API temporarily fails.

Return:

```go
return ctrl.Result{}, err
```

controller-runtime generally requeues on errors.

You can also explicitly requeue:

```go
return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
```

Use this for cases where time itself matters, but don't blindly poll everything.

A key interview point:

> **Controllers should be resilient to transient API errors and eventually converge.**

---

# 13. Optimistic concurrency

Suppose two actors update an object.

Kubernetes objects have:

```text
resourceVersion
```

The API server uses optimistic concurrency.

You might get:

```text
409 Conflict
```

because someone updated the object between your GET and UPDATE.

Your controller shouldn't assume it owns all mutations.

Good controller design:

```text
GET latest object
↓
modify only fields you own
↓
UPDATE/PATCH
↓
retry if conflict
```

This is especially important with status and shared resources.

---

# 14. Patch vs Update

Full:

```go
r.Update(ctx, obj)
```

can overwrite more than you intended.

Patch can be safer for targeted changes:

```go
patch := client.MergeFrom(obj.DeepCopy())

obj.Labels["example.com/managed"] = "true"

if err := r.Patch(ctx, obj, patch); err != nil {
	return err
}
```

Interview point:

> **Prefer narrow mutations and avoid unnecessarily overwriting fields owned by other controllers.**

---

# 15. Finalizers

What if your CR represents an external cloud resource?

Example:

```text
Database CR
     ↓
Cloud database
```

When the CR is deleted, you may need:

```text
CR deleted
   ↓
Controller sees deletionTimestamp
   ↓
Delete external database
   ↓
Remove finalizer
   ↓
Kubernetes completes deletion
```

Example:

```go
if !controllerutil.ContainsFinalizer(&obj, finalizerName) {
	controllerutil.AddFinalizer(&obj, finalizerName)
	return ctrl.Result{}, r.Update(ctx, &obj)
}
```

On deletion:

```go
if !obj.DeletionTimestamp.IsZero() {
	if err := cleanupExternalResource(); err != nil {
		return ctrl.Result{}, err
	}

	controllerutil.RemoveFinalizer(&obj, finalizerName)

	return ctrl.Result{}, r.Update(ctx, &obj)
}
```

Important interview phrase:

> **Finalizers implement cleanup-before-delete semantics.**

---

# 16. CoreDNS

Now the addons.

CoreDNS provides DNS inside the cluster.

If you have:

```text
Service:
payments
Namespace:
prod
```

a Pod can resolve something like:

```text
payments.prod.svc.cluster.local
```

Conceptually:

```text
Pod
 ↓
DNS query
 ↓
CoreDNS
 ↓
Kubernetes Service/Endpoints information
 ↓
ClusterIP / service records
```

Without CoreDNS, a huge amount of Kubernetes service discovery breaks.

### What would you monitor?

```text
DNS query latency
SERVFAIL
NXDOMAIN
CPU/memory
CoreDNS replicas
upstream DNS failures
cache hit ratio
```

A classic troubleshooting sequence:

```bash
kubectl get pods -n kube-system
kubectl get svc -n kube-system
kubectl logs -n kube-system deploy/coredns
```

Then test from a pod:

```bash
nslookup kubernetes.default.svc.cluster.local
```

or:

```bash
dig kubernetes.default.svc.cluster.local
```

---

# 17. Metrics Server

Metrics Server provides resource metrics such as:

```text
CPU
memory
```

for nodes and pods.

This is what enables things like:

```bash
kubectl top pods
kubectl top nodes
```

and contributes to HPA resource-metric workflows.

Architecture:

```text
Kubelets
   ↓
Metrics Server
   ↓
Metrics API
   ↓
HPA / kubectl top
```

Important distinction:

> **Metrics Server is not Prometheus.**

Metrics Server is primarily for short-lived resource metrics used by Kubernetes mechanisms such as HPA.

Prometheus is generally used for broader observability, historical metrics, alerting, etc.

---

# 18. ExternalDNS

ExternalDNS synchronizes Kubernetes resources with external DNS providers.

Suppose:

```yaml
kind: Ingress
spec:
  rules:
    - host: api.example.com
```

ExternalDNS can create/update the external DNS record.

Architecture:

```text
Ingress / Service
      ↓
ExternalDNS
      ↓
DNS provider
      ↓
api.example.com → Load Balancer IP
```

It watches Kubernetes resources and reconciles DNS records.

Again the same controller pattern:

```text
desired:
api.example.com → LB address

actual:
record missing

ExternalDNS:
create record
```

### Security concern

ExternalDNS has significant DNS mutation permissions.

So you'd want:

```text
least-privilege IAM
limited zones
appropriate RBAC
ownership records
audit logs
```

---

# 19. Cluster Autoscaler

Cluster Autoscaler usually works with **predefined node groups**.

Example:

```text
NodeGroup:
min = 3
max = 10
current = 3
```

Suppose there are unschedulable Pods:

```text
Pod
 ↓
Scheduler
 ↓
Can't fit on existing nodes
 ↓
Cluster Autoscaler
 ↓
increase node group
 ↓
new node
 ↓
Pod schedules
```

Scale-down works in the opposite direction:

```text
underutilized nodes
      ↓
safe to remove?
      ↓
drain/evict
      ↓
reduce node group
```

Cluster Autoscaler generally answers:

> **How many nodes should this node group have?**

---

# 20. Karpenter

Karpenter takes a somewhat different approach.

Instead of primarily saying:

```text
increase node group from 3 → 4
```

it considers pending workload requirements and provisions an appropriate node.

For example:

```text
Pod requires:
8 CPU
32 GiB RAM
GPU
region constraint
instance architecture
```

Karpenter can select an appropriate instance type based on available cloud capacity and constraints.

Conceptually:

```text
Pending Pod
    ↓
Karpenter
    ↓
What node can fit this Pod?
    ↓
choose instance type
    ↓
provision node
    ↓
Pod schedules
```

That's one reason Karpenter is attractive for dynamic workloads.

---

# 21. Cluster Autoscaler vs Karpenter

Memorize this table:

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| Main idea | Resize node groups | Provision right-sized nodes |
| Scaling model | Existing node-group abstraction | More dynamic provisioning |
| Granularity | Node group | Individual node selection |
| Scheduling awareness | Less direct | Stronger workload-driven provisioning |
| Cost optimization | Good | Often more flexible |
| Instance selection | Node-group based | Dynamic |

Interview-safe summary:

> **Cluster Autoscaler usually changes the size of predefined node groups, while Karpenter provisions nodes dynamically based on pending workloads and their scheduling requirements.**

---

# 22. How Karpenter interacts with the scheduler

Important nuance:

Karpenter does not simply replace the Kubernetes scheduler.

The flow is roughly:

```text
Pod created
   ↓
Scheduler attempts placement
   ↓
Pod remains Pending
   ↓
Karpenter observes unschedulable demand
   ↓
select/provision capacity
   ↓
Node joins cluster
   ↓
Scheduler places Pod
```

So:

```text
Scheduler → "Where can this Pod run?"
Karpenter → "What capacity should I create so it can run?"
```

---

# 23. Very common interview debugging scenario

**Question: "Pods are Pending. How do you debug autoscaling?"**

I'd walk it in this order:

```text
1. kubectl describe pod
2. inspect scheduling events
3. understand why Pod is unschedulable
4. check node capacity/taints/affinity
5. check autoscaler/Karpenter logs
6. check node provisioning events
7. verify cloud IAM/quota/capacity
8. verify new node joined cluster
```

Commands:

```bash
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
kubectl get nodes
kubectl describe node <node>
```

Then for Karpenter:

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system <karpenter-pod>
```

The key is not just:

> "Autoscaler is broken."

You determine **why the scheduler cannot place the Pod** first.

---

# 24. Core addons together

These components form a pretty interesting control plane/platform stack:

```text
                    Kubernetes

                       API
                        │
        ┌───────────────┼────────────────┐
        │               │                │
     CoreDNS       Metrics Server    ExternalDNS
        │               │                │
   Service DNS      Resource data     External DNS
                        │
                        ▼
                       HPA
                        │
                        ▼
                Pending workload
                        │
                        ▼
              Cluster Autoscaler /
                   Karpenter
                        │
                        ▼
                      Nodes
```

---

# 25. A realistic custom operator

Suppose you build:

```yaml
kind: Application
spec:
  replicas: 3
  image: nginx:1.27
  autoscaling:
    enabled: true
```

Your operator might create:

```text
Application
   │
   ├── Deployment
   ├── Service
   ├── ConfigMap
   ├── HPA
   └── NetworkPolicy
```

Then continuously reconcile:

```text
Application.spec
       ↓
desired Deployment
       ↓
actual Deployment
       ↓
fix drift
       ↓
update status
```

This is essentially how many Kubernetes-native systems are designed.

---

# 26. Full controller skeleton

This is worth knowing for an interview:

```go
package controllers

import (
	"context"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"

	examplev1 "example.com/api/v1"
)

type ApplicationReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

func (r *ApplicationReconciler) Reconcile(
	ctx context.Context,
	req ctrl.Request,
) (ctrl.Result, error) {

	var app examplev1.Application

	// 1. Read the custom resource.
	if err := r.Get(ctx, req.NamespacedName, &app); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// 2. Handle deletion / finalizer if required.

	// 3. Compute desired state.
	desired := buildDesiredDeployment(&app)

	// 4. Set ownership.
	if err := ctrl.SetControllerReference(
		&app,
		desired,
		r.Scheme,
	); err != nil {
		return ctrl.Result{}, err
	}

	// 5. Create or update dependent resources.
	var existing Deployment
	err := r.Get(ctx, req.NamespacedName, &existing)

	if err != nil {
		// Simplified example.
		_ = err
		if err := r.Create(ctx, desired); err != nil {
			return ctrl.Result{}, err
		}
	}

	// 6. Update status.
	// app.Status.ReadyReplicas = ...

	// 7. Requeue only when useful.
	return ctrl.Result{}, nil
}

func (r *ApplicationReconciler) SetupWithManager(
	mgr ctrl.Manager,
) error {

	return ctrl.NewControllerManagedBy(mgr).
		For(&examplev1.Application{}).
		Owns(&appsv1.Deployment{}).
		Complete(r)
}
```

In a real implementation you'd handle:

- `IsNotFound`
- update vs create properly
- status conditions
- finalizers
- conflicts
- immutable fields
- idempotency
- adoption/orphaning
- deletion
- retries
- metrics

---

# 27. What interviewers really test

They usually aren't testing whether you memorized every `controller-runtime` method.

They're testing whether you understand:

### Reconciliation

```text
desired → actual → diff → corrective action
```

### Idempotency

```text
reconcile()
reconcile()
reconcile()
```

should converge safely.

### Eventual consistency

The API may temporarily be:

```text
desired = 3
actual = 2
```

but the controller keeps driving toward:

```text
desired = 3
actual = 3
```

### Ownership

```text
CR → dependent resources
```

### Failure recovery

Controller crashes?

```text
restart
↓
read current cluster state
↓
reconcile
↓
recover
```

That's why controllers are powerful.

---

# 28. Strong interview response

For your resume line, I'd explain it like this:

> **"I built Kubernetes-native operators using CRDs and controller-runtime, with reconcile loops that watched custom resources, created and updated dependent objects, maintained status conditions, handled finalizers and ownership, and were designed to be idempotent and eventually consistent. I also managed core cluster addons such as CoreDNS, Metrics Server, ExternalDNS, Cluster Autoscaler and Karpenter, including deployment, RBAC, health monitoring and troubleshooting. For autoscaling, I understand the distinction between Cluster Autoscaler, which primarily adjusts predefined node groups, and Karpenter, which dynamically provisions capacity based on pending workload requirements."**

## The one diagram I'd memorize

```text
                 ┌───────────────┐
                 │     CRD       │
                 └──────┬────────┘
                        │
                        ▼
                 Custom Resource
                        │
                        ▼
                   Controller
                        │
                  Reconcile()
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
        Deployment    Service      HPA
             │
             ▼
           Pods
             │
             ▼
        Scheduler
             │
         Pending?
             │
       ┌─────┴─────┐
       ▼           ▼
 Cluster         Karpenter
Autoscaler
       │           │
       └─────┬─────┘
             ▼
           Nodes
```

And alongside that:

```text
CoreDNS      → DNS
Metrics      → CPU/memory metrics
ExternalDNS  → external DNS records
Autoscaler   → node-group size
Karpenter    → dynamic node provisioning
Controller   → desired-state reconciliation
```
