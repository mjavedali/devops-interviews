# I have 100s of pods, I want to ensure I dont run these as a root user

To ensure containers do **not run as root**, enforce it at three layers:

1. Image
2. Pod specification
3. Cluster admission policy

A Dockerfile `USER` line helps, but it is not enough. The Pod spec can override it unless policy blocks that override.

## Required Pod security settings

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
  namespace: prod
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: payments
          image: registry.company.com/payments:v1.4.2
          ports:
            - containerPort: 8080

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            privileged: false
            capabilities:
              drop:
                - ALL

          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}
```

## What each setting protects

| Setting | Why it matters |
|---|---|
| `runAsNonRoot: true` | Kubernetes refuses to start the container if it resolves to UID `0`. |
| `runAsUser: 10001` | Explicitly runs the process as a non-root UID. |
| `runAsGroup: 10001` | Sets a non-root primary group. |
| `fsGroup: 10001` | Gives the process group access to mounted volumes where required. |
| `allowPrivilegeEscalation: false` | Blocks gaining privileges through `setuid`, `setgid`, and similar paths. |
| `privileged: false` | Prevents host-level privileged execution. |
| `capabilities.drop: [ALL]` | Removes Linux capabilities such as network administration and kernel module access. |
| `readOnlyRootFilesystem: true` | Limits persistence and writes to the container filesystem. |
| `seccompProfile: RuntimeDefault` | Restricts available Linux system calls. |

## Build the image correctly

Example Dockerfile:

```dockerfile
FROM gcr.io/distroless/java17-debian12:nonroot

WORKDIR /app
COPY --chown=10001:10001 build/app.jar /app/app.jar

USER 10001:10001

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

For a Debian or Alpine base:

```dockerfile
RUN addgroup --gid 10001 appgroup \
 && adduser --uid 10001 --gid 10001 --disabled-password appuser

WORKDIR /app
COPY --chown=10001:10001 . /app
USER 10001:10001
```

Do not rely only on a username such as `USER appuser`. Use a numeric UID, especially in Kubernetes, because the user may not exist consistently across images or mounted volumes.

## Common failures after enforcing non-root

| Failure | Cause | Fix |
|---|---|---|
| Container cannot write logs/files | App writes under `/`, `/var`, or `/app` | Write to stdout, or mount a writable `emptyDir` |
| Permission denied on PVC | Volume ownership differs from container UID/GID | Use `fsGroup`, provision correct ownership, or a controlled init step |
| App tries to bind port 80/443 | Non-root normally cannot bind ports below 1024 | Run app on `8080`; Service/Ingress exposes `80/443` |
| `runAsNonRoot` fails at startup | Image defaults to root and no non-root UID is configured | Set `USER` in image and `runAsUser` in workload |
| Package/install command fails | Container tries to mutate filesystem at runtime | Build dependencies into the image |
| Java/Node/Python cache write failure | Runtime writes home or temp data | Set writable `/tmp` or application cache mount |

## Use an init container only when needed

Some volume preparation requires changing permissions. Avoid making the primary application container root.

```yaml
initContainers:
  - name: init-volume-permissions
    image: busybox:1.36
    command: ["sh", "-c", "chown -R 10001:10001 /data"]
    securityContext:
      runAsUser: 0
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
        add: ["CHOWN"]
    volumeMounts:
      - name: data
        mountPath: /data
```

Treat root init containers as an exception:

- document why `fsGroup` cannot solve it
- grant only the required capability
- use a pinned image
- limit its volume mounts
- prevent network access if possible
- remove it once storage provisioning supports correct ownership

## Enforce across all workloads

### Pod Security Admission

Label production namespaces:

```bash
kubectl label namespace prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

`restricted` blocks common unsafe settings, including privileged containers and privilege escalation. It does not replace your own UID rules in every case.

### Policy-as-code

Use Kyverno or OPA Gatekeeper to require:

- `runAsNonRoot: true`
- non-zero `runAsUser`
- `allowPrivilegeEscalation: false`
- `privileged: false`
- dropped `ALL` capabilities
- `RuntimeDefault` seccomp
- no host namespaces
- no hostPath volumes, except approved cases
- read-only root filesystem where the application supports it

Example Kyverno validation intent:

```text
Deny workload if any application or init container:
- has UID 0
- lacks runAsNonRoot: true
- allows privilege escalation
- is privileged
- adds dangerous Linux capabilities
- lacks RuntimeDefault or Localhost seccomp
```

Apply policies in stages:

```text
Audit → report violating workloads → fix images/manifests → enforce in non-prod → enforce in production
```

Do not enable enforcement blindly on shared clusters. System namespaces, CSI drivers, CNI components, and node-level agents may legitimately need elevated privileges.

## Verify the effective identity

```bash
kubectl -n prod exec deploy/payments -- id
```

Expected:

```text
uid=10001 gid=10001 groups=10001
```

Check the effective Pod settings:

```bash
kubectl -n prod get pod <pod-name> -o yaml
kubectl -n prod describe pod <pod-name>
```

The enterprise standard is: **application containers run as a fixed non-zero UID, cannot elevate privileges, have no Linux capabilities, use RuntimeDefault seccomp, and are prevented from bypassing these rules by admission policy.**