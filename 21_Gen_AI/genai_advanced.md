# Generative AI — Advanced Topics

> Deep dive into advanced Gen AI concepts: KV Cache Math, Inference Servers (vLLM), Speculative Decoding, Multi-Modal architectures, Long-Context scaling (RoPE), and next-gen frameworks like DSPy.

---

## Table of Contents

- [🟢 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Medium (Intermediate)

### Q1: What is Speculative Decoding?

**Answer:**

Speculative Decoding is an inference optimization technique that drastically speeds up LLM text generation without changing the model's weights or degrading output quality.

**The Problem:** Generating text autoregressively (one token at a time) using a massive 70B parameter model is extremely slow because memory bandwidth is the bottleneck. You have to load all 70B weights into the GPU compute cores for *every single token*.

**The Solution:**
1. You use a tiny, fast "Draft Model" (e.g., 1B parameters) to quickly generate (speculate) the next 5 tokens.
2. You pass these 5 draft tokens to the massive "Target Model" (70B) in *one single batch pass*.
3. The Target Model evaluates all 5 tokens simultaneously (in parallel).
4. If the Target Model agrees with the Draft Model's tokens, it accepts them. If it disagrees at token 3, it rejects tokens 3, 4, 5, and generates the correct token 3 itself.
*Result:* You get the exact same mathematical output as the 70B model, but potentially 2x to 3x faster because you batched the memory reads.

---

### Q2: Explain RAG Evaluation Frameworks (e.g., RAGAS, TruLens).

**Answer:**

How do you know if your RAG system is actually good? You cannot use traditional ML metrics like Accuracy or F1-Score for generative text.

Frameworks like **RAGAS** (Retrieval Augmented Generation Assessment) use LLMs as judges to score the system on 3 core pillars:

1. **Context Relevance (Retrieval Metric):** Did the Vector DB retrieve chunks that are actually relevant to the user's query, or did it pull in garbage?
2. **Faithfulness (Generation Metric):** Did the LLM answer strictly using the provided context, or did it hallucinate facts from its pre-training?
3. **Answer Relevance (End-to-End Metric):** Does the final answer directly address the user's query?

---

## 🔴 Hard (Advanced / MAANG-level)

### Q3: LLM Inference Math — How do you estimate KV Cache Memory?

**Answer:**

A classic MAANG System Design question: *"How much VRAM do you need to serve a model?"*

During text generation, the LLM caches the Key and Value matrices of previous tokens to avoid recomputing them. This is the **KV Cache**. As the sequence grows, the KV cache grows linearly, often consuming more memory than the model weights themselves!

**The KV Cache Equation:**
Memory per token (in bytes) = $2 \times \text{Layers} \times \text{Heads} \times \text{Head\_Dim} \times \text{Bytes\_per\_parameter}$
* Note: The $2$ is because we store both Keys and Values.
* Note: $\text{Heads} \times \text{Head\_Dim} = \text{Hidden\_Size}$.

**Example for LLaMA-3 8B (FP16):**
- Layers: 32
- Hidden Size: 4096
- Bytes per parameter (FP16): 2 bytes
- **Memory per token** = $2 \times 32 \times 4096 \times 2 = 524,288$ bytes (approx 0.5 MB).

If you have a batch size of 32 requests, and each request has a context window of 4,000 tokens:
- **Total KV Cache Size** = $32 \text{ batch} \times 4000 \text{ tokens} \times 0.5 \text{ MB} = 64 \text{ GB of VRAM!}$
*(This is why 8B models require massive GPUs in production, not just to hold the 16GB of weights, but to hold the massive KV cache).*

---

### Q4: What is vLLM, PagedAttention, and Continuous Batching?

**Answer:**

Standard PyTorch/HuggingFace is highly inefficient for serving LLMs to hundreds of concurrent users. Production relies on specialized Inference Servers like **vLLM** or **TGI**.

1. **Continuous Batching:** 
   - Standard batching waits for all requests in a batch to finish generating before starting a new batch. If one user requests a 10-word summary and another requests a 1000-word essay, the GPU sits idle waiting for the essay to finish.
   - Continuous Batching injects new requests into the batch the exact millisecond an old request finishes, keeping GPU utilization at nearly 100%.
2. **PagedAttention:**
   - Because we don't know how long a user's prompt or response will be, standard inference pre-allocates contiguous chunks of VRAM for the maximum possible KV cache length (e.g., 4K tokens). This results in massive memory fragmentation (wasted VRAM).
   - Inspired by OS Virtual Memory, PagedAttention divides the KV Cache into fixed-size "pages" (e.g., 16 tokens). Memory is allocated dynamically, page-by-page, as generation occurs, virtually eliminating fragmentation and allowing you to fit 2x-4x more concurrent users on a single GPU.

---

### Q5: How do models scale context windows? Explain RoPE (Rotary Position Embeddings).

**Answer:**

Standard Transformers (like BERT) used absolute positional encodings (adding a fixed sine/cosine wave to the embedding). This fails when you try to input sequences longer than what the model was trained on.

**RoPE (Rotary Position Embeddings):**
Instead of adding a fixed vector, RoPE *rotates* the Query and Key vectors in a multi-dimensional space based on their position in the sequence. 
- *Why it's brilliant:* The dot product (attention score) between a Query at position 2 and a Key at position 5 depends *only* on their relative distance (3 steps apart), not their absolute positions. 
- This allows the model to extrapolate to sequence lengths it has never seen before (e.g., fine-tuning a 4k context model to 128k context using RoPE scaling techniques like YaRN or Linear Interpolation).

---

### Q6: Explain Multi-Modal Architectures (e.g., LLaVA, GPT-4V).

**Answer:**

How does an LLM "see" an image? It doesn't use standard text tokenizers. It uses a composite architecture.

1. **The Vision Encoder:** An image is passed through a pre-trained Vision Transformer (ViT) or CLIP model. This model chops the image into "patches" (e.g., 16x16 pixels) and converts them into a sequence of dense vector embeddings.
2. **The Projection Layer:** The dimensions of the Vision Encoder's output usually don't match the LLM's expected input dimension. A simple projection matrix (or MLP) maps the image embeddings into the exact same dimensional space as the text embeddings.
3. **The LLM:** The text prompt ("What is in this image?") is tokenized and embedded. The image embeddings and text embeddings are concatenated together into one long sequence and passed into the standard Transformer LLM.
4. To the LLM, the image just looks like a sequence of highly descriptive "foreign language" words. It processes them using standard Self-Attention and generates text.

---

### Q7: What is DSPy and how does it differ from LangChain?

**Answer:**

**DSPy (Demonstrate-Search-Predict)** is a radically different approach to building LLM applications.

- **LangChain:** Relies heavily on fragile, hand-written Prompt Engineering. If you change your model from GPT-3.5 to Llama-3, your prompts might break because the models react differently to instructions.
- **DSPy:** Replaces prompt engineering with **programming and compilation**. You define the *architecture* of your program (e.g., "Retrieve then Answer"). You provide a metric (e.g., "Answer must be exact match"). 
- *The Compiler:* DSPy acts like PyTorch. It "compiles" your program by automatically running hundreds of iterations, finding the optimal prompt instructions and few-shot examples that maximize your metric for the *specific* LLM you are using. If you swap models, you just recompile.

---

### Q8: Explain Tensor Parallelism vs Pipeline Parallelism.

**Answer:**

When a model is too large to fit on a single GPU (e.g., a 70B model = 140GB in FP16 vs a GPU with 80GB VRAM), you must split it across multiple GPUs.

**Tensor Parallelism (TP):**
- Splits individual **layers** across GPUs. Each GPU holds a fraction of every layer's weight matrix.
- *Example:* A 4096×4096 weight matrix is split into 4 chunks of 4096×1024 across 4 GPUs.
- **Advantage:** Very low latency (GPUs process each token together in lockstep).
- **Requirement:** GPUs must be connected via ultra-fast NVLink (not PCIe). Communication overhead is very high.

**Pipeline Parallelism (PP):**
- Splits **layers** across GPUs sequentially. GPU 1 holds layers 1-16, GPU 2 holds layers 17-32.
- The output of GPU 1 is sent to GPU 2 as input.
- **Advantage:** Lower communication bandwidth needed (only inter-layer activation tensors are sent).
- **Disadvantage:** "Pipeline bubbles" — GPU 2 sits idle while GPU 1 processes, and vice versa.

**In Practice:** Large-scale training (like LLaMA-3 405B) uses a combination: TP within a single server (8 GPUs connected via NVLink) + PP across servers (connected via InfiniBand).

---

### Q9: What is Grouped Query Attention (GQA)?

**Answer:**

Standard Multi-Head Attention (MHA) has separate Q, K, V projections for each head. For a 32-head model, that means 32 Key heads and 32 Value heads. The KV cache stores all of them, consuming massive VRAM.

**Multi-Query Attention (MQA):** All 32 Query heads share a **single** Key and Value head. This reduces KV cache by 32x but degrades quality.

**Grouped Query Attention (GQA) — Used by LLaMA-3, Gemini:**
A compromise. Instead of 32 KV heads or 1 KV head, it uses 8 KV groups (each shared by 4 Query heads).
- **Memory:** KV cache is reduced by 4x compared to MHA.
- **Quality:** Much better than MQA (nearly matching MHA in benchmarks).

**Why it matters for production:** GQA is the reason modern 70B models can serve reasonable batch sizes. Without it, the KV cache would consume all available VRAM at batch_size > 4.

---

### Q10: Explain SGLang and RadixAttention.

**Answer:**

**SGLang** is a next-generation inference framework that focuses on structured generation and prefix caching efficiency.

**RadixAttention:** SGLang's key innovation. Instead of storing KV caches in a flat list, it organizes them in a **Radix Tree** (prefix tree) data structure.

**Why this matters:**
In production, many requests share common prefixes:
- All requests to a RAG chatbot share the same system prompt (e.g., 2000 tokens).
- Multi-turn conversations share the entire previous conversation history.

RadixAttention automatically detects these shared prefixes across ALL concurrent requests and stores their KV cache only once, enabling:
1. **Automatic prefix caching** without any user configuration.
2. **Near-instant Time-to-First-Token** for requests with cached prefixes.
3. **Much higher throughput** because VRAM isn't wasted on duplicate KV caches.

---

### Q11: What is Mixture of Depths (MoD)?

**Answer:**

Mixture of Depths (Google, 2024) is a dynamic compute allocation technique that extends the MoE concept from "which expert?" to "which layer?"

**The Insight:** Not all tokens need the same amount of computation. A function word like "the" is trivial to predict; a complex math token requires deep reasoning.

**How it works:**
At each layer, a lightweight router decides whether to process a token through the full Transformer block or skip it (pass it through a residual shortcut only).

**Result:**
- Simple tokens skip many layers, getting processed by only 50% of the network.
- Complex tokens use the full depth.
- Overall FLOPs are reduced by 30-50% with minimal quality degradation.

**Connection to Speculative Decoding:** Both techniques exploit the observation that most tokens are "easy" and don't need the full model's compute power.

---

### Q12: How do you implement Streaming / Server-Sent Events (SSE) for LLM APIs?

**Answer:**

Users expect to see tokens appear one-by-one (like ChatGPT), not wait 10 seconds for the complete response. This is implemented via **Server-Sent Events (SSE)**.

**FastAPI Implementation:**
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio

app = FastAPI()

async def generate_stream(prompt: str):
    """Simulates streaming tokens from an LLM."""
    # In production: use vLLM/OpenAI streaming API
    for token in ["Hello", " ", "World", "!", " How", " can", " I", " help?"]:
        yield f"data: {token}\n\n"  # SSE format
        await asyncio.sleep(0.1)  # Simulate generation delay
    yield "data: [DONE]\n\n"

@app.get("/stream")
async def stream_response(prompt: str):
    return StreamingResponse(
        generate_stream(prompt),
        media_type="text/event-stream"
    )
```

**Client-side JavaScript:**
```javascript
const source = new EventSource("/stream?prompt=Hello");
source.onmessage = (event) => {
    if (event.data === "[DONE]") { source.close(); return; }
    document.getElementById("output").textContent += event.data;
};
```

**Why SSE over WebSockets for LLMs?** SSE is simpler (HTTP-based, no connection upgrade), unidirectional (server → client, which is all you need for token streaming), and works through most proxies/CDNs.

---

*End of Generative AI Advanced Topics — 12 deep dives covering Speculative Decoding, RAGAS, KV Cache Math, vLLM/PagedAttention, RoPE, Multi-modal Architecture, DSPy, Tensor/Pipeline Parallelism, GQA, SGLang, Mixture of Depths, and Streaming.*

