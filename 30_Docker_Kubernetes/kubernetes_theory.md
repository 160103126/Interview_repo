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

*End of Kubernetes Theory — 7 questions covering Pods, Deployments (Self-healing), Services/Ingress (Networking), Autoscaling (HPA), and StatefulSets.*
