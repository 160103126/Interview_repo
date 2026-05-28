# Evaluation — Coding Patterns

> Practical implementation of LLM Evaluation frameworks. Focuses on the "LLM-as-a-Judge" architecture for programmatically scoring RAG pipelines and Agentic workflows.

---

## Table of Contents

- [Q1: LLM-as-a-Judge (Context Relevance)](#q1-llm-as-a-judge-context-relevance)
- [Q2: Agent Trajectory Evaluation](#q2-agent-trajectory-evaluation)

---

### Q1: LLM-as-a-Judge (Context Relevance)

**Problem:** You have a RAG pipeline. How do you programmatically evaluate if the retrieved context actually contains the answer to the user's question without human grading?

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate

# 1. Define the Grading Schema
class Grade(BaseModel):
    score: int = Field(description="Score from 1 to 5. 5 means the context perfectly answers the question.")
    reasoning: str = Field(description="Step-by-step reasoning for the assigned score.")

# 2. Set up the Judge LLM (Use a powerful model like GPT-4)
judge_llm = ChatOpenAI(model="gpt-4", temperature=0)
structured_judge = judge_llm.with_structured_output(Grade)

# 3. Create the Evaluation Prompt
eval_prompt = PromptTemplate.from_template("""
You are an impartial judge evaluating a Retrieval Augmented Generation (RAG) system.
Your task is to evaluate the relevance of the retrieved context to the user's question.

User Question: {question}
Retrieved Context: {context}

Evaluate if the context contains sufficient information to fully answer the question.
Provide a reasoning, and then a score from 1 to 5.
""")

eval_chain = eval_prompt | structured_judge

# 4. Run the Evaluation
question = "What is the company policy on remote work?"
context = "Employees must be in the office Tuesday through Thursday. Monday and Friday are flexible remote days."

result = eval_chain.invoke({"question": question, "context": context})

print(f"Score: {result.score}/5")
print(f"Reasoning: {result.reasoning}")
```

#### 🧠 Architectural Walkthrough
- **The Judge Model:** You should always use a highly capable model (GPT-4 or Claude 3.5 Sonnet) as the judge, even if your actual application uses a smaller, faster model (like Llama-3 8B or GPT-3.5) for generation.
- **Structured Output:** By forcing the judge to output a Pydantic `Grade` model, we can easily run this in a `for` loop over thousands of test queries, average the `score` integer, and track our RAG pipeline's retrieval accuracy over time in an automated CI/CD pipeline.

---

### Q2: Agent Trajectory Evaluation

**Problem:** Evaluating an Agent is harder than evaluating RAG because the Agent takes multiple steps (Tool Calls). How do you evaluate the *trajectory* of the Agent?

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate

class TrajectoryGrade(BaseModel):
    is_efficient: bool = Field(description="True if the agent took the optimal number of steps. False if it hallucinated tools or looped.")
    did_hallucinate: bool = Field(description="True if the agent hallucinated a tool output instead of relying on the actual tool response.")
    reasoning: str

judge_llm = ChatOpenAI(model="gpt-4", temperature=0).with_structured_output(TrajectoryGrade)

trajectory_prompt = PromptTemplate.from_template("""
You are auditing an AI Agent. Review its trajectory (the sequence of thoughts, tool calls, and tool outputs).

User Goal: {goal}
Agent Trajectory:
{trajectory}

Determine if the agent was efficient (no redundant tool calls) and if it relied strictly on tool outputs without hallucinating data.
""")

trajectory_log = """
1. Action: SearchWeather(location="San Francisco")
1. Observation: 75 degrees and sunny.
2. Action: SearchWeather(location="San Francisco") # REDUNDANT!
2. Observation: 75 degrees and sunny.
3. Final Answer: It is 80 degrees in SF. # HALLUCINATION!
"""

result = judge_llm.invoke({
    "goal": "What is the weather in San Francisco?",
    "trajectory": trajectory_log
})

print(f"Efficient? {result.is_efficient}")    # False
print(f"Hallucinated? {result.did_hallucinate}") # True
```

#### 🧠 Architectural Walkthrough
- **Trajectory Logs:** An Agent's output is not just its final answer. The sequence of actions is the "trajectory." 
- **Evaluating Logic over Content:** In this pattern, we don't just ask if the final answer is correct. We ask the Judge LLM to look for **loops** (calling the same tool twice), **hallucinations** (ignoring the observation), and **tool misuse** (passing bad JSON). This is how frameworks like LangSmith evaluate agent performance.
