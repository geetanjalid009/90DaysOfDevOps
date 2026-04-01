  ```mermaid
graph TD
    User[User] --> Ingress[Ingress]
    Ingress --> Service[Service]
    Service --> Pod1[Pod 1]
    Service --> Pod2[Pod 2]
    Pod1 --> Node1[Worker Node 1]
    Pod2 --> Node2[Worker Node 2]
    Node1 --> Cluster[Kind / Kubernetes Cluster]
    Node2 --> Cluster
