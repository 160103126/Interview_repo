# Natural Language Processing (NLP) — Theory & Concepts

> Traditional NLP and modern deep learning NLP: TF-IDF, Word Embeddings, Transformers, Attention Mechanism, and BERT vs GPT architectures.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is Tokenization? What are Stemming and Lemmatization?

**Answer:**

- **Tokenization:** The process of breaking raw text into smaller pieces (tokens). Tokens can be words, characters, or subwords (like BPE used in GPT).
  - *Example:* "I love NLP" -> `["I", "love", "NLP"]`.
- **Stemming:** A crude heuristic process that chops off the ends of words to reduce them to their root form. It's fast but often results in non-words.
  - *Example:* "running", "runs" -> "run". "caring" -> "car".
- **Lemmatization:** A smarter process that uses a vocabulary and morphological analysis to return the base dictionary form (Lemma) of a word.
  - *Example:* "better" -> "good". "was" -> "be".

---

### Q2: What is Bag of Words (BoW) and TF-IDF?

**Answer:**

Machine learning models cannot read text; they need numbers.

**Bag of Words (BoW):**
Creates a vocabulary of all unique words in the corpus. Each document is represented as a vector of word counts.
- *Problem:* Ignores word order and grammar. Gives too much weight to frequent but useless words like "the", "and".

**TF-IDF (Term Frequency - Inverse Document Frequency):**
A statistical measure that evaluates how important a word is to a document within a corpus.
- **TF:** How often does the word appear in this specific document?
- **IDF:** How rare is the word across *all* documents?
- **Result:** Words like "the" get a score near 0. A rare word like "XGBoost" in a machine learning article gets a very high score.

---

## 🟡 Medium (Intermediate)

### Q3: What are Word Embeddings (Word2Vec / GloVe)?

**Answer:**

While TF-IDF vectors are huge, sparse (mostly zeros), and contain no semantic meaning (the word "king" has no mathematical relation to "queen"), Word Embeddings solve this.

**Word Embeddings:**
Dense, low-dimensional vectors (e.g., 300 dimensions) learned by a neural network. 
They capture semantic meaning and relationships based on the context in which words appear.
- *Famous result:* `Vector(King) - Vector(Man) + Vector(Woman) ≈ Vector(Queen)`

**Word2Vec (Skip-Gram vs CBOW):**
- *Skip-Gram:* Predicts the surrounding context words given a target word.
- *CBOW (Continuous Bag of Words):* Predicts the target word given the surrounding context words.

---

### Q4: What is the vanishing context problem in standard RNNs?

**Answer:**

Standard RNNs process text sequentially, one word at a time. The hidden state is passed along and updated at each step. 
Because of the Vanishing Gradient problem, by the time the RNN reaches the 50th word in a sentence, the hidden state has entirely "forgotten" the context of the 1st word. 

This makes RNNs terrible for long-range dependencies (e.g., matching a pronoun at the end of a paragraph to a noun at the beginning). LSTMs and GRUs mitigate this, but do not perfectly solve it for very long documents.

---

### Q5: Explain the Encoder-Decoder Architecture (Seq2Seq).

**Answer:**

Designed for tasks where the input and output have different lengths (e.g., Machine Translation: English to French).

1. **Encoder:** An RNN reads the input sentence word by word and compresses the entire meaning into a single, fixed-size vector called the **Context Vector** (or hidden state).
2. **Decoder:** Another RNN takes this Context Vector and generates the output sentence word by word until it outputs a `<STOP>` token.

*The Flaw:* Compressing an entire 100-word paragraph into a single vector creates an massive information bottleneck. This led to the invention of Attention.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q6: What is the Attention Mechanism?

**Answer:**

Attention solves the bottleneck of the standard Encoder-Decoder.

Instead of the Encoder sending just *one* final context vector to the Decoder, it sends **all** the hidden states from every single step of the input sentence.

At each step of decoding, the Decoder looks at all the Encoder hidden states and calculates an **Attention Weight** (a probability distribution) indicating which input words it should focus on *right now* to generate the next output word.
*(e.g., When translating the French word "pomme", the model learns to place 99% of its attention on the English input word "apple").*

---

### Q7: Explain the Transformer Architecture. Why is it better than LSTMs?

**Answer:**

Introduced in the famous 2017 paper *"Attention Is All You Need"*, the Transformer completely eliminated RNNs/LSTMs in favor of pure Self-Attention mechanisms.

**Why it destroyed LSTMs:**
1. **Parallelization:** LSTMs process words sequentially (word 2 must wait for word 1 to finish). Transformers process *all words in the sentence simultaneously*. This allows massive parallelization on GPUs, enabling the training of massive models (LLMs).
2. **Long-Range Dependencies:** In an LSTM, the distance between word 1 and word 50 is 50 steps. In a Transformer (via Self-Attention), the distance between *any* two words is exactly 1 step. Context is never lost.

**Positional Encoding:**
Because Transformers process everything simultaneously, they inherently lose the order of words. Positional Encodings are sine/cosine vectors added to the input embeddings to mathematically inject information about word order.

---

### Q8: What is Self-Attention (Query, Key, Value)?

**Answer:**

Self-attention allows every word in a sentence to look at every *other* word in the same sentence to understand its own context. (e.g., In "The animal didn't cross the street because **it** was too tired", self-attention helps the word "it" attend heavily to "animal" rather than "street").

It uses a database retrieval analogy:
1. **Query (Q):** What I am looking for (The current word being processed).
2. **Key (K):** What I contain (All other words in the sentence).
3. **Value (V):** The actual information (The embeddings of all words).

The attention score is calculated by taking the Dot Product of the Query and all Keys. The scores are passed through a Softmax function, which are then used to multiply the Values. The result is a context-rich embedding for the current word.

---

### Q9: Compare BERT and GPT architectures.

**Answer:**

Both are based on the Transformer, but they use different parts of it.

**BERT (Bidirectional Encoder Representations from Transformers):**
- **Architecture:** Uses only the **Encoder** stack of the Transformer.
- **Direction:** Bidirectional. It looks at the left and right context simultaneously.
- **Pre-training:** Masked Language Modeling (MLM). It randomly hides 15% of the words in a sentence and learns to guess them based on surrounding context.
- **Use Case:** Natural Language Understanding (NLU) — Sentiment analysis, Question Answering, Named Entity Recognition.

**GPT (Generative Pre-trained Transformer):**
- **Architecture:** Uses only the **Decoder** stack of the Transformer.
- **Direction:** Unidirectional (Left-to-Right). It uses "Masked Self-Attention" to ensure it cannot peek at future words.
- **Pre-training:** Causal Language Modeling. It predicts the *next word* in a sequence.
- **Use Case:** Natural Language Generation (NLG) — Chatbots, text generation, summarization.

---

### Q10: What is BLEU and ROUGE?

**Answer:**

They are evaluation metrics for generative NLP tasks.

- **BLEU (Bilingual Evaluation Understudy):** Primarily used for Machine Translation. It measures **Precision**—how many n-grams (sequences of words) in the machine-generated text appear in the human reference text. (It checks if the words the machine used are correct).
- **ROUGE (Recall-Oriented Understudy for Gisting Evaluation):** Primarily used for Text Summarization. It measures **Recall**—how many n-grams in the human reference text appear in the machine-generated text. (It checks if the machine managed to capture all the important concepts).

---

### Q11: Explain the different Tokenization algorithms: BPE vs WordPiece vs SentencePiece.

**Answer:**

Tokenization is how raw text is split into "tokens" (the units the model processes). Each algorithm produces different vocabularies.

**BPE (Byte-Pair Encoding) — Used by GPT, LLaMA:**
- Starts with individual characters. Iteratively merges the most frequent adjacent pair of characters into a new token.
- *Example:* "low" → "l", "o", "w" → "lo", "w" → "low" (after enough merges).
- Pure data-driven; no linguistic knowledge.

**WordPiece — Used by BERT:**
- Similar to BPE but uses a language-model-based scoring (maximizes the likelihood of the training data) instead of raw frequency.
- Prepends `##` to subword tokens that are continuations of a word.
- *Example:* "unhappiness" → "un", "##happiness"

**SentencePiece — Used by T5, LLaMA:**
- Treats the input as a raw stream of bytes (including spaces). Spaces are replaced with `▁` (unicode underscore). This means it works for ANY language without pre-tokenization.
- Can use BPE or Unigram as the underlying algorithm.
- *Example:* "I love NLP" → "▁I", "▁love", "▁NLP"

---

### Q12: What is Flash Attention? Why does it matter?

**Answer:**

Standard Self-Attention computes a full N×N attention matrix, which is stored in GPU HBM (High Bandwidth Memory). For long sequences (N=128K), this matrix is enormous and memory bandwidth becomes the bottleneck.

**Flash Attention (Dao et al., 2022):**
Instead of materializing the full N×N matrix in HBM, Flash Attention computes attention in **tiles** (blocks), keeping intermediate results in the GPU's fast SRAM (on-chip memory) and never writing the full attention matrix to slow HBM.

**Impact:**
- **Memory:** Reduces memory from O(N²) to O(N) — enabling much longer context windows.
- **Speed:** 2-4x faster than standard attention by reducing HBM reads/writes.
- **Exact:** Unlike sparse attention approximations, Flash Attention computes *exact* attention (no accuracy loss).

**Why it matters:**
Flash Attention is what enabled the jump from 4K → 128K context windows. Without it, models like GPT-4-Turbo and Claude 3 could not process long documents.

---

### Q13: Explain Multi-Head Attention math. Why multiple heads?

**Answer:**

Single-Head Attention computes one set of Q, K, V projections and one attention pattern. **Multi-Head Attention** runs `h` parallel attention heads, each with its own learned Q, K, V projections.

**Math:**
- Input: `X ∈ R^(seq_len × d_model)` (e.g., d_model = 512)
- Each head `i` has its own weight matrices: `W_Q_i, W_K_i, W_V_i ∈ R^(d_model × d_k)` where `d_k = d_model / h`
- `head_i = Attention(X·W_Q_i, X·W_K_i, X·W_V_i)`
- Final: `MultiHead(X) = Concat(head_1, ..., head_h) · W_O`

**Why multiple heads?**
Each head learns to attend to **different types of relationships**:
- Head 1 might learn syntactic relationships ("The cat **sat** on the **mat**").
- Head 2 might learn coreference ("**She** said **her** name was Alice").
- Head 3 might learn positional patterns (attending to nearby words).

Without multiple heads, a single attention pattern can only capture one type of relationship per layer, severely limiting the model's representational power.

---

### Q14: What is the Positional Encoding math? (Sinusoidal Encodings)

**Answer:**

Transformers process all tokens in parallel (no inherent order). Positional Encodings are added to the input embeddings to inject sequence order information.

**Sinusoidal Encodings (Original Transformer):**
```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```
Where `pos` = position in the sequence, `i` = dimension index, `d_model` = embedding dimension.

**Why sin/cos functions?**
1. **Relative Position:** For any fixed offset `k`, `PE(pos+k)` can be expressed as a linear function of `PE(pos)`. This allows the model to learn relative positions ("3 words apart") rather than just absolute positions.
2. **Bounded Values:** sin/cos are bounded between [-1, 1], keeping the magnitude similar to the embeddings.
3. **Unique Encoding:** Each position gets a unique encoding pattern.

**Modern Alternative: RoPE** (Rotary Position Embeddings) applies rotations to the Q and K vectors instead of adding to embeddings. This makes attention scores depend only on *relative* distance, enabling better generalization to longer sequences.

---

### Q15: Explain Layer Normalization vs Batch Normalization for Transformers.

**Answer:**

**Batch Normalization** normalizes across the batch dimension (averaging over all samples). **Layer Normalization** normalizes across the feature dimension (averaging over all features within a single sample).

**Why Transformers use Layer Norm, not Batch Norm:**
1. **Variable Sequence Lengths:** Sentences in a batch have different lengths. Batch statistics are meaningless when padding is involved.
2. **Batch Size Independence:** Layer Norm computes statistics per-sample. It works identically whether your batch size is 1 or 1000. Batch Norm requires sufficiently large batches for stable statistics.
3. **Sequential/Autoregressive Models:** During generation, the model processes one token at a time (batch size = 1). Batch Norm would fail; Layer Norm works perfectly.

**Pre-Norm vs Post-Norm:**
- **Post-Norm (Original Transformer):** `output = LayerNorm(x + Sublayer(x))` — Harder to train, requires careful learning rate warmup.
- **Pre-Norm (Modern Standard):** `output = x + Sublayer(LayerNorm(x))` — More stable training, gradients flow more easily through residual connections.

---

### Q16: Explain the three main Transformer Architectures: Encoder-only, Decoder-only, and Encoder-Decoder.

**Answer:**

While the original 2017 Transformer contained both an Encoder and a Decoder, modern NLP models typically drop one of these components depending on the required task.

**1. Encoder-Only (e.g., BERT, RoBERTa, ALBERT)**
- **How it works:** Uses **Bidirectional Self-Attention**. Every token attends to all other tokens (past and future) simultaneously to build a rich representation of the entire sequence.
- **Pre-training Objective:** Masked Language Modeling (MLM). 15% of the words are randomly masked, and the model learns to predict them based on the surrounding context.
- **Strengths:** Deep Natural Language Understanding (NLU).
- **Use Cases:** Text classification, Sentiment Analysis, Named Entity Recognition (NER), Extractive Question Answering (finding the answer span within a document).

**2. Decoder-Only (e.g., GPT-4, LLaMA 3, Claude 3)**
- **How it works:** Uses **Causal (Masked) Self-Attention**. A token can only look at *past* tokens. Future tokens are mathematically masked out (set to negative infinity before softmax) to prevent cheating.
- **Pre-training Objective:** Next-Token Prediction (Causal Language Modeling). Given tokens 1 to $N$, predict token $N+1$.
- **Strengths:** Natural Language Generation (NLG). It perfectly mimics how text is generated sequentially.
- **Use Cases:** Chatbots, open-ended text generation, code generation, instruction following. *(Note: Almost all modern "Generative AI" LLMs use this architecture).*

**3. Encoder-Decoder (e.g., T5, BART, Original Transformer)**
- **How it works:** Uses both stacks. The **Encoder** processes the input sequence bidirectionally. The **Decoder** generates the output sequence autoregressively, using **Cross-Attention** to look back at the Encoder's hidden states during generation.
- **Pre-training Objective:** Sequence-to-Sequence tasks (e.g., span corruption in T5, where multiple sequences of text are removed and the model must generate the missing spans).
- **Strengths:** Tasks where the input and output have fundamentally different structures or are in different languages.
- **Use Cases:** Machine Translation (e.g., English to French), Abstractive Summarization (rewriting an article into a shorter summary).

---

*End of NLP Theory — 16 comprehensive questions covering Tokenization algorithms, Embeddings, Attention, Transformers, Architecture Variants (Encoder vs Decoder), Flash Attention, BERT vs GPT, Multi-Head Attention math, Positional Encodings, Layer Normalization, and evaluation metrics.*

