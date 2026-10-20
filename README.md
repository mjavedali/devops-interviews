# DevOps Interview

Topic-wise notes, interview questions, and practice exercises for DevOps and SRE interview preparation.

Use this repository to revise core concepts, practice explaining technical decisions, and work through operational scenarios.

## Topics

The folders below organize the repository. The focus areas describe a suggested scope for each topic.

| Folder | Focus areas |
| --- | --- |
| [Ansible](./ansible/) | Inventories, playbooks, roles, variables, idempotency, and configuration management |
| [Argo CD](./argo-cd/) | GitOps, application sync, drift detection, deployment workflows, and rollback |
| [AWS](./aws/) | IAM, networking, compute, storage, availability, and troubleshooting |
| [Istio](./istio/) | Traffic management, service-to-service security, observability, and service mesh troubleshooting |
| [Kubernetes Upgrades](./K8s-upgrade/) | Upgrade planning, compatibility checks, node draining, validation, and recovery |
| [Kubernetes](./k8s/) | Workloads, scheduling, networking, storage, RBAC, and cluster troubleshooting |
| [KEDA](./keda/) | Event-driven autoscaling, triggers, ScaledObjects, and interaction with HPA |
| [Linux](./linux/) | Processes, permissions, networking, filesystems, services, and troubleshooting |
| [Miscellaneous](./misc/) | Cross-topic questions, interview notes, and supporting concepts |
| [Monitoring](./monitoring/) | Metrics, logs, traces, alerting, dashboards, and incident investigation |
| [Python](./python/) | Automation scripts, file handling, APIs, error handling, and coding exercises |

## How to Use This Repository

1. Pick a topic and review the fundamentals.
2. Explain each answer aloud in your own words.
3. Practice commands and exercises in a disposable lab.
4. Work through troubleshooting scenarios and explain how you would verify a fix.
5. Revisit weak areas and add what you learned.

For experienced roles, connect answers to real systems: what failed, what evidence you collected, which tradeoffs you considered, and how you prevented recurrence.

## Suggested Structure Within Each Topic

Use the same layout across topics as the content grows:

| File or folder | Purpose |
| --- | --- |
| `README.md` | Topic overview and links to the material |
| `questions.md` | Interview questions with concise answers |
| `scenarios.md` | Troubleshooting and design scenarios |
| `commands.md` | Useful commands with explanations |
| `labs/` | Hands-on exercises, scripts, and sample configurations |

Create only the files that a topic needs.

## Answer Format

Keep answers short enough to explain in an interview, with extra detail available for follow-up questions.

- **Question:** What is being asked?
- **Short answer:** Give the main point in two or three sentences.
- **Example:** Add a useful command, configuration, or real situation.
- **Tradeoffs:** Explain when the approach works and where it has limits.
- **Follow-up:** Add one or two likely follow-up questions.

For troubleshooting scenarios, include symptoms, checks, findings, the fix, verification, and prevention. Keep assumptions separate from confirmed findings.

## Practice Safely

- Use disposable environments for labs and review commands before running them.
- Keep credentials, tokens, private keys, kubeconfig files, and sensitive infrastructure details out of commits.
- Record tool versions when an answer or exercise depends on version-specific behavior.
- Use official documentation to verify commands and behavior before applying them to a live system.

## Contributing

Corrections, clearer explanations, new questions, and reproducible labs are welcome.

1. Put the material in the relevant topic folder.
2. Check for duplicate questions.
3. Explain commands and expected results.
4. Add documentation references where useful.
5. Submit a pull request with a short description of the change.
