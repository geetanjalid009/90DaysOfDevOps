# Day 52 – Kubernetes Namespaces and Deployments

## What Namespaces Actually Are

A namespace is a way to carve up one cluster into isolated sections. Same cluster, different boundaries. Resources in `dev-ns` don't see resources in `staging-ns` unless you explicitly wire them together.

The practical use: separate environments (dev, staging, production) on the same cluster, or separate teams so they don't accidentally step on each other's deployments.

Kubernetes ships with four built-in namespaces:

- `default` — where everything lands if you don't specify a namespace
- `kube-system` — the cluster's own internals (API server, scheduler, etcd, coredns, kube-proxy)
- `kube-public` — readable by all users, mostly for cluster discovery
- `kube-node-lease` — node heartbeat tracking

The `kube-system` namespace had 14 pods running, all healthy at 43 minutes old:

```
coredns-7d764666f9-j4qh2              1/1   Running   0   43m
coredns-7d764666f9-smc4g              1/1   Running   0   43m
etcd-practice-cluster-control-plane   1/1   Running   0   43m
kindnet-bdx4m / -l2frk / -t9vzc / -zqvkg  (4 nodes, one each)
kube-apiserver-practice-cluster-control-plane
kube-controller-manager-practice-cluster-control-plane
kube-proxy-6wc2k / -rv87l / -xjtc6 / -xkpsz  (one per node)
kube-scheduler-practice-cluster-control-plane
```

Don't touch anything in `kube-system`.

---

## Creating Namespaces

Two ways — both work, pick whichever fits the situation.

**YAML (declarative):**

```yaml
# staging-ns.yml
apiVersion: v1
kind: Namespace
metadata:
  name: staging-ns
```

```bash
kubectl apply -f staging-ns.yml
kubectl apply -f dev-ns.yml
```

**Imperative:**

```bash
kubectl create ns production
```

After creating all three:

```
NAME              STATUS   AGE
default           Active   47m
dev-ns            Active   71s
kube-node-lease   Active   47m
kube-public       Active   47m
kube-system       Active   47m
local-path-storage Active  47m
nginx-ns          Active   37m
production        Active   5s
staging-ns        Active   75s
two-tier-ns       Active   3m50s
```

---

## Namespace Errors I Actually Hit

### Wrong namespace name

```bash
kubectl run nginx-dev --image=nginx:latest -n dev
# Error from server (NotFound): namespaces "dev" not found
```

I'd created `dev-ns`, not `dev`. The namespace name has to match exactly. Fixed:

```bash
kubectl run nginx-dev --image=nginx:latest -n dev-ns
# pod/nginx-dev created
```

### Typo: `kubect` instead of `kubectl`

```
Command 'kubect' not found, did you mean:
  command 'kubectx' from deb kubectx (0.9.5-1ubuntu0.4)
```

Fast typing. The terminal suggested `kubectx`, which is actually a useful tool for switching contexts — but that's not what I needed here.

### `-A` is the only way to see everything

```bash
kubectl get pods          # only shows current namespace (nginx-ns in my case)
kubectl get pods -A       # shows everything across all namespaces
```

This matters more than it sounds. When pods aren't showing up, the first thing to check is whether you're looking in the right namespace.

**`kubectl get pods -A` output:**

```
NAMESPACE       NAME                                              READY   STATUS
default         busybox-pod                                       1/1     Running
dev-ns          nginx-dev                                         1/1     Running
kube-system     coredns-7d764666f9-j4qh2                         1/1     Running
...
staging-ns      nginx-staging                                     0/1     ContainerCreating
```

nginx-staging was still pulling the image — ContainerCreating — while everything else was already up.

---

## Deployments

A Deployment tells Kubernetes: "keep N replicas of this pod running at all times." If a pod dies, the Deployment controller notices the gap and creates a replacement. That's the whole point.

A standalone pod has no one watching it. Delete it, it's gone. A Deployment always brings it back.

### nginx-deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: nginx-ns
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.24
        ports:
        - containerPort: 80
```

Key differences from a Pod manifest:

- `kind: Deployment` and `apiVersion: apps/v1` (not `v1`)
- `replicas: 3` — desired count
- `selector.matchLabels` — how the Deployment identifies which pods it owns. This must match `template.metadata.labels` exactly, or nothing works
- `template` — the pod blueprint. The Deployment stamps out pods from this

```bash
kubectl apply -f nginx-deployment.yml
# deployment.apps/nginx-app created

kubectl get deployments -n nginx-ns
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   3/3     3            3           4m42s
```

READY 3/3 means all three replicas are up and healthy. UP-TO-DATE means all three match the current spec. AVAILABLE means all three are serving traffic.

---

## Namespace Confusion with the Deployment

The manifest landed in `nginx-ns` because my active context namespace was still `nginx-ns` from Day 50. I thought it was going into `dev-ns` but:

```bash
kubectl get deployments -n dev
# No resources found in dev namespace.

kubectl get deployments -n dev-ns
# No resources found in dev-ns namespace.

kubectl get deployments -n nginx-ns
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   3/3     3            3           4m42s
```

The namespace in the YAML file is what matters, and my YAML had `namespace: nginx-ns`. Worth checking the file, not just assuming.

---

## Self-Healing

Deleted the standalone `nginx-dev` pod from dev-ns:

```bash
kubectl delete pod nginx-dev -n dev-ns
# pod "nginx-dev" deleted

kubectl get pods -n dev-ns
# No resources found in dev-ns namespace.
```

Gone. No controller, no replacement. That's a bare pod.

The Deployment pods in `nginx-ns` are different. If I delete one of those, the Deployment controller notices within seconds and creates a new one with a different name. The old name is gone, the new pod has a fresh suffix, but the replica count stays at whatever you declared.

---

## Scaling

```bash
# Scale up to 5
kubectl scale deployment nginx-app --replicas=5 -n nginx-ns
# deployment.apps/nginx-app scaled

kubectl get pods -n nginx-ns
NAME                         READY   STATUS    RESTARTS   AGE
nginx-app-5b76fb7664-8snnb   1/1     Running   0          16m
nginx-app-5b76fb7664-pchfm   1/1     Running   0          16m
nginx-app-5b76fb7664-sr4q2   1/1     Running   0          25s
nginx-app-5b76fb7664-vp8fl   1/1     Running   0          16m
nginx-app-5b76fb7664-xbkk8   1/1     Running   0          25s
```

The two new pods (`sr4q2`, `xbkk8`) have age 25s — Kubernetes spun them up within seconds of the scale command. Same ReplicaSet hash `5b76fb7664`, same pod template.

```bash
# Scale down to 2
kubectl scale deployment nginx-app --replicas=2 -n nginx-ns

kubectl get pods -n nginx-ns
NAME                         READY   STATUS    RESTARTS   AGE
nginx-app-5b76fb7664-8snnb   1/1     Running   0          22m
nginx-app-5b76fb7664-vp8fl   1/1     Running   0          22m
```

Kubernetes terminated the three newer pods and kept the two oldest (`8snnb`, `vp8fl`). The three extras just stopped — no drama, no data loss (there was no state to lose in this case).

---

## Rolling Update

Updated the image from `nginx:1.24` to `nginx:1.25`:

```bash
kubectl set image deployment/nginx-app nginx-container=nginx:1.25 -n nginx-ns
# deployment.apps/nginx-app image updated
```

During the rollout, a new pod from a new ReplicaSet appeared while the old ones were still running:

```
nginx-app-54c987db79-7gxp8   0/1   ContainerCreating   0   6s
nginx-app-5b76fb7664-8snnb   1/1   Running             0   26m
nginx-app-5b76fb7664-vp8fl   1/1   Running             0   26m
```

This is the zero-downtime part. Kubernetes brings up a new pod first, waits for it to be healthy, then terminates one old pod. It repeats until all replicas are on the new version. The rollout status command shows this live:

```bash
kubectl rollout status deployment/nginx-app -n nginx-ns
# Waiting for deployment "nginx-app" rollout to finish: 1 old replicas are pending termination...
# Waiting for deployment "nginx-app" rollout to finish: 1 old replicas are pending termination...
# deployment "nginx-app" successfully rolled out
```

After the rollout, both pods had the new hash `54c987db79`:

```
nginx-app-54c987db79-7gxp8   1/1   Running   0   69s
nginx-app-54c987db79-xrqh6   1/1   Running   0   38s
```

---

## Rollback

```bash
kubectl rollout history deployment/nginx-app -n nginx-ns
REVISION   CHANGE-CAUSE
1          <none>
2          <none>

kubectl rollout undo deployment/nginx-app -n nginx-ns
# deployment.apps/nginx-app rolled back

kubectl describe deployment nginx-app -n nginx-ns | grep Image
Image: nginx:1.24
```

Back to `nginx:1.24`. The `<none>` change-cause is because I didn't use `--record` (deprecated now) or annotate the deployment with a cause. In real usage, teams add a `kubernetes.io/change-cause` annotation to make the history readable.

---

## Cleanup Errors

Tried deleting namespaces with wrong names:

```bash
kubectl delete namespace dev staging production -ns
# Error: Command '-ns' not found...
# namespace "production" deleted
# Error: namespaces "dev" not found
# Error: namespaces "staging" not found
```

Three mistakes in one command — `-ns` was treated as a separate argument (not a flag), `dev` and `staging` didn't exist (the real names were `dev-ns` and `staging-ns`), so only `production` got deleted. The correct approach:

```bash
kubectl delete namespace dev-ns staging-ns two-tier-ns
```

After cleanup — `kubectl get pods -A` showed only the cluster internals and `busybox-pod` in default (the bare pod from Day 51 that nothing ever cleaned up):

```
default         busybox-pod   1/1   Running   4 (21m ago)   4h45m
```

4 restarts — busybox-pod kept exiting and coming back because WSL probably restarted or something killed the container. No controller is recreating it intentionally; `restartPolicy: Always` is the default for pods, which is what's keeping it around.

---

## What `kubectl get all -A` Shows

After all the cleanup:

```bash
kubectl get all -A
```

This is a good command to know. It shows pods, services, daemonsets, deployments, and replicasets across all namespaces in one shot. The cluster's own infrastructure appeared — coredns as a Deployment with a ReplicaSet behind it, kindnet and kube-proxy as DaemonSets (one pod per node, automatically), and two Services (kubernetes API and kube-dns).

Deployments don't manage pods directly — they manage ReplicaSets, which manage pods. The chain: Deployment → ReplicaSet → Pods. When you do a rolling update, Kubernetes creates a new ReplicaSet and scales it up while scaling the old one down.

---

## Key Commands

```bash
# Namespaces
kubectl get namespaces
kubectl apply -f namespace.yml
kubectl create ns production

# Pods across namespaces
kubectl get pods -n dev-ns
kubectl get pods -A

# Deployment
kubectl apply -f nginx-deployment.yml
kubectl get deployments -n nginx-ns
kubectl describe deployment nginx-app -n nginx-ns

# Scaling
kubectl scale deployment nginx-app --replicas=5 -n nginx-ns

# Rolling update
kubectl set image deployment/nginx-app nginx-container=nginx:1.25 -n nginx-ns
kubectl rollout status deployment/nginx-app -n nginx-ns
kubectl rollout history deployment/nginx-app -n nginx-ns
kubectl rollout undo deployment/nginx-app -n nginx-ns

# Check image after rollback
kubectl describe deployment nginx-app -n nginx-ns | grep Image

# Cleanup
kubectl delete deployment nginx-app -n nginx-ns
kubectl delete namespace dev-ns staging-ns
kubectl get all -A
```

---

`#90DaysOfDevOps` `#DevOpsKaJosh` `#TrainWithShubham`