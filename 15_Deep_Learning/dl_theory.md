# Deep Learning — Theory & Concepts

> Core concepts of neural networks: Backpropagation, Activation Functions, CNNs, RNNs/LSTMs, and optimization techniques.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is a Neural Network and what are its main components?

**Answer:**

A Neural Network is a machine learning model inspired by the human brain, composed of interconnected layers of "neurons" (nodes).

**Components:**
1. **Input Layer:** Receives the raw data (e.g., pixels of an image).
2. **Hidden Layers:** One or more layers between input and output where the computation (feature extraction) happens. (Networks with multiple hidden layers are "Deep" Neural Networks).
3. **Output Layer:** Produces the final prediction (e.g., a probability score between 0 and 1).
4. **Weights & Biases:** The learnable parameters. Weights determine the strength of the connection between neurons, and biases shift the activation threshold.

---

### Q2: What is an Activation Function? Name common ones.

**Answer:**

An activation function decides whether a neuron should "fire" or not. More importantly, it introduces **non-linearity** into the network. 
*Without non-linear activation functions, a neural network with 100 layers would just be one giant linear regression model.*

**Common Functions:**
- **Sigmoid:** Squashes values between 0 and 1. Used in the output layer for binary classification. (Suffers from vanishing gradients).
- **Tanh:** Squashes values between -1 and 1. Zero-centered, better than Sigmoid, but still has vanishing gradients.
- **ReLU (Rectified Linear Unit):** `max(0, x)`. If input is negative, outputs 0. If positive, outputs the input. The standard default for hidden layers because it's fast and mitigates vanishing gradients.
- **Softmax:** Used in the output layer for Multi-class classification. Converts a vector of numbers into a probability distribution that sums to 1.

---

### Q3: Explain Epochs, Batches, and Iterations.

**Answer:**

Assume you have a dataset of 10,000 images.

- **Batch Size:** The number of samples processed before updating the model's weights. (e.g., Batch Size = 100).
- **Iterations (Steps):** The number of batches needed to complete one Epoch. (10,000 / 100 = 100 Iterations).
- **Epoch:** One complete pass through the *entire* training dataset. (Training usually runs for 10 to 100+ epochs).

---

## 🟡 Medium (Intermediate)

### Q4: Explain Forward Propagation and Backpropagation.

**Answer:**

- **Forward Propagation:** The data moves forward through the network from input to output. The network calculates a prediction. The prediction is compared to the true label using a **Loss Function** (e.g., Mean Squared Error or Cross-Entropy) to calculate the error.
- **Backpropagation:** The process of moving backwards through the network to update the weights. Using the **Chain Rule of Calculus**, it calculates the gradient (partial derivative) of the Loss Function with respect to every single weight in the network. The optimizer (like Adam or SGD) then adjusts the weights in the opposite direction of the gradient to minimize the loss.

---

### Q5: What is the Vanishing Gradient problem? How is it solved?

**Answer:**

**The Problem:** In very deep networks using Sigmoid or Tanh activations, the derivative is always less than 1 (max 0.25 for Sigmoid). During backpropagation, we multiply these gradients together. Multiplying many small fractions makes the gradient exponentially smaller until it "vanishes" to near zero. 
As a result, the weights in the early layers (near the input) stop updating, and the network stops learning.

**Solutions:**
1. **Use ReLU:** The derivative of ReLU is 1 for all positive inputs. Gradients do not vanish.
2. **Residual Networks (ResNet):** Introduce "Skip Connections" that allow gradients to bypass layers entirely, flowing unaltered to earlier layers.
3. **Batch Normalization:** Normalizes the outputs of hidden layers, keeping them in a range where the gradient doesn't vanish.

---

### Q6: What is Dropout?

**Answer:**

Dropout is a highly effective Regularization technique used to prevent overfitting in neural networks.

**How it works:**
During training, at each iteration, a percentage of neurons (e.g., 50%) are randomly "dropped out" (ignored entirely). 

**Why it works:**
It forces the network to not rely too heavily on any single neuron or specific feature. The network must learn redundant, robust representations because it never knows which neurons will be available on the next iteration. 
*(Note: Dropout is turned off during inference/testing).*

---

## 🔴 Hard (Advanced / MAANG-level)

### Q7: Explain Convolutional Neural Networks (CNNs). What are Convolutions and Pooling?

**Answer:**

CNNs are specialized networks for processing grid-like data (images). They preserve spatial relationships (unlike dense layers which flatten everything into a 1D line).

1. **Convolution Layer:** Applies a set of learnable "Filters" (kernels) that slide across the image. 
   - Early layers learn simple features (edges, corners).
   - Deeper layers learn complex features (eyes, wheels, faces).
2. **Pooling Layer (e.g., Max Pooling):** Downsamples the image. A 2x2 Max Pool takes a 2x2 grid of pixels and keeps only the maximum value.
   - *Purpose:* Reduces computational load, reduces parameters (preventing overfitting), and provides Translation Invariance (if a cat shifts 2 pixels to the left, Max Pooling still detects the cat feature).

---

### Q8: Explain Recurrent Neural Networks (RNNs) and LSTMs. What problem do LSTMs solve?

**Answer:**

Standard neural networks assume all inputs are independent. **RNNs** are designed for sequential data (Time series, Text, Audio) because they possess an internal memory (hidden state) that is passed from one time-step to the next.

**The Problem with Standard RNNs:**
They suffer terribly from the **Vanishing Gradient** problem. They have short-term memory and forget the beginning of a long sentence by the time they reach the end.

**Long Short-Term Memory (LSTM):**
An advanced RNN cell that solves this using a "Cell State" (a conveyor belt running straight down the entire sequence) and three **Gates** (neural network layers outputting 0 to 1 via Sigmoid):
1. **Forget Gate:** Decides what information to throw away from the past.
2. **Input Gate:** Decides what new information to add to the cell state.
3. **Output Gate:** Decides what to output based on the cell state.
Because of the additive nature of the cell state, gradients can flow backwards without vanishing.

---

### Q9: Compare the Optimizers: SGD, Momentum, RMSprop, and Adam.

**Answer:**

- **SGD (Stochastic Gradient Descent):** Updates weights based purely on the current gradient. Prone to getting stuck in local minima and oscillating wildly.
- **SGD with Momentum:** Simulates a ball rolling down a hill. It adds a fraction of the *previous* update to the current update. This dampens oscillations and accelerates convergence.
- **RMSprop:** Introduces an adaptive learning rate. It divides the learning rate by an exponentially decaying average of squared gradients. It slows down updates for features with steep gradients and speeds up updates for features with flat gradients.
- **Adam (Adaptive Moment Estimation):** The industry standard. It combines the best of Momentum and RMSprop. It computes adaptive learning rates for each parameter using the first moment (mean of gradients, like Momentum) and the second moment (variance of gradients, like RMSprop).

---

### Q10: What is Transfer Learning and Fine-Tuning?

**Answer:**

Training a deep CNN like ResNet-50 from scratch requires millions of images (ImageNet) and weeks of GPU time.

**Transfer Learning:**
Take a pre-trained model (like ResNet-50). The convolutional layers have already learned how to extract edges, textures, and shapes. 
1. "Freeze" the weights of the convolutional layers.
2. Replace the final output layer with a new one for your specific task (e.g., classifying Cats vs Dogs).
3. Train *only* the new output layer on your small dataset.

**Fine-Tuning:**
After Transfer Learning converges, "Unfreeze" the last few convolutional layers and train the whole model at a very small learning rate. This allows the high-level feature extractors to adjust specifically to your new dataset.

---

### Q11: Explain Batch Normalization internals.

**Answer:**

Batch Normalization (BatchNorm) normalizes the inputs of each layer to have zero mean and unit variance across the mini-batch.

**The Math:**
1. Compute mean and variance across the batch: `μ_B = mean(x)`, `σ²_B = var(x)`
2. Normalize: `x̂ = (x - μ_B) / sqrt(σ²_B + ε)` (ε is a tiny constant for numerical stability)
3. Scale and Shift: `y = γ * x̂ + β` (γ and β are learnable parameters)

**Why step 3?** Without learnable parameters, BatchNorm would force every layer's activations into the same distribution, destroying the network's ability to represent complex functions. γ and β let the network "undo" the normalization if needed.

**During Inference:** You cannot compute batch statistics from a single input. BatchNorm uses running averages of mean and variance computed during training instead.

**Benefits:** Stabilizes training, allows higher learning rates, provides a slight regularization effect (because batch statistics introduce noise).

---

### Q12: What is the Exploding Gradient problem? How do you solve it?

**Answer:**

The opposite of Vanishing Gradients. In deep networks, if weight matrices have eigenvalues > 1, gradients grow exponentially during backpropagation.

**Symptoms:** Loss becomes `NaN` or `inf`. Weights jump wildly. Training crashes.

**Solutions:**
1. **Gradient Clipping:** The most common fix. If the gradient norm exceeds a threshold (e.g., 1.0), scale it down proportionally.
   ```python
   torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
   ```
2. **Weight Initialization:** Use He initialization (for ReLU) or Xavier/Glorot initialization (for Sigmoid/Tanh). These set initial weights to values that preserve gradient variance.
3. **Batch Normalization:** Keeps activations in a controlled range.
4. **LSTMs / GRUs:** Their gating mechanisms naturally prevent gradients from exploding in sequential models.

---

### Q13: Compare Learning Rate Schedulers.

**Answer:**

A fixed learning rate is suboptimal. Start with a high learning rate (fast convergence) and decay it over time (fine-grained optimization near the minimum).

| Scheduler | Behavior | Best For |
|---|---|---|
| **Step Decay** | Drops LR by a factor every N epochs (e.g., /10 every 30 epochs) | Simple baseline |
| **Cosine Annealing** | LR follows a cosine curve from high to low | Modern deep learning (used in most LLM training) |
| **Warmup + Cosine** | Starts near 0, linearly ramps up for N steps, then cosine decays | Transformers/LLMs (prevents early instability from large gradients) |
| **ReduceLROnPlateau** | Monitors val loss; if it plateaus for N epochs, reduce LR | Tabular data, CNNs |
| **One-Cycle Policy** | LR goes up then down in one super-cycle | Fast convergence (fastai) |

**The Warmup Pattern (Critical for LLMs):**
```python
# Linear warmup for 1000 steps, then cosine decay
scheduler = get_cosine_schedule_with_warmup(
    optimizer, num_warmup_steps=1000, num_training_steps=100000
)
```

---

### Q14: What is Mixed Precision Training (FP16)?

**Answer:**

Modern GPUs (NVIDIA A100, H100) have special hardware (Tensor Cores) that compute FP16 operations **2x-8x faster** than FP32.

**Mixed Precision Training:**
- **Forward Pass:** Use FP16 for activations and weights → faster compute, less memory.
- **Backward Pass:** Compute gradients in FP16 → faster.
- **Weight Update:** Keep a **master copy** of weights in FP32 for numerical precision during the optimizer step.

**Why not just use FP16 everywhere?**
FP16 has a limited range (max ~65,504). Very small gradients (< 6e-8) become zero ("underflow"), causing training to stall. **Loss Scaling** solves this by multiplying the loss by a large factor before backprop, and dividing the gradients by the same factor afterward.

**Implementation (PyTorch):**
```python
scaler = torch.cuda.amp.GradScaler()
with torch.cuda.amp.autocast():
    output = model(input)
    loss = criterion(output, target)
scaler.scale(loss).backward()
scaler.step(optimizer)
scaler.update()
```

**Impact:** Reduces GPU memory by ~50% and speeds up training by 2x-3x with virtually no accuracy loss.

---

### Q15: What is Gradient Accumulation? When do you need it?

**Answer:**

**The Problem:** Training large models (LLMs, ViTs) often requires batch sizes of 512+ for stable convergence. But a batch size of 512 on a single GPU might require 80GB+ VRAM — more than most GPUs have.

**Gradient Accumulation:** Simulates a large batch size by splitting it across multiple forward passes.

```python
accumulation_steps = 4  # Effective batch = 4 * 32 = 128
optimizer.zero_grad()

for i, (input, target) in enumerate(dataloader):  # dataloader batch_size = 32
    output = model(input)
    loss = criterion(output, target) / accumulation_steps  # Scale loss
    loss.backward()  # Gradients are ACCUMULATED (not zeroed)
    
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()  # Update weights using accumulated gradients
        optimizer.zero_grad()
```

**Key Insight:** Instead of processing 128 samples at once (OOM), you process 4 mini-batches of 32 and accumulate the gradients before updating weights. Mathematically equivalent to a batch size of 128, but uses 4x less memory.

---

*End of Deep Learning Theory — 15 comprehensive questions covering Neural Networks, Activation Functions, Backpropagation, Vanishing/Exploding Gradients, Dropout, CNNs, LSTMs, Optimizers, Transfer Learning, Batch Normalization, Learning Rate Schedulers, Mixed Precision Training, and Gradient Accumulation.*

