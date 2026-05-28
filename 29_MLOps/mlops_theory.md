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

*End of MLOps Theory — 6 questions covering CI/CD/CT, Model Decay (Drift), MLflow architecture, and Feature Stores.*
