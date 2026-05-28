# Chunking Strategies for RAG

> The most underrated aspect of RAG systems. How you split your text dictates the quality of your embeddings and the success of your entire AI app.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is Chunking and why is it necessary?

**Answer:**

Chunking is the process of breaking large documents (like a 50-page PDF) into smaller, bite-sized pieces of text.

**Why is it necessary?**
1. **Token Limits:** Embedding models (like OpenAI's `text-embedding-3`) have a maximum input token limit (usually 8192 tokens). You physically cannot embed a whole book at once.
2. **Context Limits:** LLMs have context windows. You can't feed 5 entire books into a prompt to answer one question. You must find the relevant paragraphs.
3. **Semantic Density:** If you embed an entire 50-page document into a single 1536-dimensional vector, the specific meaning of a single sentence on page 42 is completely drowned out (diluted). To make that sentence easily searchable, it needs to be in a small, focused chunk.

---

### Q2: What is Fixed-Size Chunking? What is Overlap?

**Answer:**

The most basic chunking strategy. You define a hard character or token limit.

- **Size:** e.g., 500 characters per chunk.
- **Overlap:** e.g., 50 characters overlap between chunks.

**Why Overlap?**
If a critical sentence is: *"The CEO, John Doe, announced his resignation."*
Without overlap, the chunk boundary might split the sentence down the middle: 
- Chunk 1: *"The CEO, John"* 
- Chunk 2: *"Doe, announced his resignation."*
Neither chunk contains enough context to answer "Who resigned?". Overlap ensures that the context spans across the boundary.

---

## 🟡 Medium (Intermediate)

### Q3: Explain Recursive Character Text Splitting (LangChain Default).

**Answer:**

Fixed-size chunking is blind; it cuts words in half. **Recursive Splitting** tries to respect the structure of the document.

It takes a list of separators (default in LangChain: `["\n\n", "\n", " ", ""]`).
1. It tries to split the document by paragraphs (`\n\n`).
2. If a paragraph is still larger than the `chunk_size`, it recursively drops down to the next separator and splits by sentences (`\n`).
3. If a sentence is too long, it splits by words (` `).

*Result:* It keeps semantically related text together, ensuring chunks are complete paragraphs or sentences rather than arbitrary character cut-offs.

---

### Q4: How do you chunk Code and Markdown?

**Answer:**

Standard text splitters destroy the syntax of code.

- **Markdown Splitter:** Splits specifically on markdown headers (e.g., `#`, `##`, `###`). This ensures an entire section under an H2 tag stays together, which is semantically ideal.
- **Code Splitter (AST Parsing):** LangChain provides `Language` splitters (Python, JS, Go). They use Abstract Syntax Tree parsers to split code specifically by classes and functions, rather than arbitrarily splitting a `while` loop in half.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q5: What is Parent-Child Chunking (Auto-merging Retrieval)?

**Answer:**

**The Paradox of Chunk Size:**
- Small chunks (100 tokens) create highly precise embeddings (great for searching). But they don't provide enough context for the LLM to generate a good answer.
- Large chunks (1000 tokens) provide great context for the LLM, but their embeddings are noisy and hard to search.

**The Parent-Child Solution:**
1. Split the document into **Large Parent chunks** (1000 tokens). Assign an ID (`parent_123`).
2. Split that parent into **Small Child chunks** (200 tokens). Tag them with `parent_id="parent_123"`.
3. Embed and store **ONLY the Child chunks** in the Vector DB.
4. Store the raw text of the Parent chunks in a standard Document/Key-Value Store.
5. **Retrieval:** Search the Vector DB. It matches a precise Child chunk. Before sending it to the LLM, use the `parent_id` to fetch the Large Parent chunk.
6. **Result:** You get the precision of small-chunk search, but the rich context of large-chunk LLM generation.

---

### Q6: What is Semantic Chunking?

**Answer:**

Even recursive chunking relies on formatting (`\n\n`). What if the text has no formatting (e.g., a raw transcript from a Zoom call)?

**Semantic Chunking** uses NLP/Embeddings to determine where a topic changes.
1. Split the document into raw sentences.
2. Embed every single sentence into a vector.
3. Calculate the Cosine Distance between consecutive sentences (Sentence 1 vs 2, Sentence 2 vs 3).
4. If the distance between two sentences spikes (exceeds a threshold), it means the topic has changed. That is where you draw the chunk boundary.

*Result:* Chunks are defined strictly by changes in topical meaning, not arbitrary character counts.

---

### Q7: Explain Metadata Enrichment / Summary Chunking.

**Answer:**

A chunk might say: *"The project was delayed by 3 weeks due to budget cuts."*
If the user asks: *"Why was Project Apollo delayed?"*, the chunk might not be retrieved because it lacks the keyword "Apollo".

**Summary Enrichment:**
Before embedding a chunk, pass it to an LLM: *"Given this chunk and the whole document title, write a 1-sentence summary of what this chunk is about."*
Append this summary to the chunk (or store it in metadata). Now the chunk is embedded with much richer semantic meaning, drastically improving retrieval recall.

---

*End of Chunking Strategies — 7 questions covering the spectrum from naive fixed-size cutting to advanced Parent-Child and Semantic boundary detection.*
