# boostflow-thesis-config

Kubernetes manifests and platform configuration for the BoostFlow thesis project. This repository is the GitOps source of truth - [ArgoCD](https://argo-cd.readthedocs.io/) watches it and syncs changes to the cluster automatically.

For the CI/CD pipeline and application source code, see [boostflow-thesis](https://github.com/KDim67/boostflow-thesis).

## How Deployment Works

1. The Jenkins CI pipeline in `boostflow-thesis` builds, scans, signs, and pushes a Docker image to GHCR
2. Jenkins clones this repo, updates the image digest in `app-deployment.yaml`, commits, and pushes
3. ArgoCD detects the change and syncs the new manifests to the cluster
4. The CI pipeline verifies the rollout is healthy before proceeding to DAST scanning

## Repository Structure

```
.
├── app-deployment.yaml              # Application Deployment (2 replicas, distroless, non-root)
├── app-service.yaml                 # ClusterIP Service
├── app-ingress.yaml                 # Ingress with TLS
├── external-services.yaml           # ExternalName services
├── hpa.yaml                         # Horizontal Pod Autoscaler
├── limits.yaml                      # LimitRange
├── namespace.yaml                   # Namespaces (boostflow, monitoring, security) + ServiceAccounts
├── network-policies.yaml            # NetworkPolicies for application namespace
├── platform-ingresses.yaml          # Ingress rules for platform services
├── rbac.yaml                        # RBAC configuration
├── vault-auth.yaml                  # Vault authentication
├── vault-static-secret-app.yaml     # VSO: application secrets from Vault
├── vault-static-secret-ghcr.yaml    # VSO: GHCR pull secret from Vault
├── vault-static-secret-minio.yaml   # VSO: MinIO secrets from Vault
├── vso-vault-network-policies.yaml  # NetworkPolicies for Vault Secrets Operator
├── minio-statefulset.yaml           # MinIO StatefulSet
├── minio-service.yaml               # MinIO Service
│
├── argocd/                          # ArgoCD Application CRs (App-of-Apps)
│   ├── root-app.yaml                # Root Application - syncs this directory
│   ├── project-boostflow.yaml       # AppProject definition
│   ├── app-boostflow.yaml           # Application: main app manifests
│   ├── app-monitoring.yaml          # Application: monitoring stack
│   ├── app-security.yaml            # Application: security components
│   ├── app-compliance.yaml          # Application: compliance checks
│   ├── app-backup.yaml              # Application: backup jobs
│   ├── app-cert-manager.yaml        # Application: cert-manager
│   └── app-neuvector.yaml           # Application: NeuVector
│
├── monitoring/                      # Observability stack
│   ├── prometheus-config.yaml       # Prometheus configuration
│   ├── prometheus-deployment.yaml   # Prometheus Deployment
│   ├── grafana-deployment.yaml      # Grafana Deployment
│   ├── grafana-dashboards.yaml      # Grafana dashboard ConfigMaps
│   ├── elasticsearch-statefulset.yaml
│   ├── logstash-deployment.yaml
│   ├── filebeat-daemonset.yaml
│   ├── kibana-deployment.yaml
│   ├── jaeger-deployment.yaml       # Jaeger distributed tracing
│   ├── vault-static-secret-basic-auth.yaml
│   └── vault-static-secret-grafana.yaml
│
├── security/                        # Runtime security
│   ├── falco-daemonset.yaml         # Falco runtime threat detection
│   ├── falco-rules.yaml             # Custom Falco rules
│   ├── neuvector-security-rules.yaml # NeuVector security policies
│   └── cluster-issuer.yaml          # cert-manager ClusterIssuer
│
├── compliance/                      # Compliance auditing
│   └── kube-bench-cronjob.yaml      # CIS Kubernetes Benchmark (scheduled)
│
└── backup/                          # Backup jobs
    ├── minio-backup-cronjob.yaml    # MinIO data backup CronJob
    ├── network-policies.yaml        # Backup namespace NetworkPolicies
    └── vault-auth.yaml              # Vault auth for backup namespace
```

## ArgoCD App-of-Apps

The `argocd/root-app.yaml` is the root Application that recursively syncs the `argocd/` directory. Each Application CR in that directory points to a subdirectory or set of manifests in this repo, creating an App-of-Apps hierarchy:

| Application | Source Path | Namespace | Description |
|---|---|---|---|
| `app-boostflow` | Root manifests | `boostflow` | Application deployment, services, ingress, secrets, networking |
| `app-monitoring` | `monitoring/` | `monitoring` | Prometheus, Grafana, EFK stack, Jaeger |
| `app-security` | `security/` | `security` | Falco, NeuVector, cert-manager |
| `app-compliance` | `compliance/` | `compliance` | kube-bench CIS benchmark |
| `app-backup` | `backup/` | `backup` | MinIO backup CronJob |
| `app-cert-manager` | Helm chart | `cert-manager` | TLS certificate management |
| `app-neuvector` | Helm chart | `cattle-neuvector-system` | Container security platform |

All applications use automated sync with pruning and self-healing enabled.

## Namespaces

| Namespace | Pod Security Standard | Purpose |
|---|---|---|
| `boostflow` | `restricted` | Application workloads (runs as non-root, no privilege escalation) |
| `monitoring` | *unenforced* | Observability stack (EFK requires host mounts to collect node logs) |
| `security` | *unenforced* | Runtime security (Falco requires eBPF/host access; NeuVector requires deep packet inspection) |
| `compliance` | `privileged` | CIS benchmarking (kube-bench requires host filesystem access to audit node config) |
| `backup` | *unenforced* | MinIO data backup CronJobs |

*Note: Where PSS is not strictly enforced (`restricted`/`privileged`), it is due to the low-level host access requirements of the respective agents (e.g., Falco, Filebeat).*

Default ServiceAccounts in `boostflow`, `monitoring`, and `security` have `automountServiceAccountToken: false`.

## Platform Components & Tools

This repository manages a comprehensive DevSecOps Kubernetes platform. The following tools are deployed and configured via GitOps:

### Continuous Deployment (GitOps)
- **[ArgoCD](https://argo-cd.readthedocs.io/)**: Synchronizes manifests from this config repository to the cluster, managing the entire deployment lifecycle via the App-of-Apps pattern.

### Observability & Monitoring
- **[Prometheus](https://prometheus.io/) & [Grafana](https://grafana.com/)**: Scrapes metrics from the BoostFlow application and cluster components. Grafana dashboards provide visibility into runtime performance and pipeline health.
- **EFK Stack ([Elasticsearch](https://www.elastic.co/elasticsearch/), [Filebeat](https://www.elastic.co/beats/filebeat), [Logstash](https://www.elastic.co/logstash/), [Kibana](https://www.elastic.co/kibana/))**: Collects application and system logs, aggregating them into Kibana for auditing and debugging.
- **[Jaeger](https://www.jaegertracing.io/)**: Traces Next.js API requests and backend services to pinpoint performance bottlenecks.

### Security & Compliance
- **[Falco](https://falco.org/)**: Deployed as a DaemonSet to detect anomalous runtime activity (e.g., unexpected shell executions in containers) using custom thesis-specific rules.
- **[NeuVector](https://neuvector.com/)**: Enforces zero-trust network policies, preventing unauthorized lateral movement and providing runtime container protection.
- **[kube-bench](https://github.com/aquasecurity/kube-bench)**: Runs as a scheduled CronJob to audit cluster compliance against CIS Kubernetes benchmarks, with reports fetched and verified by the Jenkins CI pipeline.
- **[Vault Secrets Operator](https://developer.hashicorp.com/vault/docs/platform/k8s/vso)**: Dynamically injects sensitive API keys (Firebase, Gemini, OAuth) into the application namespace without exposing them in Git.
- **[cert-manager](https://cert-manager.io/)**: Automatically provisions and manages TLS certificates for secure ingress routing to the BoostFlow web application.
