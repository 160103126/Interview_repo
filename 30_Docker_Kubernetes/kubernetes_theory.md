# Kubernetes (K8s) — Theory & Architecture

> Container Orchestration fundamentals: Pods, Deployments, Services, Ingress, and self-healing systems.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is Kubernetes (K8s) and why do we need it?

**Answer:**

Docker allows you to package and run a single container. Docker Compose allows you to run a few containers on a single machine. 

But what if you are Netflix, and you need to run 10,000 containers across 500 physical servers? What happens if a server crashes? How do you route traffic to the 500 servers evenly?

**Kubernetes is a Container Orchestration platform.** It automates the deployment, scaling, and management of containerized applications across a massive cluster of machines. It handles load balancing, auto-scaling, and "self-healing" (restarting failed containers automatically).

---

### Q2: What is the difference between a Node, a Cluster, and a Pod?

**Answer:**

- **Node:** A physical or virtual machine (e.g., an AWS EC2 instance).
- **Cluster:** A group of Nodes grouped together to share resources. Managed by a Control Plane (Master Node).
- **Pod:** The smallest deployable unit in Kubernetes. **K8s does not run containers directly; it runs Pods.** A Pod usually contains one container (e.g., a FastAPI app), but it can contain multiple tightly coupled containers that share the same IP address and storage (e.g., an app container + a logging sidecar container).

---

## 🟡 Medium (Intermediate)

### Q3: Explain the difference between a Deployment and a Pod.

**Answer:**

You should almost never create a Pod directly. Pods are mortal—if a Node dies, the Pods on it die and do not come back.

**Deployment:** A higher-level abstraction that *manages* Pods. 
You tell the Deployment: *"I want exactly 3 replicas of the FastAPI Pod running at all times."*
- If a Node crashes and takes down 1 Pod, the Deployment Controller notices the current state (2 Pods) does not match the desired state (3 Pods), and it automatically spins up a new Pod on a healthy Node.
- Deployments also handle rolling updates (e.g., updating from v1 to v2 with zero downtime).

---

### Q4: How do Pods communicate with each other? What is a Service?

**Answer:**

Pods are ephemeral. They die and get recreated constantly, and every time they are recreated, they get a new internal IP address. 
If your Frontend Pod tries to talk to your Backend Pod using a hardcoded IP, it will break when the Backend Pod restarts.

**Service:** An abstraction that provides a stable, static IP address and a DNS name (e.g., `http://backend-service`) that points to a group of dynamic Pods. 
The Service acts as an internal load balancer. It routes incoming traffic to whichever underlying Pods are currently healthy, abstracting away their changing IP addresses.

---

### Q5: What is Ingress vs LoadBalancer?

**Answer:**

How do users on the internet access your application inside the private K8s cluster?

1. **LoadBalancer Service:** Provisions a cloud-provider load balancer (like AWS ELB). It exposes exactly *one* service to the internet. If you have 10 microservices, you would need 10 expensive AWS Load Balancers.
2. **Ingress:** A smart API gateway / Reverse Proxy (like Nginx) that sits at the edge of the cluster. It requires only 1 cloud Load Balancer. It routes traffic based on URL paths or subdomains to different internal Services.
   - e.g., `myapp.com/api` -> routes to Backend Service.
   - e.g., `myapp.com/ui` -> routes to Frontend Service.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q6: Explain K8s Auto-scaling (HPA vs CA).

**Answer:**

1. **Horizontal Pod Autoscaler (HPA):** Scales the number of *Pods*. If CPU utilization of your Deployment hits 80%, HPA automatically changes the replica count from 3 to 10. (App-level scaling).
2. **Cluster Autoscaler (CA):** Scales the number of *Nodes*. If HPA requests 10 new Pods, but the existing physical Nodes are out of RAM, the Pods go into a `Pending` state. The Cluster Autoscaler notices this and provisions brand new EC2 instances (Nodes) from AWS, adds them to the cluster, and schedules the pending Pods on them. (Infrastructure-level scaling).

---

### Q7: What are StatefulSets? Why are they needed for Databases?

**Answer:**

Deployments are designed for **stateless** applications (like a Python web API). In a Deployment, all Pods are identical and interchangeable.

If you try to run a distributed database (like PostgreSQL or Cassandra) using a standard Deployment, it will fail miserably. Databases require:
- Stable, unique network identifiers (e.g., `db-0`, `db-1`).
- Persistent storage that remains attached to the *exact same Pod*, even if it gets rescheduled to a different Node.

**StatefulSets:** A controller specifically for stateful apps. It guarantees strict ordering and uniqueness. If `db-1` crashes, K8s will spin up a new Pod, strictly name it `db-1`, and reattach the exact same storage volume that belonged to the old `db-1`.

---

### Q8: How do you manage Configuration and Secrets in Kubernetes?

**Answer:**

Hardcoding database passwords or API keys in Docker images is a massive security risk. Kubernetes provides two primitives:

**ConfigMaps (Non-sensitive data):**
Store configuration data as key-value pairs (e.g., `DATABASE_HOST=postgres-service`, `LOG_LEVEL=INFO`).
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  LOG_LEVEL: "INFO"
```

**Secrets (Sensitive data):**
Store passwords, API keys, and TLS certificates. Values are Base64-encoded (NOT encrypted by default — you must enable encryption-at-rest).
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64 encoded
```

Both can be injected into Pods as **environment variables** or **mounted files**.

---

### Q9: Explain Persistent Volumes (PV) and Persistent Volume Claims (PVC).

**Answer:**

Pods are ephemeral — when they die, their filesystem is destroyed. Databases running in Pods would lose all data.

**Persistent Volume (PV):** A piece of storage provisioned by an admin (e.g., a 100GB SSD on AWS EBS or GCP Persistent Disk). It exists independently of any Pod.

**Persistent Volume Claim (PVC):** A *request* for storage by a Pod. The PVC says "I need 50GB of fast SSD storage." Kubernetes finds a matching PV and binds them together.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 50Gi
```

**Dynamic Provisioning:** In cloud environments, you don't pre-create PVs. You define a `StorageClass`, and Kubernetes automatically provisions the cloud disk when a PVC is created.

---

### Q10: What are Resource Requests and Limits? Why are they critical?

**Answer:**

Without resource constraints, a single rogue Pod can consume all CPU/Memory on a Node, starving other Pods.

```yaml
resources:
  requests:
    cpu: "500m"      # Guaranteed minimum: 0.5 CPU cores
    memory: "256Mi"  # Guaranteed minimum: 256 MB RAM
  limits:
    cpu: "2000m"     # Maximum allowed: 2 CPU cores
    memory: "1Gi"    # Maximum allowed: 1 GB RAM (OOMKilled if exceeded)
```

**Requests:** The scheduler uses these to decide *which Node* has enough capacity to place the Pod. If no Node can satisfy the request, the Pod stays in `Pending`.

**Limits:** The hard ceiling. If a Pod exceeds its memory limit, Kubernetes kills it (`OOMKilled`). If it exceeds its CPU limit, it is throttled (slowed down, not killed).

**For ML workloads:** GPU resources are also requestable:
```yaml
resources:
  limits:
    nvidia.com/gpu: 1  # Request exactly 1 GPU
```

---

### Q11: What are Readiness and Liveness Probes?

**Answer:**

Probes are health checks that Kubernetes uses to manage Pod lifecycle.

**Liveness Probe:** "Is this Pod alive?"
- If the probe fails, Kubernetes **restarts** the Pod (kills it and creates a new one).
- *Use case:* Detecting deadlocks where the process is running but stuck.

**Readiness Probe:** "Is this Pod ready to receive traffic?"
- If the probe fails, Kubernetes **removes the Pod from the Service's load balancer** (stops routing traffic to it) but does NOT restart it.
- *Use case:* An ML model server that takes 30 seconds to load weights into GPU memory. During loading, it's alive but not ready.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 15

readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 30  # Wait for model to load
  periodSeconds: 5
```

**Startup Probe (K8s 1.18+):** For slow-starting applications. Disables liveness/readiness probes until the startup probe succeeds, preventing premature restarts.

---

### Q12: What is Helm? Why is it called "the package manager for Kubernetes"?

**Answer:**

Deploying a production application on K8s requires writing many YAML files: Deployment, Service, Ingress, ConfigMap, Secret, PVC, HPA, etc. Managing these across dev/staging/production environments is a nightmare.

**Helm** solves this by packaging all K8s manifests into a single deployable unit called a **Chart**.

**Key Concepts:**
- **Chart:** A bundle of pre-configured Kubernetes resource manifests (like an `apt` package for Linux).
- **Values:** A `values.yaml` file that parameterizes the chart. Different environments override different values.
- **Release:** A running instance of a Chart on a cluster.

```bash
# Install a production-ready PostgreSQL with one command:
helm install my-db bitnami/postgresql --set auth.postgresPassword=secret

# Upgrade to a new version:
helm upgrade my-db bitnami/postgresql --set image.tag=16.0

# Rollback if broken:
helm rollback my-db 1
```

**Why it matters for interviews:** It shows you understand production K8s deployment patterns, not just writing standalone YAML files.

---

*End of Kubernetes Theory — 12 comprehensive questions covering Pods, Deployments, Services, Ingress, Autoscaling, StatefulSets, ConfigMaps/Secrets, Persistent Volumes, Resource Management, Health Probes, and Helm.*

