# Kubernetes Interview Questions — ELI7 Edition

Think of Kubernetes as a giant city for apps:

- **Pods** = tiny houses where apps live.
- **Nodes** = apartment buildings that hold those houses.
- **Cluster** = the whole city.
- **Control plane** = city hall that decides where houses go and keeps rules working.

## Architecture and cluster design

### 2. Multi-region Kubernetes for disaster recovery

Build the same app city in two different places.

- If Region A has a problem, traffic can go to Region B.
- Use global DNS/load balancing as the traffic police.
- Keep app code and Kubernetes configuration in Git so both cities look the same.
- Copy or replicate important data safely between regions.
- Regularly practice failover; a backup plan is only useful if it works during an emergency.

Simple answer: “I run independent clusters in separate regions, replicate data based on RPO/RTO needs, and use DNS failover so users can reach the healthy region.”

### 3. Managed Kubernetes vs self-managed Kubernetes

- **Managed Kubernetes** is like renting an apartment where the building owner fixes elevators and plumbing.
  - EKS, AKS, and GKE manage much of the control plane.
  - Easier upgrades and less operational work.
- **Self-managed Kubernetes** is like owning the building yourself.
  - More control.
  - You must manage etcd, API servers, upgrades, backups, security, and high availability.

For most companies, managed Kubernetes is safer because the team can focus on running applications rather than repairing Kubernetes itself.

### 4. Designing for 10k+ pods

A city with 10,000 houses needs neighborhoods and rules.

- Split workloads into separate clusters when needed; do not put everything in one giant cluster.
- Use namespaces like neighborhoods.
- Use quotas so one team cannot take all CPU or memory.
- Use multiple node pools for different house types: normal apps, batch jobs, system apps, memory-heavy apps.
- Ensure networking, DNS, metrics, logs, and control-plane capacity can handle the size.
- Test large rollouts and failure scenarios before production.

### 5. Control-plane bottlenecks in large clusters

City hall can become slow when too many people ask it questions at once.

- **API server:** too many `kubectl`, controller, or application requests.
- **etcd:** too many updates stored in Kubernetes’ database.
- **Scheduler:** too many pods waiting for a place to run.
- **Controllers:** too many things changing too frequently.
- **CoreDNS:** too many apps asking, “Where is this service?”

### 6. Kubernetes leader election

Imagine several traffic police officers, but only one should control an intersection.

- Multiple scheduler or controller copies run for high availability.
- They use a small Kubernetes record called a **Lease**.
- One becomes the leader.
- The leader keeps renewing its lease.
- If it disappears, another copy becomes leader.

This avoids two controllers trying to do the same job at the same time.

---

## Networking

### 7. Kubernetes networking model and CNI

Every pod gets its own IP address, like each house getting its own address.

- Pods should be able to talk to other pods.
- Services give apps a stable name and virtual address.
- Pods may come and go, but the Service name stays the same.
- A **CNI plugin** is the networking helper that gives pods IP addresses and connects them to the network.

CNI is like the road-building team: it creates roads, assigns addresses, and may enforce traffic rules.

### 8. Calico vs Flannel vs Cilium

| Tool | ELI7 explanation |
|---|---|
| Flannel | Builds simple roads between houses. Good for basic networking. |
| Calico | Builds roads and also provides strong traffic rules, like “this neighborhood cannot talk to that one.” |
| Cilium | Uses a faster, modern way to manage roads and inspect traffic using eBPF. Great for security and visibility. |

Simple answer: “Flannel is simple networking, Calico is strong for network policy, and Cilium adds modern eBPF-based networking, observability, and security.”

### 9. Debug pod-to-pod communication failures

Check the path step by step:

1. Is the source pod healthy?
2. Is the destination pod healthy and listening on the right port?
3. Does DNS resolve the Service name?
4. Does the Service point to the right pods?
5. Is a NetworkPolicy blocking traffic?
6. Is the CNI plugin healthy?
7. Is TLS or the application itself rejecting the request?

Think of it as checking whether the sender, road, address, gatekeeper, and receiver are all working.

### 10. What happens when you create a Kubernetes Service?

A Service is like a company’s main phone number.

- You create a Service with a selector, such as `app: payment-api`.
- Kubernetes finds pods with that label.
- Kubernetes stores those pod addresses in EndpointSlices.
- DNS creates a stable name for the Service.
- Network rules send traffic from the Service to one healthy pod.

Users call the same number even when workers change desks.

### 11. Zero-trust networking

Zero trust means: “Do not trust anyone just because they are already inside the cluster.”

- Start with **deny all** network traffic.
- Allow only the exact traffic each app needs.
- Use NetworkPolicies.
- Use mTLS to encrypt app-to-app traffic.
- Use service identity, not just IP addresses.
- Give each workload only the permissions it needs.

Example: the billing service can talk to the database, but a random frontend pod cannot.

---

## Security

### 12. Securing secrets beyond Kubernetes Secrets

A Kubernetes Secret is like a locked envelope, but you still need to protect the mailbox, key, and delivery route.

- Store sensitive values in a real secret manager or vault.
- Encrypt Kubernetes Secrets at rest.
- Use short-lived credentials instead of long-lived passwords.
- Use workload identity so pods get only the access they need.
- Rotate secrets regularly.
- Limit who can read Secrets with RBAC.
- Never place secrets directly in Git, logs, or container images.

### 13. PSA vs PSP

- **PSP (PodSecurityPolicy)** was the old security guard system. It was complicated and has been removed from Kubernetes.
- **PSA (Pod Security Admission)** is the newer built-in guard system.

PSA provides simple safety levels:

- **Privileged:** few restrictions.
- **Baseline:** basic safety rules.
- **Restricted:** strongest common safety rules.

PSA is like putting clear safety rules on the entrance to each neighborhood.

### 14. Runtime security

Runtime security means watching apps after they start running.

- Prevent containers from running as root.
- Use read-only filesystems when possible.
- Remove unnecessary Linux permissions.
- Monitor suspicious actions, such as opening shells unexpectedly.
- Detect strange outbound network traffic.
- Alert security teams if a container behaves like malware.

It is like having cameras and alarms inside the building, not just checking IDs at the front door.

### 15. OPA Gatekeeper or Kyverno

They are rule-checkers at the cluster entrance.

They can block things like:

- Containers running as root.
- Images from unknown registries.
- Missing CPU/memory limits.
- Use of `latest` image tags.
- Missing labels, owners, or security settings.

- **Gatekeeper** uses a policy language called Rego.
- **Kyverno** uses Kubernetes-style YAML and is often easier for teams already comfortable with manifests.

### 16. Securing images before deployment

Treat container images like packages entering a hospital warehouse.

- Use small, trusted base images.
- Scan images for known vulnerabilities.
- Create an SBOM: a list of everything inside the image.
- Sign images so the cluster can verify who built them.
- Use fixed image versions or digests.
- Block serious vulnerabilities unless there is an approved exception.

---

## Performance and scaling

### 17. How Cluster Autoscaler works

Cluster Autoscaler adds more buildings when there is no room for new houses.

1. A pod asks for space.
2. Kubernetes cannot find a node with enough room.
3. The pod stays Pending.
4. Cluster Autoscaler notices.
5. It increases a configured node group.
6. A new node joins the cluster.
7. The pod gets scheduled.

When the cluster is quiet, it can remove empty or safely removable nodes.

### 18. HPA vs VPA vs KEDA

| Tool | ELI7 explanation |
|---|---|
| HPA | Adds more copies of an app when existing copies are busy. |
| VPA | Suggests or changes how much CPU/memory each app copy should receive. |
| KEDA | Adds app copies when outside work arrives, such as queue messages. |

Example:

- HPA: “CPU is high; add more API pods.”
- KEDA: “There are 1,000 messages waiting; add more worker pods.”
- VPA: “Each worker asked for too much memory; right-size it.”

### 19. Optimizing resources in a large cluster

- Give pods realistic CPU and memory requests.
- Set limits to stop one pod from using everything.
- Use HPA/KEDA so pod count matches demand.
- Use node autoscaling so node count matches demand.
- Use VPA recommendations to find waste.
- Delete unused workloads and stale environments.
- Watch requested resources versus actual usage.

The key idea: do not rent a huge apartment for an app that only needs a small room.

### 20. Noisy neighbor problems

A noisy neighbor is one app using too much shared CPU, memory, disk, or network.

Fix it with:

- CPU and memory requests/limits.
- ResourceQuotas per namespace.
- Separate node pools for important apps.
- PriorityClasses for critical workloads.
- Monitoring CPU throttling, OOMKills, disk I/O, and network usage.

### 21. Tuning kube-scheduler performance

The scheduler is the person deciding which house goes into which building.

Make its job easier:

- Use simple scheduling rules.
- Avoid excessive pod affinity/anti-affinity rules.
- Use labels and node pools to narrow choices.
- Give pods accurate resource requests.
- Reduce unnecessary pod churn.
- Monitor how long pods remain Pending.

Too many complicated rules make the scheduler check too many possible homes.

---

## Observability

### 22. Centralized logging

Instead of reading logs from thousands of pods one by one, collect them in one place.

- Apps write logs to standard output.
- A node-level log agent such as Fluent Bit or Vector collects them.
- The agent sends logs to a central logging platform.
- Teams search logs, create alerts, and investigate incidents there.

Always avoid logging secrets, passwords, tokens, or sensitive health information.

### 23. Critical cluster health metrics

Watch the city’s health:

- API server latency and errors.
- etcd health and latency.
- Node readiness and disk/memory pressure.
- Pending pods.
- Pod restarts and OOMKills.
- CPU throttling.
- DNS errors.
- Network errors.
- Deployment availability.
- Autoscaler actions.
- Volume attach/mount failures.

### 24. High CPU in kube-apiserver

The API server is city hall. High CPU usually means too many requests or slow dependencies.

Check:

- Which clients make the most API calls.
- Excessive `LIST` and `WATCH` requests.
- Noisy custom controllers.
- Slow admission webhooks.
- Slow etcd.
- Huge numbers of Kubernetes objects.
- Audit logs and API server metrics.

Fix the noisy caller or slow dependency instead of only increasing API server size.

### 25. Prometheus and Kubernetes

Prometheus is a robot that regularly asks systems, “How are you doing?”

- Apps expose a `/metrics` endpoint.
- Prometheus discovers Kubernetes pods/services.
- It collects numbers such as CPU, latency, errors, and queue size.
- Grafana shows dashboards.
- Alertmanager sends alerts.

Prometheus helps answer: “Is the cluster healthy, and what changed?”

### 26. Monitoring etcd

etcd is Kubernetes’ important database.

Watch:

- Does it have a leader?
- How long do writes take?
- How often does the leader change?
- Is disk latency high?
- Is the database growing too large?
- Do all etcd members have quorum?

If etcd is unhealthy, the whole cluster can become slow or unavailable.

---

## CI/CD and GitOps

### 27. GitOps with Argo CD or Flux

GitOps means Git is the instruction book for the cluster.

1. A developer changes Kubernetes YAML in Git.
2. The change is reviewed in a pull request.
3. Argo CD or Flux notices the approved change.
4. It makes the cluster match Git.
5. If something is wrong, roll back Git and the cluster returns to the earlier state.

This makes changes auditable, repeatable, and easier to recover.

### 28. Blue-green deployments

Blue-green is like building a new store next to the old store.

- **Blue** = current version.
- **Green** = new version.
- Test green while users still use blue.
- When ready, move traffic from blue to green.
- If green fails, move traffic back to blue quickly.

### 29. Canary deployments

A canary is like letting a few people try a new feature first.

- Send 5% of traffic to the new version.
- Watch errors, latency, and user impact.
- If healthy, increase to 20%, 50%, then 100%.
- If unhealthy, stop and roll back.

### 30. Benefits of progressive delivery

- Smaller blast radius.
- Safer releases.
- Real production testing with limited users.
- Faster rollback.
- Better confidence before full rollout.

### 31. Automatic rollback on failure

- Define what “healthy” means: error rate, latency, readiness, availability.
- During a canary or rollout, continuously check these metrics.
- If thresholds fail, stop sending traffic to the new version.
- Send traffic back to the stable version automatically.

Tools such as Argo Rollouts or Flagger can automate this.

---

## Storage

### 32. CSI architecture

CSI is like a universal adapter between Kubernetes and storage systems.

- Kubernetes asks for storage.
- The CSI driver talks to the cloud disk, file system, or storage appliance.
- A controller creates or attaches the storage.
- A node plugin mounts it into the pod.

This lets Kubernetes use many storage vendors in a standard way.

### 33. Persistent volume provisioning

1. A developer asks for storage using a PVC.
2. Kubernetes checks the StorageClass.
3. The CSI driver creates a real disk or volume.
4. Kubernetes connects that volume to the pod.
5. The app can store data even if the pod restarts.

A PVC is like asking for a locker; a PV is the actual locker that gets created.

### 34. Challenges of databases in Kubernetes

Databases remember important information, so they are harder than stateless apps.

Challenges:

- Data must not be lost.
- Storage must be fast and reliable.
- Backups and restores must be tested.
- Failover must avoid split-brain.
- Upgrades need care.
- Replication and quorum need correct design.

Kubernetes can run databases, but it does not automatically solve database operations.

### 35. Migrating stateful workloads between clusters

- First, back up and test restoring the data.
- Replicate data to the new location when possible.
- Deploy the application configuration to the new cluster.
- Stop or limit writes during final data sync.
- Switch traffic only after validation.
- Keep a rollback plan.

Do not assume you can simply move a disk from one cluster to another.

---

## Troubleshooting

### 37. Pod stuck in `ContainerCreating`

`ContainerCreating` means Kubernetes is trying to prepare the pod but is blocked.

Check:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Common reasons:

- Image cannot be downloaded.
- Secret or ConfigMap is missing.
- Volume cannot attach or mount.
- CNI cannot give the pod an IP.
- Node has disk pressure.
- Security policy blocked the pod.

### 38. Pods in `CrashLoopBackOff`

CrashLoopBackOff means: “The app starts, crashes, waits, then tries again.”

Check:

```bash
kubectl logs <pod-name> -n <namespace> --previous
kubectl describe pod <pod-name> -n <namespace>
```

Then check:

- Application logs.
- Exit code.
- Wrong environment variables or secrets.
- Missing dependency.
- Bad startup command.
- OOMKilled due to low memory.
- Wrong liveness/readiness probe.
- Incorrect port or configuration.

### 39. Why `kubectl` can become slow

`kubectl` talks to the API server. It becomes slow when city hall is overloaded.

Possible causes:

- API server is busy.
- etcd is slow.
- Too many objects in the cluster.
- Slow authentication or admission webhooks.
- Too many clients watching resources.
- Broad commands such as `kubectl get pods -A`.
- Network latency to the cluster.

### 40. Debugging DNS issues

DNS is the cluster’s phone book.

Test it:

```bash
kubectl run dns-test --rm -it --image=busybox:1.36 -- nslookup kubernetes.default
```

Check:

- Are CoreDNS pods healthy?
- Does the `kube-dns` Service have endpoints?
- Is DNS blocked by NetworkPolicy?
- Is the pod’s `/etc/resolv.conf` correct?
- Can CoreDNS reach upstream DNS?
- Is the CNI/network working?
- Is CoreDNS overloaded?

Simple interview summary: “For DNS, I check the pod resolver configuration, CoreDNS health, Service endpoints, NetworkPolicies, upstream DNS, and node networking in that order.”