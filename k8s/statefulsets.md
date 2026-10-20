## Why Not Use a Deployment for Stateful Applications? Kubernetes StatefulSet vs Deployment

# Why Not Use a Deployment for Stateful Applications? Kubernetes StatefulSet vs Deployment

You can run a stateful application in a Deployment. The problem is that Kubernetes treats Deployment replicas as disposable and interchangeable, while databases and clustered systems depend on stable identity, per-node storage, and ordered membership changes. The failure is rarely at install time. It shows up on the first restart, reschedule, or rolling update.

## What breaks with a Deployment

### 1. Pod identity is not stable

A Deployment creates a ReplicaSet, which creates Pods with generated names:

```text
orders-api-7d89cc9f76-k8zjl
orders-api-7d89cc9f76-jhsmq
```

On a Pod restart, the suffix changes. On a template change, the ReplicaSet hash changes too:

```text
orders-api-7d89cc9f76-k8zjl
→ orders-api-5f799689db-4pqsh
```

For a stateless API this is fine, since the name identifies a runtime instance and nothing else. For PostgreSQL replicas, Kafka brokers, Elasticsearch nodes, or ZooKeeper members, identity is part of cluster membership. A Pod that returns under a new name can look like a new node rather than a restarted one, which means re-registration, resync, or operator intervention.

A StatefulSet uses stable ordinals:

```text
postgres-0, postgres-1, postgres-2
postgres-1 crashes → recreated as postgres-1
```

### 2. Storage can't be mapped per replica

A stateful replica needs its own disk:

```text
postgres-0 → data-postgres-0
postgres-1 → data-postgres-1
postgres-2 → data-postgres-2
```

A Deployment has one Pod template and no `volumeClaimTemplates`, so there's no built-in way to create and bind one PVC per replica. The usual outcome is every replica mounting the same volume:

```text
replica A ─┐
replica B ─┼→ same disk
replica C ─┘
```

Unless the application and storage explicitly support multi-writer access, that corrupts data. A PostgreSQL data directory must never be shared by multiple database Pods.

You can work around this with separate PVCs and one workload per replica, but then you're rebuilding StatefulSet behavior by hand, and it's hard to operate safely.

### 3. No stable per-Pod DNS

A Deployment sits behind one load-balanced Service name:

```text
orders-api.prod.svc.cluster.local
```

That's what clients want. Cluster members, however, need to address specific peers. A StatefulSet with a headless Service gives each member a predictable name:

```text
postgres-0.postgres.prod.svc.cluster.local
postgres-1.postgres.prod.svc.cluster.local
postgres-2.postgres.prod.svc.cluster.local
```

### 4. No ordered lifecycle

Deployment rollouts create and delete Pods in parallel, bounded only by `maxSurge` and `maxUnavailable`. For quorum-based systems, order matters:

```text
start primary → replicas join → validate replication → update one member at a time
```

A StatefulSet defaults to ordered operations:

```text
Create: node-0 → node-1 → node-2
Delete: node-2 → node-1 → node-0
```

That's safer for bootstrapping and controlled recovery.

### 5. Rolling updates can damage quorum

A Deployment optimizes for replica-count availability. It has no concept of:

- Kafka controller/broker quorum
- PostgreSQL primary vs. replica roles
- Elasticsearch shard relocation
- Leader election state
- WAL recovery
- Replication lag

A StatefulSet gives you identity and ordering, but it doesn't understand your database either. For serious production databases, use an operator that handles failover, backup, restore, replication, and upgrades in a database-aware way.

## Why the name matters

The name itself isn't magic. What matters is that Kubernetes preserves the **name, DNS identity, and storage mapping together** across Pod recreation.

| | Deployment | StatefulSet |
|---|---|---|
| Pod name | Generated, replaceable | Stable ordinal |
| What the name means | One temporary runtime instance | One logical cluster member |
| Name after replacement | Different | Same |
| DNS | One Service name for all replicas | One stable name per replica |
| Storage | No per-Pod identity | PVC tied to ordinal |
| Client behavior | Call any healthy Pod | Can call a specific peer |

A database may persist its own member ID, data directory, and peer address:

```text
member ID: postgres-1
data directory: /var/lib/postgresql/data
peer address: postgres-1.postgres.prod.svc.cluster.local
```

If `postgres-1` came back under a random name, the cluster would see a stranger. An API Pod like `orders-api-7d89cc9f76-k8zjl` has no such identity. A new Pod starts, registers behind the Service, and takes traffic.

## When a Deployment is still the right choice

Use a Deployment when the state is externalized:

```text
API Pods → managed PostgreSQL / Redis / object storage / Kafka
```

The Pods can be recreated with no local data loss. A Deployment also works for workers if:

- Work is stored durably in a queue
- Messages are idempotent
- Duplicate processing is handled
- No worker needs a permanent identity or local disk

## The rule

```text
State lives outside the Pod → Deployment
State is tied to each Pod's identity and disk → StatefulSet
State needs database-aware operations → StatefulSet + operator
```


---

## Common follow-up questions

### Does a StatefulSet delete its PVCs when you scale down or delete it?

Not by default. PVCs created from `volumeClaimTemplates` are retained on both scale-down and StatefulSet deletion. That's deliberate: data outlives the Pod. The side effect is that scaling `3 → 2 → 3` reattaches `data-postgres-2` with whatever stale data it held, so your application must handle a member rejoining with old state.

Retention is configurable with `persistentVolumeClaimRetentionPolicy` (GA in 1.32):

```yaml
persistentVolumeClaimRetentionPolicy:
  whenDeleted: Retain   # or Delete
  whenScaled: Retain    # or Delete
```

This only fires on scale-down or StatefulSet deletion. Pod restarts, rescheduling, and node drains never delete PVCs. `whenScaled: Delete` is the dangerous one, since an accidental scale-down destroys that member's data.

### When would you use `podManagementPolicy: Parallel`?

The default, `OrderedReady`, waits for `node-N` to be Running and Ready before creating `node-N+1`. `Parallel` launches and terminates Pods without waiting. Use it when you need stable identity and storage but not ordered bootstrap, which is common for Kafka brokers or Cassandra-style systems that coordinate among themselves.

Two points interviewers like:

- `OrderedReady` can **deadlock**. If `node-0` only becomes Ready once it can reach quorum, it never gets there because `node-1` is never created. `Parallel` fixes this.
- It only affects scaling. Rolling updates still go one Pod at a time in reverse ordinal order, regardless of this setting.

Also, `OrderedReady` blocks on a broken Pod. If `node-1` is crash-looping, `node-2` never starts, and a rolling update stalls on the unhealthy Pod. Recovery is often manual (fix the template, then delete the stuck Pod).

### Can a single-replica Deployment with an RWO PVC work?

Yes, with caveats. For a one-instance, non-HA workload (Grafana on SQLite, a small Prometheus, Jenkins), it's a common pattern, but you must set the strategy explicitly:

```yaml
strategy:
  type: Recreate
```

The default `RollingUpdate` creates the new Pod before terminating the old one. With an RWO volume, two things can go wrong:

- The new Pod lands on a different node and sticks in `ContainerCreating` with a Multi-Attach error.
- The new Pod lands on the **same** node, where RWO permits multiple Pods, so both mount the volume concurrently. That's a brief two-writer window.

`Recreate` terminates the old Pod first, which removes the overlap at the cost of downtime during every rollout. `accessModes: [ReadWriteOncePod]` (GA in 1.29) enforces single-Pod access at the storage level and is a good safeguard to add.

### So what does StatefulSet guarantee that this setup doesn't?

At-most-one semantics per identity. A Deployment optimizes for availability: if a node becomes unreachable, the controller creates a replacement while the old Pod may still be running on the partitioned node. Whether you get two writers then depends entirely on the storage layer. Block storage with attach/detach enforcement usually saves you. NFS, RWX volumes, or any storage without attach enforcement won't.

A StatefulSet won't create a replacement for `postgres-1` until the old Pod is confirmed terminated. Note that this means a node failure leaves the member down until the node is recovered or the Pod is force-deleted, because StatefulSet trades availability for safety. Neither controller makes this automatic and safe for databases, which is why the operator line in the rule still applies.

---

The three questions are written as standalone answers, so you can lift any of them into an interview-prep cheat sheet unchanged. Version numbers (1.29, 1.32) are worth verifying against your target cluster version before publishing.