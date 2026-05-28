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

*End of GCP Theory — 6 questions mapping core ML operations to Google Cloud services, covering BigQuery, Vertex AI, Dataflow, and deployment architectures.*
