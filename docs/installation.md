# Installation & Bootstrap Guide

This document describes the **end-to-end bootstrap flow** for deploying the observability stack using **Argo CD and GitOps**.

It assumes a **pre-existing Kubernetes cluster** and focuses exclusively on the **GitOps control plane and platform reconciliation**.

---

## Prerequisites

- Kubernetes cluster (tested with **AWS EKS**)
- `kubectl` configured with **cluster-admin** access
- Git installed
- (Optional) `argocd` CLI for local interaction

---

## 1. Clone the Repository

Clone the repository locally:

```bash
git clone https://github.com/LaurisNeimanis/gitops-observability-stack.git
cd gitops-observability-stack
```

This repository is the **single source of truth** for the observability platform state.

---

## 2. Install Argo CD (Bootstrap Layer)

Argo CD is installed using a **Kustomize-based bootstrap** located under `gitops/bootstrap`.

Apply it once:

```bash
kubectl apply -k gitops/bootstrap/argocd
```

Verify that Argo CD pods are running:

```bash
kubectl get pods -n argocd
```

Wait until all components are in `Running` state before proceeding.

---

## 3. (Optional) Access Argo CD UI & CLI

This step is **optional** and intended for **local inspection and demo purposes**.

### Port-forward Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Access the UI at:

```
https://localhost:8080
```

### Login Credentials

**Username**
```
admin
```

**Password**
```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### (Optional) CLI Login

```bash
argocd login localhost:8080
```

---

## 4. Install Argo CD Projects (Required)

Before deploying any applications, the **Argo CD Projects** must be created.

These define:

- allowed source repositories
- allowed destination namespaces
- cluster-level permissions (CRDs, RBAC)

Apply the projects manifest:

```bash
kubectl apply -k gitops/argo/projects
```

This step is **mandatory**.
Without it, subsequent applications will fail authorization and sync.

---

## 5. Bootstrap the Platform (GitOps Entry Point)

Apply the **root application**, which uses the **App-of-Apps** pattern:

```bash
kubectl apply -f gitops/argo/root-application.yaml
```

From this point onward, **Argo CD takes full control**.

It will reconcile, in order:

- Prometheus Operator CRDs (sync-wave `-1`)
- Observability platform components
- Custom alerting rules

No further manual `kubectl apply` commands are required.

---

## 6. Reconciliation Behavior

Once bootstrapped, Argo CD enforces:

- Automated sync
- Self-healing
- Controlled ordering via `sync-wave`
- Explicit CRD lifecycle management

This ensures **deterministic and reproducible platform state**.

---

## 7. Configure Grafana Admin Credentials (Required)

Grafana is configured to read admin credentials from an **externally managed Kubernetes Secret**.

The secret **must exist before Grafana can be accessed**:

```bash
kubectl -n observability create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='CHANGE_ME'
```

Grafana access will not be available until this secret exists.

---

## 8. Configure Alertmanager Slack Webhooks (Required)

Alertmanager requires **valid Slack Incoming Webhook URLs** to deliver notifications.

The following files must be updated with real webhook values:

- `gitops/apps/platform/observability/kube-prometheus-stack/values/values.yaml`
- `gitops/apps/platform/observability/kube-prometheus-stack/values/values-dev.yaml`

Locate the `slack_configs` section and replace the placeholder URLs:

```yaml
slack_configs:
  - api_url: "https://hooks.slack.com/services/..."
```

After updating the files, **commit and push** the changes so Argo CD can reconcile them.

---

## 9. (Optional) Access Platform Components

These steps are **not required for installation**, but useful for validation and demos.

### Grafana

Grafana uses the **externally provided admin credentials** created earlier.

- **Username:** `admin`
- **Password:** value defined in the `grafana-admin` Secret

Port-forward Grafana:

```bash
kubectl -n observability port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Access:

```
http://localhost:3000
```

---

### Alertmanager

```bash
kubectl -n observability port-forward svc/kube-prometheus-stack-alertmanager 9093:9093
```

Access:

```
http://localhost:9093
```

---

## Notes & Design Considerations

- Argo CD Projects are a **hard prerequisite** for application sync
- Grafana admin credentials are **provided externally by design**
- Alertmanager requires **explicit notification configuration**
- CRDs are managed explicitly and **not pruned**
- All applications use automated sync with self-healing enabled
- Namespace creation is handled automatically by Argo CD
- Infrastructure dependencies (S3, IAM, StorageClasses) are **out of scope**

---

## Summary

After completing these steps:

- Argo CD becomes the **authoritative control plane**
- The observability stack is fully GitOps-managed
- Dashboards and alerting become available once all required runtime configuration is provided
- All future changes are performed via Git commits

This bootstrap flow is designed to be:

- minimal
- deterministic
- production-aligned

