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

### Q4: Implement Multi-Head Attention from scratch in PyTorch.

**Answer:**

Multi-Head Attention is the core of Transformer models. Instead of one attention head, it runs multiple attention heads in parallel, each learning different types of relationships.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class MultiHeadAttention(nn.Module):
    def __init__(self, embed_size, num_heads):
        super().__init__()
        self.num_heads = num_heads
        self.head_dim = embed_size // num_heads
        assert embed_size % num_heads == 0, "embed_size must be divisible by num_heads"
        
        self.W_q = nn.Linear(embed_size, embed_size)
        self.W_k = nn.Linear(embed_size, embed_size)
        self.W_v = nn.Linear(embed_size, embed_size)
        self.W_o = nn.Linear(embed_size, embed_size)  # Output projection
    
    def forward(self, x, mask=None):
        batch_size, seq_len, embed_size = x.shape
        
        # 1. Linear projections
        Q = self.W_q(x)  # (batch, seq, embed)
        K = self.W_k(x)
        V = self.W_v(x)
        
        # 2. Reshape into multiple heads: (batch, seq, embed) -> (batch, heads, seq, head_dim)
        Q = Q.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        K = K.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        V = V.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        
        # 3. Scaled Dot-Product Attention
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.head_dim)
        
        # 4. Apply causal mask (for decoder / GPT-style models)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attention_weights = F.softmax(scores, dim=-1)
        context = torch.matmul(attention_weights, V)
        
        # 5. Concatenate heads: (batch, heads, seq, head_dim) -> (batch, seq, embed)
        context = context.transpose(1, 2).contiguous().view(batch_size, seq_len, embed_size)
        
        # 6. Final linear projection
        output = self.W_o(context)
        return output

# Usage:
# mha = MultiHeadAttention(embed_size=512, num_heads=8)
# x = torch.randn(2, 10, 512)  # (batch=2, seq_len=10, embed=512)
# out = mha(x)  # (2, 10, 512) — each head_dim = 64
```

---

### Q5: Implement Sinusoidal Positional Encoding from scratch.

**Answer:**

Transformers process all tokens simultaneously (no sequential order). Positional Encodings inject word-order information.

```python
import torch
import math

class PositionalEncoding(torch.nn.Module):
    def __init__(self, embed_size, max_seq_len=5000):
        super().__init__()
        
        # Create a matrix of shape (max_seq_len, embed_size)
        pe = torch.zeros(max_seq_len, embed_size)
        
        # Position indices: [0, 1, 2, ..., max_seq_len-1]
        position = torch.arange(0, max_seq_len).unsqueeze(1).float()
        
        # Compute the divisor term: 10000^(2i/d_model)
        div_term = torch.exp(
            torch.arange(0, embed_size, 2).float() * -(math.log(10000.0) / embed_size)
        )
        
        # Apply sin to even indices, cos to odd indices
        pe[:, 0::2] = torch.sin(position * div_term)  # Even dimensions
        pe[:, 1::2] = torch.cos(position * div_term)  # Odd dimensions
        
        # Register as buffer (not a trainable parameter)
        pe = pe.unsqueeze(0)  # (1, max_seq_len, embed_size)
        self.register_buffer('pe', pe)
    
    def forward(self, x):
        # x shape: (batch_size, seq_len, embed_size)
        seq_len = x.size(1)
        # Add positional encoding to input embeddings
        return x + self.pe[:, :seq_len, :]

# Usage:
# pe = PositionalEncoding(embed_size=512)
# embeddings = torch.randn(2, 20, 512)
# encoded = pe(embeddings)  # Now contains position information
```

**Why sin/cos?** The sinusoidal functions allow the model to learn relative positions because `PE(pos+k)` can be represented as a linear function of `PE(pos)`, enabling generalization to unseen sequence lengths.

---

### Q6: Implement a simple Byte-Pair Encoding (BPE) Tokenizer from scratch.

**Answer:**

BPE is the tokenization algorithm used by GPT models. It iteratively merges the most frequent pair of characters/tokens.

```python
from collections import Counter

def get_pair_counts(vocab):
    """Count frequency of adjacent symbol pairs across all words."""
    pairs = Counter()
    for word, freq in vocab.items():
        symbols = word.split()
        for i in range(len(symbols) - 1):
            pairs[(symbols[i], symbols[i + 1])] += freq
    return pairs

def merge_vocab(pair, vocab):
    """Merge all occurrences of the most frequent pair."""
    merged = {}
    bigram = ' '.join(pair)
    replacement = ''.join(pair)
    for word, freq in vocab.items():
        new_word = word.replace(bigram, replacement)
        merged[new_word] = freq
    return merged

def learn_bpe(corpus, num_merges=10):
    """Learn BPE merge rules from a corpus."""
    # Initialize vocab: each word is split into characters + end-of-word marker
    word_freqs = Counter(corpus.split())
    vocab = {}
    for word, freq in word_freqs.items():
        # "low" -> "l o w _"
        vocab[' '.join(list(word)) + ' _'] = freq
    
    merges = []
    for i in range(num_merges):
        pairs = get_pair_counts(vocab)
        if not pairs:
            break
        best_pair = max(pairs, key=pairs.get)
        vocab = merge_vocab(best_pair, vocab)
        merges.append(best_pair)
        print(f"Merge {i+1}: {best_pair} -> {''.join(best_pair)}")
    
    return merges, vocab

# Example:
# corpus = "low low low low low lowest lowest newer newer newer newer newer newer wider wider wider"
# merges, vocab = learn_bpe(corpus, num_merges=10)
```

---

### Q7: Build a Named Entity Recognition (NER) pipeline using spaCy.

**Answer:**

```python
import spacy

# Load pre-trained NER model
nlp = spacy.load("en_core_web_sm")

def extract_entities(text):
    """Extract named entities with their labels and positions."""
    doc = nlp(text)
    entities = []
    for ent in doc.ents:
        entities.append({
            "text": ent.text,
            "label": ent.label_,
            "start": ent.start_char,
            "end": ent.end_char,
            "description": spacy.explain(ent.label_)
        })
    return entities

# Example:
# text = "Apple Inc. was founded by Steve Jobs in Cupertino, California in 1976."
# entities = extract_entities(text)
# Output:
# [
#   {"text": "Apple Inc.", "label": "ORG", "description": "Companies, agencies..."},
#   {"text": "Steve Jobs", "label": "PERSON", "description": "People, including fictional"},
#   {"text": "Cupertino", "label": "GPE", "description": "Countries, cities, states"},
#   {"text": "California", "label": "GPE", "description": "Countries, cities, states"},
#   {"text": "1976", "label": "DATE", "description": "Absolute or relative dates"}
# ]
```

**For Custom NER (domain-specific entities like drug names or legal terms):**
Fine-tune a HuggingFace `token-classification` model (like BERT) on your labeled dataset using the `Trainer` API.

---

### Q8: Implement Text Classification with HuggingFace Transformers.

**Answer:**

A production-grade text classification pipeline using a pre-trained transformer model.

```python
from transformers import pipeline, AutoTokenizer, AutoModelForSequenceClassification
import torch

# Method 1: Zero-code pipeline (fastest to prototype)
classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased-finetuned-sst-2-english")
result = classifier("I absolutely loved this movie!")
# Output: [{'label': 'POSITIVE', 'score': 0.9998}]

# Method 2: Full control (production use)
model_name = "distilbert-base-uncased-finetuned-sst-2-english"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)

def classify_text(text: str) -> dict:
    """Classify text with confidence scores for each label."""
    inputs = tokenizer(text, return_tensors="pt", truncation=True, max_length=512)
    
    with torch.no_grad():
        outputs = model(**inputs)
    
    # Convert logits to probabilities
    probabilities = torch.softmax(outputs.logits, dim=1)
    
    # Get predicted class
    predicted_class = torch.argmax(probabilities, dim=1).item()
    confidence = probabilities[0][predicted_class].item()
    
    return {
        "label": model.config.id2label[predicted_class],
        "confidence": round(confidence, 4),
        "all_scores": {
            model.config.id2label[i]: round(p.item(), 4) 
            for i, p in enumerate(probabilities[0])
        }
    }

# Usage:
# result = classify_text("This product is terrible and broke after one day.")
# Output: {"label": "NEGATIVE", "confidence": 0.9995, "all_scores": {"NEGATIVE": 0.9995, "POSITIVE": 0.0005}}
```

---

*End of NLP Coding — 8 comprehensive implementations covering TF-IDF from scratch, RNN cell in NumPy, Self-Attention, Multi-Head Attention, Positional Encoding, BPE Tokenizer, Named Entity Recognition, and Text Classification with Transformers.*

