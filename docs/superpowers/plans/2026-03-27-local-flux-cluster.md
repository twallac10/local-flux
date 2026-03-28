# local-flux Cluster Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local Kind cluster (k8s 1.34.x, 1 control-plane + 3 workers) with Flux 2.5.1 GitOps, Kyverno policy governance with full metrics, and OTel/Prometheus/Grafana observability, all managed from `https://github.com/twallac10/local-flux.git`.

**Architecture:** Classic Flux monorepo pattern. Helmfile bootstraps only Flux 2.5.1; all other components are GitOps-managed via three Flux Kustomization layers with explicit `dependsOn` chains: `infrastructure/controllers` → `infrastructure/configs` → `apps`. OTel Operator + a single cluster-wide Collector CR scrapes all metric targets and remote-writes to Prometheus.

**Tech Stack:** Kind 1.34.x, Flux v2.5.1 (fluxcd-community/flux2 chart), Kyverno 3.x, OpenTelemetry Operator + Collector, kube-prometheus-stack, cert-manager, Taskfile v3, Helmfile

---

## File Map

```
local-flux/
├── kind-config.yaml
├── taskfile.yaml
├── helmfile.yaml
├── bootstrap.yaml
├── values/flux.yaml
├── clusters/local-flux/
│   ├── kustomization.yaml
│   ├── infrastructure.yaml
│   ├── infrastructure-configs.yaml
│   ├── apps.yaml
│   └── flux-system/
│       ├── flux.yaml
│       └── kustomization.yaml
├── infrastructure/
│   ├── controllers/
│   │   ├── kustomization.yaml
│   │   ├── cert-manager/{namespace,helmrepository,helmrelease,kustomization}.yaml
│   │   ├── kyverno/{namespace,helmrepository,helmrelease,kyverno-policies-helmrelease,kustomization}.yaml
│   │   ├── otel-operator/{namespace,helmrepository,helmrelease,kustomization}.yaml
│   │   └── prometheus-stack/{helmrepository,helmrelease,kustomization}.yaml
│   └── configs/
│       ├── kustomization.yaml
│       ├── kyverno-policies/{clusterpolicies,policyexception,kustomization}.yaml
│       ├── kyverno-reporting/kustomization.yaml
│       ├── otel-collector/{rbac,collector,kustomization}.yaml
│       └── grafana-dashboards/{flux-cluster-stats,flux-control-plane,kyverno-policy-reports,kustomization}.yaml
└── apps/
    ├── kustomization.yaml
    ├── namespaces.yaml
    ├── team-a/kustomization.yaml
    └── team-b/kustomization.yaml
```

---

### Task 1: Scaffold directory structure

**Files:** Directory tree only

- [ ] **Step 1: Create directories**
  ```bash
  cd /Users/wally/projects/local-flux
  mkdir -p clusters/local-flux/flux-system
  mkdir -p infrastructure/controllers/{cert-manager,kyverno,otel-operator,prometheus-stack}
  mkdir -p infrastructure/configs/{kyverno-policies,kyverno-reporting,otel-collector,grafana-dashboards}
  mkdir -p apps/{team-a,team-b}
  mkdir -p values
  ```

- [ ] **Step 2: Commit**
  ```bash
  git add -A && git commit -m "chore: scaffold directory structure"
  ```

---

### Task 2: Kind cluster config

**Files:**
- Create: `kind-config.yaml`

- [ ] **Step 1: Look up the latest kindest/node image for k8s 1.34.x**

  Visit https://github.com/kubernetes-sigs/kind/releases and find the image tag + SHA256 digest for the latest v1.34.x patch. The format is `kindest/node:v1.34.X@sha256:<digest>`.

- [ ] **Step 2: Write kind-config.yaml** (replace `<IMAGE>` with the full `kindest/node:v1.34.X@sha256:<digest>` string)
  ```yaml
  kind: Cluster
  apiVersion: kind.x-k8s.io/v1alpha4
  nodes:
  - role: control-plane
    image: <IMAGE>
    extraPortMappings:
    - containerPort: 80
      hostPort: 80
      protocol: TCP
    - containerPort: 443
      hostPort: 443
      protocol: TCP
  - role: worker
    image: <IMAGE>
  - role: worker
    image: <IMAGE>
  - role: worker
    image: <IMAGE>
  ```

- [ ] **Step 3: Commit**
  ```bash
  git add kind-config.yaml && git commit -m "feat: Kind cluster config (1 control-plane + 3 workers, k8s 1.34.x)"
  ```

---

### Task 3: Bootstrap files

**Files:**
- Create: `values/flux.yaml`
- Create: `helmfile.yaml`
- Create: `bootstrap.yaml`
- Create: `taskfile.yaml`

- [ ] **Step 1: Write values/flux.yaml**
  ```yaml
  imageAutomationController:
    create: false
  imageReflectionController:
    create: false
  ```

- [ ] **Step 2: Write helmfile.yaml**
  ```yaml
  repositories:
    - name: fluxcd-community
      url: https://fluxcd-community.github.io/helm-charts

  releases:
    - name: flux
      chart: fluxcd-community/flux2
      version: 2.5.1
      namespace: flux-system
      createNamespace: true
      values:
        - ./values/flux.yaml
      hooks:
        - events: [postsync]
          command: kubectl
          args:
            - apply
            - -f
            - bootstrap.yaml
  ```

- [ ] **Step 3: Write bootstrap.yaml**
  ```yaml
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: GitRepository
  metadata:
    name: flux-system
    namespace: flux-system
  spec:
    interval: 1m
    url: https://github.com/twallac10/local-flux.git
    ref:
      branch: main
    secretRef:
      name: flux-git-repo
  ---
  apiVersion: kustomize.toolkit.fluxcd.io/v1
  kind: Kustomization
  metadata:
    name: flux-system
    namespace: flux-system
  spec:
    interval: 10m
    sourceRef:
      kind: GitRepository
      name: flux-system
    path: ./clusters/local-flux
    prune: true
    wait: true
    timeout: 5m
  ```

- [ ] **Step 4: Write taskfile.yaml**
  ```yaml
  version: '3'
  vars:
    KIND_CLUSTER_NAME: local-flux
  tasks:
    build_cluster:
      desc: Build the Kind cluster
      cmds:
        - kind create cluster --name {{.KIND_CLUSTER_NAME}} --config kind-config.yaml

    install_flux:
      desc: Install Flux and bootstrap GitOps
      vars:
        GITHUB_USERNAME: $(gh api user | jq -r .login)
        GITHUB_TOKEN: $(gh auth token)
      cmds:
        - helmfile apply
        - |
          if ! kubectl get secret flux-git-repo -n flux-system 2>/dev/null; then
            echo "Creating secret flux-git-repo"
            kubectl create secret generic flux-git-repo \
              --from-literal=username={{.GITHUB_USERNAME}} \
              --from-literal=password={{.GITHUB_TOKEN}} \
              -n flux-system
          else
            echo "Secret flux-git-repo already exists"
          fi

    build:
      desc: Build cluster and install Flux
      cmds:
        - task: build_cluster
        - task: install_flux

    delete_cluster:
      desc: Delete the Kind cluster
      cmds:
        - kind delete cluster --name {{.KIND_CLUSTER_NAME}}
  ```

- [ ] **Step 5: Commit**
  ```bash
  git add values/ helmfile.yaml bootstrap.yaml taskfile.yaml
  git commit -m "feat: add bootstrap files (Taskfile, Helmfile, Flux values, bootstrap.yaml)"
  ```

---

### Task 4: Flux cluster entry point

**Files:**
- Create: `clusters/local-flux/flux-system/flux.yaml`
- Create: `clusters/local-flux/flux-system/kustomization.yaml`
- Create: `clusters/local-flux/infrastructure.yaml`
- Create: `clusters/local-flux/infrastructure-configs.yaml`
- Create: `clusters/local-flux/apps.yaml`
- Create: `clusters/local-flux/kustomization.yaml`

- [ ] **Step 1: Write clusters/local-flux/flux-system/flux.yaml**
  ```yaml
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: HelmRepository
  metadata:
    name: flux-system
    namespace: flux-system
  spec:
    interval: 1h
    url: https://fluxcd-community.github.io/helm-charts
  ---
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: flux-system
    namespace: flux-system
  spec:
    interval: 1h
    releaseName: flux
    chart:
      spec:
        chart: flux2
        version: 2.5.1
        sourceRef:
          kind: HelmRepository
          name: flux-system
          namespace: flux-system
    values:
      imageAutomationController:
        create: false
      imageReflectionController:
        create: false
    install:
      createNamespace: true
      remediation:
        retries: 3
    upgrade:
      remediation:
        retries: 3
  ```

- [ ] **Step 2: Write clusters/local-flux/flux-system/kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - flux.yaml
  ```

- [ ] **Step 3: Write clusters/local-flux/infrastructure.yaml**
  ```yaml
  apiVersion: kustomize.toolkit.fluxcd.io/v1
  kind: Kustomization
  metadata:
    name: infrastructure
    namespace: flux-system
  spec:
    interval: 10m
    sourceRef:
      kind: GitRepository
      name: flux-system
    path: ./infrastructure/controllers
    prune: true
    wait: true
    timeout: 15m
  ```

- [ ] **Step 4: Write clusters/local-flux/infrastructure-configs.yaml**
  ```yaml
  apiVersion: kustomize.toolkit.fluxcd.io/v1
  kind: Kustomization
  metadata:
    name: infrastructure-configs
    namespace: flux-system
  spec:
    dependsOn:
      - name: infrastructure
    interval: 10m
    sourceRef:
      kind: GitRepository
      name: flux-system
    path: ./infrastructure/configs
    prune: true
    wait: true
    timeout: 10m
  ```

- [ ] **Step 5: Write clusters/local-flux/apps.yaml**
  ```yaml
  apiVersion: kustomize.toolkit.fluxcd.io/v1
  kind: Kustomization
  metadata:
    name: apps
    namespace: flux-system
  spec:
    dependsOn:
      - name: infrastructure-configs
    interval: 10m
    sourceRef:
      kind: GitRepository
      name: flux-system
    path: ./apps
    prune: true
    wait: true
    timeout: 5m
  ```

- [ ] **Step 6: Write clusters/local-flux/kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - flux-system/
  - infrastructure.yaml
  - infrastructure-configs.yaml
  - apps.yaml
  ```

- [ ] **Step 7: Commit**
  ```bash
  git add clusters/
  git commit -m "feat: add Flux cluster entry point with layered Kustomizations"
  ```

---

### Task 5: cert-manager controller

**Files:**
- Create: `infrastructure/controllers/cert-manager/namespace.yaml`
- Create: `infrastructure/controllers/cert-manager/helmrepository.yaml`
- Create: `infrastructure/controllers/cert-manager/helmrelease.yaml`
- Create: `infrastructure/controllers/cert-manager/kustomization.yaml`

- [ ] **Step 1: Write namespace.yaml**
  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: cert-manager
  ```

- [ ] **Step 2: Write helmrepository.yaml**
  ```yaml
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: HelmRepository
  metadata:
    name: cert-manager
    namespace: cert-manager
  spec:
    interval: 1h
    url: https://charts.jetstack.io
  ```

- [ ] **Step 3: Write helmrelease.yaml**
  ```yaml
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: cert-manager
    namespace: cert-manager
  spec:
    interval: 1m
    chart:
      spec:
        chart: cert-manager
        version: ">=1.14.0 <2.0.0"
        sourceRef:
          kind: HelmRepository
          name: cert-manager
          namespace: cert-manager
    values:
      installCRDs: true
    install:
      remediation:
        retries: 3
    upgrade:
      remediation:
        retries: 3
  ```

- [ ] **Step 4: Write kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
  ```

- [ ] **Step 5: Commit**
  ```bash
  git add infrastructure/controllers/cert-manager/
  git commit -m "feat: add cert-manager HelmRelease"
  ```

---

### Task 6: Kyverno controller

**Files:**
- Create: `infrastructure/controllers/kyverno/namespace.yaml`
- Create: `infrastructure/controllers/kyverno/helmrepository.yaml`
- Create: `infrastructure/controllers/kyverno/helmrelease.yaml`
- Create: `infrastructure/controllers/kyverno/kyverno-policies-helmrelease.yaml`
- Create: `infrastructure/controllers/kyverno/kustomization.yaml`

- [ ] **Step 1: Write namespace.yaml**
  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: policy
  ```

- [ ] **Step 2: Write helmrepository.yaml**
  ```yaml
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: HelmRepository
  metadata:
    name: kyverno
    namespace: policy
  spec:
    interval: 1h
    url: https://kyverno.github.io/kyverno/
  ```

- [ ] **Step 3: Write helmrelease.yaml**

  All five Kyverno metric services are enabled. `dependsOn: cert-manager` because Kyverno's webhook TLS can be managed by cert-manager.
  ```yaml
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: kyverno
    namespace: policy
  spec:
    interval: 1m
    timeout: 5m
    dependsOn:
      - name: cert-manager
        namespace: cert-manager
    chart:
      spec:
        chart: kyverno
        version: ">=3.2.0 <4.0.0"
        sourceRef:
          kind: HelmRepository
          name: kyverno
          namespace: policy
    values:
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
    install:
      remediation:
        retries: 3
    upgrade:
      remediation:
        retries: 3
  ```

- [ ] **Step 4: Write kyverno-policies-helmrelease.yaml**
  ```yaml
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: kyverno-policies
    namespace: policy
  spec:
    interval: 1m
    timeout: 3m
    dependsOn:
      - name: kyverno
    chart:
      spec:
        chart: kyverno-policies
        version: ">=3.2.0 <4.0.0"
        sourceRef:
          kind: HelmRepository
          name: kyverno
          namespace: policy
    values:
      podSecurityStandard: baseline
      background: true
  ```

- [ ] **Step 5: Write kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
  - kyverno-policies-helmrelease.yaml
  ```

- [ ] **Step 6: Commit**
  ```bash
  git add infrastructure/controllers/kyverno/
  git commit -m "feat: add Kyverno HelmRelease with reporting server and all metrics enabled"
  ```

---

### Task 7: OTel operator controller

**Files:**
- Create: `infrastructure/controllers/otel-operator/namespace.yaml`
- Create: `infrastructure/controllers/otel-operator/helmrepository.yaml`
- Create: `infrastructure/controllers/otel-operator/helmrelease.yaml`
- Create: `infrastructure/controllers/otel-operator/kustomization.yaml`

- [ ] **Step 1: Write namespace.yaml**
  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: observability
  ```

- [ ] **Step 2: Write helmrepository.yaml**
  ```yaml
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: HelmRepository
  metadata:
    name: open-telemetry
    namespace: observability
  spec:
    interval: 1h
    url: https://open-telemetry.github.io/opentelemetry-helm-charts
  ```

- [ ] **Step 3: Write helmrelease.yaml**
  ```yaml
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: otel-operator
    namespace: observability
  spec:
    interval: 1m
    timeout: 5m
    dependsOn:
      - name: cert-manager
        namespace: cert-manager
    chart:
      spec:
        chart: opentelemetry-operator
        version: ">=0.60.0"
        sourceRef:
          kind: HelmRepository
          name: open-telemetry
          namespace: observability
    values:
      manager:
        collectorImage:
          repository: otel/opentelemetry-collector-contrib
      admissionWebhooks:
        certManager:
          enabled: true
    install:
      remediation:
        retries: 3
    upgrade:
      remediation:
        retries: 3
  ```

- [ ] **Step 4: Write kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
  ```

- [ ] **Step 5: Commit**
  ```bash
  git add infrastructure/controllers/otel-operator/
  git commit -m "feat: add OpenTelemetry Operator HelmRelease"
  ```

---

### Task 8: Prometheus stack controller

**Files:**
- Create: `infrastructure/controllers/prometheus-stack/helmrepository.yaml`
- Create: `infrastructure/controllers/prometheus-stack/helmrelease.yaml`
- Create: `infrastructure/controllers/prometheus-stack/kustomization.yaml`

Note: `observability` namespace is created by the otel-operator task.

- [ ] **Step 1: Write helmrepository.yaml**
  ```yaml
  apiVersion: source.toolkit.fluxcd.io/v1
  kind: HelmRepository
  metadata:
    name: prometheus-community
    namespace: observability
  spec:
    interval: 1h
    url: https://prometheus-community.github.io/helm-charts
  ```

- [ ] **Step 2: Write helmrelease.yaml**

  `enableRemoteWriteReceiver: true` allows the OTel collector to push metrics in. Grafana sidecar watches all namespaces for ConfigMaps with `grafana_dashboard: "1"`.
  ```yaml
  apiVersion: helm.toolkit.fluxcd.io/v2
  kind: HelmRelease
  metadata:
    name: prometheus-stack
    namespace: observability
  spec:
    interval: 1m
    timeout: 10m
    dependsOn:
      - name: otel-operator
    chart:
      spec:
        chart: kube-prometheus-stack
        version: ">=60.0.0"
        sourceRef:
          kind: HelmRepository
          name: prometheus-community
          namespace: observability
    values:
      prometheus:
        prometheusSpec:
          enableRemoteWriteReceiver: true
          scrapeInterval: 30s
      kubeStateMetrics:
        enabled: true
      nodeExporter:
        enabled: true
      grafana:
        enabled: true
        adminPassword: admin
        sidecar:
          dashboards:
            enabled: true
            label: grafana_dashboard
            labelValue: "1"
            searchNamespace: ALL
    install:
      crds: create
      remediation:
        retries: 3
    upgrade:
      crds: CreateReplace
      remediation:
        retries: 3
  ```

- [ ] **Step 3: Write kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - helmrepository.yaml
  - helmrelease.yaml
  ```

- [ ] **Step 4: Commit**
  ```bash
  git add infrastructure/controllers/prometheus-stack/
  git commit -m "feat: add kube-prometheus-stack HelmRelease with remote write receiver"
  ```

---

### Task 9: Wire infrastructure controllers root kustomization

**Files:**
- Create: `infrastructure/controllers/kustomization.yaml`

- [ ] **Step 1: Write infrastructure/controllers/kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - cert-manager/
  - kyverno/
  - otel-operator/
  - prometheus-stack/
  ```

- [ ] **Step 2: Commit**
  ```bash
  git add infrastructure/controllers/kustomization.yaml
  git commit -m "feat: wire infrastructure controllers root kustomization"
  ```

---

### Task 10: Kyverno policies config

**Files:**
- Create: `infrastructure/configs/kyverno-policies/clusterpolicies.yaml`
- Create: `infrastructure/configs/kyverno-policies/policyexception.yaml`
- Create: `infrastructure/configs/kyverno-policies/kustomization.yaml`

- [ ] **Step 1: Write clusterpolicies.yaml**
  ```yaml
  apiVersion: kyverno.io/v1
  kind: ClusterPolicy
  metadata:
    name: require-labels
    annotations:
      policies.kyverno.io/title: Require Labels
      policies.kyverno.io/description: All pods must have 'app' and 'owner' labels.
  spec:
    validationFailureAction: Audit
    background: true
    rules:
      - name: check-for-labels
        match:
          any:
            - resources:
                kinds:
                  - Pod
        validate:
          message: "The labels 'app' and 'owner' are required on all pods."
          pattern:
            metadata:
              labels:
                app: "?*"
                owner: "?*"
  ---
  apiVersion: kyverno.io/v1
  kind: ClusterPolicy
  metadata:
    name: disallow-latest-tag
    annotations:
      policies.kyverno.io/title: Disallow Latest Tag
      policies.kyverno.io/description: Images must use a specific tag, not ':latest'.
  spec:
    validationFailureAction: Enforce
    background: true
    rules:
      - name: require-image-tag
        match:
          any:
            - resources:
                kinds:
                  - Pod
        validate:
          message: "An image tag is required and ':latest' is not allowed."
          pattern:
            spec:
              containers:
                - image: "*:*"
      - name: validate-image-tag
        match:
          any:
            - resources:
                kinds:
                  - Pod
        validate:
          message: "Using ':latest' as the image tag is not allowed."
          deny:
            conditions:
              any:
                - key: "{{ request.object.spec.containers[].image }}"
                  operator: AnyIn
                  value: ["*:latest"]
  ```

- [ ] **Step 2: Write policyexception.yaml**

  Exceptions cover all system namespaces so infrastructure pods are not blocked or flagged.
  ```yaml
  apiVersion: kyverno.io/v2
  kind: PolicyException
  metadata:
    name: flux-system-exception
    namespace: flux-system
  spec:
    exceptions:
      - policyName: require-labels
        ruleNames:
          - check-for-labels
      - policyName: disallow-latest-tag
        ruleNames:
          - require-image-tag
          - validate-image-tag
    match:
      any:
        - resources:
            namespaces:
              - flux-system
            kinds:
              - Pod
  ---
  apiVersion: kyverno.io/v2
  kind: PolicyException
  metadata:
    name: cert-manager-exception
    namespace: cert-manager
  spec:
    exceptions:
      - policyName: require-labels
        ruleNames:
          - check-for-labels
    match:
      any:
        - resources:
            namespaces:
              - cert-manager
            kinds:
              - Pod
  ---
  apiVersion: kyverno.io/v2
  kind: PolicyException
  metadata:
    name: observability-exception
    namespace: observability
  spec:
    exceptions:
      - policyName: require-labels
        ruleNames:
          - check-for-labels
    match:
      any:
        - resources:
            namespaces:
              - observability
            kinds:
              - Pod
  ---
  apiVersion: kyverno.io/v2
  kind: PolicyException
  metadata:
    name: policy-exception
    namespace: policy
  spec:
    exceptions:
      - policyName: require-labels
        ruleNames:
          - check-for-labels
    match:
      any:
        - resources:
            namespaces:
              - policy
            kinds:
              - Pod
  ```

- [ ] **Step 3: Write kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - clusterpolicies.yaml
  - policyexception.yaml
  ```

- [ ] **Step 4: Commit**
  ```bash
  git add infrastructure/configs/kyverno-policies/
  git commit -m "feat: add Kyverno ClusterPolicies and PolicyExceptions for system namespaces"
  ```

---

### Task 11: Kyverno reporting placeholder

**Files:**
- Create: `infrastructure/configs/kyverno-reporting/kustomization.yaml`

The reporting server is configured via Helm values (Task 6). This directory is a placeholder for future `ClusterAdmissionReport` queries or custom RBAC.

- [ ] **Step 1: Write kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  # Kyverno reporting server is enabled via Helm values in infrastructure/controllers/kyverno.
  # Add ClusterAdmissionReport configurations or reporting RBAC here as needed.
  resources: []
  ```

- [ ] **Step 2: Commit**
  ```bash
  git add infrastructure/configs/kyverno-reporting/
  git commit -m "feat: add kyverno-reporting config placeholder"
  ```

---

### Task 12: OTel collector config + RBAC

**Files:**
- Create: `infrastructure/configs/otel-collector/rbac.yaml`
- Create: `infrastructure/configs/otel-collector/collector.yaml`
- Create: `infrastructure/configs/otel-collector/kustomization.yaml`

- [ ] **Step 1: Write rbac.yaml**
  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: otel-collector
    namespace: observability
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: otel-collector
  rules:
    - apiGroups: [""]
      resources:
        - nodes
        - nodes/proxy
        - nodes/metrics
        - services
        - endpoints
        - pods
      verbs: ["get", "list", "watch"]
    - nonResourceURLs:
        - /metrics
        - /metrics/cadvisor
      verbs: ["get"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: otel-collector
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: otel-collector
  subjects:
    - kind: ServiceAccount
      name: otel-collector
      namespace: observability
  ```

- [ ] **Step 2: Write collector.yaml**

  Key notes:
  - Kyverno metrics service names follow the pattern `kyverno-<component>-metrics` — verify with `kubectl get svc -n policy` after Kyverno is up if targets show connection refused.
  - kube-state-metrics service name is `prometheus-stack-kube-state-metrics` when the HelmRelease is named `prometheus-stack`.
  - node-exporter service name is `prometheus-stack-prometheus-node-exporter`.
  - `prometheus-operated` is the standard service created by the Prometheus operator for the StatefulSet (port 9090).
  - The OTel collector CR API version may be `v1alpha1` on older operator versions — verify with `kubectl get crd opentelemetrycollectors.opentelemetry.io -o jsonpath='{.spec.versions[*].name}'`.

  ```yaml
  apiVersion: opentelemetry.io/v1beta1
  kind: OpenTelemetryCollector
  metadata:
    name: cluster
    namespace: observability
  spec:
    mode: deployment
    replicas: 1
    serviceAccount: otel-collector
    config:
      receivers:
        prometheus:
          config:
            scrape_configs:
              - job_name: flux-source-controller
                static_configs:
                  - targets: ['source-controller.flux-system.svc.cluster.local:8080']
              - job_name: flux-kustomize-controller
                static_configs:
                  - targets: ['kustomize-controller.flux-system.svc.cluster.local:8080']
              - job_name: flux-helm-controller
                static_configs:
                  - targets: ['helm-controller.flux-system.svc.cluster.local:8080']
              - job_name: flux-notification-controller
                static_configs:
                  - targets: ['notification-controller.flux-system.svc.cluster.local:8080']
              - job_name: kyverno-admission-controller
                static_configs:
                  - targets: ['kyverno-admission-controller-metrics.policy.svc.cluster.local:8000']
              - job_name: kyverno-background-controller
                static_configs:
                  - targets: ['kyverno-background-controller-metrics.policy.svc.cluster.local:8000']
              - job_name: kyverno-cleanup-controller
                static_configs:
                  - targets: ['kyverno-cleanup-controller-metrics.policy.svc.cluster.local:8000']
              - job_name: kyverno-reports-controller
                static_configs:
                  - targets: ['kyverno-reports-controller-metrics.policy.svc.cluster.local:8000']
              - job_name: kyverno-reports-server
                static_configs:
                  - targets: ['kyverno-reports-server.policy.svc.cluster.local:8080']
              - job_name: cert-manager
                static_configs:
                  - targets: ['cert-manager.cert-manager.svc.cluster.local:9402']
              - job_name: kube-state-metrics
                static_configs:
                  - targets: ['prometheus-stack-kube-state-metrics.observability.svc.cluster.local:8080']
              - job_name: node-exporter
                static_configs:
                  - targets: ['prometheus-stack-prometheus-node-exporter.observability.svc.cluster.local:9100']
              - job_name: otel-operator
                static_configs:
                  - targets: ['opentelemetry-operator.observability.svc.cluster.local:8080']
              - job_name: kubelet-cadvisor
                scheme: https
                tls_config:
                  ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                  insecure_skip_verify: true
                bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
                kubernetes_sd_configs:
                  - role: node
                relabel_configs:
                  - action: labelmap
                    regex: __meta_kubernetes_node_label_(.+)
                  - target_label: __address__
                    replacement: kubernetes.default.svc.cluster.local:443
                  - source_labels: [__meta_kubernetes_node_name]
                    regex: (.+)
                    target_label: __metrics_path__
                    replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
      processors:
        batch: {}
        resource:
          attributes:
            - action: insert
              key: cluster
              value: local-flux
      exporters:
        prometheusremotewrite:
          endpoint: http://prometheus-operated.observability.svc.cluster.local:9090/api/v1/write
        debug:
          verbosity: basic
      service:
        pipelines:
          metrics:
            receivers: [prometheus]
            processors: [batch, resource]
            exporters: [prometheusremotewrite, debug]
  ```

- [ ] **Step 3: Write kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - rbac.yaml
  - collector.yaml
  ```

- [ ] **Step 4: Commit**
  ```bash
  git add infrastructure/configs/otel-collector/
  git commit -m "feat: add OTel Collector CR scraping all 14 metric targets"
  ```

---

### Task 13: Grafana dashboard ConfigMaps

**Files:**
- Create: `infrastructure/configs/grafana-dashboards/flux-cluster-stats.yaml`
- Create: `infrastructure/configs/grafana-dashboards/flux-control-plane.yaml`
- Create: `infrastructure/configs/grafana-dashboards/kyverno-policy-reports.yaml`
- Create: `infrastructure/configs/grafana-dashboards/kustomization.yaml`

- [ ] **Step 1: Fetch dashboard JSON files**
  ```bash
  curl -sL "https://grafana.com/api/dashboards/16714/revisions/latest/download" \
    -o /tmp/flux-cluster-stats.json
  curl -sL "https://grafana.com/api/dashboards/16715/revisions/latest/download" \
    -o /tmp/flux-control-plane.json
  curl -sL "https://grafana.com/api/dashboards/15987/revisions/latest/download" \
    -o /tmp/kyverno-policy-reports.json
  ```

- [ ] **Step 2: Generate ConfigMap YAML for each dashboard**
  ```bash
  kubectl create configmap flux-cluster-stats-dashboard \
    --from-file=flux-cluster-stats.json=/tmp/flux-cluster-stats.json \
    --dry-run=client -o yaml \
    > infrastructure/configs/grafana-dashboards/flux-cluster-stats.yaml

  kubectl create configmap flux-control-plane-dashboard \
    --from-file=flux-control-plane.json=/tmp/flux-control-plane.json \
    --dry-run=client -o yaml \
    > infrastructure/configs/grafana-dashboards/flux-control-plane.yaml

  kubectl create configmap kyverno-policy-reports-dashboard \
    --from-file=kyverno-policy-reports.json=/tmp/kyverno-policy-reports.json \
    --dry-run=client -o yaml \
    > infrastructure/configs/grafana-dashboards/kyverno-policy-reports.yaml
  ```

- [ ] **Step 3: Add namespace and label to each generated ConfigMap**

  Edit each of the three files. Under `metadata:`, add:
  ```yaml
  metadata:
    name: <existing-name>
    namespace: observability
    labels:
      grafana_dashboard: "1"
  ```

  Do this for all three files: `flux-cluster-stats.yaml`, `flux-control-plane.yaml`, `kyverno-policy-reports.yaml`.

- [ ] **Step 4: Write kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - flux-cluster-stats.yaml
  - flux-control-plane.yaml
  - kyverno-policy-reports.yaml
  ```

- [ ] **Step 5: Commit**
  ```bash
  git add infrastructure/configs/grafana-dashboards/
  git commit -m "feat: add Grafana dashboard ConfigMaps (Flux x2, Kyverno)"
  ```

---

### Task 14: Wire infrastructure configs root kustomization

**Files:**
- Create: `infrastructure/configs/kustomization.yaml`

- [ ] **Step 1: Write infrastructure/configs/kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - kyverno-policies/
  - kyverno-reporting/
  - otel-collector/
  - grafana-dashboards/
  ```

- [ ] **Step 2: Commit**
  ```bash
  git add infrastructure/configs/kustomization.yaml
  git commit -m "feat: wire infrastructure configs root kustomization"
  ```

---

### Task 15: Apps scaffolding

**Files:**
- Create: `apps/namespaces.yaml`
- Create: `apps/team-a/kustomization.yaml`
- Create: `apps/team-b/kustomization.yaml`
- Create: `apps/kustomization.yaml`

- [ ] **Step 1: Write apps/namespaces.yaml**
  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: team-a
    labels:
      app.kubernetes.io/managed-by: flux
      team: team-a
  ---
  apiVersion: v1
  kind: Namespace
  metadata:
    name: team-b
    labels:
      app.kubernetes.io/managed-by: flux
      team: team-b
  ```

- [ ] **Step 2: Write apps/team-a/kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  # Add team-a HelmReleases, Deployments, Services here.
  resources: []
  ```

- [ ] **Step 3: Write apps/team-b/kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  # Add team-b HelmReleases, Deployments, Services here.
  resources: []
  ```

- [ ] **Step 4: Write apps/kustomization.yaml**
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
  - namespaces.yaml
  - team-a/
  - team-b/
  ```

- [ ] **Step 5: Commit**
  ```bash
  git add apps/
  git commit -m "feat: add apps scaffolding with team-a and team-b namespace placeholders"
  ```

---

### Task 16: Push to GitHub and bootstrap the cluster

- [ ] **Step 1: Push all commits to GitHub**
  ```bash
  cd /Users/wally/projects/local-flux
  git remote add origin https://github.com/twallac10/local-flux.git
  git push -u origin main
  ```
  Expected: All commits pushed, `main` branch created.

- [ ] **Step 2: Build the Kind cluster**
  ```bash
  task build_cluster
  ```
  Expected: `Creating cluster "local-flux" ...` completes successfully.

  Verify:
  ```bash
  kubectl get nodes
  ```
  Expected: 4 nodes (1 control-plane, 3 workers) all in `Ready` state.

- [ ] **Step 3: Bootstrap Flux**
  ```bash
  task install_flux
  ```
  Expected: Helmfile applies Flux 2.5.1, `bootstrap.yaml` applied, `flux-git-repo` secret created.

- [ ] **Step 4: Verify Flux controllers are running**
  ```bash
  kubectl get pods -n flux-system
  ```
  Expected: `source-controller`, `kustomize-controller`, `helm-controller`, `notification-controller` all `Running`.

- [ ] **Step 5: Watch Flux reconcile all layers**
  ```bash
  flux get kustomizations --watch
  ```
  Expected (allow 15-25 minutes for all HelmReleases to install):
  ```
  NAME                    READY   STATUS
  flux-system             True    Applied revision: main@sha1:...
  infrastructure          True    Applied revision: main@sha1:...
  infrastructure-configs  True    Applied revision: main@sha1:...
  apps                    True    Applied revision: main@sha1:...
  ```

---

### Task 17: Verify full stack

- [ ] **Step 1: Verify all namespaces**
  ```bash
  kubectl get namespaces
  ```
  Expected: `cert-manager`, `policy`, `observability`, `flux-system`, `team-a`, `team-b` all present.

- [ ] **Step 2: Verify cert-manager**
  ```bash
  kubectl get pods -n cert-manager
  ```
  Expected: `cert-manager-*`, `cert-manager-cainjector-*`, `cert-manager-webhook-*` all `Running`.

- [ ] **Step 3: Verify all Kyverno components**
  ```bash
  kubectl get pods -n policy
  ```
  Expected: 5 Deployments running — `kyverno-admission-controller`, `kyverno-background-controller`, `kyverno-cleanup-controller`, `kyverno-reports-controller`, `kyverno-reports-server`.

- [ ] **Step 4: Verify Kyverno metrics services exist**
  ```bash
  kubectl get svc -n policy
  ```
  Expected: Services ending in `-metrics` for each component plus `kyverno-reports-server`.

  If any metrics service is missing, the service name in `collector.yaml` will need updating. Run:
  ```bash
  kubectl get svc -n policy -o name
  ```
  and update the corresponding `targets` entry in `infrastructure/configs/otel-collector/collector.yaml`.

- [ ] **Step 5: Verify OTel operator and collector**
  ```bash
  kubectl get pods -n observability | grep -E 'opentelemetry|cluster-collector'
  ```
  Expected: `opentelemetry-operator-*` and `cluster-collector-*` both `Running`.

- [ ] **Step 6: Verify OTel collector scrape targets**
  ```bash
  kubectl logs -n observability -l app.kubernetes.io/name=cluster-collector --tail=100 | grep -E 'scrape|error|refused'
  ```
  Expected: No `connection refused` errors. Each job should show successful scrape lines.

- [ ] **Step 7: Verify metrics are reaching Prometheus**
  ```bash
  kubectl port-forward -n observability svc/prometheus-operated 9090:9090 &
  sleep 3
  curl -s "http://localhost:9090/api/v1/query?query=up" | jq '.data.result | length'
  ```
  Expected: Integer >= 10 (one `up` metric per scrape target).

  Kill the port-forward when done:
  ```bash
  kill %1
  ```

- [ ] **Step 8: Access Grafana and verify dashboards**
  ```bash
  kubectl port-forward -n observability svc/prometheus-stack-grafana 3000:80 &
  ```
  Open `http://localhost:3000`. Login: `admin` / `admin`.

  Navigate to **Dashboards** and verify these three dashboards appear and show data:
  - Flux Cluster Stats
  - Flux Control Plane
  - Kyverno Policy Reports

  Kill port-forward when done:
  ```bash
  kill %1
  ```

- [ ] **Step 9: Verify Kyverno policies are active**
  ```bash
  kubectl get clusterpolicies
  ```
  Expected:
  ```
  NAME                  ADMISSION   BACKGROUND   READY
  disallow-latest-tag   true        true         True
  require-labels        true        true         True
  ```

- [ ] **Step 10: Smoke test enforce policy**
  ```bash
  kubectl run test-latest --image=nginx:latest -n team-a --dry-run=server 2>&1
  ```
  Expected: Error containing `"Using ':latest' as the image tag is not allowed."`

- [ ] **Step 11: Verify PolicyExceptions**
  ```bash
  kubectl get policyexceptions -A
  ```
  Expected: `flux-system-exception`, `cert-manager-exception`, `observability-exception`, `policy-exception` all present.
