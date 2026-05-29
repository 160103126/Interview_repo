# ML Deployment — Theory & Strategies

> Transitioning models to production: REST APIs, gRPC, Batch vs Real-time, and Deployment Strategies (Blue-Green, Canary).

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: Compare Batch (Offline) vs Real-Time (Online) Inference.

**Answer:**

**Batch Inference:**
- **How it works:** The model makes predictions on a large dataset all at once on a schedule (e.g., a nightly Airflow job).
- **Latency:** High (hours/minutes). Does not matter.
- **Throughput:** Maximized.
- **Use case:** Netflix generating tomorrow's movie recommendations for all 200 million users at 2:00 AM. 

**Real-Time Inference:**
- **How it works:** The model is deployed as an API (e.g., FastAPI). A user sends a request, and the model must return a prediction instantly.
- **Latency:** Must be extremely low (milliseconds).
- **Use case:** Credit card fraud detection. The model must approve or decline a swipe while the customer is standing at the cash register.

---

### Q2: What is Model Serialization? (Pickle vs ONNX).

**Answer:**

Serialization is saving a model object from memory into a file so it can be loaded later or on another machine.

- **Pickle (`.pkl`):** The standard Python serialization format (used by Scikit-Learn). 
  - *Warning:* Extremely insecure. Unpickling a malicious file can execute arbitrary code on your server. It is also tied tightly to specific Python versions.
- **TorchScript / SavedModel (TensorFlow):** Framework-specific serialization formats that allow models to be run in C++ environments without a Python interpreter.
- **ONNX (Open Neural Network Exchange):** The industry standard for interoperability. You can train a model in PyTorch, export it to ONNX, and run it in a highly optimized C++ or Rust environment (ONNX Runtime) in production, resulting in massive speedups.

---

## 🟡 Medium (Intermediate)

### Q3: Compare REST vs gRPC for ML Model Serving.

**Answer:**

When wrapping a model in an API, you must choose a communication protocol.

- **REST (HTTP/JSON):** The most common standard (e.g., FastAPI). Easy to debug, human-readable JSON payloads.
  - *Problem:* JSON is a text format. Sending a 1MB image array as text is massive and slow to serialize/deserialize.
- **gRPC (HTTP/2 / Protobuf):** Used heavily internally at MAANG. It uses Protocol Buffers (binary data format).
  - *Benefits:* Binary payloads are incredibly small and fast. It supports bidirectional streaming (great for video/audio ML models). Much faster than REST for high-throughput internal microservices.

---

### Q4: Explain Dynamic Batching.

**Answer:**

GPUs are terrible at processing single items (Batch Size 1). They are designed to process massive arrays in parallel. 

If your Real-Time API receives 10 concurrent requests from 10 different users at the exact same millisecond, processing them sequentially (one by one) wastes 90% of the GPU's compute power.

**Dynamic Batching (e.g., NVIDIA Triton, Ray Serve):**
The inference server briefly intercepts the 10 incoming requests, waits a few milliseconds (e.g., 5ms window), groups all 10 requests into a single Batch of size 10, runs the GPU forward pass *once*, splits the results back up, and returns them to the respective users. 
*Result:* Massive increase in throughput with minimal latency penalty.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q5: Compare Deployment Strategies: Blue-Green vs Canary vs Shadow.

**Answer:**

How do you replace `Model V1` with `Model V2` in production without breaking the app?

1. **Blue-Green Deployment:** 
   - You spin up a completely separate environment running V2 (Green). 
   - V1 (Blue) is still handling 100% of live traffic.
   - Once Green is fully tested, you flip a router switch. 100% of traffic instantly goes to Green. If it crashes, you flip the switch back to Blue. (Zero downtime, easy rollback).
2. **Canary Deployment:**
   - You release V2 to a tiny fraction of users (e.g., 5%). 
   - You monitor error rates and latency. If stable, you slowly ramp it up (25% -> 50% -> 100%). Minimizes the blast radius of a bad model.
3. **Shadow Deployment (Silent launch):**
   - V1 handles the user request and returns the answer to the user.
   - The same request is secretly mirrored to V2. V2's answer is NOT shown to the user, but is logged to a database.
   - *Why?* Allows you to compare V1 and V2 on real-world production data without ever risking the user experience.

---

### Q6: What is A/B Testing vs Multi-Armed Bandit?

**Answer:**

How do you know if V2 is *better* than V1 (e.g., drives more clicks)?

- **A/B Testing (Static):** You route 50% of traffic to V1 and 50% to V2 for 2 weeks. You then do statistical analysis (p-values) to determine a winner, and route 100% to the winner.
  - *Drawback:* For 2 weeks, you are actively sending 50% of your users to the inferior model, losing money.
- **Multi-Armed Bandit (Dynamic RL):** Uses Reinforcement Learning (e.g., Thompson Sampling). It starts 50/50. As it gathers real-time data that V2 is performing slightly better, it dynamically shifts traffic (e.g., 70/30 -> 90/10) *during* the test. 
  - *Benefit:* Minimizes Regret (lost revenue) by exploiting the winning model faster while still exploring.

---

### Q7: What is Model Quantization? Why is it critical for deployment?

**Answer:**

Quantization reduces the precision of model weights from 32-bit floating point (FP32) to lower precision formats, dramatically reducing model size and inference latency.

**Types:**
1. **FP16 (Half Precision):** Reduces weights from 32-bit to 16-bit. ~2x memory savings with minimal accuracy loss. Supported natively by modern GPUs.
2. **INT8 Quantization:** Reduces to 8-bit integers. ~4x memory savings. Requires calibration data to map float ranges to integer ranges.
3. **INT4 / GPTQ / AWQ:** Extreme compression for LLMs. Reduces a 70B model from 140GB (FP16) to ~35GB (INT4). Makes it possible to run massive models on consumer GPUs.

**Post-Training Quantization (PTQ):** Applied *after* training. Fast but can degrade accuracy for aggressive quantization.
**Quantization-Aware Training (QAT):** Simulates quantization *during* training. The model learns to compensate, resulting in higher accuracy but requiring retraining.

---

### Q8: Compare Model Serving Frameworks: TorchServe vs TFServing vs Triton.

**Answer:**

| Feature | TorchServe | TF Serving | NVIDIA Triton |
|---------|-----------|------------|---------------|
| **Framework** | PyTorch only | TensorFlow only | Framework-agnostic (PyTorch, TF, ONNX, TensorRT) |
| **Protocol** | REST + gRPC | REST + gRPC | REST + gRPC |
| **Dynamic Batching** | ✅ | ✅ | ✅ (Best-in-class) |
| **Model Ensemble** | ❌ | ❌ | ✅ (Chain multiple models in a pipeline) |
| **Best For** | PyTorch-native teams | TensorFlow-native teams | Multi-framework production environments |

**NVIDIA Triton** is the industry standard at MAANG companies because it supports any framework and provides the most sophisticated batching, GPU scheduling, and model pipeline capabilities.

---

### Q9: What is Model Versioning with Feature Flags?

**Answer:**

Feature Flags (LaunchDarkly, Unleash) allow you to control which model version serves predictions without deploying new code.

**Architecture:**
```python
@app.post("/predict")
async def predict(request: PredictRequest):
    if feature_flag.is_enabled("use_model_v2", user_id=request.user_id):
        prediction = model_v2.predict(request.features)
    else:
        prediction = model_v1.predict(request.features)
    return prediction
```

**Benefits:**
- **Instant Rollback:** If V2 shows degraded metrics, flip the flag — no redeployment needed.
- **Gradual Rollout:** Enable V2 for 5% of users, monitor, then ramp to 100%.
- **Targeted Testing:** Enable V2 only for internal employees or specific regions first.

---

### Q10: How do you Load Test ML Endpoints?

**Answer:**

ML endpoints have unique performance characteristics (GPU memory limits, batch queuing) that standard web load testing doesn't capture.

**Tools:** Locust (Python-native), k6, Apache JMeter.

**What to Measure:**
1. **Latency Percentiles (p50, p95, p99):** Not just average! A p99 of 5 seconds means 1 in 100 users waits 5 seconds.
2. **Throughput under Concurrency:** How many requests/sec can the endpoint handle before latency degrades? This reveals the GPU saturation point.
3. **Cold Start Latency:** If using serverless (Cloud Run, Lambda), how long does the first request take to load the model into memory?
4. **Memory Under Load:** Does GPU VRAM usage grow linearly with concurrent requests? When does it OOM (Out of Memory)?

**Key Pattern:** Run the load test with increasing concurrency (10 → 50 → 100 → 500 users) and plot latency vs throughput to find the optimal batch size and replica count.

---

### Q11: Explain ONNX Runtime optimizations for production inference.

**Answer:**

ONNX Runtime (ORT) is a high-performance inference engine maintained by Microsoft.

**Why use it over raw PyTorch?**
1. **Graph Optimization:** ORT fuses multiple operations (e.g., Conv → BatchNorm → ReLU) into a single kernel, reducing overhead.
2. **Quantization Integration:** ORT provides built-in INT8 quantization with calibration tools.
3. **Hardware Acceleration:** ORT automatically dispatches to the fastest available backend (CUDA, TensorRT, DirectML, CPU AVX-512).
4. **Cross-Platform:** Train in PyTorch → Export to ONNX → Deploy with ORT on any platform (Windows, Linux, Edge devices, browsers via ONNX.js).

**Typical Speedup:** 2x-5x over raw PyTorch inference with zero accuracy loss.

---

### Q12: Compare Serverless vs Dedicated GPU for ML Inference.

**Answer:**

| Feature | Serverless (Cloud Run, Lambda) | Dedicated GPU (GKE, EC2 + GPU) |
|---------|------|------|
| **Cold Start** | Seconds to minutes (loading model) | None (always warm) |
| **Cost Model** | Pay-per-request | Pay 24/7 regardless of traffic |
| **Scaling** | Auto scales to 0 (no traffic = no cost) | Manual or HPA-based |
| **Best For** | Bursty, low-frequency workloads | High-throughput, latency-sensitive APIs |

**The Hybrid Pattern (Common at scale):**
- **Always-on GPU instances** for the core high-traffic model (e.g., main recommendation engine).
- **Serverless containers** for long-tail, low-traffic models (e.g., rarely-used language translation endpoints).
- **Queue-based batch processing** for background tasks (e.g., nightly report generation).

---

*End of ML Deployment Theory — 12 comprehensive questions covering Batch vs Real-time, ONNX, gRPC, Dynamic Batching, Blue-Green/Canary, Quantization, Serving Frameworks, Feature Flags, Load Testing, and Serverless vs GPU tradeoffs.*

