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

### Q7: How do you detect and measure Hallucinations?

**Answer:**

Hallucination detection is one of the hardest problems in LLM evaluation. There are two types:

**1. Intrinsic Hallucination:** The LLM contradicts the source material it was given.
- *Example:* Context says "Revenue was $5M." LLM says "Revenue was $50M."
- **Detection:** Use an NLI (Natural Language Inference) model to check if each claim in the output is "Entailed" by the context. If it's "Contradicted," it's a hallucination.

**2. Extrinsic Hallucination:** The LLM generates plausible but unverifiable information not present in the source.
- *Example:* Context says nothing about market share. LLM says "The company holds 30% market share."
- **Detection:** Extract all claims from the output. Check if each claim can be traced back to a chunk in the context. Claims with no supporting evidence are flagged.

**Metrics:**
- **Faithfulness Score (RAGAS):** Ratio of claims that are entailed by the context vs total claims.
- **SelfCheckGPT:** Ask the LLM the same question multiple times. If it generates inconsistent facts across runs, those facts are likely hallucinated.

---

### Q8: What is BERTScore? How does it improve over BLEU/ROUGE?

**Answer:**

BLEU and ROUGE rely on exact n-gram matching. "The canine sprinted" gets a 0% BLEU score when compared to "The dog ran" — even though they mean the same thing.

**BERTScore** uses pre-trained BERT embeddings to compute semantic similarity:
1. Each token in the candidate text and reference text is embedded using BERT.
2. For each candidate token, it finds the most similar reference token using cosine similarity.
3. Precision, Recall, and F1 are computed based on these maximum similarity scores.

**Advantages:**
- Captures semantic equivalence ("canine" ≈ "dog").
- Handles paraphrasing, synonyms, and word order variations.
- Correlates much better with human judgment than BLEU/ROUGE.

**Usage:**
```python
from bert_score import score
P, R, F1 = score(candidates, references, lang="en")
```

---

### Q9: How do you design a Human Evaluation framework for LLMs?

**Answer:**

Automated metrics (BLEU, RAGAS) provide scale but miss nuance. For critical applications (medical, legal), human evaluation is essential.

**Framework Design:**
1. **Define Criteria:** Create a rubric with 3-5 axes (e.g., Helpfulness, Factual Accuracy, Toxicity, Formatting, Relevance). Each axis scored 1-5.
2. **Select Evaluators:** Use domain experts (doctors for medical AI) or trained crowd workers. Minimum 3 evaluators per sample for statistical reliability.
3. **Blind Evaluation:** Evaluators should NOT know which model generated which response. Randomize the order of Model A vs Model B outputs.
4. **Inter-Annotator Agreement:** Calculate Cohen's Kappa or Krippendorff's Alpha. If agreement is low (< 0.6), the rubric is ambiguous and must be refined.
5. **Sample Size:** Evaluate at least 200-500 randomly sampled outputs for statistical significance.

**Pairwise Comparison (Simpler alternative):**
Instead of scoring on a rubric, show evaluators two responses side-by-side and ask: "Which is better?" This is faster and produces clearer signals. (This is how Chatbot Arena / LMSYS works.)

---

### Q10: How do you A/B Test LLM-powered features?

**Answer:**

A/B Testing LLMs is fundamentally harder than A/B testing traditional software because outputs are non-deterministic.

**The Process:**
1. **Control Group (A):** Current production prompt/model.
2. **Treatment Group (B):** New prompt/model variant.
3. **Randomization:** Route 50% of users to each group using consistent user-level hashing (same user always sees the same variant).
4. **Metrics:**
   - **Online Metrics (Business KPIs):** Click-through rate, task completion rate, user satisfaction score (thumbs up/down).
   - **Offline Metrics (Quality):** Run LLM-as-a-Judge on a sample of outputs from both groups.
5. **Duration:** Run for at least 1-2 weeks to account for day-of-week effects and user behavior variance.
6. **Statistical Significance:** Use a two-proportion z-test or chi-squared test. Require p < 0.05 before declaring a winner.

**Guardrails:** Always monitor safety metrics (toxicity, PII leaks) for the treatment group. If safety degrades, kill the experiment immediately.

---

### Q11: What are LLM Benchmarks? (MMLU, HumanEval, HellaSwag)

**Answer:**

Benchmarks provide standardized comparisons between LLMs.

| Benchmark | What it measures | Format |
|-----------|-----------------|--------|
| **MMLU** | World knowledge across 57 subjects (math, history, law) | Multiple choice (A/B/C/D) |
| **HumanEval** | Code generation (Python function completion) | Write code, run unit tests |
| **HellaSwag** | Common sense reasoning (sentence completion) | Multiple choice |
| **GSM8K** | Grade school math reasoning | Open-ended math problems |
| **MT-Bench** | Multi-turn conversation quality | LLM-as-a-Judge scoring |
| **TruthfulQA** | Resistance to generating false but popular claims | Open-ended factual questions |

**Caveats for interviews:**
1. **Data Contamination:** If a model saw benchmark questions during pre-training, scores are inflated. (A major concern with closed-source models).
2. **Benchmark Saturation:** Models are now scoring 90%+ on many benchmarks, making them less useful for differentiation.
3. **Real-world performance ≠ Benchmark performance:** A model might ace MMLU but fail on your specific domain (legal, medical).

---

### Q12: Design an End-to-End Evaluation Pipeline for a RAG system.

**Answer:**

A production RAG system needs continuous, automated evaluation — not one-off notebook checks.

**Pipeline Architecture:**
```
[Synthetic Test Set Generator] → [RAG Pipeline Under Test] → [Multi-Metric Evaluator] → [Dashboard + Alerting]
```

**Step 1: Test Set Generation**
- Use an LLM to generate synthetic question-answer pairs from your document corpus.
- Manually curate 50-100 "golden" Q&A pairs for regression testing.

**Step 2: Automated Evaluation Metrics**
Run each test question through the RAG pipeline and compute:
- **Retrieval Metrics:** Context Relevance (RAGAS), MRR@K, NDCG@K.
- **Generation Metrics:** Faithfulness (RAGAS), Answer Relevance, BERTScore vs golden answer.
- **Latency Metrics:** End-to-end response time, retrieval time, generation time.

**Step 3: Regression Detection**
- Compare current scores to the baseline from the last deployment.
- If Faithfulness drops by > 5%, block the deployment and alert the team.

**Step 4: Continuous Monitoring**
- Sample 1% of production traffic daily.
- Run the evaluation pipeline on sampled queries.
- Dashboard tracks scores over time to detect gradual drift.

---

*End of Evaluation Theory — 12 comprehensive questions covering ML metrics (F1, AUC), NLP metrics (BLEU/ROUGE/BERTScore), LLM-as-a-Judge, RAGAS, Hallucination Detection, Human Evaluation, A/B Testing, Benchmarks, and E2E Evaluation Pipelines.*

