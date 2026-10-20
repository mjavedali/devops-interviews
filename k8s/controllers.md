```
Walk me through to create an operator/crd etc granular steps , coding wherever required and everything they can ask in an interview .
```
Yes. Let's turn this into a **hands-on Kubernetes Operator course**, not just theory.

We will build a real `WebApp` Operator in Go, step by step:

```text
WebApp CR
   ↓
WebApp Controller
   ↓
Deployment + Service
   ↓
Pods
```

Then we'll progressively make it production-grade:

```text
CRD
→ CR
→ Reconcile()
→ Deployment
→ Service
→ OwnerReferences
→ Status
→ Conditions
→ Finalizers
→ RBAC
→ Watches
→ Errors/retries
→ Idempotency
→ Tests
→ Metrics
→ Leader election
→ HA
→ Upgrades/versioning
```

At every stage I'll give you:

**ELI5 explanation → exact commands → code → what Kubernetes is doing → debugging → interview questions → interview answer.**

The current Kubebuilder quick-start uses Go, Docker, kubectl, and a Kubernetes cluster, and scaffolds an API/controller with `kubebuilder create api`. [Kubebuilder Book](https://book.kubebuilder.io/quick-start.html?utm_source=chatgpt.com)

---

# PART 0 — The one picture you must understand

Before writing a single line of Go, memorize this:

```text
                    YOU
                     │
                     │ kubectl apply
                     ▼
              ┌───────────────┐
              │  API SERVER   │
              └───────┬───────┘
                      │
                      ▼
                WebApp CR
                      │
              "I want 3 nginx"
                      │
                      ▼
          ┌──────────────────────┐
          │  WEBAPP CONTROLLER   │
          └──────────┬───────────┘
                     │
                     ▼
                Deployment
                     │
                     ▼
                   Pods
```

The **CR is the request**.

The **controller is the worker**.

The **API server is the database + front door**.

The **controller's job is to continuously make reality match the CR**.

This is the core Kubernetes controller/operator pattern. Kubernetes explicitly describes a custom resource plus a custom controller as a declarative API, with operators using the control-loop pattern. [Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/?utm_source=chatgpt.com)

---

# PART 1 — CRD vs CR vs Controller vs Operator

This gets asked constantly.

## CRD

**CustomResourceDefinition**

It tells Kubernetes:

> "There is a new type of object called `WebApp`."

Think:

```text
CRD = blueprint
```

Example:

```text
WebApp
```

---

## CR

**Custom Resource**

An actual object created using that blueprint.

```yaml
kind: WebApp
metadata:
  name: my-webapp
```

Think:

```text
CRD = blueprint
CR  = actual house
```

---

## Controller

A program that watches resources and takes action.

```text
WebApp
   ↓
Controller
   ↓
Deployment
```

---

## Operator

Usually:

```text
CRD(s)
+
Controller(s)
+
Operational knowledge
```

Kubernetes describes operators as software extensions that use custom resources to manage applications/components and implement control loops. [Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/?utm_source=chatgpt.com)

---

# PART 2 — Our project

We're going to create:

```text
WebApp
```

The user will write:

```yaml
apiVersion: apps.example.com/v1
kind: WebApp
metadata:
  name: hello
spec:
  image: nginx:1.27
  replicas: 3
```

Our controller will make this happen:

```text
WebApp
   │
   ├── Deployment
   │      └── 3 Pods
   │
   └── Service
```

And eventually our CR should show:

```yaml
status:
  replicas: 3
  readyReplicas: 3
  conditions:
    - type: Ready
      status: "True"
```

---

# PART 3 — Set up your machine

You need:

```text
Go
kubectl
Docker
Kubernetes cluster
Kubebuilder
```

For learning, use a local Kubernetes cluster such as **kind**.

You can think:

```text
kind
=
Kubernetes cluster running inside Docker
```

The current Kubebuilder documentation recommends a Kubernetes cluster for development/testing and supports local clusters such as kind. [Kubebuilder Book](https://book.kubebuilder.io/quick-start.html?utm_source=chatgpt.com)

Check:

```bash
go version
kubectl version --client
docker version
kubectl cluster-info
```

You want this to work:

```bash
kubectl get nodes
```

Example:

```text
NAME                 STATUS   ROLES
kind-control-plane   Ready    control-plane
```

---

# PART 4 — Create the Operator project

Create a directory:

```bash
mkdir webapp-operator
cd webapp-operator
```

Initialize Kubebuilder:

```bash
kubebuilder init \
  --domain example.com \
  --repo example.com/webapp-operator
```

Kubebuilder scaffolds the project and Go module for you. [Kubebuilder Book](https://book.kubebuilder.io/quick-start.html?utm_source=chatgpt.com)

Now:

```bash
ls
```

You'll see a project structure roughly like:

```text
webapp-operator/
├── api/
├── config/
├── internal/
├── cmd/
├── go.mod
├── go.sum
├── Makefile
└── PROJECT
```

Don't worry about every file yet.

---

# PART 5 — Create our CRD + Controller

Run:

```bash
kubebuilder create api \
  --group apps \
  --version v1 \
  --kind WebApp
```

When asked:

```text
Create Resource [y/n]
```

answer:

```text
y
```

When asked:

```text
Create Controller [y/n]
```

answer:

```text
y
```

Kubebuilder's current quick start uses this exact API-scaffolding workflow. [Kubebuilder Book](https://book.kubebuilder.io/quick-start.html?utm_source=chatgpt.com)

Now you get something conceptually like:

```text
api/v1/
    webapp_types.go

internal/controller/
    webapp_controller.go
```

These are two files you should become extremely comfortable with.

---

# PART 6 — Understand the Go API definition

Open:

```text
api/v1/webapp_types.go
```

You'll see something roughly like:

```go
type WebAppSpec struct {
    // desired state
}

type WebAppStatus struct {
    // observed state
}

type WebApp struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   WebAppSpec   `json:"spec,omitempty"`
    Status WebAppStatus `json:"status,omitempty"`
}
```

Pause.

This is extremely important.

A Kubernetes object looks conceptually like:

```text
WebApp
│
├── metadata
│
├── spec
│
└── status
```

---

# PART 7 — Spec vs Status

Interviewers LOVE this.

Suppose:

```yaml
spec:
  replicas: 3
```

This means:

> "I want 3."

Suppose your controller sees:

```text
Deployment replicas = 3
ready replicas = 2
```

Then:

```yaml
status:
  replicas: 3
  readyReplicas: 2
```

means:

> "I know you asked for 3, but currently only 2 are ready."

So:

```text
spec
=
DESIRED STATE

status
=
OBSERVED STATE
```

### Interview question

**Q: Why shouldn't you put desired state inside status?**

Answer:

> `spec` represents what the user wants; `status` represents what the controller observes. Keeping them separate follows Kubernetes declarative API conventions and lets the controller continuously reconcile actual state toward desired state.

Kubernetes' custom-resource documentation explicitly describes `.spec` and `.status` as standard API conventions for declarative resources. [Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/?utm_source=chatgpt.com)

---

# PART 8 — Define our WebApp API

Let's make this:

```go
type WebAppSpec struct {
    Image    string `json:"image"`
    Replicas *int32 `json:"replicas,omitempty"`
}

type WebAppStatus struct {
    Replicas      int32 `json:"replicas,omitempty"`
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`
}
```

Conceptually:

```text
WebApp

spec:
    image
    replicas

status:
    replicas
    readyReplicas
```

Now our CR can be:

```yaml
apiVersion: apps.example.com/v1
kind: WebApp
metadata:
  name: hello
spec:
  image: nginx:1.27
  replicas: 3
```

---

# PART 9 — Why `*int32` instead of `int32`?

Interview trap.

Why:

```go
Replicas *int32
```

instead of:

```go
Replicas int32
```

Because pointers allow you to distinguish:

```text
not provided
```

from:

```text
provided as 0
```

With:

```go
int32
```

the zero value is:

```text
0
```

With:

```go
*int32
```

you can have:

```text
nil
```

meaning:

> user didn't specify it.

This becomes particularly useful when designing Kubernetes APIs.

---

# PART 10 — Kubebuilder markers

You will see comments such as:

```go
// +kubebuilder:validation:Minimum=1
// +kubebuilder:validation:Maximum=10
```

These are instructions to **controller-gen**.

For example:

```go
// +kubebuilder:validation:Minimum=1
// +kubebuilder:validation:Maximum=10
Replicas *int32 `json:"replicas,omitempty"`
```

means:

```text
replicas >= 1
replicas <= 10
```

Then Kubebuilder generates the CRD's OpenAPI schema.

So:

```text
Go type
   ↓
Kubebuilder markers
   ↓
controller-gen
   ↓
CRD YAML
   ↓
Kubernetes API validation
```

This is important because validation happens at the API boundary instead of relying only on your controller to reject bad input.

---

# PART 11 — Generate the CRD

Run:

```bash
make manifests
```

Kubebuilder generates the CRD manifests from your Go API definitions. [Kubebuilder Book](https://book.kubebuilder.io/quick-start.html?utm_source=chatgpt.com)

You'll get something under:

```text
config/crd/bases/
```

You may see:

```text
config/crd/bases/apps.example.com_webapps.yaml
```

Now Kubernetes can understand:

```text
WebApp
```

---

# PART 12 — Install the CRD

Run:

```bash
make install
```

Then:

```bash
kubectl get crd
```

You should see:

```text
webapps.apps.example.com
```

Notice the naming:

```text
webapps.apps.example.com
```

Break it down:

```text
webapps
   │
   └── plural resource name

apps
   │
   └── API group

example.com
   │
   └── domain
```

And the API version:

```text
apps.example.com/v1
```

---

# PART 13 — CRD vs API Group vs Version vs Kind

Interviewers may give you:

```yaml
apiVersion: apps.example.com/v1
kind: WebApp
```

and ask you to explain it.

Answer:

```text
apps.example.com
    =
API group

v1
    =
API version

WebApp
    =
Kind
```

Resource is usually plural:

```text
webapps
```

So:

```bash
kubectl get webapps
```

or:

```bash
kubectl get webapp
```

may work depending on resource aliases.

---

# PART 14 — Create an actual Custom Resource

Create:

```text
config/samples/apps_v1_webapp.yaml
```

with:

```yaml
apiVersion: apps.example.com/v1
kind: WebApp
metadata:
  name: hello
spec:
  image: nginx:1.27
  replicas: 3
```

Apply:

```bash
kubectl apply -f config/samples/apps_v1_webapp.yaml
```

Then:

```bash
kubectl get webapps
```

You'll see:

```text
NAME     AGE
hello    10s
```

Congratulations.

You just added a **new API object to Kubernetes**.

---

# PART 15 — But NOTHING actually happens yet

This is a crucial realization.

You created:

```text
CRD
```

Then:

```text
CR
```

But no Deployment is created.

Why?

Because a CRD only tells Kubernetes:

> "This object type exists."

A CRD does not magically know your business logic.

Kubernetes' documentation explicitly notes that custom resources by themselves provide storage/API behavior; combining them with a custom controller gives you the declarative automation. [Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/?utm_source=chatgpt.com)

So:

```text
CRD
+
CR
=
data/API object

CRD
+
CR
+
Controller
=
automation
```

---

# PART 16 — The controller

Open:

```text
internal/controller/webapp_controller.go
```

You'll find:

```go
func (r *WebAppReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,
) (ctrl.Result, error) {
```

This function is the heart of your operator.

Think:

```text
Kubernetes:
"Hey controller, something concerning WebApp hello happened."

Controller:
"Okay, let me inspect the world."

Reconcile():

1. Find WebApp
2. Read desired state
3. Find Deployment
4. Compare desired vs actual
5. Fix differences
6. Return
```

---

# PART 17 — First useful Reconcile()

Start simple:

```go
func (r *WebAppReconciler) Reconcile(
    ctx context.Context,
    req ctrl.Request,
) (ctrl.Result, error) {

    var webApp appsv1.WebApp

    err := r.Get(ctx, req.NamespacedName, &webApp)
    if err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    fmt.Println(
        "Reconciling WebApp:",
        webApp.Name,
    )

    return ctrl.Result{}, nil
}
```

What happened?

```text
WebApp created
       ↓
Kubernetes event
       ↓
controller
       ↓
Reconcile()
       ↓
Get WebApp
       ↓
print
```

---

# PART 18 — Run the controller locally

Run:

```bash
make run
```

Kubebuilder supports running the controller locally against the cluster selected by your kubeconfig. [Kubebuilder Book](https://book.kubebuilder.io/quick-start.html?utm_source=chatgpt.com)

Now in another terminal:

```bash
kubectl apply -f config/samples/apps_v1_webapp.yaml
```

Watch your controller terminal.

You should see a reconciliation event.

This is your first working controller.

---

# PART 19 — Understand the event pipeline

When you run:

```bash
kubectl apply -f webapp.yaml
```

this happens:

```text
kubectl
  │
  ▼
API Server
  │
  ▼
stores WebApp
  │
  ▼
watch event
  │
  ▼
controller cache/informer machinery
  │
  ▼
queue
  │
  ▼
worker
  │
  ▼
Reconcile()
```

This is where **client-go** enters the picture.

---

# PART 20 — Client-Go

`client-go` is the Go client ecosystem for talking to Kubernetes.

At a high level:

```text
Go program
    │
    ▼
client-go
    │
    ▼
Kubernetes API Server
```

client-go includes machinery such as clients, informers, caches and workqueues; the official examples include a controller built from an informer, indexer/cache, and rate-limited workqueue. [GitHub](https://github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go?utm_source=chatgpt.com)

---

# PART 21 — Why informers?

Imagine your controller doing this every second:

```text
GET all WebApps
GET all WebApps
GET all WebApps
GET all WebApps
```

Terrible idea.

Instead:

```text
API Server
     │
     │ watch
     ▼
Informer
     │
     ▼
Local cache
```

The informer watches Kubernetes changes and maintains a local view.

Think of it as:

> "Kubernetes, tell me when something changes. Don't make me repeatedly ask."

---

# PART 22 — Why a work queue?

Suppose 500 resources change.

You don't necessarily want 500 reconciliation operations randomly running at once.

Instead:

```text
events
  │
  ▼
work queue
  │
  ├── webapp/foo
  ├── webapp/bar
  ├── webapp/baz
  └── ...
```

Workers process queue items.

Conceptually:

```go
for {
    item := queue.Get()

    reconcile(item)

    queue.Done(item)
}
```

The official client-go controller example uses an informer/cache and a rate-limiting workqueue, and specifically notes that a queue item may cause reconciliation to see a newer object version than the event that originally enqueued it. [GitHub](https://github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go?utm_source=chatgpt.com)

That leads to a major interview concept:

> **Events are notifications, not the source of truth.**

Your source of truth is the current Kubernetes state.

---

# PART 23 — Controller-runtime

Manually doing client-go controller plumbing requires a lot of code.

Controller-runtime gives you the higher-level building blocks.

Think:

```text
client-go
   │
   ├── API client
   ├── informer
   ├── cache
   ├── queue
   └── low-level machinery
          │
          ▼
controller-runtime
   │
   ├── Manager
   ├── Client
   ├── Controller
   ├── Reconciler
   └── higher-level abstractions
```

This is why modern operator development commonly uses:

```text
Kubebuilder
+
controller-runtime
```

rather than writing all informer/queue machinery yourself.

---

# PART 24 — Create a Deployment from the WebApp

Now we're actually becoming an operator.

Desired:

```yaml
spec:
  image: nginx:1.27
  replicas: 3
```

We want:

```text
Deployment
    replicas = 3
    image = nginx:1.27
```

Inside `Reconcile()`:

```go
deployment := &appsv1.Deployment{}

deployment.Name = webApp.Name
deployment.Namespace = webApp.Namespace
```

Now configure it.

A simplified example:

```go
deployment.Spec = appsv1.DeploymentSpec{
    Replicas: webApp.Spec.Replicas,
    Selector: &metav1.LabelSelector{
        MatchLabels: map[string]string{
            "app": webApp.Name,
        },
    },
    Template: corev1.PodTemplateSpec{
        ObjectMeta: metav1.ObjectMeta{
            Labels: map[string]string{
                "app": webApp.Name,
            },
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{
                {
                    Name:  "web",
                    Image: webApp.Spec.Image,
                },
            },
        },
    },
}
```

Now the important part:

```text
Desired:
Deployment exists

Actual:
Maybe Deployment doesn't exist

→ create it
```

---

# PART 25 — Create vs Update

A controller shouldn't blindly call:

```text
Create Deployment
```

every time.

Because on second reconciliation:

```text
Deployment already exists
```

So we'd get:

```text
AlreadyExists
```

Instead:

```go
var deployment appsv1.Deployment

err := r.Get(
    ctx,
    types.NamespacedName{
        Name:      webApp.Name,
        Namespace: webApp.Namespace,
    },
    &deployment,
)
```

Then:

```text
Does it exist?
     │
 ┌───┴────┐
 │        │
No       Yes
 │        │
 ▼        ▼
Create   Compare
          │
          ▼
       Update if needed
```

This is the essence of reconciliation.

---

# PART 26 — The interviewer word: idempotency

**Q: What does idempotent reconciliation mean?**

Excellent answer:

> An idempotent reconciler can be invoked multiple times and still converge to the same desired state without producing incorrect duplicate side effects.

Example:

```text
Reconcile #1
Deployment missing
→ create it

Reconcile #2
Deployment exists
→ verify it

Reconcile #3
Deployment exists
→ no change

Reconcile #100
Deployment exists
→ no change
```

That's exactly what we want.

---

# PART 27 — Don't use "create once"

This is a common beginner mistake.

Bad mental model:

```go
if firstTime {
    createDeployment()
}
```

Why bad?

Because Kubernetes is continuously changing.

Deployment might be:

```text
deleted manually
modified manually
scaled manually
recreated
rolled back
```

Your controller should continuously ask:

```text
"What should exist RIGHT NOW?"
```

not:

```text
"Did I already do this?"
```

---

# PART 28 — OwnerReferences

Now we create:

```text
WebApp
   ↓ owns
Deployment
```

Set the owner:

```go
err := controllerutil.SetControllerReference(
    &webApp,
    deployment,
    r.Scheme(),
)
```

Then:

```text
WebApp
  │
  │ ownerReference
  ▼
Deployment
```

Why?

Because Kubernetes can understand that the Deployment belongs to the WebApp.

This is useful for:

```text
garbage collection
ownership
event relationships
controller watches
```

Interview question:

**Q: Why use ownerReferences?**

Answer:

> To establish ownership between Kubernetes objects so lifecycle management, garbage collection, and controller relationships can be handled consistently by Kubernetes.

---

# PART 29 — `Owns()`

Now configure:

```go
func (r *WebAppReconciler) SetupWithManager(
    mgr ctrl.Manager,
) error {

    return ctrl.NewControllerManagedBy(mgr).
        For(&appsv1.WebApp{}).
        Owns(&appsv1.Deployment{}).
        Complete(r)
}
```

Read it as English:

```text
For(WebApp)
    =
watch WebApps

Owns(Deployment)
    =
watch Deployments owned by WebApps
```

So:

```text
WebApp changes
    ↓
Reconcile

Owned Deployment changes
    ↓
Reconcile
```

This is a huge controller-runtime interview topic.

---

# PART 30 — Why watch the Deployment?

Imagine:

```text
WebApp says:
replicas = 3
```

but someone manually does:

```bash
kubectl scale deployment hello --replicas=1
```

Now:

```text
Desired = 3
Actual = 1
```

The controller should notice and restore:

```text
1 → 3
```

That's reconciliation.

---

# PART 31 — Service

Now create a Service too:

```text
WebApp
 │
 ├── Deployment
 │      └── Pods
 │
 └── Service
```

Service:

```go
service := &corev1.Service{
    ObjectMeta: metav1.ObjectMeta{
        Name:      webApp.Name,
        Namespace: webApp.Namespace,
    },
    Spec: corev1.ServiceSpec{
        Selector: map[string]string{
            "app": webApp.Name,
        },
        Ports: []corev1.ServicePort{
            {
                Port: 80,
                TargetPort: intstr.FromInt(80),
            },
        },
    },
}
```

Set owner:

```go
controllerutil.SetControllerReference(
    &webApp,
    service,
    r.Scheme(),
)
```

Then:

```text
WebApp
 ├── Deployment
 └── Service
```

---

# PART 32 — Our controller now looks like this

Conceptually:

```go
func (r *WebAppReconciler) Reconcile(...) {

    // 1. Read WebApp
    getWebApp()

    // 2. Ensure Deployment
    ensureDeployment()

    // 3. Ensure Service
    ensureService()

    // 4. Update status
    updateStatus()

    // 5. Done
}
```

That's an operator.

Not magical.

Just:

```text
watch
+
reconcile
+
Kubernetes API
```

---

# PART 33 — Status

Now imagine:

```text
WebApp:
spec.replicas = 3
```

Deployment:

```text
desired replicas = 3
ready replicas = 2
```

Your controller can update:

```yaml
status:
  replicas: 3
  readyReplicas: 2
```

This gives users visibility:

```bash
kubectl get webapp hello -o yaml
```

They can distinguish:

```text
"Controller hasn't done anything"
```

from:

```text
"Controller created everything but Pods aren't ready."
```

---

# PART 34 — Why Status is dangerous to update incorrectly

This is a very common interview topic.

Suppose your controller does:

```go
r.Update(ctx, &webApp)
```

just to update status.

That modifies the main object and can trigger additional reconciliation.

Instead, Kubernetes APIs generally support the status subresource, and controller-runtime exposes:

```go
r.Status().Update(ctx, &webApp)
```

Conceptually:

```text
spec update
    =
user desired state

status update
    =
controller observed state
```

Keeping them separate reduces accidental spec modification and preserves API semantics.

---

# PART 35 — Conditions

A production controller shouldn't just say:

```yaml
status:
  ready: true
```

Often you want conditions such as:

```yaml
status:
  conditions:
    - type: Available
      status: "True"

    - type: Progressing
      status: "False"

    - type: Degraded
      status: "False"
```

Think of conditions as:

> "Little health statements about different aspects of the object."

This becomes extremely useful in:

```bash
kubectl describe
```

and automation.

---

# PART 36 — Finalizers

Now imagine our WebApp controller creates an **external resource**:

```text
WebApp
   ↓
Cloud Load Balancer
```

User deletes:

```bash
kubectl delete webapp hello
```

What if the CR disappears before the controller cleans up the load balancer?

You could leak infrastructure.

Finalizer solves this.

Conceptually:

```text
Delete WebApp
      ↓
deletionTimestamp set
      ↓
FINALIZER still present
      ↓
Controller sees deletion
      ↓
cleanup external resource
      ↓
remove finalizer
      ↓
object finally disappears
```

Interview answer:

> A finalizer is a deletion guard. Kubernetes marks the object for deletion but keeps it around until the controller removes the finalizer after performing required cleanup.

---

# PART 37 — Finalizer code

Conceptually:

```go
if webApp.DeletionTimestamp.IsZero() {

    if !controllerutil.ContainsFinalizer(
        &webApp,
        webAppFinalizer,
    ) {

        controllerutil.AddFinalizer(
            &webApp,
            webAppFinalizer,
        )

        return ctrl.Result{}, r.Update(
            ctx,
            &webApp,
        )
    }

} else {

    cleanupExternalResource()

    controllerutil.RemoveFinalizer(
        &webApp,
        webAppFinalizer,
    )

    return ctrl.Result{}, r.Update(
        ctx,
        &webApp,
    )
}
```

---

# PART 38 — RBAC

Now you have a real security issue.

Your controller needs permission to:

```text
get WebApps
list WebApps
watch WebApps

get Deployments
create Deployments
update Deployments
patch Deployments
delete Deployments

get Services
create Services
update Services
...
```

Kubernetes won't automatically allow that.

This produces:

```text
Forbidden
```

errors if RBAC isn't configured.

Kubebuilder uses RBAC markers to generate permissions.

For example:

```go
//+kubebuilder:rbac:groups=apps.example.com,resources=webapps,verbs=get;list;watch;create;update;patch;delete
```

and:

```go
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch
```

Then:

```bash
make manifests
```

updates the RBAC manifests.

---

# PART 39 — Interview question: Why not cluster-admin?

**Bad approach:**

```text
Give operator cluster-admin
```

Why?

Huge blast radius.

Better:

```text
Least privilege
```

Only grant:

```text
resources needed
+
verbs needed
+
namespaces needed
```

Interview answer:

> Controllers should use least-privilege RBAC because a compromised controller pod otherwise has unnecessarily broad access to cluster resources.

---

# PART 40 — Errors

Imagine:

```text
Create Deployment
      ↓
API server temporarily unavailable
```

Don't panic.

Return an error:

```go
return ctrl.Result{}, err
```

controller-runtime will retry according to its workqueue/rate-limiting behavior.

Another case:

```text
Everything fine,
but Pods need time to become ready.
```

You may requeue:

```go
return ctrl.Result{
    RequeueAfter: 10 * time.Second,
}, nil
```

But don't use:

```go
time.Sleep(10 * time.Second)
```

inside reconciliation.

Why?

Because you've just tied up a worker.

---

# PART 41 — Interview question: event vs state

This is one of my favorite controller interview questions.

**Q: If a Deployment changed five times before your controller processed the queue, do you need to process all five events?**

Answer:

> No. The queue is a trigger to reconcile the current state, not a reliable event log. When reconciliation runs, the controller reads the latest state and converges it toward the desired state.

That is why Kubernetes controllers are **level-triggered** rather than being designed as simple event processors.

---

# PART 42 — What happens when the controller crashes?

Excellent interview question.

Suppose:

```text
WebApp exists

controller crashes
```

Does the WebApp disappear?

No.

API server still contains:

```text
WebApp
```

When controller comes back:

```text
controller starts
    ↓
initial cache/list
    ↓
sees WebApp
    ↓
reconcile
    ↓
repair world
```

This is another reason reconciliation is powerful.

The controller doesn't have to remember every historical event.

---

# PART 43 — What if Deployment gets deleted?

Suppose:

```text
WebApp
 └── Deployment
```

Someone executes:

```bash
kubectl delete deployment hello
```

Desired state:

```text
Deployment exists
```

Actual state:

```text
Deployment missing
```

Controller:

```text
detect difference
      ↓
create Deployment
```

That's your operator healing the cluster.

---

# PART 44 — What if user changes the image?

User:

```yaml
spec:
  image: nginx:1.28
```

Controller sees:

```text
desired image = nginx:1.28
actual image = nginx:1.27
```

Then:

```text
update Deployment
```

Deployment performs its rolling update.

So the chain becomes:

```text
kubectl edit WebApp
       ↓
API server
       ↓
watch event
       ↓
Reconcile
       ↓
Deployment update
       ↓
Deployment controller
       ↓
ReplicaSet
       ↓
new Pods
```

Now you've just seen **multiple controllers cooperating**.

---

# PART 45 — This is extremely important

Your controller does **not** directly create Pods.

Ideally:

```text
Your WebApp Controller
        ↓
Deployment
        ↓
Deployment Controller
        ↓
ReplicaSet
        ↓
Pods
```

Why?

Because Kubernetes already provides machinery for:

```text
rolling updates
replica management
pod replacement
```

Don't reinvent Kubernetes.

Your operator should operate at the right abstraction level.

---

# PART 46 — The controller hierarchy

Memorize this:

```text
WebApp Controller
      │
      ▼
Deployment
      │
      ▼
Deployment Controller
      │
      ▼
ReplicaSet
      │
      ▼
Pod
      │
      ▼
Scheduler
      │
      ▼
Node
```

Kubernetes is a collection of controllers interacting through APIs.

---

# PART 47 — Core interview questions: CRDs

You should be able to answer these immediately.

### Q: What is a CRD?

> A CRD defines a new Kubernetes API resource type. Once installed, instances of that type are stored and served through the Kubernetes API server.

### Q: CRD vs CR?

> CRD defines the schema/type; CR is an instance of that type.

### Q: Does a CRD implement business logic?

> No. It defines and exposes the API. A controller/operator implements the behavior.

### Q: Why use a CRD?

> When I want a declarative Kubernetes-native API that can be manipulated with kubectl, watched by controllers, and used to represent configuration or desired state.

### Q: CRD vs ConfigMap?

> ConfigMap is general-purpose key/value configuration. A CRD gives me a typed Kubernetes API resource with schema, `.spec`, `.status`, watch semantics, RBAC, and controller integration.

Kubernetes' current documentation explicitly calls out custom resources when you want Kubernetes API/CLI support, controllers watching them, and declarative API conventions. [Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/?utm_source=chatgpt.com)

---

# PART 48 — Core interview questions: Controllers

### Q: What is reconciliation?

> Reconciliation is the process of observing the current state, comparing it with desired state, and taking actions to make actual state converge toward desired state.

### Q: Is Reconcile called only once?

> No. It may be invoked many times due to resource events, dependent-resource changes, retries, requeues, or startup/cache synchronization.

### Q: Should Reconcile be idempotent?

> Yes.

### Q: Why?

> Because reconciliation is repeatedly invoked and must safely converge without producing duplicate or incorrect side effects.

### Q: What is the source of truth?

> Current state from the Kubernetes API/cache, combined with the resource's desired state—not the event itself.

---

# PART 49 — Core interview questions: controller-runtime

### Q: What is Manager?

Think:

```text
Manager
 ├── Client
 ├── Cache
 ├── Controllers
 ├── Scheme
 └── Leader election/configuration
```

It's the central runtime that wires your controllers and supporting components together.

### Q: What does `For()` mean?

> Define the primary resource being reconciled/watched.

Example:

```go
For(&WebApp{})
```

### Q: What does `Owns()` mean?

> Watch resources owned by the primary resource and enqueue the related reconciliation.

Example:

```go
Owns(&Deployment{})
```

### Q: Why use owner references?

> To associate child resources with their controller/owner and enable lifecycle and garbage-collection semantics.

---

# PART 50 — Client-Go interview questions

### Q: What is client-go?

> The Go client library and supporting machinery for interacting with Kubernetes APIs.

### Q: What are informers?

> Components that list/watch Kubernetes resources and maintain a local cache, emitting change events to consumers.

### Q: What is a lister?

Think:

```text
Informer → local cache
Lister   → efficient reads from cache
```

### Q: Why use a cache?

> To avoid repeatedly hitting the API server for reads and to make controllers scalable and efficient.

### Q: What's a workqueue?

> A queue used by controllers to serialize or parallelize reconciliation work, support retries, and apply rate limiting.

The official client-go controller example demonstrates the informer/indexer plus rate-limited workqueue pattern. [GitHub](https://github.com/kubernetes/client-go/blob/master/examples/workqueue/main.go?utm_source=chatgpt.com)

---

# PART 51 — CR lifecycle

A CR generally travels through:

```text
kubectl apply
      ↓
API server validation
      ↓
stored in etcd
      ↓
watch event
      ↓
controller
      ↓
Reconcile
      ↓
child resources
      ↓
status update
```

On deletion:

```text
kubectl delete
      ↓
deletionTimestamp
      ↓
finalizer?
   /      \
 yes       no
  │         │
cleanup    delete
  │
remove finalizer
  │
delete
```

---

# PART 52 — Important concept: etcd

The API server isn't the long-term database itself.

Very simplified:

```text
kubectl
   ↓
API Server
   ↓
etcd
```

The API server handles:

```text
authentication
authorization
validation
admission
API semantics
watch
```

etcd stores the control-plane state.

Interview question:

**Q: Where are CRs stored?**

Answer:

> Custom resources are stored by the Kubernetes API server using the cluster's backing storage, typically etcd.

---

# PART 53 — Why controllers don't directly talk to etcd

Your controller should do:

```text
controller
   ↓
Kubernetes API
```

not:

```text
controller
   ↓
etcd
```

Because the Kubernetes API server provides the consistency, authorization, validation, versioning, watch and API semantics your controller needs.

---

# PART 54 — Optimistic concurrency

Suppose:

```text
Controller A reads object version 10

Controller B updates object
version becomes 11

Controller A tries update using old object
```

You can get a conflict:

```text
409 Conflict
```

This is normal.

Controllers must be designed to handle concurrency.

Common pattern:

```text
Get latest object
    ↓
Modify
    ↓
Update
    ↓
Conflict?
    ↓
reconcile again
```

Interviewers may ask:

> "What happens if two actors update the same object?"

Talk about:

```text
resourceVersion
optimistic concurrency
conflict/retry
```

---

# PART 55 — Don't blindly Update entire resources

Controllers can accidentally fight with:

```text
kubectl
HPA
other controllers
admission controllers
```

Suppose you replace the entire object based on stale state.

You could overwrite changes made by another actor.

Safer patterns include carefully updating only what your controller owns and handling conflicts appropriately.

This becomes especially important with:

```text
Status
metadata
spec fields managed by your controller
```

---

# PART 56 — What about multiple replicas of the operator?

Suppose:

```text
operator-1
operator-2
operator-3
```

all running.

Do you want all three reconciling the same resources simultaneously?

Usually controller-runtime uses leader election when configured so only one active leader performs the controller-management role at a time while others remain available for failover.

Conceptually:

```text
operator-1 ──┐
operator-2 ──┼── leader election
operator-3 ──┘
                 │
                 ▼
              leader
                 │
                 ▼
             reconcile
```

If leader dies:

```text
another replica
      ↓
becomes leader
      ↓
continues reconciliation
```

This is an important HA interview topic.

---

# PART 57 — Tests

A serious operator needs tests.

At minimum:

```text
unit tests
reconcile tests
envtest
integration tests
```

You want to test cases such as:

```text
CR doesn't exist
CR exists
Deployment missing
Deployment exists
Deployment incorrect
Service missing
Image changed
Replica count changed
Deletion
Finalizer cleanup
API error
RBAC failure
status update
```

Interview question:

**Q: How do you test a Kubernetes controller without a real production cluster?**

Answer:

> Use unit tests for pure reconciliation logic and controller-runtime's test tooling/envtest for exercising API interactions against a test control plane/API environment.

---

# PART 58 — Debugging a broken Operator

Suppose:

```bash
kubectl apply -f webapp.yaml
```

but nothing happens.

Don't randomly restart things.

Walk the chain:

```text
1. Does CRD exist?
2. Does CR exist?
3. Is controller running?
4. Is controller watching that type?
5. Did Reconcile run?
6. Are there RBAC errors?
7. Did controller create Deployment?
8. Is Deployment correct?
9. Are Pods scheduled?
10. Are Pods healthy?
11. Is status updated?
```

Commands:

```bash
kubectl get crd
kubectl get webapps
kubectl describe webapp hello
kubectl get deployment
kubectl describe deployment hello
kubectl get pods
kubectl logs ...
```

And inspect controller logs.

---

# PART 59 — Debugging using events

This is a very useful habit:

```bash
kubectl describe webapp hello
```

and:

```bash
kubectl get events --sort-by=.lastTimestamp
```

You want to follow:

```text
CR event
  ↓
controller log
  ↓
Deployment event
  ↓
ReplicaSet event
  ↓
Pod event
  ↓
Node/scheduler event
```

This is essentially distributed-systems debugging across control loops.

---

# PART 60 — Now understand all the add-ons using the same brain

Let's connect this to the second half of your original question.

## CoreDNS

Think:

```text
"I need to find another Service."
```

CoreDNS provides cluster DNS/service discovery.

```text
Pod
 ↓
DNS query
 ↓
CoreDNS
 ↓
Service IP
```

So:

> **CoreDNS = phonebook**

---

# PART 61 — Metrics Server

Think:

```text
"How much CPU/memory is being used?"
```

Metrics Server provides resource metrics used by Kubernetes components such as `kubectl top` and HPA.

```text
kubelet
   ↓
Metrics
   ↓
Metrics Server
   ↓
Metrics API
```

So:

> **Metrics Server = basic CPU/memory meter**

It is not your full observability stack.

---

# PART 62 — ExternalDNS

Think:

```text
Kubernetes says:

"I have this external address."

DNS says:

"I don't have the record."

ExternalDNS:
"Let me reconcile that."
```

```text
Ingress / Service / Gateway
          ↓
       ExternalDNS
          ↓
      DNS provider
```

This is basically another controller loop.

---

# PART 63 — Cluster Autoscaler

Think:

```text
Pod needs a seat
     ↓
No seat available
     ↓
Need more machines
```

Cluster Autoscaler typically reasons about node groups.

```text
Pending Pods
     ↓
Cluster Autoscaler
     ↓
increase node-group capacity
     ↓
new node
     ↓
scheduler
     ↓
Pod runs
```

---

# PART 64 — Karpenter

Think more dynamically:

```text
Pending Pod says:

"I need:
8 CPU
32 GiB
amd64
zone A
GPU? no
certain labels
certain taints/tolerations
"
```

Karpenter reasons about what capacity would satisfy scheduling requirements and can provision nodes accordingly.

So the simplified interview distinction is:

```text
Cluster Autoscaler
    =
grow/shrink existing node groups

Karpenter
    =
dynamically provision suitable capacity
```

Do not memorize vendor-specific implementation details yet.

Master the control-loop model first.

---

# PART 65 — HPA vs Autoscaler

Another extremely common interview question.

Imagine traffic goes:

```text
10 requests/sec
        ↓
1000 requests/sec
```

HPA might decide:

```text
Pods:
3 → 20
```

Now perhaps:

```text
20 Pods
↓
not enough nodes
↓
some Pods Pending
```

Then node capacity must grow.

Therefore:

```text
HPA
=
Pod scaling

Cluster Autoscaler / Karpenter
=
Node capacity scaling
```

This is a beautiful example of **controllers composing together**.

---

# PART 66 — The complete system picture

This is the architecture I want you eventually to be able to draw in an interview:

```text
                       USER
                        │
                        ▼
                 Kubernetes API
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
             CRDs               Built-ins
              │                   │
              ▼                   ▼
          Custom CRs          Deployments
              │                   │
              ▼                   ▼
         Custom Controller    Built-in Controllers
              │                   │
              └─────────┬─────────┘
                        ▼
                  Desired state
                        │
                        ▼
                  Actual cluster
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
       Scheduler      kubelet       Add-ons
                                       │
              ┌────────────────────────┼───────────────┐
              ▼                        ▼               ▼
            CoreDNS              ExternalDNS       Autoscaling
              │                        │               │
        Service DNS              DNS records      Node/Pod capacity
```

---

# PART 67 — The interview "deep dive" questions

Once you've mastered the implementation above, expect questions like these.

### Architecture

**"Explain Kubernetes control plane."**

Be ready for:

```text
API Server
Scheduler
Controller Manager
etcd
kubelet
kube-proxy/CNI
```

---

### Controller

**"Walk me through Reconcile."**

You should answer:

```text
Get primary resource
↓
Handle NotFound
↓
Check deletion/finalizer
↓
Read desired state
↓
Read dependent resources
↓
Compare actual vs desired
↓
Create/update/delete
↓
Update status/conditions
↓
Return
```

---

### Idempotency

**"Why does Kubernetes need idempotent controllers?"**

Because reconciliation happens repeatedly and events may be duplicated/coalesced/out of order.

---

### Event-driven vs reconciliation

**"Would you process every event?"**

No.

You use events to trigger reconciliation, then read current state and converge.

---

### CRD

**"When wouldn't you use a CRD?"**

If you don't need a Kubernetes-native declarative API/watch/controller model, or if the object is really just application configuration better represented by ConfigMap/Secret or external storage.

Kubernetes' current documentation gives similar guidance around declarative APIs, object size, access patterns, and whether controller automation is required. [Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/?utm_source=chatgpt.com)

---

### Finalizer

**"How do you safely clean external resources?"**

Use finalizers.

---

### OwnerReference

**"How do you connect child resources to the parent?"**

OwnerReferences plus controller-runtime ownership watches where appropriate.

---

### RBAC

**"What happens when your controller tries to create a Deployment without permission?"**

The API server returns `Forbidden`; inspect ServiceAccount/RBAC/Role/ClusterRole and required verbs/resources.

---

### Performance

**"How do you prevent API-server overload?"**

Talk about:

```text
informers/cache
listers
event-driven reconciliation
workqueues
rate limiting
minimal API calls
idempotent updates
predicates/filtering
```

---

### Scaling

**"What if there are 1 million resources?"**

Think:

```text
cache
queue
worker concurrency
rate limiting
sharding/partitioning
namespace scope
predicates
avoiding unnecessary writes
status churn
```

You don't just say:

> "Add more goroutines."

---

### HA

**"How do you run multiple replicas?"**

Talk about:

```text
leader election
stateless/reconstructible controller state
shared Kubernetes API state
failover
```

---

# PART 68 — The most important mistakes to avoid

### Mistake 1

Thinking:

```text
Controller = webhook
```

No.

Controller is fundamentally:

```text
control loop
```

---

### Mistake 2

Thinking:

```text
CRD = Operator
```

No.

```text
CRD
=
API definition

Operator
=
CRD + controller/business logic
```

---

### Mistake 3

Thinking:

```text
Event = truth
```

No.

```text
Event = trigger

Current state = truth
```

---

### Mistake 4

Thinking:

```text
Reconcile = one-time function
```

No.

```text
Reconcile = repeatedly converge
```

---

### Mistake 5

Creating Pods directly when a Deployment expresses what you need.

Use Kubernetes abstractions whenever possible.

---

### Mistake 6

Using `time.Sleep()` inside reconciliation.

Return/requeue instead.

---

### Mistake 7

Giving your operator `cluster-admin`.

Use least privilege.

---

### Mistake 8

Forgetting finalizers when external resources exist.

That can cause resource leaks.

---

# PART 69 — Your hands-on progression

Don't try to memorize this whole thing.

Actually build these versions:

```text
VERSION 1

WebApp CRD
+
Controller prints:
"Reconciling WebApp"
```

Then:

```text
VERSION 2

WebApp
 ↓
Deployment
```

Then:

```text
VERSION 3

WebApp
 ├── Deployment
 └── Service
```

Then:

```text
VERSION 4

OwnerReferences
```

Then:

```text
VERSION 5

Status
```

Then:

```text
VERSION 6

Conditions
```

Then:

```text
VERSION 7

Finalizer
```

Then:

```text
VERSION 8

RBAC
```

Then:

```text
VERSION 9

Tests
```

Then:

```text
VERSION 10

Run operator inside Kubernetes
```

Then:

```text
VERSION 11

Two operator replicas + leader election
```

Then:

```text
VERSION 12

Metrics + production logging
```

---

# PART 70 — The mental model I want you to carry into interviews

Suppose the interviewer says:

> "Design an operator that manages Redis."

Don't immediately start talking about Go structs.

Start here:

```text
What is the desired state?
```

Maybe:

```yaml
kind: RedisCluster
spec:
  replicas: 3
  version: "7"
  persistence: true
```

Then:

```text
What is actual state?
```

Maybe:

```text
StatefulSet exists
3 desired
2 ready
PVCs exist
Service exists
```

Then:

```text
What controller should reconcile?
```

Then:

```text
What child resources does it own?
```

Then:

```text
What happens on:
create?
update?
delete?
failure?
partial failure?
controller restart?
manual tampering?
external dependency outage?
```

Then:

```text
What RBAC does it need?
```

Then:

```text
How do I observe and debug it?
```

Then:

```text
How does it behave with 10,000 CRs?
```

That is how a strong Kubernetes engineer thinks.

---

# PART 71 — One interview answer worth memorizing

When they ask:

> **"Explain how you would build a Kubernetes operator."**

A strong concise answer is:

> "I'd first define the desired-state API as a CRD with a versioned schema and validation. I'd define typed Go structs for Spec and Status, then implement a controller using controller-runtime. The controller would watch the CR and owned resources, fetch the latest state from the cache/API, compare desired and actual state, and idempotently create/update/delete child resources. I'd use owner references for managed Kubernetes resources, finalizers for external-resource cleanup, status conditions for observability, least-privilege RBAC, rate-limited retries, and metrics/logging. I'd test reconciliation with unit tests and envtest, then package and deploy the controller with HA/leader election where required."

That single answer covers a **lot** of interview territory.

---

## What we should do next

The best way to continue is **not another wall of theory**.

Next, we should take the `WebApp` example and literally build the first working version together, file by file:

```text
webapp_types.go
       ↓
CRD generation
       ↓
webapp_controller.go
       ↓
Reconcile()
       ↓
Deployment creation
       ↓
Service creation
       ↓
kubectl apply
       ↓
watch controller logs
       ↓
break it deliberately
       ↓
debug it
```

Then we'll add **Status → Conditions → OwnerReferences → Finalizers → RBAC → tests**, with each addition followed immediately by the interview questions that correspond to it.