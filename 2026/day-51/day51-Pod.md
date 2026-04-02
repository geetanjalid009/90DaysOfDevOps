# Day 51 – Kubernetes Manifests and First Pods

## The Four Fields You Need to Know

Every Kubernetes resource is a YAML file with four required top-level fields. Miss any one of them and the API server won't accept it.

**`apiVersion`** — which API group handles this resource. For a Pod it's `v1`. For a Deployment it'll be `apps/v1`. Think of it like specifying which version of a form you're filling out.

**`kind`** — what you're creating. `Pod`, `Deployment`, `Service`, etc. The API server uses this to know which controller should handle the request.

**`metadata`** — the identity of the resource. `name` is required. `labels` are optional key-value pairs, but you'll use them constantly for filtering and selection.

**`spec`** — the actual definition of what you want. For a Pod, this is which containers to run, which images, which ports. Everything else in the file exists just to frame this section.

---

## Pod Manifests

### nginx-pod.yml

```yaml
kind: Pod
apiVersion: v1

metadata:
  namespace: nginx-ns
  name: nginx-pod

spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

One thing I noticed — the `nginx-pod.yml` didn't have labels in the metadata. That bit me later when I tried to filter by `app=nginx` and got nothing back. Adding labels is a habit worth building from day one.

---

### busybox-pod.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
  namespace: nginx-ns
  labels:
    app: busybox
    environment: dev
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sh", "-c", "echo Hello from BusyBox && sleep 3600"]
```

The `command` field is important here. BusyBox has no long-running process by default — without the `sleep 3600`, the container exits immediately and the pod goes into `CrashLoopBackOff`. The command keeps it alive long enough to be useful.

Also got burned by a typo on the first attempt. Used `cmd` instead of `command`:

```
Error from server (BadRequest): Pod in version "v1" cannot be handled as a Pod:
strict decoding error: unknown field "spec.containers[0].cmd"
```

Kubernetes is strict about field names. `cmd` doesn't exist. Fixed it to `command` and it worked.

---

### redis-pod (imperative)

```bash
kubectl run redis-pod --image=redis:latest
```

Created without a YAML file. This is the imperative approach — faster for quick tests, but you can't version-control it or reuse it.

---

## Errors I Actually Hit

### Namespace didn't exist yet

```
Error from server (NotFound): error when creating "busybox-pod.yml":
namespaces "nginx-ns" not found
```

The manifest had `namespace: nginx-ns` in it, but I hadn't created that namespace yet. Applied `namespace.yml` first, then the pod applied fine.

```bash
kubectl apply -f namespace.yml
kubectl apply -f busybox-pod.yml
# pod/busybox-pod created
```

### Passed filename instead of pod name to logs

```bash
kubectl logs busybox-pod.yml
# error: pods "busybox-pod.yml" not found in namespace "default"
```

`kubectl logs` takes a pod name, not a filename. The pod was also in `nginx-ns`, not `default`, which made the error message slightly misleading.

```bash
kubectl logs busybox-pod -n nginx-ns
```

### set-context syntax

```bash
kubectl config set-context --namespace=nginx-ns
# error: you must specify a non-empty context name or --current
```

Needed `--current` to modify the active context:

```bash
kubectl config set-context --current --namespace=nginx-ns
# Context "kind-practice-cluster" modified.
```

---

## Watching a Pod Start

redis-pod took about 25–27 seconds to go from `ContainerCreating` to `Running`. The image pull is what takes time on a fresh node. Once it's cached, subsequent pods with the same image start much faster.

```
NAME          READY   STATUS              RESTARTS   AGE
busybox-pod   1/1     Running             0          2m34s
redis-pod     0/1     ContainerCreating   0          21s
...
redis-pod     1/1     Running             0          27s
```

*(Screenshot: redis-pod polling from ContainerCreating → Running)*

---

## Dry Run

```bash
kubectl run test-pod --image=nginx --dry-run=client -o yaml
```

Output:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test-pod
  name: test-pod
spec:
  containers:
  - image: nginx
    name: test-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Compared to a hand-written manifest, Kubernetes auto-adds: `dnsPolicy`, `restartPolicy`, `resources: {}`, and `status: {}`. Those are all defaults. Your YAML doesn't need them, but they show up in the generated output and in `kubectl get pod -o yaml` on a live pod.

Saved it to a file for reference:

```bash
kubectl run test-pod --image=nginx --dry-run=client -o yaml > dry-run.txt
```

This is the fastest way to scaffold a manifest template without having to remember the exact field structure.

---

## Imperative vs Declarative

| | Imperative | Declarative |
|---|---|---|
| Command | `kubectl run redis-pod --image=redis:latest` | `kubectl apply -f redis-pod.yml` |
| Version control | No — it's gone after the terminal session | Yes — the YAML file is the source of truth |
| Reproducible | No | Yes |
| Good for | Quick tests, one-off debugging | Everything in production |

The dry-run trick sits between both worlds — use imperative syntax to generate declarative output, then customize the YAML.

---

## Labels and Filtering

```bash
kubectl get pods --show-labels
NAME          READY   STATUS    RESTARTS   AGE    LABELS
busybox-pod   1/1     Running   0          16m    app=busybox,environment=dev
redis-pod     1/1     Running   0          14m    run=redis-pod
```

Filtering by label:

```bash
kubectl get pods -l app=nginx
# No resources found in nginx-ns namespace.

kubectl get pods -l environment=dev
# busybox-pod   1/1   Running   0   10m
```

The `app=nginx` filter returned nothing because the nginx-pod manifest didn't have labels. That's the practical reason labels matter — if you don't add them, you can't select by them later. Services, Deployments, and network policies all rely on label selectors.

---

## Cleanup

```bash
kubectl delete pod nginx-pod
# pod "nginx-pod" deleted from nginx-ns namespace

kubectl delete pod busybox-pod
# pod "busybox-pod" deleted from nginx-ns namespace

kubectl delete pod redis-pod
# pod "redis-pod" deleted from nginx-ns namespace

kubectl get pods
# No resources found in nginx-ns namespace.
```

Bare pods don't come back. There's no controller watching them. Delete a standalone Pod and it's gone — there's nothing to recreate it. This is exactly why Deployments exist. A Deployment has a controller that constantly checks: "did I ask for 3 replicas? are 3 running?" and fixes the gap if not. Day 52 covers that.

---

## Key Commands

```bash
# Apply a manifest
kubectl apply -f pod.yml

# Imperative pod creation
kubectl run redis-pod --image=redis:latest

# Dry run — generate YAML without creating anything
kubectl run test-pod --image=nginx --dry-run=client -o yaml

# Check running pods
kubectl get pods
kubectl get pods -n nginx-ns
kubectl get pods --show-labels
kubectl get pods -o wide

# Filter by label
kubectl get pods -l app=busybox
kubectl get pods -l environment=dev

# Debug
kubectl describe pod nginx-pod
kubectl logs busybox-pod -n nginx-ns
kubectl exec -it nginx-pod -- /bin/bash

# Delete
kubectl delete pod nginx-pod
kubectl delete -f nginx-pod.yml
```

---

`#90DaysOfDevOps` `#DevOpsKaJosh` `#TrainWithShubham`