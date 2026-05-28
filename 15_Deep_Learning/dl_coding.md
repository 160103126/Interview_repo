# Deep Learning — Coding Patterns

> Practical PyTorch implementations of Deep Learning architectures. MAANG interviews focus on your ability to translate research papers and mathematical equations directly into tensor operations.

---

## Table of Contents

- [Q1: Multi-Layer Perceptron (MLP)](#q1-multi-layer-perceptron-mlp)
- [Q2: Convolutional Neural Network (CNN)](#q2-convolutional-neural-network-cnn)
- [Q3: Custom PyTorch Dataset & DataLoader](#q3-custom-pytorch-dataset--dataloader)
- [Q4: Recurrent Neural Network (RNN) Step](#q4-recurrent-neural-network-rnn-step)
- [Q5: Scaled Dot-Product Attention (Self-Attention)](#q5-scaled-dot-product-attention-self-attention)
- [Q6: PyTorch Training Loop](#q6-pytorch-training-loop)

---

### Q1: Multi-Layer Perceptron (MLP)

**Problem:** Build a standard Feed-Forward Neural Network with 2 hidden layers for classification.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MLP(nn.Module):
    def __init__(self, input_size, hidden_size1, hidden_size2, num_classes):
        super(MLP, self).__init__()
        # Linear layer: y = Wx + b
        self.fc1 = nn.Linear(input_size, hidden_size1)
        self.fc2 = nn.Linear(hidden_size1, hidden_size2)
        self.fc3 = nn.Linear(hidden_size2, num_classes)
        
        # Dropout to prevent overfitting
        self.dropout = nn.Dropout(p=0.5)

    def forward(self, x):
        # Flatten the input if it's a 2D image (e.g., MNIST)
        x = x.view(x.shape[0], -1) 
        
        x = F.relu(self.fc1(x))
        x = self.dropout(x)
        x = F.relu(self.fc2(x))
        x = self.dropout(x)
        
        # Raw logits out. (CrossEntropyLoss handles the Softmax internally).
        out = self.fc3(x)
        return out
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **`nn.Module`:** The base class for all neural networks in PyTorch. You must define layers in `__init__` so PyTorch registers their weights (parameters) for backpropagation.
- **`forward` vs `__call__`:** You never explicitly call `.forward(x)`. You call `model(x)`. The internal `__call__` method handles hooks and then runs your `forward` logic.
- **Softmax Warning:** We do *not* apply Softmax at the end. PyTorch's `nn.CrossEntropyLoss` combines `LogSoftmax` and `NLLLoss` into a single function for numerical stability.

#### ⏱️ Complexity
- **Time Complexity:** $O(N \times (I \times H1 + H1 \times H2 + H2 \times C))$ per forward pass.
- **Space Complexity:** $O(I \times H1 + \dots)$ to store the weights.

---

### Q2: Convolutional Neural Network (CNN)

**Problem:** Build a CNN for image classification (e.g., CIFAR-10).

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SimpleCNN(nn.Module):
    def __init__(self, num_classes=10):
        super(SimpleCNN, self).__init__()
        # 3 input channels (RGB), 16 output channels, 3x3 kernel
        self.conv1 = nn.Conv2d(in_channels=3, out_channels=16, kernel_size=3, padding=1)
        self.conv2 = nn.Conv2d(in_channels=16, out_channels=32, kernel_size=3, padding=1)
        
        self.pool = nn.MaxPool2d(kernel_size=2, stride=2)
        
        # After two 2x2 pools, a 32x32 image becomes 8x8.
        # 32 channels * 8 width * 8 height
        self.fc1 = nn.Linear(32 * 8 * 8, 128)
        self.fc2 = nn.Linear(128, num_classes)

    def forward(self, x):
        # Conv -> ReLU -> Pool
        x = self.pool(F.relu(self.conv1(x)))
        x = self.pool(F.relu(self.conv2(x)))
        
        # Flatten for the linear layer
        x = x.view(-1, 32 * 8 * 8)
        
        x = F.relu(self.fc1(x))
        x = self.fc2(x)
        return x
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Spatial Hierarchies:** Convolutions extract spatial features (edges, textures). Max pooling halves the spatial dimensions (translation invariance) while retaining the strongest activations.
- **The Flattening Math:** The hardest part of CNNs is transitioning to the Linear layer. If you start with $32 \times 32$ pixels, pooling twice with a stride of 2 halves it twice ($32 \rightarrow 16 \rightarrow 8$). Since `conv2` outputs $32$ filters, the flattened tensor size is strictly $32 \times 8 \times 8$.

#### ⏱️ Complexity
- **Time Complexity:** Heavily dominated by the matrix multiplications in the convolutional sliding windows.
- **Space Complexity:** Driven by the number of output channels and the dense `fc1` layer.

---

### Q3: Custom PyTorch Dataset & DataLoader

**Problem:** Write a custom dataset to load images and labels from a CSV file.

```python
import torch
from torch.utils.data import Dataset, DataLoader
import pandas as pd
from PIL import Image

class CustomImageDataset(Dataset):
    def __init__(self, annotations_file, img_dir, transform=None):
        self.img_labels = pd.read_csv(annotations_file)
        self.img_dir = img_dir
        self.transform = transform

    def __len__(self):
        # Required: Returns the total number of samples
        return len(self.img_labels)

    def __getitem__(self, idx):
        # Required: Returns a single sample (tensor, label) at index idx
        img_path = f"{self.img_dir}/{self.img_labels.iloc[idx, 0]}"
        image = Image.open(img_path).convert("RGB")
        label = self.img_labels.iloc[idx, 1]
        
        if self.transform:
            image = self.transform(image)
            
        # Ensure label is a tensor
        return image, torch.tensor(label, dtype=torch.long)

# Usage:
# dataset = CustomImageDataset("labels.csv", "images/")
# dataloader = DataLoader(dataset, batch_size=64, shuffle=True, num_workers=4)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Lazy Loading:** We DO NOT load all images into memory during `__init__`. We only load the CSV file (which is tiny). The massive image files are read from the SSD one-by-one inside `__getitem__`.
- **`DataLoader`:** The DataLoader utilizes multiprocessing (`num_workers`) to asynchronously pre-fetch data, apply transforms (augmentations), and collate them into batched Tensors for the GPU.

---

### Q4: Recurrent Neural Network (RNN) Step

**Problem:** Implement the mathematical forward pass of a basic vanilla RNN cell from scratch (without `nn.RNN`).

```python
import torch
import torch.nn as nn

class CustomRNNCell(nn.Module):
    def __init__(self, input_size, hidden_size):
        super(CustomRNNCell, self).__init__()
        self.hidden_size = hidden_size
        
        # Weights for the input
        self.W_x = nn.Linear(input_size, hidden_size, bias=False)
        # Weights for the previous hidden state
        self.W_h = nn.Linear(hidden_size, hidden_size, bias=True)
        
    def forward(self, x_t, h_prev):
        # The core RNN equation: h_t = tanh(W_x * x_t + W_h * h_{t-1} + b)
        h_t = torch.tanh(self.W_x(x_t) + self.W_h(h_prev))
        return h_t

# Usage over a sequence of length T:
# h_t = torch.zeros(batch_size, hidden_size)
# for t in range(T):
#     h_t = rnn_cell(x_seq[:, t, :], h_t)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Statefulness:** An RNN cell receives both the current token $x_t$ and the "memory" of previous tokens $h_{prev}$. 
- **The Tanh Activation:** `tanh` crushes values between $-1$ and $1$. It prevents the hidden state from exploding to infinity as it loops through hundreds of sequence steps.
- **Vanishing Gradients:** Vanilla RNNs struggle with long sequences because multiplying gradients through the `tanh` function repeatedly causes them to shrink to zero. This is why LSTMs were invented.

---

### Q5: Scaled Dot-Product Attention (Self-Attention)

**Problem:** Implement the core mechanism behind the Transformer (Attention Is All You Need).

```python
import torch
import torch.nn as nn
import math

class SelfAttention(nn.Module):
    def __init__(self, embed_size, heads):
        super(SelfAttention, self).__init__()
        self.embed_size = embed_size
        self.heads = heads
        self.head_dim = embed_size // heads
        
        assert (self.head_dim * heads == embed_size), "Embed size needs to be divisible by heads"

        # Linear transformations for Query, Key, Value
        self.queries = nn.Linear(self.head_dim, self.head_dim, bias=False)
        self.keys = nn.Linear(self.head_dim, self.head_dim, bias=False)
        self.values = nn.Linear(self.head_dim, self.head_dim, bias=False)
        
        self.fc_out = nn.Linear(heads * self.head_dim, embed_size)

    def forward(self, values, keys, queries, mask=None):
        N = queries.shape[0] # Batch size
        value_len, key_len, query_len = values.shape[1], keys.shape[1], queries.shape[1]

        # Split embedding into multi-heads
        # (N, seq_len, heads, head_dim)
        values = values.reshape(N, value_len, self.heads, self.head_dim)
        keys = keys.reshape(N, key_len, self.heads, self.head_dim)
        queries = queries.reshape(N, query_len, self.heads, self.head_dim)

        values = self.values(values)
        keys = self.keys(keys)
        queries = self.queries(queries)

        # Einsum matrix multiplication:
        # queries: (N, query_len, heads, head_dim)
        # keys: (N, key_len, heads, head_dim)
        # result energy: (N, heads, query_len, key_len)
        energy = torch.einsum("nqhd,nkhd->nhqk", [queries, keys])

        if mask is not None:
            # Mask out future tokens in decoders
            energy = energy.masked_fill(mask == 0, float("-1e20"))

        # Scaled Dot-Product Attention
        attention = torch.softmax(energy / math.sqrt(self.head_dim), dim=3)

        # Multiply attention weights by values
        out = torch.einsum("nhqk,nvhd->nqhd", [attention, values]).reshape(
            N, query_len, self.heads * self.head_dim
        )

        return self.fc_out(out)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Q, K, V:** Queries (what I'm looking for), Keys (what I have), Values (what I actually output).
- **The Dot Product:** Multiplying $Q \times K^T$ calculates how much "attention" each word should pay to every other word in the sequence. High dot products mean strong semantic relevance.
- **The Scaling Factor:** Dividing by $\sqrt{d_{k}}$ prevents the dot products from growing too large, which would push the Softmax function into regions with near-zero gradients (halting learning).
- **`einsum`:** Einstein summation convention is a powerful way to define complex multi-dimensional tensor multiplications cleanly.

#### ⏱️ Complexity
- **Time/Space Complexity:** $O(N^2 \times D)$ where $N$ is sequence length. This $N^2$ scaling is the primary bottleneck of modern LLMs (handled in modern times by techniques like FlashAttention).

---

### Q6: PyTorch Training Loop

**Problem:** Write a standard PyTorch training loop for a single epoch.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from tqdm import tqdm

def train_one_epoch(model, dataloader, optimizer, criterion, device):
    # Set model to training mode (enables Dropout and BatchNorm)
    model.train()
    
    total_loss = 0.0
    correct = 0
    total = 0
    
    loop = tqdm(dataloader, leave=True)
    for batch_idx, (inputs, targets) in enumerate(loop):
        # 1. Move data to GPU
        inputs = inputs.to(device)
        targets = targets.to(device)
        
        # 2. Forward pass
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        
        # 3. Backward pass
        optimizer.zero_grad()  # CLEAR PREVIOUS GRADIENTS!
        loss.backward()        # Calculate new gradients
        optimizer.step()       # Update weights
        
        # Metrics calculation
        total_loss += loss.item()
        _, predicted = outputs.max(1)
        total += targets.size(0)
        correct += predicted.eq(targets).sum().item()
        
        # Update progress bar
        loop.set_postfix(loss=loss.item(), acc=100.*correct/total)
        
    return total_loss / len(dataloader)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **`model.train()`:** Critical. If you forget this, Dropout layers won't work, and BatchNorm layers will use frozen running statistics.
- **`optimizer.zero_grad()`:** Critical. PyTorch *accumulates* gradients by default. If you don't zero them out, step 2's gradients will be added to step 1's gradients, completely corrupting the math.
- **`.to(device)`:** The tensors (data) and the model parameters must physically reside on the exact same GPU VRAM to perform matrix multiplication.
- **`loss.item()`:** Extracts the scalar python float from the 1-dimensional tensor. If you just do `total_loss += loss`, you append the entire computational graph history to a list, resulting in a massive GPU memory leak.

---

*End of Deep Learning Coding — Mastering Tensors, Gradients, CNNs, RNNs, and the core mathematics of Transformers.*
