# kubernetes_nodeport_service_and_ingress

# Kubernetes NodePort, Service & Ingress — Complete Guide

A simple practical guide to understand:

* Kubernetes Service
* NodePort
* `port`, `targetPort`, and `nodePort`
* Multiple Nodes and same NodePort
* Private Nodes and external access
* Node failure and replacement
* LoadBalancer
* Ingress
* Ingress Controller
* AWS EKS architecture

---

# 1. Kubernetes mein Basic Problem

Suppose Kubernetes cluster mein application ke multiple Pods hain:

```text
Kubernetes Cluster
│
├── Node 1
│   └── Pod
│
├── Node 2
│   └── Pod
│
└── Node 3
    └── Pod
```

Pods ke IP addresses generally dynamic hote hain.

For example:

```text
Pod 1 → 10.244.1.10
Pod 2 → 10.244.2.10
Pod 3 → 10.244.3.10
```

Agar Pod restart ho jaye, uska IP change ho sakta hai.

Isliye clients ko directly Pod IP nahi dena chahiye.

Yahan **Kubernetes Service** ka role aata hai.

---

# 2. Kubernetes Service

Service Pods ke liye ek **stable network endpoint** provide karti hai.

```text
Client
   |
   v
Service
   |
   +------ Pod 1
   |
   +------ Pod 2
   |
   +------ Pod 3
```

Service Pods ko selector ke through identify karti hai:

```yaml
selector:
  app: nginx
```

Agar Pods ke labels hain:

```yaml
labels:
  app: nginx
```

to Service in Pods ko backend ke taur par use karegi.

---

# 3. Service ke Important Ports

Kubernetes Service mein commonly teen ports samajhna important hai:

```text
port
targetPort
nodePort
```

Example:

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30008
```

Inka meaning:

| Port         | Meaning                                          |
| ------------ | ------------------------------------------------ |
| `targetPort` | Pod/application kis port par listen kar rahi hai |
| `port`       | Kubernetes Service ka port                       |
| `nodePort`   | Node par externally exposed port                 |

---

# 4. `targetPort`

Suppose application Pod ke andar port `8080` par chal rahi hai:

```text
Pod
└── Application
    └── :8080
```

To:

```yaml
targetPort: 8080
```

ka matlab:

> Service traffic ko Pod ke port `8080` par forward karegi.

---

# 5. `port`

Suppose:

```yaml
port: 80
```

To ye Service ka port hai.

Conceptually:

```text
Service:80
      |
      v
Pod:8080
```

---

# 6. `nodePort`

Agar Service ka type `NodePort` hai:

```yaml
type: NodePort
```

to Kubernetes Node par ek port expose karta hai.

Example:

```yaml
nodePort: 30008
```

External request:

```text
NodeIP:30008
```

par aa sakti hai.

---

# 7. NodePort Complete Example

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-service

spec:
  type: NodePort

  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30008
```

Suppose Node ka IP:

```text
192.168.1.10
```

To external request:

```text
http://192.168.1.10:30008
```

ho sakti hai.

---

# 8. Most Important Traffic Flow

Ye flow yaad rakho:

```text
External User
      |
      | NodeIP:NodePort
      v
   Kubernetes Node
      |
      v
    Service
      |
      | targetPort
      v
      Pod
```

Example:

```text
External User
      |
      | 192.168.1.10:30008
      v
     Node
      |
      | NodePort 30008
      v
   Service:80
      |
      | targetPort 8080
      v
    Pod:8080
```

### Important

External user **pehle Service ke port 80 par nahi aata**.

External user:

```text
NodeIP:30008
```

par aata hai.

Phir Kubernetes us traffic ko Service ke through Pod tak route karta hai.

---

# 9. Simple Formula

Agar:

```yaml
port: 80
targetPort: 8080
nodePort: 30008
```

to yaad rakho:

```text
External User
     ↓
NodeIP:30008
     ↓
Service:80
     ↓
Pod:8080
```

---

# 10. Multiple Nodes mein NodePort

Suppose cluster mein 3 Nodes hain:

```text
Node 1 → 10.0.1.10
Node 2 → 10.0.1.11
Node 3 → 10.0.1.12
```

Aur Service:

```yaml
type: NodePort

ports:
  - port: 80
    targetPort: 8080
    nodePort: 30008
```

To same NodePort teeno Nodes par available hota hai:

```text
10.0.1.10:30008
10.0.1.11:30008
10.0.1.12:30008
```

### Node IP different

```text
10.0.1.10
10.0.1.11
10.0.1.12
```

### NodePort same

```text
30008
30008
30008
```

---

# 11. Kubernetes ko Teen Different Ports kyun chahiye?

Example:

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30008
```

Traffic:

```text
NodePort
  30008
     |
     v
Service
  port 80
     |
     v
Pod
targetPort 8080
```

Iska matlab:

* `30008` → Node par entry point
* `80` → Service ka port
* `8080` → Application/Pod ka port

---

# 12. NodePort Default Range

By default Kubernetes NodePort range generally:

```text
30000 - 32767
```

hoti hai.

Example:

```yaml
nodePort: 30008
```

Valid hai.

Normally:

```yaml
nodePort: 8080
```

valid nahi hoga unless cluster ka NodePort range customize kiya gaya ho.

---

# 13. NodePort ki Problem in Production

Ab ek important real-world scenario.

Suppose:

```text
Node 1
10.0.1.10
```

NodePort:

```text
30008
```

External user access karta hai:

```text
10.0.1.10:30008
```

Ab Node crash ho gaya:

```text
Node 1
10.0.1.10
     ❌
```

AWS/EKS new Node create karta hai:

```text
Node 2
10.0.1.35
```

Ab old address:

```text
10.0.1.10:30008
```

available nahi hoga.

New node ka IP different hai:

```text
10.0.1.35:30008
```

Agar external user directly Node IP use kar raha tha, to endpoint change ho gaya.

### Isi wajah se production mein users ko individual worker-node IPs directly expose karna generally desirable architecture nahi hai.

---

# 14. Private Subnet ka Scenario

AWS EKS mein worker Nodes commonly private subnets mein hote hain.

Example:

```text
VPC
│
├── Public Subnet
│
│   └── Load Balancer
│
└── Private Subnet
    │
    ├── Node 1
    ├── Node 2
    └── Node 3
```

Private node ka IP:

```text
10.0.1.10
```

Internet se directly reachable nahi hota.

So:

```text
Internet
    |
    X
    |
Private Node
```

direct access nahi hoga.

---

# 15. Production mein Load Balancer

Instead of exposing Node IP directly:

```text
Internet
   |
   v
NodeIP:30008
   |
   v
Node
```

we can use:

```text
Internet
   |
   v
Load Balancer
   |
   v
Private Nodes
   |
   v
Service
   |
   v
Pods
```

External user ko Node IP nahi pata.

User kisi stable endpoint ko use karta hai:

```text
https://myapp.example.com
```

---

# 16. Node Crash ke Baad Load Balancer

Initial state:

```text
Load Balancer
      |
      +------ Node 1
      |
      +------ Node 2
      |
      +------ Node 3
```

Suppose Node 2 crash:

```text
Load Balancer
      |
      +------ Node 1
      |
      +------ Node 2 ❌
      |
      +------ Node 3
```

Node group / autoscaling mechanism new Node create kar sakta hai:

```text
Node 4
```

Updated architecture:

```text
Load Balancer
      |
      +------ Node 1
      |
      +------ Node 3
      |
      +------ Node 4
```

External user ke liye endpoint same rehta hai:

```text
https://myapp.example.com
```

User ko Node IP change hone ka concern nahi hota.

---

# 17. Important AWS/EKS Correction

Kubernetes control plane generally khud EC2 instance create nahi karta.

EKS mein Node replacement commonly:

```text
Node failure
    ↓
Managed Node Group / Auto Scaling
    ↓
New EC2 instance
    ↓
New Kubernetes Node
    ↓
Pods scheduled
```

Control plane Kubernetes workloads ko manage/orchestrate karta hai, jabke EC2 instance provisioning AWS infrastructure/autoscaling mechanisms handle karte hain.

---

# 18. NodePort vs LoadBalancer

## NodePort

```text
Internet
   |
   v
NodeIP:30008
   |
   v
Service
   |
   v
Pod
```

External user ko Node IP use karna pad sakta hai.

---

## LoadBalancer

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Service
   |
   v
Pods
```

User stable Load Balancer endpoint/domain use karta hai.

Example:

```text
https://myapp.example.com
```

---

# 19. Multiple Applications ka Problem

Suppose ek cluster mein:

```text
Medical Coding
Billing
AI Assistant
```

hain.

Agar har application ke liye separate `LoadBalancer` Service bana dein:

```text
Internet
   |
   +---- LoadBalancer → Medical
   |
   +---- LoadBalancer → Billing
   |
   +---- LoadBalancer → AI Assistant
```

Multiple Load Balancers provision ho sakte hain.

Is scenario mein **Ingress** useful hota hai.

---

# 20. Ingress kya hai?

Ingress Kubernetes ka ek API object hai jo HTTP/HTTPS traffic ke routing rules define karta hai.

Example:

```text
medical.example.com
        ↓
Medical Service

billing.example.com
        ↓
Billing Service

assistant.example.com
        ↓
AI Assistant Service
```

Ingress ka kaam basically ye rules define karna hai:

```text
"Is hostname/path ki request kis Service ko jani chahiye?"
```

---

# 21. Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: app-ingress

spec:
  rules:

    - host: medical.example.com
      http:
        paths:
          - path: /
            pathType: Prefix

            backend:
              service:
                name: medical-service
                port:
                  number: 80

    - host: billing.example.com
      http:
        paths:
          - path: /
            pathType: Prefix

            backend:
              service:
                name: billing-service
                port:
                  number: 80
```

Ab:

```text
medical.example.com
        ↓
medical-service
```

and:

```text
billing.example.com
        ↓
billing-service
```

---

# 22. Ingress khud Traffic Handle karta hai?

**Nahi.**

Ye bahut important distinction hai.

### Ingress

Rules/configuration hai:

```text
medical.example.com → Medical Service
billing.example.com → Billing Service
```

### Ingress Controller

Actual component hai jo traffic receive aur route karta hai.

---

# 23. Ingress Controller

Examples:

* AWS Load Balancer Controller
* NGINX Ingress Controller
* Traefik
* HAProxy

Simple analogy:

```text
Ingress
=
Routing instructions

Ingress Controller
=
Actual traffic handler
```

Ingress Controller Ingress resources ko watch karta hai aur un rules ke mutabiq traffic route karta hai.

---

# 24. Complete Ingress Flow

```text
User
 |
 | https://medical.example.com
 v
DNS
 |
 v
Load Balancer
 |
 v
Ingress Controller
 |
 v
Ingress Rules
 |
 v
Medical Service
 |
 v
Medical Pod
```

---

# 25. Host-Based Routing

Example:

```text
medical.example.com
billing.example.com
assistant.example.com
```

Routing:

```text
medical.example.com
        ↓
Medical Service

billing.example.com
        ↓
Billing Service

assistant.example.com
        ↓
Assistant Service
```

---

# 26. Path-Based Routing

Ek domain ke under multiple paths bhi use kar sakte hain:

```text
example.com/medical
example.com/billing
example.com/assistant
```

Ingress:

```text
/medical
   ↓
medical-service

/billing
   ↓
billing-service

/assistant
   ↓
assistant-service
```

Example:

```yaml
rules:
  - host: example.com
    http:
      paths:

        - path: /medical
          pathType: Prefix
          backend:
            service:
              name: medical-service
              port:
                number: 80

        - path: /billing
          pathType: Prefix
          backend:
            service:
              name: billing-service
              port:
                number: 80
```

---

# 27. AWS EKS + Ingress

AWS EKS mein ek common architecture:

```text
                       INTERNET
                           |
                           v
                    DNS / Domain
                           |
                           v
                +--------------------+
                | AWS Load Balancer  |
                +---------+----------+
                          |
                          v
                Ingress Controller
                          |
                          v
                  Ingress Rules
                          |
          +---------------+---------------+
          |               |               |
          v               v               v
    Medical SVC      Billing SVC     AI SVC
          |               |               |
         Pods            Pods            Pods
```

Worker Nodes private subnets mein ho sakte hain.

---

# 28. AWS Load Balancer Controller

EKS mein **AWS Load Balancer Controller** Kubernetes resources ko AWS load-balancing resources ke saath integrate karta hai.

Conceptually:

```text
Kubernetes Ingress
        |
        v
AWS Load Balancer Controller
        |
        v
AWS Application Load Balancer
        |
        v
Kubernetes Services / Pods
```

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: medical-ingress

spec:
  ingressClassName: alb

  rules:
    - host: medical.example.com

      http:
        paths:
          - path: /
            pathType: Prefix

            backend:
              service:
                name: medical-service
                port:
                  number: 80
```

---

# 29. Complete Production Architecture

A typical EKS-style architecture can look like:

```text
                         INTERNET
                            |
                            v
                    medical.example.com
                            |
                            v
                   +----------------+
                   | DNS            |
                   +-------+--------+
                           |
                           v
                   +----------------+
                   | AWS ALB        |
                   | Public          |
                   +-------+--------+
                           |
                           v
                +----------------------+
                | Ingress Controller /  |
                | AWS LB integration    |
                +----------+-----------+
                           |
                    Ingress Rules
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
    Medical Service   Billing Service   AI Service
          |                |                |
       +--+--+          +--+--+         +--+--+
       |     |          |     |         |     |
      Pod   Pod        Pod   Pod       Pod   Pod
       |     |          |     |         |     |
       +-----+----------+-----+---------+-----+
                         |
                  Private Subnets
                         |
                +--------+--------+
                |        |        |
              Node 1   Node 2   Node 3
```

---

# 30. NodePort vs Ingress

| Feature             | NodePort                         | Ingress                       |
| ------------------- | -------------------------------- | ----------------------------- |
| Purpose             | Expose Service through Node port | HTTP/HTTPS routing            |
| Port                | Usually `30000-32767`            | Usually `80/443` externally   |
| Routing             | Service-level                    | Host/path-based               |
| Node IP exposed     | Often involved                   | User normally doesn't need it |
| Multiple apps       | Less convenient                  | Very convenient               |
| TLS                 | Not its main purpose             | Commonly handled              |
| Layer               | Mainly L4-style Service exposure | Layer 7 HTTP/HTTPS            |
| Controller required | No separate Ingress Controller   | Yes                           |

---

# 31. Service Types

Kubernetes commonly provides:

```text
ClusterIP
NodePort
LoadBalancer
```

### ClusterIP

Default Service type.

```text
Pod
 |
 v
Service
 |
 v
Other Pod
```

Mainly cluster-internal access.

---

### NodePort

```text
External/Node Client
        |
        v
NodeIP:NodePort
        |
        v
Service
        |
        v
Pod
```

---

### LoadBalancer

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Service
   |
   v
Pods
```

---

# 32. NodePort vs Ingress — Important Difference

Do not think:

```text
NodePort = Ingress
```

They solve different problems.

### NodePort

Question:

> "How can I expose this Service through a port on Kubernetes Nodes?"

Answer:

```text
NodeIP:30008
```

### Ingress

Question:

> "How should HTTP/HTTPS requests be routed to different Services based on hostname/path?"

Answer:

```text
medical.example.com → medical-service

billing.example.com → billing-service
```

---

# 33. Final Mental Model

Remember the hierarchy:

```text
                    EXTERNAL USER
                         |
                         v
                 Load Balancer / ALB
                         |
                         v
                Ingress Controller
                         |
                         v
                     Ingress
                  Routing Rules
                         |
             +-----------+-----------+
             |                       |
             v                       v
      Medical Service          Billing Service
             |                       |
             v                       v
           Pods                    Pods
```

And for a direct NodePort setup:

```text
External User
      |
      | NodeIP:30008
      v
     Node
      |
      v
   Service:80
      |
      v
   Pod:8080
```

---

# 34. One-Line Definitions for Interviews

### Service

> A Kubernetes Service provides a stable network endpoint for a group of Pods.

### NodePort

> NodePort exposes a Service on a specific port on each Kubernetes Node, allowing traffic to reach the Service through `NodeIP:NodePort`.

### Ingress

> Ingress is a Kubernetes API resource that defines HTTP/HTTPS routing rules for directing external traffic to Services.

### Ingress Controller

> An Ingress Controller is the actual component that implements the Ingress rules and handles HTTP/HTTPS traffic routing.

### LoadBalancer

> A LoadBalancer Service exposes an application through a cloud-provider load balancer and provides a stable external entry point.

---

# 35. Most Important Diagram to Remember

```text
                 USER
                  |
                  |
          https://app.example.com
                  |
                  v
             DNS / Domain
                  |
                  v
          AWS Load Balancer
                  |
                  v
         Ingress Controller
                  |
                  v
           Ingress Rules
                  |
         +--------+--------+
         |                 |
         v                 v
   Medical Service   Billing Service
         |                 |
         v                 v
       Pods              Pods
         |
         |
    Private Nodes
```

The user does **not** need to know:

```text
Node IP
Pod IP
Pod name
NodePort
```

The user only needs the stable application endpoint:

```text
https://app.example.com
```

---

# 36. Quick Revision

```text
Pod
 ↓
Service
 ↓
Stable internal endpoint
```

For NodePort:

```text
NodeIP:NodePort
       ↓
    Service
       ↓
      Pod
```

For production-style external HTTP/HTTPS access:

```text
Internet
   ↓
DNS
   ↓
Load Balancer
   ↓
Ingress Controller
   ↓
Ingress Rules
   ↓
Service
   ↓
Pods
```

### Golden Rule

> **Pods are temporary, Node IPs can change, but users should normally interact with a stable Service/Load Balancer/Ingress endpoint rather than individual Pod or Node IPs.**
