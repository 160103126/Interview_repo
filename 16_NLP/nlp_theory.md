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

*End of NLP Theory — 10 advanced questions covering Embeddings, Attention mechanisms, Transformers, BERT vs GPT, and evaluation metrics.*
