# Hugging Face — Coding Patterns

> Practical implementation of Hugging Face `transformers` library, including Pipelines, Tokenizer manipulation, basic training loops, and Advanced Parameter-Efficient Fine-Tuning (PEFT/LoRA).

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

### Q4: Implement LoRA (Low-Rank Adaptation) using the PEFT library.

**Answer:**

Fine-tuning a 7B parameter model requires massive VRAM because you have to calculate gradients and optimizer states for all 7 Billion weights. **LoRA** (Low-Rank Adaptation) freezes the original model weights and injects tiny, trainable "Low Rank Matrices" into the Attention layers. You end up training less than 1% of the parameters, saving massive VRAM and compute.

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training

model_name = "meta-llama/Llama-2-7b-hf"

# 1. Load the original model (Normally you'd load in 8-bit or 4-bit here for QLoRA)
model = AutoModelForCausalLM.from_pretrained(model_name, device_map="auto")
tokenizer = AutoTokenizer.from_pretrained(model_name)

# 2. Prepare for int8/int4 training (if doing QLoRA)
# model = prepare_model_for_kbit_training(model)

# 3. Define LoRA Configuration
lora_config = LoraConfig(
    r=16,                       # Rank: The dimension of the low-rank matrices. (Higher = more parameters to train)
    lora_alpha=32,              # Scaling factor for the LoRA weights.
    target_modules=["q_proj", "v_proj"], # Which layers to inject LoRA into (usually Query and Value projections in Attention).
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM"
)

# 4. Wrap the base model with PEFT
peft_model = get_peft_model(model, lora_config)

# Print trainable parameters to prove we are only training a fraction of the model!
peft_model.print_trainable_parameters()
# Example Output: trainable params: 8,388,608 || all params: 6,746,804,224 || trainable%: 0.1243%

# 5. From here, you pass `peft_model` to standard Hugging Face Trainer
# trainer = Trainer(model=peft_model, ...)
# trainer.train()
```

---

*End of Hugging Face Coding — Covers basic Pipelines, manual tokenization/tensor manipulation, advanced text generation, and PEFT/LoRA.*
