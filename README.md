# argocd-workshop

A minimal but realistic GitOps example using Argo CD, meant as the advanced follow-up to our Kubernetes course.

It shows how to bootstrap a whole cluster from a single Git repository using the **app-of-apps** pattern. Argo CD even manages itself.

## What you'll learn

- How the app-of-apps pattern works
- How to group applications with `AppProject`s
- How to deploy upstream Helm charts with your own values from Git (multi-source Applications)
- How automated sync, self-heal, and prune behave
- Why Argo CD managing its own installation is useful, and why it needs care

## Repository layout

```
.
├── launcher.yml              # Root Application (the "app of apps"). Apply this once.
├── .launcher/                # Application and AppProject manifests, read recursively by launcher
│   ├── spaceship/            # Cluster platform components (project: launcher)
│   │   ├── engine/           #   Argo CD itself + the "launcher" AppProject
│   │   ├── door/             #   ingress-nginx
│   │   ├── fuel-tank/main/   #   CloudNativePG operator
│   │   └── engine-monitor/   #   kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
│   └── apollo11/             # A sample "team" project (project: apollo11)
│       ├── crews.yml         #   Application -> apps/apollo11/crews
│       └── food.yml          #   Application -> apps/apollo11/food
└── apps/                     # What actually gets deployed
    ├── spaceship/<component>/
    │   ├── *-values-base.yml     # Upstream chart defaults, for reference only
    │   └── values-override.yml   # Our overrides, the only file Argo CD uses
    └── apollo11/             # Plain manifests (ConfigMaps)
```

The naming follows a spaceship metaphor: the **engine** (Argo CD) drives everything, the **door** (ingress) lets traffic in, the **fuel tank** holds data, and the **engine monitor** watches it all.

## How it works

1. You install Argo CD once by hand.
2. You apply `launcher.yml`. It points to `.launcher/` with `directory.recurse: true`.
3. Argo CD finds every Application and AppProject in `.launcher/` and creates them.
4. Each child Application deploys its component, either a Helm chart plus our values, or plain manifests from `apps/`.
5. From then on, Git is the source of truth. Push a change, and Argo CD syncs it. Change something by hand in the cluster, and self-heal reverts it.

### Multi-source Applications

The platform components use two sources:

```yaml
sources:
  - repoURL: https://github.com/rezachalak/argocd-workshop.git
    targetRevision: HEAD
    ref: deploymentRepo                  # this repo, referenced as $deploymentRepo
  - chart: ingress-nginx
    repoURL: https://kubernetes.github.io/ingress-nginx
    targetRevision: 4.12.1
    helm:
      valueFiles:
        - $deploymentRepo/apps/spaceship/door/values-override.yml
```

The chart comes from the upstream Helm repository, and the values come from this repo. You never fork the chart.

## Components

| Component | Chart | Version | Namespace | Project |
|---|---|---|---|---|
| Argo CD | argo-cd | 8.1.3 | argocd | launcher |
| ingress-nginx | ingress-nginx | 4.12.1 | ingress-nginx | launcher |
| CloudNativePG operator | cloudnative-pg | 0.23.2 | postgres-operator | launcher |
| kube-prometheus-stack | kube-prometheus-stack | 68.4.4 | system-monitoring | launcher |
| crews | plain manifests | – | crews | apollo11 |
| food | plain manifests | – | food | apollo11 |

`apps/spaceship/` also contains values for Kafka, Kafka UI, and ClickHouse. They have no Application yet and are left as exercises (see below).

## Prerequisites

- A Kubernetes cluster (kind, k3d, minikube, or similar)
- `kubectl` and `helm`
- A way to get `LoadBalancer` IPs, for example MetalLB, `minikube tunnel`, or the built-in load balancer in k3d

## Getting started

### 1. Install Argo CD

Use the same chart version and release name that the repo uses. Then Argo CD can adopt its own installation without conflicts.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo-cd argo/argo-cd \
  --version 8.1.3 \
  --namespace argocd --create-namespace \
  -f apps/spaceship/engine/values-override.yml
```

### 2. Launch

```bash
kubectl apply -f launcher.yml
```

Then watch the apps appear:

```bash
kubectl -n argocd get applications -w
```

### 3. Open the UIs

Point these hostnames to your ingress controller's IP in `/etc/hosts`:

```
<INGRESS_IP>  argocd.local grafana.local alertmanager.local
```

Get the Argo CD admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

Then open http://argocd.local and log in as `admin`.

## Exercises

1. **Self-heal:** Delete the `neil-armstrong` ConfigMap in the `crews` namespace and watch Argo CD restore it.
2. **GitOps change:** Add a new crew member (for example `buzz.yml`) under `apps/apollo11/crews/`, push it, and watch it sync.
3. **Prune:** Remove a file from Git and confirm the resource is deleted from the cluster.
4. **New component:** Write an Application in `.launcher/spaceship/digital-bridge/` that deploys Kafka with the values in `apps/spaceship/digital-bridge/`.
5. **Lock down a project:** Restrict the `apollo11` AppProject so it can only deploy to the `crews` and `food` namespaces and only from this repo.

## Notes and gotchas

- The Argo CD Application uses `prune: false`. Automatic pruning of Argo CD's own resources could delete the controller that's doing the pruning.
- Both AppProjects allow everything (`*`). That's fine for a workshop, but never do it in production.
- `targetRevision: HEAD` tracks the default branch. In real setups, pin to a tag or a commit.

## Linting

- `.yamllint.yml` lints YAML (it ignores `*-base.yml` files).
- `.kube-linter.yaml` checks manifests against Kubernetes best practices.
- `.pre-commit-config.yaml` runs basic hooks before each commit.
