# local-flux Cluster Design

**Date:** 2026-03-27
**Repo:** https://github.com/twallac10/local-flux.git
**Status:** Approved

## Overview

A local Kind cluster running Kubernetes 1.34.x with Flux for GitOps, Kyverno for policy governance (with reporting server and full metrics), and a complete observability stack (OTel Operator + Collector, Prometheus, Grafana). The cluster is structured as a classic Flux monorepo with explicit dependency-ordered Kustomizations.

---

## 1. Repository Structure

```
local-flux/
├── clusters/
│   └── local-flux/                      # Flux entry point
│       ├── flux-system/
│       │   ├── flux.yaml                # HelmRepository + HelmRelease for flux2 v2.5.1
│       │   └── kustomization.yaml
│       ├── infrastructure.yaml          # Flux Kustomization → ./infrastructure/controllers
│       ├── infrastructure-configs.yaml  # dependsOn: infrastructure
│       └── apps.yaml                    # dependsOn: infrastructure-configs
│
├── infrastructure/
│   ├── controllers/                     # Helm releases for operators & CRDs
│   │   ├── cert-manager/
│   │   ├── kyverno/
│   │   ├── otel-operator/
│   │   └── prometheus-stack/
│   └── configs/                         # CRs that depend on controllers being ready
│       ├── kyverno-reporting/
│       ├── kyverno-policies/
│       ├── otel-collector/
│       └── grafana-dashboards/
│
├── apps/
│   ├── namespaces.yaml                  # all app Namespace definitions
│   ├── team-a/
│   └── team-b/
│
├── helmfile.yaml                        # bootstraps Flux 2.5.1
├── bootstrap.yaml                       # GitRepository + root Kustomization
├── kind-config.yaml                     # Kind cluster config
└── taskfile.yaml                        # task build / delete
```

---

## 2. Kind Cluster

**File:** `kind-config.yaml`

- **Nodes:** 1 control-plane + 3 workers
- **Image:** `kindest/node:v1.34.x` (latest patch at time of setup)
- **Kubernetes version:** 1.34.x
- **Extra port mappings:** 80 and 443 on control-plane for ingress

**Namespaces created at bootstrap (not via Flux):**

| Namespace | Purpose |
|---|---|
| `flux-system` | Flux controllers |
| `policy` | Kyverno + reporting server |
| `observability` | OTel operator, collector, Prometheus, Grafana |
| `cert-manager` | cert-manager controllers |

---

## 3. Bootstrap (Taskfile + Helmfile)

**File:** `taskfile.yaml`

```
task build_cluster   → kind create cluster --name local-flux --config kind-config.yaml
task install_flux    → helmfile apply + create flux-git-repo secret via gh CLI
task build           → build_cluster → install_flux
task delete_cluster  → kind delete cluster --name local-flux
```

**File:** `helmfile.yaml`
- Installs `fluxcd-community/flux2` chart version pinned to `2.5.1` into `flux-system`
- `imageAutomationController: false`, `imageReflectionController: false`
- Post-sync hook: `kubectl apply -f bootstrap.yaml`

**File:** `bootstrap.yaml`
- `GitRepository` pointing at `https://github.com/twallac10/local-flux.git` (main branch)
- Root `Kustomization` pointing at `clusters/local-flux/`
- Auth via `flux-git-repo` Secret (username + GitHub PAT from `gh auth token`)

---

## 4. Flux Kustomization Dependency Chain

```
flux-system
  └── infrastructure                     (controllers layer)
        ├── cert-manager
        ├── kyverno                       dependsOn: cert-manager
        ├── kyverno-policies-chart        dependsOn: kyverno
        ├── otel-operator                 dependsOn: cert-manager
        └── prometheus-stack             dependsOn: otel-operator
              └── infrastructure-configs  (configs layer)
                    ├── kyverno-policies  (custom ClusterPolicies)
                    ├── kyverno-reporting dependsOn: kyverno-policies
                    ├── otel-collector    dependsOn: otel-operator
                    └── grafana-dashboards dependsOn: prometheus-stack
                          └── apps
                                ├── namespaces
                                ├── team-a
                                └── team-b
```

Each Flux `Kustomization` uses `healthChecks` on its Deployments/HelmReleases before allowing downstream layers to reconcile.

---

## 5. Kyverno Stack

### Controllers (`infrastructure/controllers/kyverno/`)

- `kyverno` HelmRelease — chart from `https://kyverno.github.io/kyverno/`, v3.x latest, namespace `policy`
- `kyverno-policies` HelmRelease — same repo, same version, `dependsOn: kyverno`

**Kyverno Helm values — all metric services enabled:**

```yaml
admissionController:
  replicas: 1
  metricsService:
    create: true
backgroundController:
  metricsService:
    create: true
cleanupController:
  metricsService:
    create: true
reportsController:
  metricsService:
    create: true
reportsServer:
  enabled: true
  metricsService:
    create: true
features:
  policyExceptions:
    enabled: true
```

### Kyverno Metric Endpoints

| Component | Port | Metrics |
|---|---|---|
| Admission Controller | `8000` | policy decisions, webhook latency |
| Background Controller | `8000` | background scan results |
| Cleanup Controller | `8000` | cleanup policy execution |
| Reports Controller | `8000` | report generation |
| Reports Server | `8080` | policy report query metrics |

**Note on bootstrap ordering:** Kyverno is installed by Flux (not helmfile) — Flux controllers deploy before policies are enforced. This is safe because: (1) the only `enforce` policy is `disallow-latest-tag` and Flux uses pinned image digests, not `:latest`; (2) `require-labels` is `audit` only and never blocks. A `PolicyException` for `flux-system` covers remaining edge cases.

### Configs (`infrastructure/configs/kyverno-policies/`)

- Pod Security Standards (baseline + restricted) via `kyverno-policies` chart values
- Custom `ClusterPolicy`: `require-labels` — audit: all pods must have `app` and `owner` labels
- Custom `ClusterPolicy`: `disallow-latest-tag` — enforce: block images using `:latest` tag
- `PolicyException` for `flux-system` namespace

---

## 6. OTel Operator + Collector

### Controller (`infrastructure/controllers/otel-operator/`)

- `opentelemetry-operator` HelmRelease — `open-telemetry/opentelemetry-operator` chart, `observability` namespace
- `dependsOn: cert-manager` (webhook TLS requirement)

### Config (`infrastructure/configs/otel-collector/`)

Single `OpenTelemetryCollector` CR:
- `mode: deployment`, 1 replica
- Namespace: `observability`

**Full scrape target list (prometheus receiver):**

| Job | Namespace | Port | Component |
|---|---|---|---|
| `flux-source-controller` | `flux-system` | `8080` | source reconciliation |
| `flux-kustomize-controller` | `flux-system` | `8080` | kustomization reconciliation |
| `flux-helm-controller` | `flux-system` | `8080` | helm release reconciliation |
| `flux-notification-controller` | `flux-system` | `8080` | alerts & receivers |
| `kyverno-admission-controller` | `policy` | `8000` | policy decisions |
| `kyverno-background-controller` | `policy` | `8000` | background scans |
| `kyverno-cleanup-controller` | `policy` | `8000` | cleanup policies |
| `kyverno-reports-controller` | `policy` | `8000` | report generation |
| `kyverno-reports-server` | `policy` | `8080` | policy report queries |
| `cert-manager` | `cert-manager` | `9402` | certificate lifecycle |
| `kube-state-metrics` | `observability` | `8080` | k8s object state |
| `node-exporter` | `observability` | `9100` | node-level metrics |
| `kubelet-cadvisor` | `kube-system` | `10250` | pod/container metrics |
| `otel-operator` | `observability` | `8080` | operator reconciliation |

**Pipeline:**
```
receivers:   prometheus (all 14 scrape jobs)
processors:  batch, resource (adds cluster: local-flux label)
exporters:   prometheusremotewrite → http://prometheus-stack-prometheus.observability:9090/api/v1/write
             debug (for local troubleshooting)
```

---

## 7. Prometheus & Grafana

### Controller (`infrastructure/controllers/prometheus-stack/`)

- `kube-prometheus-stack` HelmRelease — `prometheus-community/kube-prometheus-stack`, `observability` namespace
- `dependsOn: otel-operator`
- `kube-state-metrics`: enabled (scraped by OTel only — disable Prometheus's built-in scrape config to prevent duplicate metrics)
- `node-exporter`: enabled (same — disable Prometheus's built-in scrape config)
- Grafana: enabled, admin credentials in a Kubernetes Secret
- Prometheus `remoteWrite` receiver enabled (for OTel collector to write to)

### Grafana Dashboards (`infrastructure/configs/grafana-dashboards/`)

ConfigMaps with label `grafana_dashboard: "1"` (picked up by Grafana sidecar):

| Dashboard | Grafana ID | Source |
|---|---|---|
| Flux Cluster Stats | `16714` | Official Flux |
| Flux Control Plane | `16715` | Official Flux |
| Kyverno Policy Reports | `15987` | Official Kyverno |
| Kubernetes Cluster Overview | bundled | kube-prometheus-stack default |

---

## 8. App Namespace Scaffolding

**`apps/namespaces.yaml`** — defines `team-a` and `team-b` namespaces with standard labels.

**`apps/team-a/`** and **`apps/team-b/`** — empty placeholder `kustomization.yaml` files ready for application HelmReleases or raw manifests.

Adding a new application means:
1. Add its namespace to `apps/namespaces.yaml`
2. Create `apps/<name>/` with a `kustomization.yaml` listing its resources
3. Flux reconciles automatically on next sync interval
