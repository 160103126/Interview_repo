# Google Cloud Platform (GCP) for Machine Learning

> Mapping ML concepts to GCP managed services: Vertex AI, BigQuery ML, Dataflow, and Pub/Sub.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is Google Cloud Storage (GCS) vs BigQuery?

**Answer:**

- **Cloud Storage (GCS):** Object storage (like AWS S3). You store unstructured data here (raw images, audio files, PDFs, or massive raw CSVs). This is your Data Lake.
- **BigQuery:** A fully-managed, serverless Enterprise Data Warehouse. You store structured, tabular data here. It allows you to run SQL queries across petabytes of data in seconds.

*ML Workflow:* Raw data lands in GCS -> Cleaned and structured -> Loaded into BigQuery for analysis and feature engineering.

---

### Q2: What is Vertex AI?

**Answer:**

Vertex AI is Google's unified MLOps platform (the equivalent of AWS SageMaker). 

Before Vertex, GCP had disjointed services (AI Platform Training, AI Platform Prediction, AutoML). Vertex AI combined them all into one UI/API to manage the entire ML lifecycle:
- Data labeling.
- Jupyter Notebooks (Vertex Workbench).
- Training (Custom code or AutoML).
- Model Registry.
- Deployment (Vertex Endpoints).

---

## 🟡 Medium (Intermediate)

### Q3: What is BigQuery ML (BQML)? Why use it?

**Answer:**

Normally, to train a model, a data scientist must extract data from the DB, move it to a Jupyter Notebook, load it into Pandas, and train with Scikit-Learn. Moving 500GB of data over the network is slow, expensive, and risks memory crashes.

**BigQuery ML** allows you to train and execute machine learning models directly inside BigQuery using standard SQL queries.
- `CREATE MODEL my_model OPTIONS(model_type='xgboost') AS SELECT * FROM my_table`
- *Benefits:* Zero data movement. Massively parallelized by BigQuery's compute engine. Excellent for quick baselines (Linear Regression, K-Means, XGBoost).

---

### Q4: Compare Cloud Run, GKE, and Compute Engine for Model Deployment.

**Answer:**

If you have a trained model wrapped in a FastAPI Docker container, where do you host it on GCP?

1. **Compute Engine (GCE):** Virtual Machines (IaaS). You manage the OS, updates, and scaling. *Best for:* Highly customized, complex legacy setups.
2. **Google Kubernetes Engine (GKE):** Managed Kubernetes. *Best for:* Large enterprise microservices architectures requiring strict control over networking, GPUs, and inter-service communication.
3. **Cloud Run:** Serverless Container Hosting. You give it a Docker image, and Google handles everything. It scales from 0 to 1000 instances instantly based on HTTP traffic. *Best for:* 90% of modern APIs. (Note: Cloud Run now supports GPUs).
4. **Vertex AI Endpoints:** Purpose-built for ML. Provides built-in model monitoring (data drift detection) and easy A/B testing splits.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q5: Design a Real-Time Streaming Data Pipeline on GCP.

**Answer:**

**The Scenario:** You need to ingest millions of real-time clicks from a website, process them, and feed them to an ML model.

**The GCP Stack:**
1. **Pub/Sub:** The messaging queue (like Kafka). Ingests the real-time clicks. It decouples the website (producer) from the processing engine (consumer).
2. **Dataflow (Apache Beam):** The stream processing engine. It reads from Pub/Sub, performs transformations (e.g., calculates "clicks in the last 5 minutes" using windowing), and outputs the features.
3. **Bigtable or Memorystore (Redis):** The ultra-low latency NoSQL database where Dataflow writes the real-time features.
4. **Vertex AI Endpoint:** The REST API hosting the model. When a prediction is requested, it fetches the real-time features from Bigtable and returns the prediction.

---

### Q6: Explain Vertex AI Pipelines vs Cloud Composer.

**Answer:**

Orchestrating MLOps requires DAGs (Directed Acyclic Graphs).

- **Cloud Composer:** Google's managed Apache Airflow service. It is a general-purpose orchestrator. *Best for:* Heavy data engineering (ETL pipelines, moving data from GCS to BigQuery). It runs continuously.
- **Vertex AI Pipelines:** Built specifically for ML pipelines based on Kubeflow Pipelines (KFP). Every step (Data Prep, Train, Evaluate, Deploy) runs as a distinct Docker container on serverless infrastructure. 
  - *Best for:* Training loops. It has native artifact tracking (metadata lineage). If the "Train" step fails, it remembers the "Data Prep" output, saving hours on retries.

---

### Q7: Explain GCP IAM (Identity and Access Management) and Service Accounts.

**Answer:**

IAM controls **who** (identity) has **what** access (role) to **which** resource.

**Key Concepts:**
- **Principal:** Who is requesting access? (A user, a group, or a Service Account).
- **Role:** A collection of permissions. (e.g., `roles/storage.objectViewer` = can read GCS files but not write).
- **Policy Binding:** Binds a Principal to a Role on a Resource. "Give `user@gmail.com` the `Storage Admin` role on bucket `my-data`."

**Service Accounts (Critical for ML):**
Service Accounts are non-human identities used by applications and VMs. Your Vertex AI training job needs a Service Account with permissions to:
1. Read training data from GCS.
2. Write model artifacts to GCS.
3. Push Docker images to Artifact Registry.
4. Deploy models to Vertex Endpoints.

**Principle of Least Privilege:** Always grant the minimum permissions needed. Never use the default compute service account with `Editor` role in production.

---

### Q8: What is VPC (Virtual Private Cloud) and why does it matter for ML?

**Answer:**

A VPC is your private, isolated network within GCP. All your resources (VMs, databases, K8s clusters) live inside a VPC.

**Why it matters for ML:**
1. **Data Privacy:** Medical/financial models must process data that CANNOT leave your VPC. Using OpenAI's API sends data to external servers. Self-hosted LLMs inside your VPC keep data private.
2. **VPC Service Controls:** Creates a security perimeter around GCP services. Even if a developer's credentials are leaked, an attacker cannot exfiltrate data from BigQuery to an external bucket.
3. **Private Google Access:** Allows VMs without external IP addresses to still access GCP APIs (GCS, BigQuery) without exposing them to the public internet.
4. **VPC Peering:** Connects two VPCs so resources in different projects can communicate privately (e.g., your ML team's VPC can access the data team's BigQuery tables).

---

### Q9: What is Cloud Functions vs Cloud Run?

**Answer:**

Both are serverless, but they serve different purposes:

**Cloud Functions (FaaS - Function as a Service):**
- Deploys a *single function* triggered by events (HTTP request, Pub/Sub message, GCS file upload).
- You write just the function code; Google manages everything else.
- **Best for:** Lightweight event-driven tasks (e.g., "When a new PDF is uploaded to GCS, trigger an embedding pipeline").
- **Limitation:** Cold starts, 60-minute timeout, limited language support.

**Cloud Run (CaaS - Container as a Service):**
- Deploys a full *Docker container* (any language, any framework).
- You control the entire runtime environment.
- **Best for:** Full web applications, APIs, ML model servers (FastAPI + PyTorch).
- **Advantage:** Scales to zero (no traffic = no cost), supports GPUs, WebSockets, and gRPC.

**Rule of thumb:** Use Cloud Functions for glue code (event triggers). Use Cloud Run for everything else.

---

### Q10: What is Artifact Registry?

**Answer:**

Artifact Registry is GCP's managed repository for storing and versioning build artifacts. It replaces the older Container Registry.

**What it stores:**
1. **Docker Images:** Your ML model containers (e.g., `us-docker.pkg.dev/my-project/ml-models/fraud-detector:v2.1`).
2. **Python Packages:** Private PyPI packages (your internal ML libraries).
3. **Language Packages:** npm, Maven, Go modules.

**Why it matters for ML:**
- **Vulnerability Scanning:** Automatically scans Docker images for known security vulnerabilities (CVEs).
- **Image Signing:** Ensures only verified, signed images can be deployed to production GKE clusters (Binary Authorization).
- **Regional Storage:** Store images close to your GKE clusters to minimize pull latency during Pod scaling.

---

### Q11: Explain Cloud Build for CI/CD in ML projects.

**Answer:**

Cloud Build is GCP's serverless CI/CD platform. It automates building, testing, and deploying code when changes are pushed to a Git repository.

**Example `cloudbuild.yaml` for an ML API:**
```yaml
steps:
  # Step 1: Run unit tests
  - name: 'python:3.11'
    entrypoint: 'bash'
    args: ['-c', 'pip install -r requirements.txt && pytest tests/']

  # Step 2: Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'us-docker.pkg.dev/$PROJECT_ID/ml-models/api:$SHORT_SHA', '.']

  # Step 3: Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'us-docker.pkg.dev/$PROJECT_ID/ml-models/api:$SHORT_SHA']

  # Step 4: Deploy to Cloud Run
  - name: 'gcr.io/cloud-builders/gcloud'
    args: ['run', 'deploy', 'ml-api', '--image', 'us-docker.pkg.dev/$PROJECT_ID/ml-models/api:$SHORT_SHA', '--region', 'us-central1']
```

**Triggers:** Automatically runs when a PR is merged to `main`, ensuring every production deployment is tested and reproducible.

---

### Q12: How do you monitor and log in GCP? (Cloud Operations Suite)

**Answer:**

GCP provides a unified observability stack (formerly Stackdriver):

**1. Cloud Logging:**
- Collects and stores logs from all GCP services (Cloud Run, GKE, Compute Engine).
- Supports structured JSON logging (critical for searching and filtering in production).
- **Log-based Metrics:** Create custom metrics from log patterns (e.g., count the number of `ERROR: model_inference_failed` per minute).

**2. Cloud Monitoring:**
- Collects system metrics (CPU, memory, request count, latency) automatically.
- **Custom Metrics:** Expose ML-specific metrics (model accuracy, prediction distribution) via the Monitoring API.
- **Dashboards:** Visualize metrics (like Grafana but fully managed).
- **Alerting Policies:** "Alert me if p99 latency exceeds 2 seconds for 5 consecutive minutes."

**3. Cloud Trace:**
- Distributed tracing for microservices. Visualizes the full request lifecycle across multiple services (e.g., API Gateway → ML Service → Database), identifying exactly which service is the latency bottleneck.

---

*End of GCP Theory — 12 comprehensive questions covering GCS, BigQuery, Vertex AI, BQML, deployment options, Pipelines, IAM, VPC, Cloud Functions/Run, Artifact Registry, Cloud Build CI/CD, and Monitoring.*

