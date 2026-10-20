## User request: Kubernetes interview questions
1. Architecture & Cluster Design
2. How do you design a multi-region Kubernetes architecture for disaster recovery?
3. What are the trade-offs between managed Kubernetes (EKS/AKS/GKE) and self-managed clusters?
4. How would you design Kubernetes for 10k+ pods in a cluster?
5. What are control plane bottlenecks in large Kubernetes clusters?
6. How does Kubernetes leader election work in the control plane?
### Networking
7. Explain the Kubernetes networking model and how CNI plugins work.
8. What are the differences between Calico, Flannel, and Cilium?
9. How do you debug pod-to-pod communication failures?
10. What happens internally when you create a Kubernetes Service?
11. How do you implement zero-trust networking in Kubernetes?
### Security
12. How do you secure secrets in Kubernetes beyond using Kubernetes Secrets?
13. Explain Pod Security Admission (PSA) vs PodSecurityPolicy (PSP).
14. How do you implement runtime security in Kubernetes?
15. What is the role of OPA Gatekeeper or Kyverno?
16. How do you secure container images before deployment?
### Performance & Scaling
17. How does the Cluster Autoscaler work internally?
18. What are the differences between HPA, VPA, and KEDA?
19. How would you optimize resource usage in a large Kubernetes cluster?
20. How do you handle noisy neighbor problems in Kubernetes?
21. How do you tune kube-scheduler performance?
### Observability
22. How do you implement centralized logging in Kubernetes?
23. What metrics are critical for monitoring Kubernetes cluster health?
24. How do you troubleshoot high CPU usage in kube-apiserver?
25. Explain how Prometheus integrates with Kubernetes.
26. How do you monitor etcd performance and health?
### CI/CD & GitOps
27. How do you implement GitOps with ArgoCD or Flux?
28. How do you perform blue-green deployments in Kubernetes?
29. How does canary deployment work in Kubernetes?
30. What are the benefits of progressive delivery?
31. How do you rollback deployments automatically on failure?
### Storage
32. Explain CSI (Container Storage Interface) architecture.
33. How does Kubernetes handle persistent volume provisioning?
34. What challenges occur when running databases in Kubernetes?
35. How do you migrate stateful workloads between clusters?
### Troubleshooting
37. A pod is stuck in ContainerCreating state. How would you debug it?
38. Pods are CrashLoopBackOff. What steps would you follow?
39. Why might kubectl commands become slow in a large cluster?
40. How do you debug DNS issues in Kubernetes?