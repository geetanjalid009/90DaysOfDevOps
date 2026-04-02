# Day 53 – Kubernetes Services

## The Problem I Was Trying to Solve

Every Pod in Kubernetes gets its own IP address. That sounds fine until you realize those IPs are completely temporary. Restart a Pod, it gets a new IP. Scale up, new Pods get new IPs. Run three replicas and suddenly you have three different IPs with no obvious way to load-balance between them.

Services fix this. A Service gives your Pods a single stable IP and DNS name that sticks around no matter how many times those Pods come and go underneath it. Traffic hits the Service, the Service figures out which Pod to send it to.

```
[Client] --> [Service (stable IP)] --> [Pod 1]
                                   --> [Pod 2]
                                   --> [Pod 3]
```

---

## Setup

I put everything in a custom namespace called `app-ns` to keep things organized. Then deployed three nginx Pods using a Deployment with `replicas: 3` and the label `app: web-app`.

Three Pods came up across three different worker nodes — each with its own IP:

| Pod | IP | Node |
|-----|----|------|
| app-deployment-6cffb4b956-jj8sh | 10.244.3.7 | practice-cluster-worker2 |
| app-deployment-6cffb4b956-kjx4c | 10.244.2.9 | practice-cluster-worker3 |
| app-deployment-6cffb4b956-z8tkd | 10.244.1.9 | practice-cluster-worker |

These IPs are the problem. Restart any of these Pods, the IPs change. A Service is what gives you a stable way to reach all three without caring which Pod actually handles the request.

---

## ClusterIP Service

ClusterIP is the default Service type. It creates a stable internal IP that only works inside the cluster — no outside access at all.

The key field is `selector`. It tells the Service which Pods to route traffic to. In my case, `app: web-app` — any Pod with that label gets traffic. If the selector doesn't match any Pod labels, the Service routes to nothing and fails silently. That's one of the more frustrating bugs to track down.

After applying:

```
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
web-app-clusterip  ClusterIP   10.96.103.194    <none>        80/TCP    30s
```

That `10.96.103.194` is now stable regardless of what happens to individual Pods underneath it.

### Testing from Inside the Cluster

ClusterIP is internal-only, so to test it I had to spin up a temporary Pod:

```bash
kubectl run test-client --image=busybox:latest --rm -it --restart=Never -- sh
wget -qO- http://web-app-clusterip
```

Got back the nginx welcome page. The Service routed my request to one of the three Pods.

---

## DNS Discovery

Kubernetes runs an internal DNS server. Every Service gets a DNS record automatically in this format:

```
<service-name>.<namespace>.svc.cluster.local
```

I ran `nslookup` from inside a test Pod in the `app-ns` namespace:

```
Name:    web-app-clusterip.app-ns.svc.cluster.local
Address: 10.96.103.194
```

That IP matches the CLUSTER-IP from `kubectl get services` exactly — DNS is just resolving the name to the same stable IP.

There were `NXDOMAIN` errors when I tried shorter forms like `web-app-clusterip.svc.cluster.local`. That's because my resources were in `app-ns`, not `default`. The short name `web-app-clusterip` works only within the same namespace. For cross-namespace calls, you need the full `<service>.<namespace>.svc.cluster.local` format.

---

## NodePort Service

NodePort opens a specific port on every node in the cluster, making the Service reachable from outside. The valid range is 30000–32767. I used port 30080.

Traffic flow: `<NodeIP>:30080` → Service → Pod:80

After applying:

```
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
web-app            NodePort    10.96.101.127    <none>        80:30080/TCP     17s
web-app-clusterip  ClusterIP   10.96.103.194    <none>        80/TCP           120m
```

Got my node IPs with `kubectl get nodes -o wide`. Control-plane was at `172.18.0.4`. Tested it:

```bash
curl 172.18.0.4:30080
```

Nginx welcome page came back. NodePort works well for local testing and development. Not something you'd expose directly in production.

---

## LoadBalancer Service

In a real cloud environment, LoadBalancer provisions an actual external load balancer through the cloud provider and assigns it a public IP automatically.

On my local Kind cluster, EXTERNAL-IP stays at `<pending>` — there's no cloud provider to fulfill the request. Expected behavior.

On AWS, GCP, or Azure, Kubernetes calls the cloud's API, provisions a load balancer, and populates EXTERNAL-IP with a real public IP or hostname without any manual steps.

---

## How the Three Types Stack

```bash
kubectl get services -o wide
```

```
NAME                  TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          SELECTOR
web-app               NodePort       10.96.101.127    <none>        80:30080/TCP     app=web-app
web-app-clusterip     ClusterIP      10.96.103.194    <none>        80/TCP           app=web-app
web-app-loadbalancer  LoadBalancer   <pending>        <pending>     80/TCP           app=web-app
```

The three types aren't really separate things — they build on each other:

- **ClusterIP** is the base. Internal IP, internal DNS, load balancing across matching Pods.
- **NodePort** adds a port on every node. Still has a ClusterIP underneath.
- **LoadBalancer** adds a cloud-managed entry point. Still has a NodePort and a ClusterIP underneath.

| Type | Reachable From | Typical Use |
|------|---------------|-------------|
| ClusterIP | Inside the cluster only | Service-to-service communication |
| NodePort | Outside via `<NodeIP>:<NodePort>` | Local testing, direct node access |
| LoadBalancer | Outside via cloud load balancer | Production on cloud providers |

---

## What Endpoints Are

A Service doesn't directly know which Pods to send traffic to — it uses an Endpoints object. This gets created and updated automatically whenever Pods matching the selector come up or go down.

```bash
kubectl get endpoints web-app-clusterip -n app-ns
```

This shows the actual Pod IPs currently receiving traffic. If a Service isn't routing correctly, checking endpoints is the first thing to do. An empty list almost always means the selector doesn't match any Pod labels.

---

## Cleanup

After deleting the Deployment and all three Services, only the built-in `kubernetes` service in the default namespace remained. Clean state.

---

## Screenshots

- Pods running across worker nodes with individual IPs
- Deployment YAML with namespace and label configuration
- ClusterIP Service applied and `kubectl get services` output
- ClusterIP tested from a busybox Pod — nginx welcome page returned
- DNS test showing `nslookup` resolving to the correct ClusterIP
- Full DNS name `web-app-clusterip.app-ns.svc.cluster.local` resolving
- NodePort Service + `kubectl get services -o wide` + curl to node IP:30080