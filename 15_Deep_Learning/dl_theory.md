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

*End of Deep Learning Theory — 10 questions covering Neural Network internals, CNNs, LSTMs, Optimization algorithms, and Transfer Learning.*
