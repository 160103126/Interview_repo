# Hugging Face — Coding Patterns

> Practical implementation of Hugging Face `transformers` library, including Pipelines, Tokenizer manipulation, and basic training loops.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: Write a script to perform Named Entity Recognition (NER) using a Pipeline.

**Answer:**

```python
from transformers import pipeline

# Load the NER pipeline (automatically downloads a default model like dbmdz/bert-large-cased-finetuned-conll03-english)
# grouped_entities=True merges subword tokens back into whole words (e.g., 'San' + 'Francisco' -> 'San Francisco')
ner_pipe = pipeline("ner", grouped_entities=True)

text = "Tim Cook is the CEO of Apple Inc. and he lives in California."
results = ner_pipe(text)

for entity in results:
    print(f"Entity: {entity['word']} | Label: {entity['entity_group']} | Confidence: {entity['score']:.2f}")

# Output:
# Entity: Tim Cook | Label: PER | Confidence: 0.99
# Entity: Apple Inc | Label: ORG | Confidence: 0.98
# Entity: California | Label: LOC | Confidence: 0.99
```

---

## 🟡 Medium (Intermediate)

### Q2: How do you manually tokenize text and run inference without a Pipeline?

**Answer:**

Pipelines are great, but in production, you often need to manually handle tokenization to implement custom batching or padding.

```python
import torch
from transformers import AutoTokenizer, AutoModelForSequenceClassification

model_name = "distilbert-base-uncased-finetuned-sst-2-english"

# 1. Load Tokenizer and Model
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name)

texts = ["I love this product!", "This is terrible."]

# 2. Manual Tokenization
# padding=True pads the shorter sentence with 0s. 
# truncation=True cuts off sentences longer than the model's max length (e.g., 512).
# return_tensors="pt" returns PyTorch tensors instead of Python lists.
inputs = tokenizer(texts, padding=True, truncation=True, return_tensors="pt")

print(inputs['input_ids'])
# tensor([[ 101, 1045, 2293, 2023, 4031,  999,  102],
#         [ 101, 2023, 2003, 6659, 1012,  102,    0]]) # 0 is the [PAD] token

# 3. Model Inference (No gradients needed for inference)
with torch.no_grad():
    outputs = model(**inputs)

# 4. Post-processing logits to probabilities
logits = outputs.logits
probabilities = torch.nn.functional.softmax(logits, dim=-1)

# 5. Extract predictions
predictions = torch.argmax(probabilities, dim=-1)
labels = [model.config.id2label[p.item()] for p in predictions]

print(labels) # ['POSITIVE', 'NEGATIVE']
```

---

## 🔴 Hard (Advanced / MAANG-level)

### Q3: Write a custom text generation loop using the `generate()` method and discuss decoding parameters.

**Answer:**

When dealing with LLMs (Causal Language Models like GPT-2 or LLaMA), you don't just do a single forward pass. You generate text autoregressively.

```python
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM

model_id = "gpt2"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id)

prompt = "The future of Artificial Intelligence is"
inputs = tokenizer(prompt, return_tensors="pt")

# Advanced Generation using various decoding strategies
with torch.no_grad():
    output_ids = model.generate(
        **inputs,
        max_new_tokens=50,          # Stop after generating 50 new tokens
        do_sample=True,             # Required for Top-K / Top-p (disables purely greedy search)
        temperature=0.7,            # Lowers randomness (1.0 is default, <1.0 is more confident)
        top_k=50,                   # Only sample from the top 50 most likely next words
        top_p=0.9,                  # Nucleus sampling: sample from words comprising 90% of probability mass
        repetition_penalty=1.2,     # Penalize words that have already been generated to prevent loops
        pad_token_id=tokenizer.eos_token_id # Prevents warnings on open-ended generation
    )

# Decode the generated tokens back to text
# skip_special_tokens=True removes <|endoftext|> and [PAD] tags
generated_text = tokenizer.decode(output_ids[0], skip_special_tokens=True)

print(generated_text)
```

---

*End of Hugging Face Coding — Covers basic Pipelines, manual tokenization/tensor manipulation, and advanced text generation algorithms.*
