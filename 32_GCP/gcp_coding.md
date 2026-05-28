# Google Cloud Platform (GCP) — Coding Patterns

> Practical implementation of MLOps pipelines using the Google Cloud SDK. Focuses on Vertex AI for training and deploying custom ML models.

---

## Table of Contents

- [Q1: Vertex AI Custom Training Job](#q1-vertex-ai-custom-training-job)
- [Q2: Vertex AI Model Deployment (Endpoint)](#q2-vertex-ai-model-deployment-endpoint)

---

### Q1: Vertex AI Custom Training Job

**Problem:** You have a local Python script `train.py` that trains a PyTorch model. Write a script to launch this training job on a powerful cloud GPU using Vertex AI.

```python
from google.cloud import aiplatform

# 1. Initialize the GCP Project connection
aiplatform.init(
    project="my-gcp-project-id",
    location="us-central1",
    staging_bucket="gs://my-mlops-bucket"
)

# 2. Define the Custom Training Job
job = aiplatform.CustomTrainingJob(
    display_name="pytorch-resnet-training",
    script_path="train.py",                  # The local script to upload and run
    container_uri="us-docker.pkg.dev/vertex-ai/training/pytorch-xla.1-13:latest", # Pre-built PyTorch Docker image
    requirements=["pandas", "scikit-learn"], # Extra pip packages needed
)

# 3. Execute the Job on a Cloud GPU
model = job.run(
    machine_type="n1-standard-8",
    accelerator_type="NVIDIA_TESLA_T4",
    accelerator_count=1,
    args=["--epochs", "50", "--batch_size", "64"], # Passed to train.py as sys.argv
    sync=True # Wait here until the cloud job finishes
)

print(f"Training completed. Model saved to: {model.uri}")
```

#### 🧠 Architectural Walkthrough
- **Serverless Training:** You don't have to SSH into a VM, install CUDA drivers, or leave a terminal open. Vertex AI automatically provisions the VM, pulls the PyTorch Docker image, copies your `train.py` script into it, installs your requirements, runs the script, saves the output to your Cloud Storage bucket, and immediately deletes the VM so you stop paying for it.
- **`container_uri`:** GCP provides pre-built, highly optimized Docker containers for TensorFlow, PyTorch, and Scikit-Learn.

---

### Q2: Vertex AI Model Deployment (Endpoint)

**Problem:** Take a trained model from a Cloud Storage bucket and deploy it as an auto-scaling REST API endpoint.

```python
from google.cloud import aiplatform

# 1. Upload the Model to the Vertex AI Model Registry
model = aiplatform.Model.upload(
    display_name="my-production-model",
    artifact_uri="gs://my-mlops-bucket/model_output/", # Path containing model.pt
    serving_container_image_uri="us-docker.pkg.dev/vertex-ai/prediction/pytorch-cpu.1-13:latest",
)

# 2. Create an Endpoint (The actual URL)
endpoint = aiplatform.Endpoint.create(display_name="my-model-endpoint")

# 3. Deploy the Model to the Endpoint
model.deploy(
    endpoint=endpoint,
    machine_type="n1-standard-4",
    min_replica_count=1,
    max_replica_count=5, # Auto-scales up to 5 VMs during traffic spikes
    traffic_split={"0": 100},
)

print(f"Endpoint ready! URL: {endpoint.resource_name}")

# 4. Make an Inference Request
# response = endpoint.predict(instances=[[5.1, 3.5, 1.4, 0.2]])
# print(response.predictions)
```

#### 🧠 Architectural Walkthrough
- **Decoupling Models and Endpoints:** In GCP, a `Model` is just a file and a Docker image stored in a registry. An `Endpoint` is the physical server. This decoupling allows you to deploy multiple versions of a model to the same endpoint (e.g., A/B testing with a 90/10 `traffic_split`).
- **Auto-scaling:** By setting `min` and `max` replicas, GCP automatically monitors the CPU/RAM utilization. If a traffic spike hits, it provisions new VMs. If traffic drops, it scales back down to `min_replica_count`, saving money.
