# Task 1: Recall the Kubernetes Story

## 1. Why was Kubernetes created?

Kubernetes was created to **automate the deployment, scaling, and management of containerized applications**.

Docker can run containers, but it **cannot manage containers across multiple servers**.

### Problems Kubernetes Solves

- Container orchestration across multiple machines
- Automatic scaling when traffic increases
- Self-healing (restart failed containers)
- Load balancing
- Rolling updates and rollbacks

### Key Idea

```

Docker = Container Runtime
Kubernetes = Container Orchestration System

```

---

## 2. Who Created Kubernetes?

- Created by: **Google Engineers**
- Inspired by: **Google's internal system called Borg**

### Important Clarification

- Google **developed Kubernetes**
- Later **donated it to the Cloud Native Computing Foundation (CNCF)** in **2015**

Therefore:

> Kubernetes was created by Google and inspired by Google's internal container management system **Borg**.

---

## 3. Meaning of Kubernetes

The word **Kubernetes** comes from **Greek** and means:

> **Helmsman** or **Ship Pilot**

This fits the analogy:

```

Containers = Ships
Kubernetes = System that steers and manages the ships

```

---

# Task 2: Kubernetes Architecture

Your architecture diagram is **mostly correct (~90%)**.

Below is the correct explanation.

---

# 1. Control Plane Components

The **Control Plane** manages the entire cluster.

### Components

- **API Server**
- **etcd**
- **Scheduler**
- **Controller Manager**

### Communication Model

```

Scheduler → API Server
Controller Manager → API Server
API Server ↔ etcd

```

✔ All control plane components communicate **through the API Server**.

---

# 2. Worker Node Components

Worker nodes run the actual applications.

### Components

- **kubelet** — Node agent that communicates with the API Server
- **kube-proxy** — Handles networking and service routing
- **Container Runtime (containerd)** — Runs containers
- **Pods** — Smallest deployable unit in Kubernetes

### Execution Flow

```

API Server → kubelet → container runtime → containers

```

---

# 3. Important Correction

Incorrect flow:

```

User → kube-proxy

```

This is **wrong**.

Correct flow:

```

User → kubectl → API Server

```

### Networking Flow

```

User → Service → kube-proxy → Pod

```

`kube-proxy` only manages **network routing**, not user access.

---

# 4. What Happens When You Run

```

kubectl apply -f pod.yaml

```

### Step-by-Step Flow

#### 1. kubectl sends request to API Server

```

kubectl → API Server

```

---

#### 2. API Server validates request

Checks:

- Authentication
- Authorization
- Schema validation

---

#### 3. API Server stores state in etcd

```

API Server → etcd

```

---

#### 4. Scheduler assigns a node

Scheduler detects a **new unscheduled Pod**.

```

Scheduler → API Server

```

Pod gets assigned to a node.

---

#### 5. kubelet detects new Pod

```

API Server → kubelet

```

kubelet watches the API server for changes.

---

#### 6. kubelet starts container

```

kubelet → containerd

```

containerd:

- pulls the image
- creates the container
- starts the pod

---

#### 7. Pod starts running

Networking handled by:

```

kube-proxy

```

---

# 5. What Happens if API Server Goes Down?

The cluster becomes **unmanageable**, but existing workloads continue running.

You **cannot**:

- create pods
- delete pods
- scale deployments
- run kubectl commands

But existing pods continue running because:

```

kubelet + container runtime keep containers alive

```

---

# 6. What Happens if a Worker Node Goes Down?

1. kubelet stops sending heartbeat
2. Control plane marks node as **NotReady**

After some time:

3. Controller Manager marks pods as **lost**
4. Scheduler schedules replacement pods on other nodes

This is **Kubernetes self-healing**.

---

# 7. Architecture Improvement

Add a **Service layer**:

```

User → Service → kube-proxy → Pod

```

Most traffic enters Kubernetes through **Services**.

---

# Final Verdict

Your architecture diagram is:

```

~90% correct

```

Correction needed:

```

❌ User → kube-proxy
✔ User → API Server

````

---

# kubectl Command Differences

## kubectl get vs kubectl describe

| Command | Purpose | Output |
|------|------|------|
| `kubectl get` | Lists Kubernetes resources | Short summary view |
| `kubectl describe` | Shows detailed resource information | Full details + events |

---

# kubectl get

Used to **quickly list resources**.

Example:

```bash
kubectl get pods
````

Example Output:

```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          5m
```

Shows:

* Name
* Status
* Restart count
* Age

More detailed view:

```bash
kubectl get pods -o wide
```

---

# kubectl describe

Used for **debugging specific resources**.

Example:

```bash
kubectl describe pod nginx-pod
```

Shows:

* Labels
* Node placement
* Container image
* Ports
* Environment variables
* Mounts
* Conditions
* **Events (very important for debugging)**

Example events:

```
Events:
Type     Reason     Age   From               Message
Normal   Scheduled  2m    default-scheduler  Successfully assigned
Normal   Pulled     2m    kubelet            Container image pulled
```

---

# Quick Interview Answer

```
kubectl get → overview of resources
kubectl describe → detailed debugging information
```

---

# Easy Way to Remember

```
get → overview
describe → debugging
```

---

