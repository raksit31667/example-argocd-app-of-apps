# Plan: ArgoCD App-of-Apps from Scratch

## TL;DR

Build a full ArgoCD app-of-apps pattern covering service management (microservices + service namespaces), cluster-addons (Helm charts fan-out across labeled clusters), cluster-core (CRDs + GatewayClass via kustomize), and GitHub self-hosted runners (controller + scale set).

---

## Phase 1: Repo & Root Bootstrap

1. Create repo skeleton directories: `charts/apps/templates/`, `charts/cluster-addons/`, `charts/gha-runner-scale-set-controller/`, `charts/gha-runner/`, `kustomize/cluster-core/crds/`, `kustomize/cluster-core/gateway-class/`, `kustomize/service-namespaces/`

2. Create `charts/apps/Chart.yaml` — minimal Helm chart metadata (apiVersion v2, type application).

3. Create `app-of-apps.yaml` — root ArgoCD Application bootstrapped with `kubectl apply`. Points to `charts/apps` in `<SOURCE_REPO_SSH_URL>`, project `default`, destination `https://kubernetes.default.svc`, namespace `argocd`.

---

## Phase 2: ArgoCD Projects

*Depends on Phase 1.*

4. Create `charts/apps/templates/projects.yaml` with three AppProjects:
   - `<DEVOPS_PROJECT>` — sync-wave `-1`, targets `https://kubernetes.default.svc` only, cluster resource whitelist: Namespace, ClusterRole, ClusterRoleBinding, CRD, apiregistration, admissionregistration, IngressClass, IngressClassParams.
   - `cluster-addons` — sync-wave `-1`, one destination entry per environment cluster (`<K8S_API_SERVER_URL>`), same cluster resource whitelist.
   - `cluster-core` — sync-wave `-3` (must precede addons), all environment cluster destinations, `clusterResourceWhitelist: [{group: '*', kind: '*'}]`.

---

## Phase 3: Cluster Core (CRDs + GatewayClass)

*Depends on Phase 2. Parallel with Phases 4–6.*

5. Create `kustomize/cluster-core/crds/kustomization.yaml` — list CRD manifests under `resources`. Place CRD YAML files alongside it.

6. Create `kustomize/cluster-core/gateway-class/gatewayclass.yaml` — GatewayClass manifest.

7. Create `charts/apps/templates/cluster-core-crds.yaml` — ApplicationSet:
   - Generator: `clusters` with `matchLabels: <CLUSTER_LABEL_KEY>: "<CLUSTER_LABEL_VALUE>"`
   - Template name: `cluster-core-crds-{{name}}`
   - Project: `cluster-core`
   - Source: `<SOURCE_REPO_SSH_URL>`, path `kustomize/cluster-core/crds`, `targetRevision: HEAD`
   - Destination: `{{server}}`, namespace `default`

8. Create `charts/apps/templates/cluster-core-gateway-class.yaml` — ApplicationSet (same generator pattern as CRDs, path `kustomize/cluster-core/gateway-class`, name `cluster-core-gateway-class-{{name}}`).

---

## Phase 4: Cluster Add-ons

*Depends on Phase 2. Parallel with Phases 3, 5, 6.*

9. For each add-on listed below, repeat the same three-file structure:
   - `Chart.yaml` — wrapper chart with upstream Helm chart as a dependency (via `dependencies:` block).
   - `values.yaml` — base values applied to all clusters.
   - `values.<CLUSTER_NAME>.yaml` — per-cluster overrides, one file per registered cluster.

   Add-ons to provision (one subdirectory each under `charts/cluster-addons/`):

   | Directory | Purpose |
   |---|---|
   | `aws-loadbalancer-controller` | AWS Load Balancer Controller |
   | `cluster-autoscaler` | Cluster Autoscaler |
   | `crowdstrike` | CrowdStrike Falcon sensor |
   | `datadog` | Datadog agent |
   | `external-dns` | ExternalDNS |
   | `metric` | Metrics server |
   | `salt-sensor` | Salt security sensor |

10. Create `charts/apps/templates/cluster-addons.yaml` — ApplicationSet:
    - Generator: `matrix` combining:
      - `git` generator scanning `charts/cluster-addons/*` directories in `<SOURCE_REPO_SSH_URL>` (revision: `main`)
      - `clusters` generator with `matchLabels: <CLUSTER_LABEL_KEY>: "<CLUSTER_LABEL_VALUE>"`
    - Template name: `{{path.basename}}-{{name}}`
    - Project: `cluster-addons`
    - Source: `<SOURCE_REPO_SSH_URL>`, path `{{path}}`, helm `releaseName: {{path.basename}}`, valueFiles `[values.yaml, values.{{name}}.yaml]`
    - Destination: `{{server}}`, namespace `{{path.basename}}`

---

## Phase 5: Service Management

*Depends on Phase 2. Parallel with Phases 3, 4, 6.*

### Service Namespaces (ApplicationSet)

11. Create `kustomize/service-namespaces/<SERVICE_NAMESPACE>.yaml` — one Namespace manifest per service. A single team owning multiple services creates one file per service.

12. Create `kustomize/service-namespaces/kustomization.yaml` — lists all service namespace YAML files under `resources`.

13. Create `charts/apps/templates/service-namespaces.yaml` — ApplicationSet:
    - Generator: `matrix` combining git directory scan of `kustomize/service-namespaces` and clusters with label `<MICROSERVICES_CLUSTER_LABEL>: "true"`
    - Template fans out each namespace to every microservice cluster.

### Microservices Delivery

14. Create `charts/apps/templates/microservices.yaml` — single Application pointing to the **deployment repo** (`<DEPLOYMENT_REPO_SSH_URL>`) at path `argocd/apps`, project `default`. This delegates all microservice ApplicationSets to the deployment repo.

    OR — for per-service ApplicationSets managed in this repo, create one file per service under `charts/apps/templates/` using a `list` generator with `(cluster, url)` elements for each environment (`<CLUSTER_NAME>` / `<K8S_API_SERVER_URL>`).

---

## Phase 6: GitHub Self-Hosted Runners

*Depends on Phase 2. Parallel with Phases 3, 4, 5.*

15. Create `charts/gha-runner-scale-set-controller/Chart.yaml` — wrapper chart depending on upstream `gha-runner-scale-set-controller` Helm chart.

16. Create `charts/gha-runner-scale-set-controller/values.yaml` — controller configuration.

17. Create `charts/gha-runner/Chart.yaml` — wrapper chart depending on upstream `actions-runner-controller` scale set chart.

18. Create `charts/gha-runner/values.yaml` — runner scale set config: `githubConfigUrl: <GITHUB_CONFIG_URL>`, `githubConfigSecret: <GITHUB_SECRET_NAME>`, `runnerGroup: <RUNNER_GROUP>`, minRunners, maxRunners, labels, service account.

19. Create `charts/apps/templates/gha-runner-scale-set-controller.yaml` — Application:
    - Project: `<DEVOPS_PROJECT>`
    - Destination: `https://kubernetes.default.svc`, namespace `<RUNNER_NAMESPACE>`
    - Source: `<SOURCE_REPO_SSH_URL>`, path `charts/gha-runner-scale-set-controller`
    - syncOptions: `ServerSideApply=true`

20. Create `charts/apps/templates/gha-runner.yaml` — Application:
    - Project: `<DEVOPS_PROJECT>`
    - Destination: `https://kubernetes.default.svc`, namespace `<RUNNER_NAMESPACE>`
    - Source: `<SOURCE_REPO_SSH_URL>`, path `charts/gha-runner`

---

## Phase 7: Bootstrap & Verify

*Depends on all prior phases being committed and pushed.*

21. Register each target cluster in ArgoCD:
    ```
    argocd cluster add <ALIAS> --label "<CLUSTER_LABEL_KEY>=<CLUSTER_LABEL_VALUE>" --upsert
    ```
    Add `<MICROSERVICES_CLUSTER_LABEL>=true` on clusters that run microservice workloads.

22. Apply root bootstrap:
    ```
    kubectl apply -f app-of-apps.yaml
    ```

23. In ArgoCD UI, sync the `apps` root application. Verify child applications are created per component.

---

## Relevant Files

| File | Purpose |
|---|---|
| `app-of-apps.yaml` | Root bootstrap Application |
| `charts/apps/Chart.yaml` | Helm chart wrapper |
| `charts/apps/templates/projects.yaml` | All AppProjects |
| `charts/apps/templates/cluster-core-crds.yaml` | CRD ApplicationSet |
| `charts/apps/templates/cluster-core-gateway-class.yaml` | GatewayClass ApplicationSet |
| `charts/apps/templates/cluster-addons.yaml` | Cluster-addons matrix ApplicationSet |
| `charts/apps/templates/service-namespaces.yaml` | Service namespace ApplicationSet |
| `charts/apps/templates/microservices.yaml` | Microservices delivery pointer |
| `charts/apps/templates/gha-runner-scale-set-controller.yaml` | Runner controller Application |
| `charts/apps/templates/gha-runner.yaml` | Runner scale set Application |
| `charts/cluster-addons/aws-loadbalancer-controller/` | AWS Load Balancer Controller wrapper chart + per-cluster values |
| `charts/cluster-addons/cluster-autoscaler/` | Cluster Autoscaler wrapper chart + per-cluster values |
| `charts/cluster-addons/crowdstrike/` | CrowdStrike sensor wrapper chart + per-cluster values |
| `charts/cluster-addons/datadog/` | Datadog agent wrapper chart + per-cluster values |
| `charts/cluster-addons/external-dns/` | ExternalDNS wrapper chart + per-cluster values |
| `charts/cluster-addons/metric/` | Metrics server wrapper chart + per-cluster values |
| `charts/cluster-addons/salt-sensor/` | Salt security sensor wrapper chart + per-cluster values |
| `charts/gha-runner-scale-set-controller/` | Controller wrapper chart |
| `charts/gha-runner/` | Runner chart |
| `kustomize/cluster-core/crds/` | CRD kustomize layer |
| `kustomize/cluster-core/gateway-class/` | GatewayClass kustomize layer |
| `kustomize/service-namespaces/` | Service namespace manifests + kustomization.yaml |

---

## Verification

1. Run `helm template charts/apps` from repo root — all templates render without errors.
2. After `kubectl apply -f app-of-apps.yaml`, confirm `apps` Application appears in ArgoCD UI.
3. After sync, verify ApplicationSets are created: `cluster-addons`, `cluster-core-crds`, `cluster-core-gateway-class`, `service-namespaces`.
4. Confirm one child Application per `<CLUSTER_LABEL_KEY>=<CLUSTER_LABEL_VALUE>` cluster exists for each ApplicationSet.
5. Confirm runner pods start in `<RUNNER_NAMESPACE>` on the devops cluster.
6. Confirm CRDs are visible on each target cluster via `kubectl get crds`.
7. Confirm service namespaces (e.g., `<SERVICE_NAMESPACE>`) exist across all `<MICROSERVICES_CLUSTER_LABEL>=true` clusters.

---

## Decisions

- Microservices delivery delegates to the deployment repo (`<DEPLOYMENT_REPO_SSH_URL>`) via `microservices.yaml`. Per-service ApplicationSets are created in the deployment repo.
- `cluster-core` project uses sync-wave `-3` (before `cluster-addons` at `-1`) to ensure CRDs exist before addon controllers start.
- `service-namespaces` ApplicationSet targets clusters labeled `<MICROSERVICES_CLUSTER_LABEL>=true` (separate from `<CLUSTER_LABEL_KEY>`) to avoid pushing service namespaces to the devops cluster.
- One namespace per service — a team owning multiple services creates one `<SERVICE_NAMESPACE>.yaml` per service under `kustomize/service-namespaces/`.
- Custom build runner is excluded from this plan.
- GatewayClass component is included in cluster-core.

---

## Placeholder Reference

| Placeholder | Example value |
|---|---|
| `<SOURCE_REPO_SSH_URL>` | `git@github.com:<ORG>/<INFRA_REPO>.git` |
| `<DEPLOYMENT_REPO_SSH_URL>` | `git@github.com:<ORG>/<DEPLOY_REPO>.git` |
| `<CLUSTER_LABEL_KEY>` | `environment` |
| `<CLUSTER_LABEL_VALUE>` | `managed` |
| `<MICROSERVICES_CLUSTER_LABEL>` | `microservices` |
| `<K8S_API_SERVER_URL>` | `https://<ID>.region.eks.amazonaws.com` |
| `<CLUSTER_NAME>` | `prod`, `staging`, `test` |
| `<ALIAS>` | `my-cluster-name` |
| `<DEVOPS_PROJECT>` | `devops-cluster` |
| `<RUNNER_NAMESPACE>` | `gha-runner-scale-set` |
| `<ADDON_NAME>` | `aws-loadbalancer-controller`, `cluster-autoscaler`, `crowdstrike`, `datadog`, `external-dns`, `metric`, `salt-sensor` |
| `<SERVICE_NAMESPACE>` | `payments`, `notifications` |
| `<GITHUB_CONFIG_URL>` | `https://github.com/<ORG>` |
| `<GITHUB_SECRET_NAME>` | `github-runner-secret` |
| `<RUNNER_GROUP>` | `default` |
