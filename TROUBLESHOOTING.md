# Troubleshooting notes

Failures introduced deliberately on a minikube cluster running an nginx Deployment
(2 replicas) behind a ClusterIP Service, then diagnosed. Scenarios 1 to 6 are plain
Kubernetes, 7 and 8 are with Argo CD managing the app.

| # | What I broke | Symptom | Command | Why / Fix |
|---|---|---|---|---|
| 1 | Bad image tag<br>`nginx:9.99` | `ImagePullBackOff` on the new pod, old pods keep serving | `describe pod` → Events | Tag does not exist in the registry. The pod *was* scheduled, then failed to pull. The rolling update stalls because the new ReplicaSet never becomes Ready, so the old one is never scaled down. Zero downtime by design.<br>**Fix:** `kubectl rollout undo deployment/web` |
| 2 | Wrong Service selector<br>`app: wrong` | Pods 1/1 Running, Service exists, apply succeeded, nothing reachable. No error anywhere | `get endpoints web` → `<none>`<br>`get pods --show-labels` → `app=web` | A Service finds its pods only through its label selector. No match means empty endpoints and no traffic. Silent failure: every status check looks green.<br>**Fix:** set the selector back to `app: web` |
| 3a | `requests` > `limits`<br>64Mi req / 10Mi limit | `apply` rejected. No pod created at all, running pods untouched | `apply` → `"requests: Invalid value 64Mi: must be less than or equal to memory limit of 10Mi"` | Admission validation: the API server rejects the object before it reaches etcd. A different class of failure from runtime errors, nothing was ever scheduled.<br>**Fix:** lower requests below the limit |
| 3b | Very low memory limit<br>4Mi req / 6Mi limit | Pods Running 1/1, no OOMKill, site keeps working | `get pods -w`<br>`get deployment web -o jsonpath="{.spec.template.spec.containers[0].resources}"` | Limits are enforced against actual consumption, not reserved upfront. An idle nginx fits in 6Mi. OOMKill fires only when the container allocates past the ceiling, which in production means under load or on a leak, not at deploy time.<br>**Fix:** revert to 64Mi / 128Mi |
| 4 | Impossible CPU request<br>`requests.cpu: "100"` | Pod stuck `Pending`, `Node: <none>`, no IP | `describe pod` → Events | `0/1 nodes are available: 1 Insufficient cpu`. Requests decide placement. Contrast with #1: there the pod was scheduled and then failed; here it is never placed at all. `Pending` means a scheduling problem, `ImagePullBackOff` or `CrashLoopBackOff` means the pod is already placed and the container is the problem.<br>**Fix:** lower to `50m` |
| 5 | Deleted a pod | Replacement appears in about 1 second, unprompted. Old pod shows `Completed`, so it shut down cleanly | `get pods -w` | Reconciliation loop: the controller compares desired state (2 replicas) against actual (1) and corrects. The replacement is created before the old pod finishes terminating.<br>**Fix:** none, working as designed |
| 6 | Scaled to 5 replicas | 3 pods created, then terminated when scaled back | `get pods` | Same reconciliation loop, driven by a changed desired count instead of a deleted pod. Scaling and self-healing are the same mechanism. HPA automates this from metrics |
| 7 | Deleted the Deployment, Argo CD self-heal on | Recreated in about 1 second, nothing clicked | `get pods -w` | Argo CD reconciles the cluster against Git, not just a ReplicaSet against its own spec. The manifest came from GitHub, no `kubectl apply` involved |
| 8 | `kubectl scale --replicas=5` while Git says 2 | 3 pods created, all terminated after about 5 seconds | `get pods -w` | Git wins. Once an app is under GitOps, `kubectl` is no longer the source of truth. To actually get 5 replicas you commit the change |

---

## Key concepts

**Requests vs limits.** Requests decide scheduling, where a pod can fit. Limits decide
enforcement, the hard ceiling. Getting this backwards explains most resource bugs.

**Label selectors.** The only link between a Service and its pods. There is no explicit
reference. Empty endpoints means a selector mismatch, every time.

**Reconciliation loop.** The controller continuously compares desired state to actual
state and corrects the difference. Declarative, self-healing, and the same mechanism
behind both scaling and pod replacement.

**Two places things break.** The API server validates and rejects an invalid object
before anything is created (`requests > limits`), or the object is valid and the failure
happens at runtime (bad image tag, OOMKill). Knowing which one you are looking at tells
you where to look.

**Manifest drift.** `rollout undo` reverted the cluster but not `deployment.yaml`, so the
next `apply` pushed the broken image straight back. This is exactly the problem GitOps
removes: Git is the single source of truth, and an uncommitted manual change is reverted
on the next sync.

**Stale port-forward.** `kubectl port-forward` keeps an existing tunnel open after the
Service selector changes or pods are replaced, so the browser can keep working while the
Service is already broken. Trust `kubectl get endpoints` over what the browser shows.

---

## Diagnostic command map

| Command | Answers |
|---|---|
| `kubectl get pods -o wide -w` | what state |
| `kubectl describe pod <name>` | why, from the Events section |
| `kubectl logs <pod> --previous` | what the app said before it crashed |
| `kubectl get endpoints <svc>` | did the Service find any pods |
| `kubectl get pods --show-labels` | labels vs the Service selector |
| `kubectl rollout status/history/undo` | check or revert a Deployment |
