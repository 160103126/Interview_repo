# Evaluation & Metrics — Theory & Concepts

> Traditional Machine Learning metrics vs Modern LLM/RAG evaluation frameworks (LLM-as-a-Judge, RAGAS).

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: Compare Classification Metrics: Accuracy vs Precision vs Recall.

**Answer:**

- **Accuracy:** (TP + TN) / Total. Good for balanced datasets. Terrible for imbalanced datasets (e.g., detecting a rare 1% disease. A model that says "everyone is healthy" is 99% accurate but useless).
- **Precision:** TP / (TP + FP). Out of all the positive predictions I made, how many were actually correct? (Focus: Minimizing False Positives).
- **Recall (Sensitivity):** TP / (TP + FN). Out of all the actual positive cases in reality, how many did I successfully find? (Focus: Minimizing False Negatives).

### Q2: What is the F1-Score?

**Answer:**

The harmonic mean of Precision and Recall. `2 * (Precision * Recall) / (Precision + Recall)`
It is the standard metric used when you have imbalanced classes and need a single number that balances the trade-off between Precision (not crying wolf) and Recall (not missing the wolf).

---

## 🟡 Medium (Intermediate)

### Q3: What is the ROC Curve and AUC?

**Answer:**

Classification models don't just output "Yes/No". They output a probability (e.g., 0.85). The default threshold is 0.5. If you change the threshold to 0.9, you get fewer False Positives but more False Negatives.

- **ROC Curve (Receiver Operating Characteristic):** A graph plotting the True Positive Rate (Recall) vs the False Positive Rate at *every possible threshold* from 0.0 to 1.0.
- **AUC (Area Under the Curve):** A single number summarizing the ROC curve.
  - AUC = 1.0 (Perfect model).
  - AUC = 0.5 (Random guessing, a diagonal line).
  - AUC measures the model's ability to rank a random positive example higher than a random negative example, independent of the threshold chosen.

### Q4: Explain NLP Metrics: BLEU vs ROUGE.

**Answer:**

Used before LLMs became standard. They rely on exact string/n-gram matching against a human reference text.

- **BLEU (Bilingual Evaluation Understudy):** Used for Machine Translation. It is a Precision-focused metric. It counts how many n-grams in the machine's output are present in the human reference. 
- **ROUGE (Recall-Oriented Understudy for Gisting Evaluation):** Used for Summarization. It is a Recall-focused metric. It counts how many n-grams from the human reference were successfully captured by the machine's output.

*Why they fail today:* They do not understand semantics. If the human reference is "The dog ran", and the LLM outputs "The canine sprinted", BLEU/ROUGE score it as a 0% match, even though it is semantically perfect.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q5: What is "LLM-as-a-Judge"?

**Answer:**

Because exact-match metrics (BLEU) fail on open-ended generation, modern evaluation uses a powerful LLM (like GPT-4) to grade the outputs of a weaker or fine-tuned LLM (like Llama-3).

**The Process:**
You provide GPT-4 with a rubric via a strict prompt:
> "You are an impartial judge. Evaluate the Assistant's response based on the User's query. Score it from 1 to 5 on 'Helpfulness' and 'Toxicity'. Provide your reasoning first, then the score in JSON format."

*Drawbacks to know for interviews:*
1. **Cost & Latency:** Running GPT-4 over 10,000 test cases is expensive.
2. **Positional Bias:** If grading two responses (A vs B), the judge often prefers whichever response was presented first. (Must swap A and B and run twice to mitigate).
3. **Verbosity Bias:** LLM judges tend to assign higher scores to longer answers, even if they are fluff.

---

### Q6: Explain the RAGAS framework (RAG Assessment).

**Answer:**

RAGAS is the industry standard for evaluating RAG pipelines without requiring human-labeled ground truth for every query. It breaks evaluation into three components, using LLMs-as-a-Judge:

**1. Context Relevance (Evaluates the Retriever & Vector DB)**
- *Metric:* Is the retrieved context actually useful for answering the query, or is it irrelevant noise? 
- *How it works:* The Judge LLM extracts all distinct statements from the retrieved chunks and calculates the ratio of statements that are directly relevant to the user's question.

**2. Faithfulness (Evaluates Hallucination in the LLM)**
- *Metric:* Did the LLM make things up, or is its answer strictly derived from the retrieved context?
- *How it works:* The Judge LLM extracts all claims from the generated answer and checks if each claim can be logically deduced *only* from the provided chunks.

**3. Answer Relevance (Evaluates End-to-End pipeline)**
- *Metric:* Does the final generated answer directly address the user's query?
- *How it works:* (Reverse-engineering). The Judge LLM reads the final answer and is asked to generate a question that this answer would belong to. If the generated question matches the user's original query, the score is high.

---

*End of Evaluation Theory — 6 questions covering traditional ML metrics (F1, AUC), legacy NLP metrics (BLEU/ROUGE), and modern LLM-as-a-judge / RAGAS frameworks.*
