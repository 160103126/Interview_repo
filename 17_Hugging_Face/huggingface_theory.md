# Hugging Face — Theory & Concepts

> Working with the `transformers` library, model hubs, pipelines, tokenizers, fine-tuning, and PEFT (LoRA).

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is Hugging Face and the `transformers` library?

**Answer:**

Hugging Face is the "GitHub for Machine Learning." It hosts models, datasets, and ML application spaces.

The `transformers` library is their open-source Python library that provides thousands of pre-trained state-of-the-art models for NLP (like BERT, GPT, T5), Computer Vision, and Audio tasks. It drastically reduces the code needed to download, fine-tune, and run these massive models using PyTorch or TensorFlow.

---

### Q2: What is a `pipeline` in Hugging Face?

**Answer:**

The `pipeline` is the easiest way to use a model for inference. It abstracts away all the complex pre-processing (tokenization) and post-processing.

```python
from transformers import pipeline

# Downloads the default sentiment-analysis model and its tokenizer
classifier = pipeline("sentiment-analysis")

result = classifier("I love Hugging Face!")
print(result) # [{'label': 'POSITIVE', 'score': 0.9998}]
```

*Under the hood, the pipeline does 3 things:*
1. Takes the raw text.
2. Passes it to the **Tokenizer** (converts text to integer IDs).
3. Passes the IDs to the **Model** (runs the neural network).
4. Converts the model's raw logit predictions back into human-readable labels.

---

### Q3: What is a Tokenizer? Why does a specific model need a specific tokenizer?

**Answer:**

A Tokenizer translates raw text strings into arrays of numbers (Token IDs) that the neural network can understand.

You **must** use the exact same tokenizer that was used to train the model. 
If BERT was trained mapping the word "apple" to ID `4021`, and you use a GPT tokenizer that maps "apple" to ID `85`, the BERT model will process `85` as an entirely different word, resulting in garbage output.

```python
from transformers import AutoTokenizer

# AutoTokenizer automatically fetches the correct tokenizer for the model
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
tokens = tokenizer("Hello world!")
print(tokens) 
# {'input_ids': [101, 7592, 2088, 999, 102], 'attention_mask': [1, 1, 1, 1, 1]}
```

---

## 🟡 Medium (Intermediate)

### Q4: Explain `AutoModel`, `AutoTokenizer`, and Model Heads.

**Answer:**

Hugging Face uses "Auto" classes as smart wrappers that automatically guess the correct model architecture (e.g., loading a `BertModel` vs `LlamaModel`) based on the model string you provide.

**Base Models vs Model Heads:**
- `AutoModel` loads the base transformer without a specific task head. It just outputs hidden state embeddings.
- `AutoModelForSequenceClassification` loads the base model but slaps a Linear classification layer on top.
- `AutoModelForCausalLM` slaps a Language Modeling head on top (used for text generation like GPT).

```python
from transformers import AutoModelForSequenceClassification

# This warns you that the classification head is randomly initialized 
# and you MUST train it on your data!
model = AutoModelForSequenceClassification.from_pretrained("bert-base-uncased", num_labels=2)
```

---

### Q5: What is `input_ids` and `attention_mask`?

**Answer:**

When you tokenize text, you get two critical arrays:

1. **`input_ids`:** The actual integers representing the words.
   *(Note: 101 is usually the `[CLS]` token, 102 is the `[SEP]` token, and 0 is the `[PAD]` token).*
2. **`attention_mask`:** A binary array (1s and 0s). 
   Because neural networks require fixed-size inputs (e.g., batches of exactly 512 length), short sentences are padded with zeros. The attention mask tells the model's Self-Attention mechanism to **ignore** the `[PAD]` tokens (0s), ensuring they don't corrupt the meaning of the real words (1s).

---

### Q6: How do you Fine-Tune a model using the `Trainer` API?

**Answer:**

Instead of writing complex PyTorch training loops manually, Hugging Face provides the `Trainer` class.

1. You load a pre-trained model and tokenizer.
2. You tokenize your dataset using the `datasets` library.
3. You define `TrainingArguments` (epochs, learning rate, batch size, output directory).
4. You pass the model, args, and dataset to the `Trainer`.
5. Run `trainer.train()`.

The `Trainer` automatically handles GPU distribution, gradient accumulation, mixed-precision training (FP16), and logging to TensorBoard/WandB.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q7: What is PEFT (Parameter-Efficient Fine-Tuning) and LoRA?

**Answer:**

Fine-tuning a 7-Billion parameter LLM requires massive amounts of VRAM (GPUs) because you have to load 7B weights, their gradients, and optimizer states.

**PEFT** techniques allow you to fine-tune massive models on consumer hardware.

**LoRA (Low-Rank Adaptation):**
1. It **freezes** all the original 7B weights of the pre-trained model.
2. It injects a tiny number of new, trainable weights (e.g., 10 million parameters) into the attention layers, configured as low-rank matrices.
3. During training, *only* these 10M parameters are updated.
4. During inference, these tiny updated matrices are mathematically merged back into the frozen 7B weights with zero latency overhead.

*Result:* You can fine-tune a massive LLM using 1/10th the GPU memory.

---

### Q8: What is Quantization (e.g., 8-bit, 4-bit) and bitsandbytes?

**Answer:**

Model weights are traditionally stored as 32-bit floats (FP32) or 16-bit floats (FP16/BF16). A 7B parameter model in FP16 requires ~14GB of VRAM just to load.

**Quantization** is the process of compressing these weights into lower precision, like 8-bit integers (INT8) or 4-bit (INT4).

- Hugging Face integrates with the `bitsandbytes` library to do this on the fly.
- `model = AutoModelForCausalLM.from_pretrained("llama-7b", load_in_4bit=True)`
- This shrinks the 14GB model down to ~4GB of VRAM.

**QLoRA:** The standard modern approach for LLMs. You load the base model in 4-bit quantization (frozen), and add 16-bit LoRA adapters on top to train.

---

### Q9: Explain Generation Strategies: Greedy, Beam Search, Top-K, and Top-p (Nucleus).

**Answer:**

When an LLM (like GPT) generates text, it outputs a probability distribution over the entire vocabulary for the *next* word. How do we choose the word?

1. **Greedy Search:** Always picks the word with the absolute highest probability. 
   - *Problem:* Often gets stuck in repetitive loops ("I went to the store and bought a bought a bought a").
2. **Beam Search:** Keeps track of the top `N` most probable *sequences* of words, rather than just the next single word. 
   - *Use case:* Great for translation, bad for creative writing.
3. **Top-K Sampling:** Randomly samples the next word from the top `K` most probable words. (e.g., K=50).
4. **Top-p (Nucleus) Sampling:** Randomly samples from the smallest pool of words whose cumulative probability exceeds `p` (e.g., p=0.9). If the top 2 words hold 90% of the probability, it only samples from those 2.
5. **Temperature:** Modifies the logits before the softmax. `T=0` makes the model strictly greedy/deterministic. `T=1` is standard. `T>1` makes the model highly random/creative.

---

*End of Hugging Face Theory — 9 questions covering basic Pipelines, Tokenizers, the Trainer API, LoRA, Quantization, and Decoding Strategies.*
