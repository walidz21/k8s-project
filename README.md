# Kubernetes, GitOps and Helm

A lab cluster used to work through Kubernetes operations, GitOps delivery with Argo CD,
and Helm packaging. This repository is the declared state of the cluster.

**Stack:** Kubernetes (minikube), Argo CD, Helm, Docker, Git

---

## Repository layout

```
.
├── manifests/          # plain Kubernetes YAML, watched by the "web" Argo CD app
│   ├── deployment.yaml
│   └── service.yaml
├── helm-chart/         # the same app as a Helm chart, watched by the "web-helm" app
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       └── service.yaml
├── TROUBLESHOOTING.md  # failure scenarios, symptoms and how they were diagnosed
└── README.md
```

Two Applications, same repository, different paths, so the same workload is delivered
both ways: raw manifests and a templated chart.

The workload itself is an nginx Deployment behind a ClusterIP Service, with:
- explicit resource `requests` and `limits`
- label-based Service to Pod wiring
- 2 replicas by default, configurable per environment in the Helm chart

---

## GitOps

Argo CD runs in its own `argocd` namespace with auto-sync and self-heal enabled, reconciling
the cluster against this repo. The practical consequence is that `kubectl` stops being the
source of truth:

```bash
kubectl scale deployment web --replicas=5   # reverted in ~5s, Git says 2
kubectl delete deployment web               # recreated in ~1s
```

Changing what runs means changing the repo:

```bash
# edit manifests/deployment.yaml
git commit -am "Scale web to 5 replicas"
git push
```

Drift disappears, and the commit history becomes the deployment record. The escape hatch
for an incident is disabling auto-sync, not reaching for kubectl and hoping it sticks.

---

## Helm

`helm-chart/` templates the Deployment and Service, with the per-environment values
pulled out into `values.yaml`:

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.25"
```

One chart, several releases:

```bash
helm install web-dev  ./helm-chart
helm install web-prod ./helm-chart --set replicaCount=3 --set image.tag=1.27
helm install web-dev  ./helm-chart --dry-run --debug   # rendering before applying
```

Rollback is atomic across every resource in the release, which is the difference that
matters against `kubectl rollout undo`:

```bash
helm history  web-dev
helm rollback web-dev 1
```

In the delivery path Helm is templating only. Argo CD renders the chart and applies it,
so no `helm install` is involved.

---

## Running it

```bash
minikube start

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

kubectl port-forward svc/argocd-server -n argocd 8081:443   # UI
kubectl port-forward svc/web 8080:80                        # app
```

---

## Troubleshooting

[`TROUBLESHOOTING.md`](TROUBLESHOOTING.md) covers eight failures introduced deliberately and
then diagnosed: image pull failures, a selector mismatch, resource misconfiguration and a
scheduling failure, with the symptom, the command that surfaced the cause, and the fix.

Two of them are worth the summary here. A Service finds its pods only through its label
selector, so a wrong selector leaves endpoints empty while every pod stays healthy and
nothing anywhere reports an error. And resource failures land in two different places:
`requests > limits` is rejected by the API server before anything is created, while an
impossible CPU request is accepted and then sits Pending, never scheduled.
