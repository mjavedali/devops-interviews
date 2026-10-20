# Kubernetes Security Recommendations

## Kubernetes Security Recommendations

### 1. Cluster Access

```text
Use RBAC with least privilege
Integrate with SSO/OIDC
Enable MFA
Avoid cluster-admin for developers
Separate admin and application access
```

---

### 2. Namespace Isolation

```text
Separate namespaces per team/application
Separate dev, QA, and production
Apply ResourceQuota and LimitRange
```

---

### 3. Pod Security

Run containers securely.

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

Avoid:

```text
Privileged containers
hostNetwork
hostPID
hostIPC
hostPath volumes (unless absolutely required)
Running as root
```

---

### 4. Image Security

```text
Use private container registry
Scan images (Trivy, Prisma, Aqua)
Use minimal/distroless images
Pin image versions
Sign images (Cosign)
Do not use :latest tag
```

---

### 5. Secrets

```text
Never store secrets in Git
Use AWS Secrets Manager / Azure Key Vault / OCI Vault / HashiCorp Vault
Use External Secrets Operator or CSI Driver
Rotate secrets regularly
Encrypt secrets at rest
```

---

### 6. Network Security

```text
Default deny NetworkPolicy
Allow only required communication
Use private clusters if possible
Expose applications through Ingress/API Gateway
Enable TLS everywhere
```

---

### 7. Admission Control

Use:

```text
Kyverno
OPA Gatekeeper
ValidatingAdmissionPolicy
```

Enforce:

```text
No privileged pods
No latest image
Mandatory resource requests/limits
Mandatory labels
Approved registries only
```

---

### 8. Runtime Security

Use:

```text
Falco
Cilium Tetragon
Microsoft Defender
Prisma
Sysdig
```

Detect:

```text
Container escape
Unexpected shell
Privilege escalation
Suspicious processes
```

---

### 9. Node Security

```text
Patch nodes regularly
Use hardened OS images
Disable unnecessary SSH access
Use dedicated node pools for sensitive workloads
Keep Kubernetes version updated
```

---

### 10. Monitoring & Audit

Enable:

```text
Kubernetes Audit Logs
Prometheus
Grafana
Centralized logging
Alerting
```

Monitor:

```text
Failed authentication
RBAC changes
Pod restarts
Image pull failures
Node status
```

---

### 11. Backup & Recovery

```text
Backup etcd
Backup persistent volumes
Test restore regularly
Have DR procedures
```

---

> Kubernetes security is built in layers: secure access, secure workloads, secure networking, secure supply chain, runtime protection, and continuous monitoring. No single control is sufficient on its own.