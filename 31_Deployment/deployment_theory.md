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

*End of ML Deployment Theory — 6 advanced questions covering Batch vs Real-time, ONNX, gRPC vs REST, Dynamic Batching, and Blue-Green/Canary/Shadow rollouts.*
