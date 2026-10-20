# Rollback strategies

Rollback means returning a system to a known-good state after a bad release.

Common strategies:

- **Re-deploy previous version**: Tag every release (for example, `v1.4.2`) and deploy the last healthy image/chart/build. This is the standard approach for stateless services.

- **Blue-green deployment**: Run old and new versions side by side. Shift traffic to the new environment; if it fails, point traffic back to the old one almost instantly.

- **Canary rollback**: Send a small percentage of traffic to the new version first. Stop and revert if error rate, latency, or business metrics degrade.

- **Feature flags / kill switches**: Keep code deployed but disable the risky feature without redeploying. Best for isolating new behavior quickly.

- **Progressive delivery rollback**: Automatically halt or roll back rollout steps when SLOs fail, such as elevated 5xx errors, latency, CPU, or checkout failures.

- **Infrastructure rollback**: Keep Terraform/Pulumi/Helm changes versioned. Revert the PR or deploy the previous approved revision. Be careful with destructive infrastructure changes.

- **Database rollback**: Prefer *expand-contract* migrations:
  1. Add backward-compatible schema changes.
  2. Deploy code that supports both schemas.
  3. Backfill data.
  4. Remove old fields only after the release is stable.

  Avoid relying on “down” migrations for production data. Restore from backups only when necessary and tested.

A practical incident sequence:

1. Stop the rollout.
2. Assess blast radius using logs, metrics, traces, and customer impact.
3. Use a flag or traffic shift if that fixes it fastest.
4. Otherwise deploy the last known-good version.
5. Verify recovery against defined health and business metrics.
6. Preserve evidence, fix forward in a PR, and document the decision.

The safest design is: immutable release artifacts, versioned configuration, automated health checks, backward-compatible database changes, and a tested rollback runbook.

## Blue-green deployment

At Staff Engineer level, rollback is a **system capability**, not a command someone remembers during an incident. Each release needs a defined recovery path, measurable rollback criteria, clear authority, and tested failure modes.

## 1. Re-deploy the previous version

**Use when:** Stateless services, containers, frontend assets, workers, or changes fully represented by a versioned artifact.

**Implementation notes**

- Build once and promote the exact immutable artifact across environments.
  - Example: `payments:v2026.09.24-3` with image digest, not `latest`.
- Keep a deployment record containing:
  - Git SHA
  - image digest
  - Helm chart version and values revision
  - configuration revision
  - database migration version
  - feature-flag state
  - deployment time and operator or pipeline ID
- Define **last known good** from observed production health, not merely the previous release.
- Roll back through the same deployment controller and pipeline used for rollout. Avoid manual `kubectl set image` except as an explicitly documented emergency path.
- Ensure configuration is compatible with both current and prior binaries. A rollback that restores code but leaves incompatible config is not a rollback.
- Make workers idempotent or version-tolerant. Old workers must safely consume events produced by newer workers.

**Decision criteria**

- Roll back when the new version causally correlates with a breach:
  - 5xx error-rate threshold
  - p95/p99 latency regression
  - saturation or queue lag
  - elevated crash-loop/restart rate
  - SLO burn rate
  - broken business KPI, such as payment-authorisation success

**Main risk**

- Code rollback does not undo side effects: writes, emitted messages, emails, payments, cache invalidations, or schema changes. Plan compensation separately.

---

## 2. Blue-green deployment

**Use when:** You need near-zero-downtime rollback and can afford duplicate production capacity.

**Architecture**

- **Blue** is the current stable production environment.
- **Green** is the new production-equivalent environment.
- Both have:
  - equivalent runtime configuration
  - identical secrets and external dependencies where appropriate
  - sufficient capacity for full production traffic
  - independent deployment and health status
- A stable traffic layer sits in front:
  - load balancer
  - API gateway
  - service mesh
  - DNS only if TTL and caching behavior are acceptable

**Release process**

1. Deploy new release to Green with no customer traffic.
2. Run smoke tests, synthetic transactions, readiness checks, security checks, and dependency validation.
3. Warm the environment:
   - scale pods and connections
   - warm caches if safe
   - load models/rules/configuration
   - verify autoscaling behavior
4. Shift production traffic from Blue to Green.
5. Observe both technical and business metrics for a defined soak period.
6. Keep Blue fully viable until Green exits the rollback window.
7. Decommission or repurpose Blue only after the window and verification complete.

**Traffic switching**

- Prefer L7 routing or a mesh/gateway to make traffic switching immediate and observable.
- Do not assume DNS is instant:
  - client and ISP caches can ignore low TTLs
  - long-lived connections may stay pinned
- Handle websockets, streaming requests, sticky sessions, background jobs, and consumer groups explicitly.
- Define connection draining:
  - stop accepting new requests on Blue
  - allow in-flight work for a bounded duration
  - force-close only after the deadline

**Rollback procedure**

1. Freeze writes or rollout activity if data compatibility is uncertain.
2. Shift new traffic from Green back to Blue.
3. Drain Green connections.
4. Pause Green workers and consumers to avoid duplicate processing.
5. Verify Blue health and business success metrics.
6. Keep Green available for investigation, but isolate it from live side effects.

**Data requirements**

- Blue and Green must work against the same database schema during the switch.
- Use backward-compatible migrations only.
- Avoid releases that require a schema downgrade to return to Blue.
- For write-heavy systems, use:
  - schema versioning
  - dual reads/writes where needed
  - event versioning
  - idempotency keys
  - compensating workflows for irreversible actions

**Cost and limits**

- Requires close to 2x capacity during deployments.
- More complex for stateful systems, large databases, and asynchronous processing.
- Best suited for critical APIs, gateways, and customer-facing applications with strict recovery-time objectives.

---

## 3. Canary deployment and rollback

**Use when:** You want to limit blast radius and validate behavior against real traffic.

**Traffic progression**

A typical policy might be:

- 1% for 10 minutes
- 5% for 15 minutes
- 25% for 30 minutes
- 50% for 30 minutes
- 100% after sustained health

The values depend on traffic volume, request criticality, and time needed for failures to appear.

**Canary evaluation signals**

Technical:

- request success rate
- p95/p99 latency
- CPU, memory, pod restarts
- dependency error rate
- queue lag and consumer failures
- cache hit ratio
- database connection/saturation metrics

Business:

- login success
- checkout completion
- payment authorisation
- search conversion
- order creation
- message delivery

Compare canary metrics to the stable version, not just to an absolute threshold.

**Automation**

- Use a progressive-delivery controller such as Argo Rollouts or Flagger, or equivalent platform tooling.
- Encode:
  - traffic steps
  - analysis windows
  - baseline comparison
  - failure thresholds
  - automatic abort behavior
  - manual approval gates for high-risk releases
- Prevent a rollout from continuing when analysis data is missing. Missing telemetry should fail closed for critical services.

**Rollback**

- Abort promotion.
- Set canary traffic to zero.
- Restore stable version to 100%.
- Scale down or quarantine canary pods after preserving diagnostics.
- Validate delayed impact such as queue lag, data integrity, or downstream rate limits.

**Main risk**

- Low traffic may not cover rare paths, large tenants, certain regions, batch jobs, or peak-load behavior. Segment canaries deliberately by geography, tenant class, API route, and workload type.

---

## 4. Feature flags and kill switches

**Use when:** The dangerous part is a behavior change, not the entire binary.

**Types**

- **Release flag:** Decouples deployment from activation.
- **Operational kill switch:** Immediately disables an expensive or unsafe path.
- **Experiment flag:** Routes users between variants.
- **Permission/tenant flag:** Enables a feature only for selected users or customers.
- **Fallback flag:** Switches to a known-safe algorithm, provider, or degraded mode.

**Design requirements**

- The flag check must be fast, reliable, and safe when the flag service is unavailable.
- Define the failure default per flag:
  - usually fail closed for risky new features
  - sometimes fail open for critical availability paths
- Include owner, purpose, expiry date, default value, and removal ticket.
- Audit every change to production flag state.
- Keep flags out of deeply nested business logic. A flag should represent one clear decision point.
- Test both states in CI and pre-production.

**Rollback procedure**

1. Disable the feature or route to the fallback.
2. Confirm new requests no longer enter the failing path.
3. Address in-flight jobs, cached responses, and asynchronous events.
4. Verify the business metric that motivated the kill switch.
5. Create a removal plan. Permanent flags become untestable configuration debt.

**Main risk**

- A flag cannot undo effects already written to data stores or third parties. Use idempotency and compensation where activation performs irreversible actions.

---

## 5. Progressive delivery with automated rollback

**Use when:** Releases are frequent and the platform can reliably judge health from telemetry.

**Required controls**

- Service-level objectives and error budgets are defined before adoption.
- Every critical service exports standardized golden signals:
  - traffic
  - errors
  - latency
  - saturation
- The deployment system can correlate metrics by:
  - service
  - version
  - region
  - tenant
  - endpoint
  - dependency
- Alerting and rollout analysis use the same metric definitions to avoid conflicting decisions.

**Policy examples**

- Abort if error rate exceeds stable by 1 percentage point for 5 minutes.
- Abort if p99 latency rises more than 20% against baseline for 10 minutes.
- Abort if checkout conversion falls below control by a statistically meaningful threshold.
- Pause, rather than roll back, if signals conflict or sample size is insufficient.
- Require manual approval before expansion beyond 25% for regulated or high-financial-impact paths.

**Staff-level concern**

Do not automate rollback solely on CPU or generic pod health. A release can be technically healthy while corrupting data or lowering successful transactions. Pair platform metrics with domain metrics.

**Main risk**

- Bad thresholds cause flapping, false rollbacks, or silent failures. Tune policies from historical incident data and review them regularly.

---

## 6. Infrastructure rollback

**Use when:** Infrastructure, networking, IAM, Kubernetes policies, routing, or configuration changes cause impact.

**Implementation notes**

- Store desired state in version control.
- Use reviewed pull requests, plan output, policy checks, and an immutable record of applied versions.
- Separate:
  - additive and reversible changes
  - destructive or state-moving changes
- Pin provider, module, and chart versions.
- Use environment promotion, not ad hoc production-only edits.
- Detect drift and document whether the rollback path restores declared state or retains a deliberate emergency change.

**Rollback approaches**

- **Declarative revert:** Revert the infrastructure commit and apply through CI.
- **Stateful recovery:** Restore a prior configuration only after checking external state.
- **Traffic-level reversal:** For routing, gateway, WAF, or mesh changes, restore the previous policy immediately while investigating.
- **Forward fix:** Use when reverse application would delete or corrupt resources.

**Critical warning**

Terraform or similar tooling does not guarantee a safe reverse operation. Reverting code may plan to replace, destroy, or recreate resources. Review the plan during an incident, especially for:

- databases
- persistent volumes
- IAM permissions
- DNS
- load balancers
- secrets
- message topics and subscriptions

**Main risk**

- “Rollback” can be more destructive than the original change. For critical stateful infrastructure, pre-approved recovery playbooks are safer than generic reversions.

---

## 7. Database rollback and data recovery

**Use when:** A release changes schema, data semantics, retention, permissions, or write behavior.

**Primary strategy: expand-contract**

1. Add new tables, columns, indexes, or APIs without removing existing behavior.
2. Deploy code compatible with old and new schema.
3. Backfill data using throttled, observable, restartable jobs.
4. Validate correctness and performance.
5. Switch reads and writes behind a flag if needed.
6. Retire old schema only after the rollback window has expired.

**Rules**

- Application code must be backward-compatible for at least one release cycle.
- Never couple deployment success to a long, blocking production migration.
- Make backfills idempotent, chunked, rate-limited, and resumable.
- Version event payloads and consumers. Old consumers must ignore unknown fields safely.
- Capture audit data for migrations that change business meaning.
- Test migration performance on realistic data volume, lock behavior, replication lag, and rollback conditions.

**When a bad write has reached production**

- Stop or fence the faulty writer.
- Identify affected records using release markers, timestamps, request IDs, event IDs, or audit trails.
- Reconcile from authoritative source-of-truth data.
- Use a reviewed, idempotent repair job with dry-run output.
- Validate sample records and aggregate counts before and after repair.
- Use point-in-time recovery only for severe scenarios where the recovery point and data loss are explicitly accepted.

**Avoid**

- Assuming `DOWN` migrations restore production data correctly.
- Restoring an entire database for a localized application defect.
- Running a repair script directly against all rows without a dry run, batching, and audit output.

---

## 8. Rollback of asynchronous and event-driven systems

**Use when:** Services publish events, process queues, run scheduled jobs, or call external providers.

**Controls**

- Idempotency keys for every externally visible or financially meaningful operation.
- Consumer version compatibility and schema evolution rules.
- Dead-letter queues with replay controls.
- Ability to pause consumers independently from APIs.
- Rate limits and circuit breakers for downstream systems.
- A clear owner and process for replaying messages.

**Rollback procedure**

1. Stop the faulty producer or consumer.
2. Prevent duplicate work by fencing consumers or acquiring a single active lease.
3. Capture message offsets, queue depth, and affected time window.
4. Return API traffic to the stable version if required.
5. Decide whether messages should be discarded, repaired, replayed, or compensated.
6. Replay in controlled batches with duplicate detection and monitoring.

**Main risk**

- Re-deploying old code does not retract events already published. Event replay is a data operation and needs its own approval and verification path.

---

## 9. Enterprise operating model

Every production service should have a short rollback runbook with:

- rollback owner and incident commander authority
- last-known-good release identifier
- exact command or pipeline action
- flag and traffic-routing locations
- health metrics and pass/fail thresholds
- database compatibility statement
- async processing and side-effect handling
- customer communication trigger
- verification checklist
- post-rollback investigation owner

**Set concrete recovery objectives**

| Measure | Example |
|---|---|
| Recovery time objective | Restore customer traffic within 10 minutes |
| Detection objective | Detect severe regression within 5 minutes |
| Rollback trigger | 5xx above 1% or p99 latency 25% above control |
| Rollback authority | On-call SRE or incident commander during active incident |
| Retention window | Keep previous environment deployable for 24 hours |
| Data recovery | Repair job available and tested before schema cutover |

The key Staff Engineer decision is choosing the smallest safe blast radius: disable a feature first, shift traffic next, revert a service version when safe, and treat data correction as a separate controlled operation.