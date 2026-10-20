```
How istio works
```

Think of **Istio** as a team of smart traffic cops for all the little services inside your Kubernetes cluster.

Imagine you have three services:

**App → Orders → Payments**

Normally, App talks directly to Orders, and Orders talks directly to Payments.

With Istio, there’s an extra helper beside each service:

**App + helper → Orders + helper → Payments + helper**

That helper is usually an **Envoy proxy**. Istio calls this setup a **service mesh**.

The important trick is: **your application usually doesn’t need to know Istio exists.** Istio intercepts network traffic and handles things like:

- **Routing:** “Send 90% of users to version 1, 10% to version 2.”
- **Security:** “Encrypt traffic between services.”
- **Retries:** “Payment service failed? Try again.”
- **Timeouts:** “Don’t wait forever.”
- **Observability:** “How long did this request take? Which service failed?”
- **Access control:** “Service A may call Service B, but Service C may not.”

There are basically two halves:

```text
                 Istio Control Plane
                       istiod
                         |
            tells proxies what rules to use
                         |
        -----------------------------------
        |                |                |
     Envoy            Envoy            Envoy
        |                |                |
      App             Orders          Payments
```

**`istiod` is the brain. Envoy proxies are the traffic cops.**

For example, suppose you deploy:

```text
payments-v1
payments-v2
```

You can tell Istio:

```text
95% → payments-v1
 5% → payments-v2
```

without changing the Orders application itself.

That makes things like **canary deployments** much easier.

One subtle point: traditionally Istio used a **sidecar Envoy proxy next to every pod**. Modern Istio can also use **Ambient Mesh**, where you don't necessarily need a sidecar inside every application pod.

So the ELI7 version is:

> **Kubernetes runs your services. Istio controls and watches the traffic traveling between those services.**

And the three words worth remembering are:

**Envoy = traffic cop**  
**istiod = brain**  
**Service mesh = the whole traffic-control system**

Absolutely. The easiest way to understand Istio properly is to **build a tiny system and follow one HTTP request through it**.

I’m going to teach the traditional **sidecar model** first because you can literally see what Istio is doing. Istio currently recommends **ambient mode for new deployments**, but sidecars are still fully supported and are much easier for learning the architecture. [Istio](https://istio.io/latest/docs/overview/quickstart/?utm_source=chatgpt.com)

## 1. Our example application

Imagine an ecommerce application:

```text
Frontend
   |
   | GET /orders
   v
Orders Service
```

Later we'll deploy two versions:

```text
                  +--> orders-v1
Frontend --> orders
                  +--> orders-v2
```

And tell Istio:

```text
90% traffic --> v1
10% traffic --> v2
```

The important thing is that **Frontend continues calling exactly the same URL**:

```text
http://orders
```

Frontend doesn't know v1 or v2 exist.

Istio decides where the request actually goes.

---

# 2. First understand plain Kubernetes

Before Istio, suppose we have two pods:

```text
frontend pod
orders pod
```

And a Kubernetes `Service`:

```text
orders
```

The frontend might make this call:

```python
requests.get("http://orders")
```

Kubernetes DNS resolves:

```text
orders
```

to the Kubernetes Service.

So normally:

```text
Frontend Pod
     |
     | HTTP GET
     v
Kubernetes Service: orders
     |
     v
Orders Pod
```

Nothing unusual yet.

---

# 3. Now install Istio

Assuming you already have Kubernetes running.

Download Istio:

```bash
curl -L https://istio.io/downloadIstio | sh -
```

Enter the downloaded directory:

```bash
cd istio-*
```

Add `istioctl`:

```bash
export PATH=$PWD/bin:$PATH
```

For a learning environment, install the demo profile.

```bash
istioctl install --set profile=demo -y
```

Then verify:

```bash
kubectl get pods -n istio-system
```

You'll see things such as:

```text
istiod-xxxxxxxxxx     1/1     Running
```

`istiod` is the **control plane** — essentially the component that calculates and distributes mesh configuration.

The official getting-started flow similarly installs Istio and then enables automatic proxy injection on a namespace. [Istio](https://istio.io/latest/docs/setup/getting-started/)

---

# 4. Tell Istio to manage our namespace

Let's create a namespace:

```bash
kubectl create namespace shop
```

Now this line is extremely important:

```bash
kubectl label namespace shop istio-injection=enabled
```

It means:

> Whenever Kubernetes creates a pod in `shop`, automatically add an Istio Envoy proxy beside the application container.

Istio's official sidecar guide uses the same `istio-injection=enabled` namespace labeling mechanism. [Istio](https://istio.io/latest/docs/setup/getting-started/)

---

# 5. Let's create Orders v1

We'll use nginx so you don't even need to write an application.

Create:

```text
orders-v1.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-v1
  namespace: shop
spec:
  replicas: 1

  selector:
    matchLabels:
      app: orders
      version: v1

  template:
    metadata:
      labels:
        app: orders
        version: v1
    spec:
      containers:
        - name: orders
          image: nginx:alpine
          ports:
            - containerPort: 80
```

Notice these labels:

```yaml
app: orders
version: v1
```

That `version: v1` will become very important.

Apply it:

```bash
kubectl apply -f orders-v1.yaml
```

Now check:

```bash
kubectl get pods -n shop
```

You should notice something interesting:

```text
NAME                         READY
orders-v1-xxxxxxxxxx         2/2
```

Why **2/2**?

We only declared **one container**!

Because Istio changed your pod roughly into:

```text
Pod
├── orders container
└── istio-proxy container
```

The second container is the Envoy proxy.

That is the famous:

# Istio Sidecar

Conceptually:

```text
+-------------------------------+
| Orders Pod                    |
|                               |
|   +---------+   +---------+   |
|   | Orders  |   |  Envoy  |   |
|   |   App   |   |  Proxy  |   |
|   +---------+   +---------+   |
|                               |
+-------------------------------+
```

You can confirm:

```bash
kubectl get pod -n shop
```

Then:

```bash
kubectl describe pod <pod-name> -n shop
```

You'll see an application container and an Istio proxy container.

---

# 6. Create the Kubernetes Service

Now create:

```text
orders-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: orders
  namespace: shop
spec:

  selector:
    app: orders

  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f orders-service.yaml
```

Notice something subtle.

The Service selects:

```yaml
app: orders
```

It does **not** say:

```yaml
version: v1
```

That's deliberate.

Soon both:

```text
orders-v1
orders-v2
```

will belong to the same Kubernetes service:

```text
orders
```

---

# 7. Create a client/frontend

Let's create a simple pod from which we can call Orders.

```text
frontend.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: shop
spec:
  replicas: 1

  selector:
    matchLabels:
      app: frontend

  template:
    metadata:
      labels:
        app: frontend
    spec:

      containers:
        - name: frontend
          image: curlimages/curl
          command:
            - sleep
            - "3600"
```

Apply:

```bash
kubectl apply -f frontend.yaml
```

Check:

```bash
kubectl get pods -n shop
```

You might see:

```text
frontend-6f8c...      2/2
orders-v1-7d5...      2/2
```

Again:

```text
2/2
```

because each pod has:

```text
Application
+
Envoy
```

---

# 8. Send a request

Enter the frontend application container:

```bash
kubectl exec -it deployment/frontend \
  -n shop \
  -c frontend \
  -- sh
```

Then:

```bash
curl http://orders
```

You'll get the nginx welcome page.

But here's the important part.

You think this happened:

```text
frontend
   |
   v
orders
```

It actually looks more like:

```text
Frontend Pod
+------------------+
|                  |
| frontend         |
|     |            |
|     v            |
| Envoy -----------+------+
+------------------+      |
                           |
                           v
                     Kubernetes network
                           |
                           v
Orders Pod                 |
+--------------------------+---+
|                          v   |
| Envoy ----------------> App  |
|                              |
+------------------------------+
```

In simplified form:

```text
Frontend App
     |
     v
Frontend Envoy
     |
     v
Orders Envoy
     |
     v
Orders App
```

**Istio controls the Envoys.**

Your applications can continue doing ordinary HTTP.

---

# 9. Where does `istiod` fit?

Now add the control plane.

```text
                   +----------------+
                   |     istiod     |
                   | Control Plane  |
                   +--------+-------+
                            |
                 configuration
                            |
              +-------------+-------------+
              |                           |
              v                           v

       Frontend Pod                  Orders Pod
    +----------------+            +----------------+
    | frontend       |            | orders         |
    |      |         |            |      ^         |
    |      v         |            |      |         |
    |    Envoy ------+----------->|    Envoy       |
    +----------------+            +----------------+
```

One critical distinction:

**Application traffic does not normally travel through `istiod`.**

Don't imagine:

```text
Frontend → istiod → Orders
```

That's wrong.

Instead:

```text
        configuration
istiod --------------------> Envoy

Frontend → Envoy → network → Envoy → Orders
```

So:

```text
istiod = control plane
Envoy  = data plane
```

---

# 10. Now let's make it interesting: Orders v2

Create another deployment.

```text
orders-v2.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-v2
  namespace: shop
spec:
  replicas: 1

  selector:
    matchLabels:
      app: orders
      version: v2

  template:
    metadata:
      labels:
        app: orders
        version: v2
    spec:

      containers:
        - name: orders
          image: nginx:alpine
          ports:
            - containerPort: 80
```

Notice:

```yaml
app: orders
version: v2
```

Apply:

```bash
kubectl apply -f orders-v2.yaml
```

Now:

```bash
kubectl get pods -n shop --show-labels
```

Conceptually:

```text
orders-v1   app=orders,version=v1
orders-v2   app=orders,version=v2
```

And both are behind:

```text
Service: orders
```

So now:

```text
                 Kubernetes Service
                       orders
                     /        \
                    /          \
                   v            v

              orders-v1     orders-v2
```

---

# 11. Here Istio starts becoming powerful

Suppose we deployed v2 yesterday.

We don't want:

```text
50% → v1
50% → v2
```

Maybe v2 has a bug.

Instead we want:

```text
90% → v1
10% → v2
```

This is a **canary deployment**.

Istio gives us two important resources:

```text
DestinationRule
VirtualService
```

These are Kubernetes custom resources used by Istio's traffic management APIs. [Istio](https://istio.io/latest/docs/concepts/traffic-management/)

The beginner-friendly interpretation is:

```text
DestinationRule
=
"Tell Istio what versions exist."


VirtualService
=
"Tell Istio how traffic should be routed."
```

---

# 12. DestinationRule

Create:

```text
orders-destination-rule.yaml
```

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule

metadata:
  name: orders
  namespace: shop

spec:

  host: orders

  subsets:

    - name: v1
      labels:
        version: v1

    - name: v2
      labels:
        version: v2
```

Apply:

```bash
kubectl apply -f orders-destination-rule.yaml
```

Read this like English:

```yaml
host: orders
```

means:

> These rules belong to the `orders` service.

Then:

```yaml
- name: v1
  labels:
    version: v1
```

means:

> Whenever I say `v1`, I mean the Orders pods whose label is `version=v1`.

Similarly:

```yaml
- name: v2
  labels:
    version: v2
```

means:

> v2 means pods with `version=v2`.

This is precisely the purpose of subsets in Istio: a `DestinationRule` can group service instances using labels such as `version: v1` and `version: v2`. [Istio](https://istio.io/latest/docs/concepts/traffic-management/)

So mentally:

```text
DestinationRule

v1 --> version=v1 pods
v2 --> version=v2 pods
```

---

# 13. VirtualService

Now tell Istio:

```text
90% → v1
10% → v2
```

Create:

```text
orders-virtual-service.yaml
```

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService

metadata:
  name: orders
  namespace: shop

spec:

  hosts:
    - orders

  http:

    - route:

        - destination:
            host: orders
            subset: v1
          weight: 90

        - destination:
            host: orders
            subset: v2
          weight: 10
```

Apply:

```bash
kubectl apply -f orders-virtual-service.yaml
```

Now Istio knows:

```text
Request for:

http://orders

        |
        v

VirtualService
        |
        +---- 90% ----> subset v1
        |
        +---- 10% ----> subset v2
```

Weighted routing between service subsets is a first-class `VirtualService` capability in Istio. [Istio](https://istio.io/latest/docs/concepts/traffic-management/)

---

# 14. But where is that rule actually executed?

This is one of the most important things to understand.

The `VirtualService` itself **doesn't handle requests**.

You create:

```text
VirtualService YAML
```

Kubernetes stores it.

Then:

```text
istiod reads it
```

Then istiod tells Envoy:

```text
"For requests to orders:

90% => v1
10% => v2"
```

So:

```text
orders-virtual-service.yaml
          |
          v
   Kubernetes API
          |
          v
       istiod
          |
          | pushes config
          v
   Frontend Envoy
```

Then when a request happens:

```text
frontend
    |
    | GET http://orders
    v
Frontend Envoy
    |
    | sees routing configuration
    |
    +--- random selection ---> v1
    |
    +------------------------> v2
```

`istiod` is **not involved in each request**.

That's a major architectural idea.

---

# 15. Let's actually test the routing

Our nginx containers currently respond identically, which isn't useful.

Let's change them to return different responses.

A nicer deployment for v1 would be:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: orders-v1
  namespace: shop

spec:

  replicas: 1

  selector:
    matchLabels:
      app: orders
      version: v1

  template:

    metadata:
      labels:
        app: orders
        version: v1

    spec:

      containers:

        - name: orders
          image: hashicorp/http-echo:1.0

          args:
            - "-text=ORDERS VERSION 1"

          ports:
            - containerPort: 5678
```

And v2:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: orders-v2
  namespace: shop

spec:

  replicas: 1

  selector:
    matchLabels:
      app: orders
      version: v2

  template:

    metadata:
      labels:
        app: orders
        version: v2

    spec:

      containers:

        - name: orders
          image: hashicorp/http-echo:1.0

          args:
            - "-text=ORDERS VERSION 2"

          ports:
            - containerPort: 5678
```

Change the Service accordingly:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: orders
  namespace: shop

spec:

  selector:
    app: orders

  ports:
    - port: 80
      targetPort: 5678
```

Then make many requests:

```bash
for i in $(seq 1 20); do
  curl -s http://orders
done
```

You should see something roughly like:

```text
ORDERS VERSION 1
ORDERS VERSION 1
ORDERS VERSION 1
ORDERS VERSION 1
ORDERS VERSION 1
ORDERS VERSION 2
ORDERS VERSION 1
ORDERS VERSION 1
ORDERS VERSION 1
ORDERS VERSION 1
...
```

It won't necessarily be *exactly* 18 v1 and 2 v2 for only 20 requests.

Over many requests it should approach the configured:

```text
90 / 10
```

---

# 16. Why can't Kubernetes alone easily do this?

This distinction matters a lot.

Suppose Kubernetes has:

```text
9 v1 pods
1 v2 pod
```

You might roughly get:

```text
90% v1
10% v2
```

But now deployment scaling and traffic policy are tied together.

What if v2 needs five pods because it's computationally expensive, but you still only want 10% of users seeing it?

Istio separates:

```text
How many pods exist
```

from:

```text
How much traffic each version receives
```

So you could have:

```text
v1 replicas = 3
v2 replicas = 10

Traffic:
v1 = 90%
v2 = 10%
```

The traffic percentage doesn't need to correspond to the pod count. Istio explicitly highlights this separation as useful for canary rollouts. [Istio](https://istio.io/latest/docs/concepts/traffic-management/)

---

# 17. Now imagine deploying progressively

Start:

```yaml
v1: 100
v2: 0
```

Then:

```yaml
v1: 95
v2: 5
```

Watch metrics.

Then:

```yaml
v1: 90
v2: 10
```

Then:

```yaml
v1: 50
v2: 50
```

Finally:

```yaml
v1: 0
v2: 100
```

You never changed:

```python
requests.get("http://orders")
```

That's the power.

---

# 18. Istio can also route by HTTP headers

Suppose developers should test v2 first.

You can say:

```text
Header:
x-user-type: tester

        ↓

orders-v2
```

Everyone else:

```text
orders-v1
```

Example:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService

metadata:
  name: orders
  namespace: shop

spec:

  hosts:
    - orders

  http:

    - match:
        - headers:
            x-user-type:
              exact: tester

      route:
        - destination:
            host: orders
            subset: v2

    - route:
        - destination:
            host: orders
            subset: v1
```

Now:

```bash
curl http://orders
```

goes to:

```text
v1
```

But:

```bash
curl \
  -H "x-user-type: tester" \
  http://orders
```

goes to:

```text
v2
```

Istio `VirtualService` rules can match headers, URIs, ports, and other request properties. [Istio](https://istio.io/latest/docs/concepts/traffic-management/)

That means you can implement things like:

```text
Employees       → v2
Beta customers  → v2
Everyone else   → v1
```

without putting that routing logic inside your application.

---

# 19. Istio can also configure retries

Suppose Orders occasionally fails.

Normally your frontend might contain:

```python
try:
    call_orders()
except:
    retry()
```

With Istio you can configure retries at the mesh level.

Conceptually:

```yaml
http:
  - route:
      - destination:
          host: orders

    retries:
      attempts: 3
      perTryTimeout: 1s
```

So Envoy can retry failed calls according to the policy rather than every application independently implementing identical networking behavior. Istio exposes retry configuration through HTTP routing rules. [Istio](https://istio.io/latest/docs/concepts/traffic-management/)

---

# 20. Another huge feature: mTLS

Suppose:

```text
Frontend → Orders
```

Without mesh-level encryption, the internal connection might simply be ordinary HTTP:

```text
GET /orders
```

With Istio mTLS, conceptually:

```text
Frontend
   |
Frontend Envoy
   |
   | encrypted mTLS
   |
Orders Envoy
   |
Orders
```

So the applications might still think:

```text
http://orders
```

while the proxies secure service-to-service transport.

This is one reason service meshes became popular:

```text
Applications focus on business logic.

Mesh focuses on networking/security policy.
```

---

# 21. Now put all the pieces together

Your application code:

```python
response = requests.get("http://orders")
```

That's it.

The complete network path becomes approximately:

```text
                     CONTROL PLANE

                  +---------------+
                  |    istiod     |
                  +-------+-------+
                          |
                  proxy configuration
                          |
             +------------+-------------+
             |                          |
             v                          v

                  DATA PLANE

+-----------------------+      +-----------------------+
| Frontend Pod          |      | Orders v1 Pod         |
|                       |      |                       |
| +----------+          |      |       +----------+    |
| | frontend |          |      |       | orders   |    |
| +----+-----+          |      |       +----^-----+    |
|      |                |      |            |          |
|      v                |      |            |          |
| +----------+          |      |       +----+-----+    |
| |  Envoy   |----------+----->|       |  Envoy   |    |
| +----------+          |      |       +----------+    |
+-----------------------+      +-----------------------+

                    OR

                               +-----------------------+
                               | Orders v2 Pod         |
                               |                       |
                               |       +----------+    |
                     --------->| Envoy | orders   |    |
                               +-----------------------+
```

And your Istio config says:

```text
DestinationRule:

v1 = pods labelled version=v1
v2 = pods labelled version=v2
```

while:

```text
VirtualService:

orders
   |
   +-- 90% --> v1
   |
   +-- 10% --> v2
```

That's the core of Istio traffic management. `VirtualService` controls **how** requests are routed, while `DestinationRule` defines destination subsets and policies that apply after routing. [Istio](https://istio.io/latest/docs/concepts/traffic-management/)

---

# 22. The Kubernetes vs Istio separation

This is worth memorizing.

```text
KUBERNETES
```

answers:

> Where are my workloads and how do I keep them running?

Examples:

```text
Deployment
Pod
Service
ReplicaSet
ConfigMap
Secret
```

Istio answers:

> How should network traffic behave between those workloads?

Examples:

```text
routing
retries
timeouts
mTLS
authorization
traffic splitting
circuit breaking
telemetry
```

So conceptually:

```text
                  Kubernetes

         frontend             orders
            |                    |
            |     networking     |
            +--------------------+
                      |
                    Istio
                      |
             controls behavior
```

---

# 23. Four Istio terms you should know first

Don't try to learn 30 Istio CRDs yet.

Start with these:

| Thing | Think of it as |
|---|---|
| **Envoy** | Proxy handling real requests |
| **istiod** | Control plane configuring proxies |
| **VirtualService** | Where should traffic go? |
| **DestinationRule** | What versions/policies exist at destination? |

Then later learn:

```text
Gateway
AuthorizationPolicy
PeerAuthentication
ServiceEntry
Waypoint
Ambient mesh
```

---

# 24. The most important mental model

When you write:

```yaml
kind: VirtualService
```

don't imagine Kubernetes itself routing the HTTP traffic.

Instead:

```text
You write YAML
     ↓
Kubernetes stores it
     ↓
istiod watches it
     ↓
istiod generates proxy configuration
     ↓
Envoy receives configuration
     ↓
Actual HTTP request arrives
     ↓
Envoy applies the rule
```

So Istio is fundamentally:

```text
                    CONFIGURATION

 YAML → Kubernetes → istiod
                         |
                         |
                         v
                       Envoy


                       TRAFFIC

 Application → Envoy → Envoy → Application
```

Once this picture clicks, most of Istio becomes much less mysterious.

One modernization note: current Istio also has **ambient mode**, where every workload doesn't need its own sidecar; Istio uses node-level secure tunnels and optional waypoint proxies instead. The project currently recommends ambient as the starting point for new users, although the traffic-management ideas above remain central. [Istio](https://istio.io/latest/docs/overview/quickstart/?utm_source=chatgpt.com)

If you're learning Istio for DevOps work, the best next progression is **`Service → Envoy → istiod → VirtualService → DestinationRule → Gateway → mTLS → AuthorizationPolicy`**.


### Elaborated explaination on
 - Gateway
 - AuthorizationPolicy
 - PeerAuthentication
 - ServiceEntry
 - Waypoint
 - Ambient mesh

 Use this one application in your head throughout:

```text
Internet User
     |
     v
 api.shop.com
     |
     v
Ingress Gateway
     |
     v
 Frontend
     |
     v
 Orders
     |
     v
External payment API
```

Now the six concepts become much easier:

```text
Gateway              = how traffic enters/leaves the mesh
AuthorizationPolicy  = who is allowed to call whom
PeerAuthentication   = must service-to-service traffic use mTLS?
ServiceEntry          = teach Istio about a service outside Kubernetes
Ambient mesh          = Istio without an Envoy sidecar in every pod
Waypoint              = optional L7 Envoy proxy used by ambient mesh
```

---

# 1. Gateway

Suppose your Kubernetes cluster contains:

```text
frontend
orders
payments
```

Inside Kubernetes, `frontend` might be reachable as:

```text
http://frontend.shop.svc.cluster.local
```

But your customer sitting on the internet cannot call that.

They want:

```text
https://api.shop.com
```

You therefore need something sitting at the **edge of the cluster/mesh**.

That is an **ingress gateway**.

Istio gateways are Envoy proxies at the edge of the mesh that can control incoming or outgoing traffic. Modern Istio also supports the Kubernetes Gateway API, which Istio intends to make its default traffic-management API over time. [Istio](https://istio.io/latest/docs/setup/additional-setup/gateway/?utm_source=chatgpt.com)

Think:

```text
Internet
   |
   | HTTPS
   v
+---------------------+
| Istio Ingress       |
| Gateway / Envoy     |
+----------+----------+
           |
           v
       frontend
```

## Gateway vs Kubernetes Service

These are different jobs.

A Kubernetes `Service`:

```yaml
kind: Service
```

primarily gives workloads a stable network identity and selects pods.

An Istio/Kubernetes Gateway deals with things like:

```text
accept traffic on port 443
TLS certificates
api.shop.com
HTTP routing
```

For example, conceptually:

```text
api.shop.com/orders
          |
          v
       orders
```

while:

```text
api.shop.com/catalog
          |
          v
       catalog
```

---

## Example Gateway

Using Istio's traditional `Gateway` API:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway

metadata:
  name: shop-gateway
  namespace: shop

spec:
  selector:
    istio: ingressgateway

  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP

      hosts:
        - "api.shop.com"
```

This means roughly:

> Configure the ingress gateway Envoy to listen on HTTP port 80 for `api.shop.com`.

Then you connect it to routing using a `VirtualService`.

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService

metadata:
  name: shop
  namespace: shop

spec:
  hosts:
    - "api.shop.com"

  gateways:
    - shop-gateway

  http:
    - match:
        - uri:
            prefix: /orders

      route:
        - destination:
            host: orders
            port:
              number: 80
```

Now:

```text
GET http://api.shop.com/orders
```

becomes approximately:

```text
Internet
   |
   v
Ingress Gateway
   |
   | VirtualService says:
   | "/orders -> orders service"
   v
orders
```

The Gateway determines what the edge proxy listens for; routing configuration determines the destination. [Istio](https://istio.io/latest/docs/reference/config/networking/gateway/?utm_source=chatgpt.com)

---

# 2. PeerAuthentication

Now forget the internet for a moment.

Look inside your cluster:

```text
frontend → orders
```

Question:

> Should frontend be allowed to communicate with Orders using unencrypted ordinary HTTP transport?

Maybe you want:

```text
NO.

All service-to-service communication must be protected using mTLS.
```

That is where `PeerAuthentication` comes in.

Istio's peer authentication controls workload-to-workload authentication, primarily mutual TLS. A namespace-level policy can require `STRICT` mTLS for workloads in that namespace. [Istio](https://istio.io/latest/docs/reference/config/security/peer_authentication/?utm_source=chatgpt.com)

---

## What is mTLS?

Normal TLS looks roughly like:

```text
Client --------> Server

                 proves:
              "I am server X"
```

The client verifies the server certificate.

With **mutual TLS**:

```text
Client <-------> Server

Client proves identity
Server proves identity
```

So both sides authenticate.

In Istio:

```text
frontend identity
       |
       | mTLS
       v
orders identity
```

Istio manages workload identities and can establish this transport authentication without requiring your application code to implement it itself. [Istio](https://istio.io/latest/docs/concepts/security/?utm_source=chatgpt.com)

---

## Example

Suppose namespace:

```text
shop
```

contains:

```text
frontend
orders
payments
```

Create:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication

metadata:
  name: default
  namespace: shop

spec:
  mtls:
    mode: STRICT
```

This means:

> Workloads covered by this policy should require mTLS for incoming peer traffic.

The `STRICT` namespace example is explicitly supported by Istio's authentication policy model. [Istio](https://istio.io/latest/docs/tasks/security/authentication/authn-policy/?utm_source=chatgpt.com)

Now:

```text
Istio workload
      |
      | mTLS
      v
orders
```

works.

But a workload trying to reach the protected service using plaintext where mTLS is required will fail authentication. [Istio](https://istio.io/latest/docs/tasks/security/authentication/authn-policy/?utm_source=chatgpt.com)

---

# PeerAuthentication does NOT mean authorization

This distinction is extremely important.

Suppose:

```text
frontend → payments
```

and mTLS tells Payments:

```text
"This really is frontend."
```

That does **not** automatically mean:

```text
"frontend is allowed to call payments."
```

Authentication asks:

> **Who are you?**

Authorization asks:

> **Are you allowed to do this?**

So:

```text
PeerAuthentication
       ↓
Who are you?


AuthorizationPolicy
       ↓
What may you do?
```

---

# 3. AuthorizationPolicy

Let's say your architecture is:

```text
frontend
   |
   v
orders
   |
   v
payments
```

Your security rule says:

```text
Frontend CAN call Orders.

Orders CAN call Payments.

Frontend CANNOT call Payments directly.
```

This is where `AuthorizationPolicy` enters.

Istio authorization policies can allow or deny traffic based on workload identity and request properties, including principals, namespaces, paths, methods, IP information, and—for L7-capable enforcement—other HTTP attributes. [Istio](https://istio.io/latest/docs/reference/config/security/authorization-policy/?utm_source=chatgpt.com)

---

## Example

Suppose Orders runs using:

```yaml
serviceAccountName: orders
```

Frontend:

```yaml
serviceAccountName: frontend
```

Payments:

```yaml
serviceAccountName: payments
```

These identities matter because Istio's mTLS identity can correspond to Kubernetes service-account-based workload identity.

Conceptually you get principals such as:

```text
cluster.local/ns/shop/sa/frontend
cluster.local/ns/shop/sa/orders
cluster.local/ns/shop/sa/payments
```

Now suppose Payments should accept calls **only from Orders**.

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy

metadata:
  name: payments-policy
  namespace: shop

spec:
  selector:
    matchLabels:
      app: payments

  action: ALLOW

  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/shop/sa/orders
```

Read this as:

```text
Target:
payments

Allow:
orders service account
```

Therefore:

```text
orders → payments
```

allowed.

While:

```text
frontend → payments
```

is not allowed by that ALLOW policy.

Istio supports authorization based on authenticated principals, and its security documentation specifically discusses combining mTLS identity with authorization policies. [Istio](https://istio.io/latest/docs/concepts/security/?utm_source=chatgpt.com)

---

# You can go deeper than service identity

Suppose Orders exposes:

```text
GET    /orders
POST   /orders
DELETE /orders
```

Maybe Frontend should be allowed:

```text
GET /orders
POST /orders
```

but not:

```text
DELETE /orders
```

Then an L7 authorization policy can express HTTP-level restrictions.

Conceptually:

```yaml
rules:
  - to:
      - operation:
          methods:
            - GET
            - POST
```

Now you're enforcing:

```text
frontend
   |
   +---- GET /orders ------> YES
   |
   +---- POST /orders -----> YES
   |
   +---- DELETE /orders ---> NO
```

AuthorizationPolicy supports request-level conditions of this kind when the enforcement point has the needed L7 information. [Istio](https://istio.io/latest/docs/reference/config/security/authorization-policy/?utm_source=chatgpt.com)

This becomes important again when we discuss **waypoints**.

---

# PeerAuthentication + AuthorizationPolicy together

This combination is worth memorizing.

Imagine someone says:

```text
"I am orders."
```

PeerAuthentication asks:

```text
Can you cryptographically prove it?
```

If yes:

```text
identity = orders
```

Then AuthorizationPolicy asks:

```text
Is orders allowed to call payments?
```

So:

```text
                mTLS
orders --------------------> payments
           identity verified
                  |
                  v
         AuthorizationPolicy
                  |
             allowed?
             /      \
           yes       no
```

Very roughly:

```text
PeerAuthentication = authentication

AuthorizationPolicy = authorization
```

---

# 4. ServiceEntry

Now imagine Orders needs Stripe—or any external API.

```text
Orders
   |
   |
   v
api.external-payments.com
```

The external service isn't a Kubernetes Service.

Kubernetes knows things like:

```text
orders.shop.svc.cluster.local
payments.shop.svc.cluster.local
```

but your external SaaS/API is outside the cluster.

A `ServiceEntry` adds additional services into **Istio's internal service registry**, including external services or services represented outside normal platform discovery. [Istio](https://istio.io/latest/docs/reference/config/networking/service-entry/?utm_source=chatgpt.com)

Think:

> ServiceEntry = "Istio, here's another service I want you to understand."

---

## Example

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry

metadata:
  name: external-payments
  namespace: shop

spec:

  hosts:
    - api.payment-company.example

  ports:
    - number: 443
      name: https
      protocol: HTTPS

  resolution: DNS

  location: MESH_EXTERNAL
```

You're telling Istio:

```text
There is a service:

api.payment-company.example

port:
443

protocol:
HTTPS

location:
outside my mesh
```

A ServiceEntry can describe host names, ports, protocols, addresses and endpoints for services that Istio should understand. [Istio](https://istio.io/latest/docs/reference/config/networking/service-entry/?utm_source=chatgpt.com)

---

# Why would I bother?

Couldn't the pod simply do:

```bash
curl https://api.payment-company.example
```

Potentially yes.

But once Istio knows about the external destination, you can bring mesh traffic-management capabilities to that service. Istio's egress documentation specifically uses ServiceEntry to control and monitor traffic to external services. [Istio](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/?utm_source=chatgpt.com)

For example:

```text
orders
   |
   v
ServiceEntry
   |
   +-- routing
   +-- observability
   +-- controlled egress behavior
   +-- TLS-related configuration
   |
   v
external API
```

There are deployments where outbound traffic is configured as `REGISTRY_ONLY`; in that configuration, registered destinations such as ServiceEntries are central to controlling allowed egress. [Istio](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/?utm_source=chatgpt.com)

---

# ServiceEntry isn't only for the internet

This is another useful point.

Suppose your infrastructure is:

```text
Kubernetes
   |
   |
   +------ legacy VM
              |
              v
        database-api.internal
```

Istio can also represent external/legacy workloads and services. `ServiceEntry`, together with mechanisms such as `WorkloadEntry`, can describe services or workload instances outside normal Kubernetes discovery. [Istio](https://istio.io/latest/docs/reference/config/networking/service-entry/?utm_source=chatgpt.com)

So:

```text
ServiceEntry
```

really means:

> Extend Istio's understanding of what counts as a service.

Not merely:

> whitelist internet domain.

---

# 5. Now Ambient Mesh

Everything we've discussed earlier used:

```text
Pod
├── application
└── Envoy
```

That is the **sidecar architecture**.

If you have:

```text
1000 pods
```

you may have roughly:

```text
1000 Envoy sidecars
```

because each participating pod carries its own proxy.

In current Istio there are two main data-plane approaches:

```text
Sidecar mode
Ambient mode
```

In sidecar mode, Istio uses Envoy proxies attached to workloads.

In ambient mode, Istio instead uses node-level **ztunnel** proxies for its basic secure L4 mesh, with optional Layer-7 proxies called **waypoints**. [Istio](https://istio.io/latest/docs/overview/dataplane-modes/?utm_source=chatgpt.com)

---

# Sidecar architecture

Imagine three pods:

```text
Node
+---------------------------------------+
|                                       |
| frontend pod                          |
| [frontend] [Envoy]                    |
|                                       |
| orders pod                            |
| [orders]   [Envoy]                    |
|                                       |
| payments pod                          |
| [payments] [Envoy]                    |
|                                       |
+---------------------------------------+
```

Every workload carries a proxy.

---

# Ambient architecture

Ambient removes the requirement for an Envoy sidecar in each participating application pod.

Instead:

```text
Node
+--------------------------------------+
|                                      |
| [frontend]                           |
|                                      |
| [orders]                             |
|                                      |
| [payments]                           |
|                                      |
|                 +----------------+   |
|                 |    ztunnel     |   |
|                 +----------------+   |
|                                      |
+--------------------------------------+
```

`ztunnel` runs as a node-level component, deployed as a DaemonSet, and provides the basic Layer-4 secure mesh capabilities for enrolled workloads. [Istio](https://istio.io/latest/docs/ambient/usage/waypoint/?utm_source=chatgpt.com)

Think:

```text
Sidecar:

pod → own Envoy


Ambient:

pod
 \
pod ----> node ztunnel
 /
pod
```

---

# So what does ztunnel do?

The important beginner model is:

```text
ztunnel = secure L4 mesh
```

Think:

```text
identity
mTLS
TCP-level traffic
basic L4 authorization
```

rather than:

```text
inspect HTTP URL
route /checkout
inspect JWT claim
do sophisticated HTTP routing
```

Ambient intentionally separates the secure Layer-4 overlay from optional Layer-7 processing. [Istio](https://istio.io/latest/docs/ambient/overview/?utm_source=chatgpt.com)

That separation is the key design.

---

# Layer 4 vs Layer 7

You need to understand this to understand Waypoint.

## Layer 4

At L4, imagine that Istio knows:

```text
source:
frontend

destination:
orders

destination port:
8080
```

It understands:

```text
TCP connection
IP
port
identity
```

But not necessarily:

```text
GET /orders/123

POST /orders

header:
x-user: john
```

Those are HTTP concepts.

---

## Layer 7

An L7 proxy understands HTTP:

```http
POST /orders/123 HTTP/1.1
Authorization: Bearer xxx
X-User-Type: beta
```

Therefore it can implement richer policies:

```text
GET /orders       → allowed
DELETE /orders    → denied

x-user-type=beta  → orders-v2

JWT admin=true    → admin service
```

And that's where **Waypoint** comes in.

---

# 6. Waypoint

In ambient mode:

```text
ztunnel = L4

waypoint = L7
```

A waypoint is an Envoy-based Layer-7 proxy used by ambient mode. Waypoints can be shared by applications with a common security boundary and typically handle traffic for services/workloads rather than requiring a proxy in every application pod. [Istio](https://istio.io/latest/docs/ambient/usage/waypoint/?utm_source=chatgpt.com)

Imagine:

```text
frontend
    |
    v
ztunnel
    |
    v
Waypoint
    |
    v
ztunnel
    |
    v
orders
```

Now the Waypoint can understand:

```text
GET /orders
POST /orders
headers
HTTP routes
L7 AuthorizationPolicy
```

---

# Why not make ztunnel do everything?

Because ambient intentionally separates:

```text
cheap/basic secure networking
```

from:

```text
full HTTP-aware proxying
```

So you can choose.

Some services only need:

```text
mTLS
identity
L4 policy
```

They need:

```text
ztunnel
```

Others need:

```text
HTTP routing
JWT-related policy
L7 authorization
HTTP telemetry
```

Then add:

```text
Waypoint
```

Istio describes ambient as incremental adoption from no mesh, to a secure L4 overlay, to full L7 processing and policy. [Istio](https://istio.io/latest/docs/ambient/overview/?utm_source=chatgpt.com)

That's one of the most important ideas behind ambient.

---

# Example

Imagine:

```text
frontend
orders
payments
redis
```

Maybe Redis only needs:

```text
encrypted traffic
workload identity
```

So:

```text
Redis
  |
ztunnel only
```

But Orders needs rules like:

```text
GET /orders → frontend allowed

DELETE /orders → admin only

/orders/v2 → canary routing
```

That requires HTTP awareness.

Therefore:

```text
orders
   |
waypoint
   |
ztunnel
```

Conceptually:

```text
             Ambient mesh

frontend
   |
   v
+---------+
| ztunnel |
+----+----+
     |
     | encrypted mesh transport
     |
     v
+-----------------------+
| Orders Waypoint       |
|                       |
| Envoy                 |
|                       |
| understands HTTP      |
| routing               |
| L7 authorization      |
+-----------+-----------+
            |
            v
        +---------+
        | ztunnel |
        +----+----+
             |
             v
           orders
```

---

# Creating a Waypoint

Istio provides `istioctl` helpers for this.

For example:

```bash
istioctl waypoint apply \
  --enroll-namespace \
  --wait
```

This can create a namespace waypoint and enroll the namespace to use it. Istio's current ambient authorization tutorial demonstrates this flow. [Istio](https://istio.io/latest/docs/ambient/getting-started/enforce-auth-policies/?utm_source=chatgpt.com)

The result conceptually becomes:

```text
Namespace: shop

             waypoint
                |
      +---------+---------+
      |         |         |
      v         v         v
   frontend   orders   payments
```

It is **not necessarily one waypoint per application pod**.

That's a major difference from sidecars.

---

# Ambient AuthorizationPolicy

This has an important architectural wrinkle.

In ambient:

### L4 policy

Can be enforced by:

```text
ztunnel
```

For example:

```text
Only namespace shop may connect to database.
```

Istio documents that L4 authorization rules involving source principals, namespaces, IP ranges and similar connection-level information can be enforced in ambient without requiring waypoint L7 processing. [Istio](https://istio.io/latest/docs/ambient/migrate/migrate-policies/?utm_source=chatgpt.com)

---

### L7 policy

Suppose:

```text
GET /orders allowed

DELETE /orders denied
```

ztunnel cannot make a sophisticated HTTP-path decision.

So:

```text
Waypoint
```

must enforce it.

Istio's ambient documentation explicitly requires a waypoint for Layer-7 authorization policies. [Istio](https://istio.io/latest/docs/ambient/getting-started/enforce-auth-policies/?utm_source=chatgpt.com)

Therefore:

```text
AuthorizationPolicy

             What does rule inspect?
                     |
             +-------+-------+
             |               |
             v               v

        TCP/identity       HTTP
        namespace          method
        principal          path
        IP                 headers
             |               |
             v               v

          ztunnel         waypoint
```

Very useful mental model.

---

# Putting all six concepts together

Now our complete shop looks like this.

```text
                        INTERNET

                           |
                           | HTTPS
                           v

                +---------------------+
                |   Ingress Gateway   |
                |       Envoy         |
                +----------+----------+
                           |
                           |
                           v


                    AMBIENT MESH


                    frontend pod
                         |
                         v
                  +-------------+
                  |   ztunnel   |
                  +------+------+
                         |
                         |
                         v

               +--------------------+
               | Orders Waypoint    |
               |       Envoy        |
               |                    |
               | HTTP routing       |
               | L7 authorization   |
               +---------+----------+
                         |
                         v
                  +-------------+
                  |   ztunnel   |
                  +------+------+
                         |
                         v

                     orders pod

                         |
                         |
                         | ServiceEntry
                         v

                external payment API
```

Now sprinkle the security policies over it.

---

# PeerAuthentication

Says:

```text
Internal mesh connection:

MUST use mTLS
```

Example:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication

metadata:
  name: default
  namespace: shop

spec:
  mtls:
    mode: STRICT
```

Conceptually:

```text
frontend ==mTLS==> orders
```

PeerAuthentication is about **peer authentication and transport security**, not deciding whether a particular HTTP operation is authorized. [Istio](https://istio.io/latest/docs/reference/config/security/peer_authentication/?utm_source=chatgpt.com)

---

# AuthorizationPolicy

Says:

```text
WHO may do WHAT?
```

Example:

```text
frontend → orders    YES

frontend → payments  NO

orders → payments    YES
```

or:

```text
frontend:

GET /orders     YES
DELETE /orders  NO
```

The latter requires L7 enforcement, which in ambient means a waypoint. [Istio](https://istio.io/latest/docs/reference/config/security/authorization-policy/?utm_source=chatgpt.com)

---

# Gateway

Says:

```text
What traffic can enter my application from outside?
```

Example:

```text
api.shop.com:443
        |
        v
     frontend
```

The gateway is an Envoy-based edge component, separate conceptually from service-to-service mesh proxies. [Istio](https://istio.io/latest/docs/setup/additional-setup/gateway/?utm_source=chatgpt.com)

---

# ServiceEntry

Says:

```text
Istio, this external thing should be part
of your service registry knowledge.
```

Example:

```text
orders
   |
   v
api.payment-company.example
```

ServiceEntry extends Istio's registry beyond automatically discovered platform services. [Istio](https://istio.io/latest/docs/reference/config/networking/service-entry/?utm_source=chatgpt.com)

---

# Ambient Mesh

Says:

```text
Don't inject Envoy into every pod.

Use shared/node-level mesh infrastructure.
```

Basic path:

```text
pod
 |
ztunnel
 |
ztunnel
 |
pod
```

Ambient's node-level ztunnel provides the base data plane, with optional waypoints for L7 capabilities. [Istio](https://istio.io/latest/docs/overview/dataplane-modes/?utm_source=chatgpt.com)

---

# Waypoint

Says:

```text
This service needs HTTP-aware Istio functionality.
```

So:

```text
ztunnel
   |
   v
waypoint Envoy
   |
   +-- HTTP routing
   +-- L7 authorization
   +-- HTTP-aware policies
   |
   v
service
```

Waypoints are Envoy-based L7 proxies in ambient mode. [Istio](https://istio.io/latest/docs/ambient/usage/waypoint/?utm_source=chatgpt.com)

---

# The easiest cheat sheet

| Object | Question it answers |
|---|---|
| **Gateway** | How does outside traffic enter/leave? |
| **PeerAuthentication** | How must workloads authenticate each other? |
| **AuthorizationPolicy** | Who may call what? |
| **ServiceEntry** | What non-Kubernetes/external service should Istio know? |
| **ztunnel** | How does ambient provide the base secure L4 mesh? |
| **Waypoint** | Where does ambient perform HTTP/L7 processing? |
| **Ambient mesh** | How do we run Istio without a sidecar in every pod? |

And don't confuse these three:

```text
PeerAuthentication
       |
       v
"Prove who you are."


AuthorizationPolicy
       |
       v
"Are you allowed?"


Gateway
       |
       v
"How did you enter?"
```

Finally, the architecture I would keep in your head as a beginner is:

```text
                            INTERNET
                               |
                               v
                        +--------------+
                        |   Gateway    |
                        +------+-------+
                               |
                               v
                           frontend
                               |
                         +-----v------+
                         |  ztunnel   |   ← L4/mTLS
                         +-----+------+
                               |
                               v
                        +--------------+
                        |   Waypoint   |   ← L7/HTTP
                        +------+-------+
                               |
                         +-----v------+
                         |  ztunnel   |
                         +-----+------+
                               |
                               v
                            orders
                               |
                               | ServiceEntry
                               v
                       external-service


PeerAuthentication
      └── protects/authenticates mesh connections

AuthorizationPolicy
      └── decides who is allowed to perform operations
```

If you understand that diagram, you're already past one of the hardest conceptual hurdles in Istio.