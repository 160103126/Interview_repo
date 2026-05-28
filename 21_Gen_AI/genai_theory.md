# Generative AI (LLMs) — Theory & Architecture

> Deep dive into Large Language Model architectures: Pre-training, RLHF, KV Caching, Mixture of Experts (MoE), and Tokenization strategies.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is the difference between a Base Model and an Instruct/Chat Model?

**Answer:**

- **Base Model (Pre-trained):** Trained purely to predict the next word on a massive corpus of internet text. If you prompt it with *"What is the capital of France?"*, it might reply with *"What is the capital of Germany?"* because it thinks it's generating a list of trivia questions. (e.g., Llama-3-8B).
- **Instruct/Chat Model (Fine-tuned):** A base model that has undergone additional training (Supervised Fine-Tuning and RLHF) to act as a helpful assistant. If you prompt it with the same question, it will reply: *"The capital of France is Paris."* (e.g., Llama-3-8B-Instruct).

---

### Q2: Explain the 3 stages of training a modern LLM (Pre-training, SFT, RLHF).

**Answer:**

1. **Pre-training (The Base):** The model reads trillions of tokens (Wikipedia, Reddit, Code) and learns grammar, facts, and reasoning by predicting the next word. (Requires thousands of GPUs for months).
2. **Supervised Fine-Tuning (SFT):** The model is trained on thousands of high-quality human-written Prompt/Response pairs to teach it the *format* of an assistant.
3. **RLHF (Reinforcement Learning from Human Feedback):** 
   - Humans rank multiple AI responses from best to worst.
   - A separate "Reward Model" learns what humans prefer (e.g., polite, concise, unbiased).
   - The LLM is optimized using an RL algorithm (like PPO) to maximize the score from the Reward Model. This makes the model safe and highly aligned.

---

## 🟡 Medium (Intermediate)

### Q3: What is Context Window and why is it hard to increase?

**Answer:**

The Context Window is the maximum number of tokens an LLM can process in a single request (prompt + response). E.g., GPT-3 was 4k, GPT-4 is 128k, Claude 3 is 200k.

**Why it's hard to increase:**
Standard Self-Attention has **O(N^2) time and memory complexity** relative to sequence length (N). 
If you double the context window, the compute required quadruples. A 100k context window generates a massive attention matrix that quickly exhausts GPU VRAM.

*Solutions:* Techniques like FlashAttention (hardware-aware memory optimization), Sparse Attention, and RoPE (Rotary Position Embeddings) scaling are used to stretch these limits.

---

### Q4: Explain KV Caching. Why is it essential for LLM inference?

**Answer:**

LLM text generation is **autoregressive** (generates one token at a time).

**Without KV Cache:**
To generate the 101st token, the model processes tokens 1-100. To generate the 102nd token, it processes tokens 1-101. It recalculates the Key and Value matrices for tokens 1-100 *again*. This is extremely wasteful and slow.

**With KV Cache:**
The model saves the computed Key and Value vectors for tokens 1-100 in GPU memory. To generate the 102nd token, it only computes the Q, K, V for the *newest* token, and fetches the historical K and V from the cache to calculate attention.
- *Trade-off:* Massive speedup in token generation (Compute-bound -> Memory-bound), but the KV cache consumes massive amounts of VRAM, heavily limiting batch sizes.

---

### Q5: What is Temperature, Top-K, and Top-p?

**Answer:**

These control the randomness of token generation.
- **Temperature:** Scales the logits before the Softmax function. `T=0` makes the model totally greedy (deterministic). `T=1` is standard. `T>1` flattens the distribution, making weird/rare words more likely (creative).
- **Top-K:** The model discards all possible next words except the top `K` (e.g., 50) most probable ones, then samples from them.
- **Top-p (Nucleus Sampling):** The model dynamically keeps the minimum number of words whose probabilities sum up to `p` (e.g., 0.90). If the top word has 99% probability, it only considers that 1 word. If the probabilities are flat, it considers hundreds of words.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q6: What is a Mixture of Experts (MoE) architecture?

**Answer:**

MoE (used in GPT-4 and Mixtral 8x7B) scales model parameters without proportionally increasing inference compute costs.

Instead of a standard dense Feed-Forward layer where every token passes through every parameter, MoE uses multiple "Expert" networks.
1. **The Router (Gating Network):** For each token, a router decides which 1 or 2 "experts" (out of e.g., 8) are best suited to process it.
2. **Sparse Execution:** The token is only processed by the selected experts. 
3. *Result:* Mixtral 8x7B has 47 Billion parameters total, but only uses ~12 Billion active parameters for any given token. It runs as fast as a 12B model but has the reasoning capability of a 47B model.
- *Trade-off:* Huge VRAM requirement (you still have to load all 47B parameters into RAM).

---

### Q7: Explain Tokenization anomalies (Why are LLMs bad at math/spelling)?

**Answer:**

LLMs do not see characters; they see Tokens (chunks of characters).

**Spelling (The "Strawberry" problem):**
If you ask an LLM "How many 'r's are in strawberry?", it often fails. Why? Because the word "strawberry" might be tokenized as a single integer ID `[14231]`. The LLM has no idea what letters make up that ID unless it memorized the spelling during pre-training.

**Math Anomalies:**
If `345` is tokenized as `[345]` and `3456` is tokenized as `[34, 56]`, the model struggles to align digits for addition.
*Modern Solution:* Models like Llama-3 force tokenizers to split all numbers into individual digits (e.g., `[3, 4, 5, 6]`), vastly improving arithmetic capabilities.

---

### Q8: What is DPO (Direct Preference Optimization)? How does it replace RLHF?

**Answer:**

Standard RLHF is incredibly complex and unstable (requires training a separate Reward Model and tuning a PPO RL policy).

**DPO (Direct Preference Optimization):**
A mathematical breakthrough that bypasses the Reward Model and RL entirely.
1. You take the human preference data (Prompt -> Winner Response, Loser Response).
2. DPO applies a modified Cross-Entropy loss directly to the LLM during standard supervised training. 
3. It essentially says: "Increase the probability of the Winner response and decrease the probability of the Loser response."

DPO is much simpler to implement, requires less memory, and is now the industry standard for aligning open-source models (e.g., Zephyr, Llama-3-Instruct).

---

### Q9: Explain Prompt Caching.

**Answer:**

Prompt Caching (introduced by Anthropic for Claude and supported by vLLM) drastically reduces costs and latency for repetitive system prompts.

If you have a 100k token system prompt (e.g., an entire codebase) and you ask 50 different questions about it, normally the LLM computes the KV cache for the 100k tokens 50 separate times.

**Prompt Caching:** The LLM computes the KV cache for the static 100k tokens once, stores it in GPU memory (or disk), and reuses it for all subsequent API calls that start with that exact same prefix. 
*Impact:* Latency drops from seconds to milliseconds, and API costs drop by up to 90%.

---

*End of Generative AI Theory — 9 advanced questions covering RLHF/DPO, MoE, KV Caching, and Tokenization anomalies.*
