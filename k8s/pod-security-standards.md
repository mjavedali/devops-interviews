# Kubernetes Pod Security Standards: From Namespace Labels to GitOps

A container can ask to run as root, use the host network, or mount directories from a node. Kubernetes needs a way to decide which of those requests are acceptable before the Pod starts.

**Pod Security Standards (PSS)** define the rules. **Pod Security Admission (PSA)** is the built-in admission controller that checks Pods against them. You select the policy through namespace labels, and Kubernetes evaluates new Pods in that namespace automatically.

PSA became stable in Kubernetes 1.25, the same release that removed PodSecurityPolicy (PSP). It provides predefined profiles rather than a direct replacement for every custom PSP rule.

The examples below target ordinary Linux containers. Windows workloads and workloads using user namespaces can have different, version-dependent rules. Use the policy documentation for your cluster version when evaluating those cases.

## Choose a profile based on the workload

PSS defines three profiles: **Privileged**, **Baseline**, and **Restricted**. They describe increasing restrictions, not environment names. A development namespace can use Restricted, and a production infrastructure component may need an exception.

| Profile | What it allows or restricts | Where it fits |
| --- | --- | --- |
| Privileged | Adds no PSS restrictions; other admission policies and permissions still apply | Infrastructure components with a justified need for host access |
| Baseline | Blocks privileged containers, host networking, host PID/IPC sharing, and `hostPath` volumes; limits added capabilities | Workloads that need a starting security boundary but are not yet Restricted-compatible |
| Restricted | Adds non-root execution, disabled privilege escalation, explicit seccomp configuration, dropped capabilities, and tighter volume restrictions | Application workloads that can meet these requirements |

Some CNI plugins, storage drivers, and node agents need elevated access. That does not mean every DaemonSet or monitoring agent needs the Privileged profile. Check the component's requirements and limit access to namespaces that permit it.

Restricted is a useful target for application namespaces. For ordinary Linux containers, it requires settings such as `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, a permitted seccomp profile, and dropping `ALL` Linux capabilities. It permits adding back `NET_BIND_SERVICE` when needed, rather than arbitrary capabilities.

Two details are easy to miss:

- **Baseline forbids `hostPath` volumes.** It does not provide an allowlist of permitted host paths.
- **Restricted does not require a read-only root filesystem.** `readOnlyRootFilesystem: true` is additional hardening you can apply yourself or enforce with another policy.

PSS also checks fields beyond this overview, including security settings on init and ephemeral containers. The [full standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) are the reference for exact requirements.

## Understand enforce, audit, and warn

Each namespace can configure three independent modes, with a different profile for each:

| Mode | What happens when its policy is violated |
| --- | --- |
| `enforce` | The violating Pod admission request is rejected |
| `audit` | Policy violations are attached to API audit events; this mode does not reject the request |
| `warn` | The API returns a warning to the client; this mode does not reject the request |

Audit annotations are useful only if API audit logging is configured and the events are retained somewhere you can inspect. They do not appear automatically in application logs.

The modes operate independently. Setting `warn=restricted` does not override a rejection from `enforce=baseline`.

## Create a namespace with Restricted enforcement

For a disposable lab, save this as `namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: psa-demo
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

Apply it and inspect the labels:

```bash
kubectl apply -f namespace.yaml
kubectl get namespace psa-demo --show-labels
```

This assumes Pod Security Admission is enabled and the request is not exempt. Labels do not install or enable an admission controller that the cluster administrator has disabled.

`latest` uses the policy version selected by the API server. For production, pin a supported policy version that you have tested, such as `v1.34` on a compatible cluster. Treat that as an example version, not a recommendation for every cluster. Review the pin during upgrades so policy changes are deliberate.

The same configuration can be applied to an existing namespace using `kubectl label --overwrite`, but review existing workloads before enabling enforcement.

## Deploy a workload that passes the policy

PSS validates the manifest; it does not fill in missing security settings. Developers still need to supply a compatible Pod template and an image that works with those settings.

Save this as `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: restricted-demo
  namespace: psa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: restricted-demo
  template:
    metadata:
      labels:
        app: restricted-demo
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: busybox:1.37.0
          command: ["sh", "-c", "sleep 86400"]
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          resources:
            requests:
              cpu: 10m
              memory: 16Mi
            limits:
              cpu: 100m
              memory: 64Mi
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

The Pod-level security context supplies a non-root identity and seccomp profile. Container-specific settings disable privilege escalation and drop capabilities. A numeric UID avoids relying on the image's default user being suitable.

The root filesystem is read-only, while `/tmp` remains writable through an `emptyDir` volume. `fsGroup` gives the process group access to that volume. For a real application, identify all required write paths and give them suitable volumes instead of making the whole filesystem writable.

Disabling service-account token mounting and declaring resource requests and limits are useful here, but neither is required by Restricted. The image tag is for the lab; use your reviewed image and immutable digest in production.

Run the example:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/restricted-demo -n psa-demo

# Confirm that the process runs with a non-root identity.
kubectl exec -n psa-demo deployment/restricted-demo -- id

# Confirm that the explicitly mounted scratch directory is writable.
kubectl exec -n psa-demo deployment/restricted-demo -- \
  sh -c 'echo "volume is writable" > /tmp/check && cat /tmp/check'
```

An admitted Pod can still fail at runtime because of image permissions, missing files, or incompatible application behavior. Passing PSA means the manifest met the selected policy, not that the application is healthy.

## Check that a privileged Pod is rejected

Save this intentionally non-compliant Pod as `privileged-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-demo
  namespace: psa-demo
spec:
  containers:
    - name: app
      image: busybox:1.37.0
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        privileged: true
```

Use a server-side dry run to submit it for admission without persisting it:

```bash
kubectl create --dry-run=server -f privileged-pod.yaml
```

Expect a rejection identifying violations of `restricted`, including the privileged container and missing required settings. The exact message depends on the Kubernetes and policy version.

### Why a Deployment can be created while its Pods are blocked

PSA enforcement applies to the resulting Pods, not directly to Deployment objects. Warning and audit modes also inspect supported workload templates to give earlier feedback.

A non-compliant Deployment can therefore be accepted while its ReplicaSet repeatedly fails to create Pods. If the Deployment exists but no Pods appear, check controller events:

```bash
kubectl get deployments,replicasets,pods -n psa-demo
kubectl describe replicasets -n psa-demo
kubectl get events -n psa-demo --sort-by=.metadata.creationTimestamp
```

Look for `FailedCreate` events mentioning PodSecurity. Changing CPU limits or adding nodes will not fix a Pod admission rejection.

## Roll out enforcement without breaking replacements

For an existing application namespace, start by identifying violations. One transition is to enforce Baseline while auditing and warning against Restricted:

```bash
# Replace payments with the namespace you have reviewed.
kubectl label namespace payments --overwrite \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

Even Baseline can block existing workload templates, so preview that change before applying it if the namespace currently allows more permissive workloads.

After fixing the templates, preview Restricted enforcement:

```bash
kubectl label namespace payments --overwrite --dry-run=server \
  pod-security.kubernetes.io/enforce=restricted
```

When an enforcement label changes, PSA checks existing Pods and can return warnings. It does not evict those Pods. However, replacements created during a rollout, scaling event, or node drain must pass the policy. Review dormant Jobs, CronJobs, and workloads currently scaled to zero too; checking running Pods alone will miss them.

Once the templates are ready, apply the change through your normal configuration workflow. If you pin policy versions, include the reviewed version labels as part of that change.

## Keep the policy in Git

Store the labeled Namespace manifest alongside your infrastructure configuration and let Argo CD, Flux, Terraform, or your namespace provisioning workflow manage it. With GitOps, create the namespace and apply its policy before syncing workloads into it.

Managing one labeled Namespace does not automatically secure every future namespace. Make these labels part of the namespace creation process, or configure appropriate cluster-wide defaults. Limit who can edit the policy labels, since someone who can weaken them can change the admission boundary.

Application templates should include compatible security contexts so each team starts from settings that work. Enforcement then catches omissions and unintended changes.

### Add a meaningful CI check

Validate the rendered output of Helm or Kustomize, including injected containers where your workflow supports that. A YAML syntax check does not evaluate Pod Security Standards.

For an integration check against a test cluster, extract the Pod template from the example Deployment and submit it as a Pod with a server-side dry run. This command requires `jq` and the labeled `psa-demo` namespace:

```bash
kubectl create --dry-run=client -f deployment.yaml -o json \
  | jq '{
      apiVersion: "v1",
      kind: "Pod",
      metadata: {
        name: "psa-ci-check",
        namespace: "psa-demo",
        labels: .spec.template.metadata.labels
      },
      spec: .spec.template.spec
    }' \
  | kubectl create --dry-run=server -f -
```

In a shell-based CI job, enable `set -o pipefail` so an earlier failure in the pipeline is not hidden. This example handles one Deployment. A general pipeline must process every rendered workload and preserve relevant template annotations, including those used by admission webhooks.

Running only a server-side dry run of the Deployment is insufficient to prove its Pods will pass `enforce`. Submitting a Pod exercises that admission check, though it still does not prove the container will start successfully.

For offline checks, tools such as Conftest or the Kyverno CLI need matching policies configured. Installing a tool alone does not make it reproduce your cluster's PSA behavior. Keep offline rules aligned with the policy version enforced in the cluster.

## Add custom rules where PSS stops

PSS covers predefined Pod security controls. It does not require approved registries, immutable image references, ownership labels, resource requests, or read-only root filesystems.

Use a policy engine such as Kyverno or OPA Gatekeeper for organization-specific requirements. Keep those rules explicit so teams know whether a rejection came from PSA or an additional policy.

## A concise interview answer

> Pod Security Standards define three profiles: Privileged, Baseline, and Restricted. The built-in Pod Security Admission controller applies them using namespace labels. I aim for Restricted in application namespaces, introduce changes through audit and warning modes, and manage the labels through GitOps or infrastructure code. Developers still need compatible security contexts because PSA validates manifests rather than fixing them. I also test the generated Pods in CI and use additional policies for requirements such as approved registries or read-only root filesystems.

## References

- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Enforce standards with namespace labels](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- [Plan a Pod Security rollout](https://kubernetes.io/docs/setup/best-practices/enforcing-pod-security-standards/)
- [Configure a Pod or container security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
