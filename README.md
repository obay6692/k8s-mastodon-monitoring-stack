# IKT210 Final Project — Full Kubernetes Platform with Mastodon, GitOps & Monitoring

End-to-end Kubernetes platform built as the final project for **IKT210 — Cloud Infrastructure** at the University of Agder.

It combines a real-world social-network application (**Mastodon**), an LLM frontend (**Open WebUI**), a complete **GitOps CI/CD stack**, and a production-grade **observability stack**, all deployed declaratively via Argo CD.

## Stacks

### `apps/` — Workloads
| App | Description |
|-----|-------------|
| **Mastodon** | Self-hosted federated social network. Includes web + nginx, PostgreSQL StatefulSet, Redis StatefulSet, PVCs, secrets, namespace, and a ConfigMap. |
| **Open WebUI** | Browser interface for local LLMs (Ollama-compatible). Deployment, Service, PVC, and Argo CD Application. |

### `ci-stack/` — GitOps / CI/CD
| Component | Role |
|-----------|------|
| **Argo CD** (`argo-kustom/`) | Continuously syncs the cluster to this Git repository |
| **Argo CD Image Updater** (`image-updater/`) | Watches container registries and bumps image tags in Git automatically |
| **Sealed Secrets** (`sealed-secrets/`) | Encrypts Kubernetes Secrets so they can be safely committed to Git |

### `monitoring-stack/` — Observability
| Component | Role |
|-----------|------|
| **Prometheus Operator** | Manages Prometheus, Alertmanager, and ServiceMonitor CRDs |
| **Prometheus** | Metrics collection and storage |
| **Alertmanager** | Routes alert notifications |
| **Grafana** | Dashboards and visualization |
| **Loki + Promtail** | Log aggregation and shipping |
| **Node Exporter** | Host-level metrics (CPU, memory, disk, network) |
| **Metrics Server** | Resource metrics for `kubectl top` and HPA |
| **CRDs** (`crds/`) | All Prometheus Operator CRDs (Alertmanager, ServiceMonitor, PodMonitor, etc.) |

Each component is its own Kustomize bundle and has an `application.yaml` so Argo CD manages it independently.

## Deployment order

1. **Install Argo CD** first — it bootstraps everything else:
   ```bash
   kubectl apply -k ci-stack/argo-kustom
   ```
2. **Apply the CI stack** (sealed-secrets must be ready before any sealed secret in Mastodon will decrypt):
   ```bash
   kubectl apply -k ci-stack
   ```
3. **Apply the monitoring stack CRDs** first, then the stack itself:
   ```bash
   kubectl apply -k monitoring-stack/crds
   kubectl apply -k monitoring-stack
   ```
4. **Apply the application Argo CD definitions** — Argo CD then deploys Mastodon and Open WebUI:
   ```bash
   kubectl apply -f apps/Mastodon/
   kubectl apply -f apps/open-webui/application.yaml
   ```

Once all `Applications` show `Synced` and `Healthy` in the Argo CD UI, the platform is up.

## Access

- **Argo CD UI** — `kubectl -n argocd port-forward svc/argocd-server 8080:443`
- **Grafana** — `kubectl -n monitoring port-forward svc/grafana 3000:3000`
- **Mastodon** — exposed via the Nginx Service in `apps/Mastodon/`
- **Open WebUI** — exposed via its Service in `apps/open-webui/`

## Architecture highlights

- **GitOps**: every change goes through Git. Argo CD pulls and reconciles. No `kubectl apply` after the bootstrap.
- **Encrypted secrets in Git**: Mastodon's database credentials, Mastodon secrets, and other sensitive values are stored as SealedSecrets, safe to commit.
- **Automated image updates**: Argo CD Image Updater promotes new container image tags by writing back to this repo, closing the GitOps loop.
- **Full observability**: metrics (Prometheus + Node Exporter), dashboards (Grafana), logs (Loki + Promtail), and alerting (Alertmanager).

## Cleanup

```bash
kubectl delete -k monitoring-stack
kubectl delete -k ci-stack
kubectl delete -f apps/Mastodon/
kubectl delete -f apps/open-webui/application.yaml
```

PVCs (Mastodon, PostgreSQL, Grafana, Loki, Prometheus) survive on purpose — delete them manually if you intend to wipe state.

## Notes

- Secrets in this repo are intended to be **SealedSecrets**. Any plain `secrets.yaml` files are placeholders or course-graded samples and should be re-sealed for a real deployment.
- Designed to run on the OpenStack-based cluster provisioned in [k8s-cluster-terraform-setup](https://github.com/obay6692/k8s-cluster-terraform-setup).
