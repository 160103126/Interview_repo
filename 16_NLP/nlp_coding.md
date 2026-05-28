# Natural Language Processing — Coding Patterns

> Practical implementation of NLP concepts. From classical TF-IDF to modern Transformer implementations using PyTorch.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: Implement a basic TF-IDF Vectorizer from scratch in Python.

**Answer:**

A fundamental question to prove you understand how text is converted to numbers without relying on Scikit-Learn.

```python
import math
from collections import Counter

def compute_tf(document):
    # TF = (Count of word in document) / (Total words in document)
    tf_dict = {}
    words = document.split()
    word_count = len(words)
    counts = Counter(words)
    for word, count in counts.items():
        tf_dict[word] = count / float(word_count)
    return tf_dict

def compute_idf(documents):
    # IDF = log(Total documents / Number of documents containing the word)
    idf_dict = {}
    total_docs = len(documents)
    
    # Get all unique words across all documents
    all_words = set(word for doc in documents for word in doc.split())
    
    for word in all_words:
        # Count how many documents contain this word
        docs_containing_word = sum(1 for doc in documents if word in doc.split())
        idf_dict[word] = math.log(total_docs / float(docs_containing_word))
    return idf_dict

def compute_tfidf(documents):
    idf_dict = compute_idf(documents)
    tfidf_scores = []
    
    for doc in documents:
        tf_dict = compute_tf(doc)
        tfidf_doc = {}
        for word, tf in tf_dict.items():
            tfidf_doc[word] = tf * idf_dict[word]
        tfidf_scores.append(tfidf_doc)
        
    return tfidf_scores

# Example Usage:
# docs = ["i love machine learning", "machine learning is hard", "i love python"]
# tfidf = compute_tfidf(docs)
# print(tfidf[0]) # {'i': 0.05, 'love': 0.05, 'machine': 0.0, 'learning': 0.0}
```

---

## 🟡 Medium (Intermediate)

### Q2: Implement a simple Recurrent Neural Network (RNN) cell from scratch in NumPy.

**Answer:**

Shows deep understanding of sequence modeling before using PyTorch abstractions.

```python
import numpy as np

class SimpleRNNCell:
    def __init__(self, input_size, hidden_size):
        self.hidden_size = hidden_size
        
        # Initialize weights randomly
        # Wxh: Weights for the input x
        self.Wxh = np.random.randn(hidden_size, input_size) * 0.01
        
        # Whh: Weights for the previous hidden state
        self.Whh = np.random.randn(hidden_size, hidden_size) * 0.01
        
        # Bias
        self.bh = np.zeros((hidden_size, 1))

    def forward(self, x_t, h_prev):
        # The core RNN equation: h_t = tanh(Wxh * x_t + Whh * h_prev + b)
        h_t = np.tanh(np.dot(self.Wxh, x_t) + np.dot(self.Whh, h_prev) + self.bh)
        return h_t

# Example Usage (Sequence of 3 words, embedded into size 4):
# rnn = SimpleRNNCell(input_size=4, hidden_size=8)
# sequence = [np.random.randn(4, 1) for _ in range(3)]
# h_t = np.zeros((8, 1)) # Initial hidden state
# 
# for word_vector in sequence:
#     h_t = rnn.forward(word_vector, h_t)
# print(h_t.shape) # (8, 1)
```

---

## 🔴 Hard (Advanced / MAANG-level)

### Q3: Implement the Self-Attention mechanism from scratch in PyTorch.

**Answer:**

The core engine of Transformers and LLMs. `Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V`

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class SelfAttention(nn.Module):
    def __init__(self, embed_size):
        super(SelfAttention, self).__init__()
        self.embed_size = embed_size
        
        # Linear layers to project input into Q, K, V
        self.queries = nn.Linear(embed_size, embed_size)
        self.keys = nn.Linear(embed_size, embed_size)
        self.values = nn.Linear(embed_size, embed_size)

    def forward(self, x):
        # x shape: (batch_size, seq_length, embed_size)
        
        # 1. Project x into Q, K, V
        Q = self.queries(x) # (batch_size, seq_length, embed_size)
        K = self.keys(x)    # (batch_size, seq_length, embed_size)
        V = self.values(x)  # (batch_size, seq_length, embed_size)
        
        # 2. Compute Q * K^T
        # We need to transpose the last two dimensions of K to multiply
        # K transposed shape: (batch_size, embed_size, seq_length)
        # Score shape: (batch_size, seq_length, seq_length)
        scores = torch.bmm(Q, K.transpose(1, 2))
        
        # 3. Scale by square root of embed_size
        scaled_scores = scores / math.sqrt(self.embed_size)
        
        # 4. Apply Softmax to get attention weights (probabilities)
        # Dim=2 applies softmax across the sequence length
        attention_weights = F.softmax(scaled_scores, dim=2)
        
        # 5. Multiply attention weights by V
        # Out shape: (batch_size, seq_length, embed_size)
        out = torch.bmm(attention_weights, V)
        
        return out

# Example Usage:
# batch_size = 2, seq_length = 5 (words), embed_size = 512
# x = torch.randn(2, 5, 512)
# attention = SelfAttention(embed_size=512)
# output = attention(x)
# print(output.shape) # torch.Size([2, 5, 512])
```

---

*End of NLP Coding — Implementations covering TF-IDF from scratch, raw NumPy RNN logic, and the core PyTorch Self-Attention equation.*
