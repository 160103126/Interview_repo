# MLOps (Machine Learning Operations) — Theory & Concepts

> Bridging the gap between Jupyter Notebooks and Production: Model Registries, CI/CD for ML, Data Drift, and MLflow.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is MLOps and why is it necessary?

**Answer:**

MLOps (Machine Learning Operations) is the intersection of Machine Learning, DevOps, and Data Engineering.

**The Problem:** Data scientists build highly accurate models in Jupyter Notebooks. But a model on a laptop is useless. Deploying it to production, keeping it running 24/7, tracking its performance over time, and retraining it when data changes is incredibly difficult. 
Only ~5% of an ML system is the actual ML code; the rest is data pipelines, serving infrastructure, and monitoring.

**The Goal of MLOps:**
To automate and standardize the entire ML lifecycle—from data ingestion and model training to deployment and continuous monitoring.

---

### Q2: What is a Model Registry?

**Answer:**

A centralized repository for managing the lifecycle of ML models. (e.g., MLflow Model Registry).

If you train a new model every week, you need to track:
- Which version of the model is currently in "Staging"?
- Which version is in "Production"?
- What were the hyper-parameters and training data used for `v2.4`?

The Model Registry stores the actual model artifacts (the `.pkl` or `.pt` files) alongside metadata, lineage, and version history, allowing seamless rollbacks if a new model fails in production.

---

## 🟡 Medium (Intermediate)

### Q3: Explain Data Drift and Concept Drift.

**Answer:**

Models degrade over time because the real world changes. This is called "Model Decay."

- **Data Drift (Feature Drift):** The statistical distribution of the *input features* changes. 
  - *Example:* A fraud detection model was trained on data where the average transaction was $50. Due to inflation, the average is now $100. The model starts throwing false positives.
- **Concept Drift:** The relationship between the input features and the *target variable* changes. The definition of what you are predicting changes.
  - *Example:* A spam filter trained in 2018 doesn't catch 2024 crypto-spam, because the "concept" of what spam looks like has fundamentally shifted.

**Solution:** Implement Continuous Monitoring that triggers automated Continuous Training (CT) when drift metrics exceed a threshold.

---

### Q4: How does CI/CD work in MLOps compared to standard software engineering?

**Answer:**

In standard DevOps, CI/CD is about building and testing code.
In MLOps, CI/CD is about code, data, *and* models.

1. **Continuous Integration (CI):** You don't just test the Python code. You test the data pipelines (e.g., catching null values) and the model architectures.
2. **Continuous Delivery (CD):** Deploying the model as a prediction service (REST API) seamlessly without downtime.
3. **Continuous Training (CT) - *The MLOps Addition*:** The pipeline is designed to automatically retrain the model on fresh data, evaluate it against the current production model, and auto-deploy it if it performs better.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q5: Explain the components of MLflow.

**Answer:**

MLflow is the industry standard open-source platform for the ML lifecycle.

1. **MLflow Tracking:** An API to log parameters, code versions, metrics (Accuracy, Loss), and output files when running your ML code. You can visually compare hundreds of training runs in the UI.
2. **MLflow Projects:** A standard format for packaging reusable data science code (using a `MLproject` file and Docker/Conda environments) so anyone can reproduce the run.
3. **MLflow Models:** A standard format for packaging models. It saves a model in multiple "flavors" (e.g., a PyTorch flavor and a standard Python function flavor) so it can be deployed to AWS SageMaker or a local Docker container using the exact same code.
4. **Model Registry:** The central hub to collaboratively manage the full lifecycle (Staging -> Production -> Archived).

---

### Q6: What is a Feature Store? (e.g., Feast, Vertex AI Feature Store)

**Answer:**

**The Problem:** 
- The Data Science team writes a complex Pandas script to calculate `user_average_spend_30_days` to train a model.
- The Engineering team writes a totally different SQL script in production to calculate the exact same feature for real-time inference. 
- Discrepancies between the two scripts cause the model to fail in production (Training-Serving Skew). Also, multiple teams keep re-calculating the same features from scratch.

**The Solution:**
A Feature Store is a centralized data management system specifically for ML features.
- **Offline Store (Data Warehouse):** Stores massive historical feature data for model training.
- **Online Store (Redis/Cassandra):** Stores the *latest* feature values for ultra-low latency access during real-time inference.
- *Benefit:* Ensures that the exact same feature definition is used for both training and serving, eliminating skew and allowing teams to reuse features.

---

### Q7: What is Data Versioning? (DVC)

**Answer:**

Git tracks code changes perfectly, but it was never designed to track datasets (a single CSV can be 50GB).

**DVC (Data Version Control):**
DVC works alongside Git to version-control datasets and model artifacts.
- **How it works:** DVC stores the actual data in remote storage (S3, GCS). In Git, it stores a tiny `.dvc` file (like a pointer) containing the hash of the data. 
- **The Workflow:** `dvc add data/train.csv` → creates `data/train.csv.dvc` (tracked by Git) → pushes the actual data to S3 via `dvc push`.
- **Reproducibility:** If you `git checkout` to a commit from 3 months ago, `dvc checkout` will automatically download the exact dataset version that was used at that point in time.

---

### Q8: What is Training-Serving Skew and how do you prevent it?

**Answer:**

Training-Serving Skew is one of the most insidious bugs in production ML. The model works perfectly in your Jupyter notebook but fails silently in production.

**Types of Skew:**
1. **Feature Skew:** The training pipeline and serving pipeline compute features differently. (E.g., training uses Pandas to calculate `avg_spend_30d`, but serving uses a SQL query with a slightly different date range).
2. **Data Distribution Skew:** The production data looks fundamentally different from training data (different user demographics, seasonality effects).
3. **Label Skew:** The definition of labels has drifted (e.g., what counts as "fraudulent" in 2024 is different from 2022).

**Prevention:**
- **Feature Stores** (Feast, Vertex AI Feature Store) ensure the *exact same* feature computation code is used for both training and serving.
- **Schema Validation:** Use TensorFlow Data Validation (TFDV) or Great Expectations to validate that incoming production data matches the statistical profile of training data.

---

### Q9: What is Shadow Deployment for ML models?

**Answer:**

Shadow Deployment (also called "Dark Launch") is a production testing strategy where the new model runs in parallel with the current production model, but its outputs are never shown to users.

**How it works:**
1. The production model (V1) receives a user request and returns the prediction to the user.
2. The same request is silently forwarded to the new model (V2).
3. V2's prediction is logged to a data warehouse — not returned to the user.
4. A batch evaluation job compares V1 vs V2 predictions on real production traffic.

**Why use it:**
- Validates the model on *real* data distributions (not just offline test sets).
- Zero risk to user experience.
- Catches issues like latency spikes, memory leaks, and edge cases that offline testing misses.

---

### Q10: How do you monitor ML models in production? (Prometheus + Grafana)

**Answer:**

Model monitoring requires tracking both **system metrics** and **ML-specific metrics**.

**System Metrics (Standard DevOps):**
- Request latency (p50, p95, p99), throughput (requests/sec), error rates, CPU/GPU utilization.
- Tools: Prometheus (metric collection) + Grafana (dashboards and alerting).

**ML-Specific Metrics:**
1. **Prediction Distribution Drift:** Is the distribution of model outputs changing? If your fraud model suddenly predicts 30% fraud instead of 2%, something is wrong.
2. **Feature Drift:** Are the input features drifting from the training distribution? (Monitored using statistical tests like KS-Test or PSI — Population Stability Index).
3. **Ground Truth Delayed Metrics:** For tasks where labels arrive later (e.g., loan default takes months), track business KPIs as proxies.

**Alerting Pipeline:**
```
Prometheus scrapes /metrics → Detects drift PSI > 0.2 → Triggers PagerDuty alert → 
Team investigates → Triggers automated retraining pipeline if confirmed
```

---

### Q11: Explain Experiment Tracking. How does MLflow Tracking work?

**Answer:**

Data scientists often run hundreds of experiments (different hyperparameters, feature sets, architectures). Without tracking, they lose the ability to reproduce their best result.

**MLflow Tracking:**
```python
import mlflow

mlflow.set_experiment("fraud-detection-v2")

with mlflow.start_run():
    # Log hyperparameters
    mlflow.log_param("learning_rate", 0.001)
    mlflow.log_param("epochs", 50)
    mlflow.log_param("model", "XGBoost")
    
    # Train model...
    model = train(...)
    
    # Log metrics
    mlflow.log_metric("accuracy", 0.94)
    mlflow.log_metric("f1_score", 0.87)
    mlflow.log_metric("auc", 0.96)
    
    # Log the model artifact
    mlflow.sklearn.log_model(model, "model")
```

**The MLflow UI** provides a visual comparison table where you can sort all 200 runs by F1-Score, compare hyperparameters side-by-side, and instantly find the best configuration.

---

### Q12: What is an ML Pipeline? How does it differ from a Jupyter Notebook?

**Answer:**

A Jupyter Notebook is a linear, manual, non-reproducible script. An **ML Pipeline** is an automated, modular, reproducible workflow.

**Why Notebooks Fail in Production:**
- No automatic retries on failure.
- No dependency management between steps.
- "Run all cells" breaks when cells have hidden state.
- No scheduling, alerting, or versioning.

**Pipeline Architecture (Airflow / Vertex AI Pipelines):**
Each step is an independent, containerized task connected in a DAG (Directed Acyclic Graph):

```
[Data Ingestion] → [Data Validation] → [Feature Engineering] → [Model Training] → [Model Evaluation] → [Deploy if Better]
```

**Key Properties:**
- **Idempotent:** Re-running a step produces the same result.
- **Cacheable:** If "Data Ingestion" hasn't changed, skip it and reuse the cached output.
- **Observable:** Each step logs metadata, metrics, and artifacts.
- **Triggerable:** Can be triggered by a schedule (daily), an event (new data arrives), or a drift alert.

---

*End of MLOps Theory — 12 comprehensive questions covering CI/CD/CT, Model Decay (Drift), MLflow, Feature Stores, DVC, Training-Serving Skew, Shadow Deployments, Production Monitoring, and Pipeline Architecture.*

