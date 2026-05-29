# Agentic AI — Theory & Concepts

> The theory behind autonomous AI agents: Planning algorithms, Memory structures, Multi-Agent orchestration, and Tool Calling mechanisms.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What defines an Agent compared to a standard LLM?

**Answer:**

An LLM is a stateless text completion engine. An **Agent** is a system built *around* an LLM that gives it agency.

To be an agent, the system must have:
1. **Perception:** The ability to take in state from the environment (e.g., reading an API response, viewing a webpage).
2. **Brain/Reasoning:** The LLM itself, used to process the perception and decide what to do next.
3. **Action:** The ability to execute tools (Python scripts, SQL queries, Web Searches) to affect the environment.
4. **Autonomy:** The ability to execute the Perceive -> Reason -> Act loop multiple times independently until a goal is reached.

---

### Q2: What is the ReAct Framework?

**Answer:**

ReAct (Reasoning + Acting) is the foundational prompt structure for single-agent systems.

It forces the LLM to interleave internal thoughts with external actions.
1. **Thought:** The LLM reasons about the current state ("I need to find the user's IP address").
2. **Action:** The LLM requests to use a tool (`get_ip()`).
3. **Observation:** The system executes the tool and returns the result ("192.168.1.1").
4. **Thought:** The LLM processes the observation ("Now that I have the IP, I can look up the location").
5. **Action:** Tool call (`get_location("192.168.1.1")`).

This loop continues until the LLM decides it has enough information to output a Final Answer.

---

## 🟡 Medium (Intermediate)

### Q3: Explain the Plan-and-Solve (Plan-and-Execute) Architecture.

**Answer:**

Standard ReAct agents suffer when given long-horizon, complex tasks. They lose track of the main goal and get stuck in local loops. 

**Plan-and-Execute** separates the reasoning into two different agents:
1. **The Planner:** Takes the user's complex goal and outputs a step-by-step sequential plan (e.g., 1. Search web for X, 2. Scrape data, 3. Summarize, 4. Save to file).
2. **The Executor:** Takes *only* Step 1 and executes it. 
3. **The Replanner:** Looks at the result of Step 1, updates the global state, and tells the Executor to start Step 2.

*Benefits:* Highly reliable for complex tasks, reduces hallucinations, and keeps context windows small.

---

### Q4: How do Agents use Memory (Short-term vs Long-term)?

**Answer:**

Agents need memory to maintain continuity across loops and sessions.

- **Short-Term Memory:** This is the LLM's context window. It contains the prompt, the recent chat history, and the scratchpad of recent tool observations. It is ephemeral and erased when the session ends.
- **Long-Term Memory:** Persistent storage used to recall facts from past sessions.
  - *Episodic Memory:* Remembering past events. ("Last week, the user asked me to prefer Python over Java"). Usually implemented via a Vector DB (RAG) that the agent can search.
  - *Semantic Memory:* Storing facts in a structured Knowledge Graph (e.g., User -> works_at -> Google).

---

### Q5: What is Tool Calling (Function Calling) at the model level?

**Answer:**

Originally, developers had to rely on brittle Prompt Engineering to get tools to work (e.g., "Output exactly `Action: search, Input: cat`"). The LLM would often hallucinate the JSON formatting.

**Native Function Calling (OpenAI / Anthropic APIs):**
Models are now explicitly fine-tuned on JSON schemas.
You pass the LLM a list of JSON schemas describing your Python functions. The LLM natively detects if a function is needed, pauses generation, and outputs a highly reliable, perfectly formatted JSON string containing the exact arguments needed to call your function.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q6: What is a Multi-Agent System (MAS)? Compare AutoGen to LangGraph.

**Answer:**

Instead of one massive agent trying to do everything, a MAS uses a society of specialized agents working together.

- **Microsoft AutoGen:** Uses a purely **Conversational** paradigm. Agents literally send chat messages to each other. You define a `UserProxy` agent, a `Coder` agent, and a `Reviewer` agent, and they argue in a chatroom until the code works. (Highly emergent, but can be chaotic and hard to control in production).
- **LangGraph:** Uses a **State Machine (Graph)** paradigm. Agents don't chat freely. They are nodes in a graph that strictly read from and write to a shared State object, governed by rigid conditional edges. (Less emergent, but highly deterministic and production-ready).

---

### Q7: Explain Reflection and Self-Critique in Agents.

**Answer:**

**Reflection** is the process where an agent evaluates its own past actions or outputs to improve them.

*Example Workflow (Self-Correction):*
1. **Generator Agent:** Writes a python script to sort an array.
2. **System:** Runs the script and captures the output/error.
3. **Critique Agent (Reflection):** Given the original prompt, the script, and the output, it writes a critique: "The script fails on edge case X. The time complexity is O(n^2), it should be O(n log n)."
4. **Generator Agent:** Receives the critique and generates V2 of the script.

This loop significantly boosts the success rate on benchmarks like HumanEval compared to zero-shot generation.

---

### Q8: What are the primary failure modes of Agentic systems?

**Answer:**

Building agents is easy; making them reliable is incredibly hard.

1. **Infinite Loops:** The agent repeatedly calls a tool with the same bad arguments, getting the same error, until the context window fills up. (Fix: Max iteration limits, tracking visited states).
2. **Tool Hallucination:** The agent tries to use a tool that doesn't exist, or passes arguments that aren't in the schema. (Fix: Native function calling fine-tuning, strict pydantic validation).
3. **Context Degradation:** As the agent executes 20 steps, the context window fills with verbose tool outputs (like full HTML from a web scrape). The agent "forgets" its original goal. (Fix: Summarization nodes between steps, LLM filtering of tool outputs).

---

### Q9: What are Human-in-the-Loop (HITL) patterns for Agents?

**Answer:**

Fully autonomous agents are dangerous in production (financial transactions, database modifications, email sending). HITL patterns add human checkpoints.

**Pattern 1: Approval Gate**
The agent pauses before executing high-risk tools and presents the proposed action to a human for explicit approval.
```
Agent: "I want to execute: DELETE FROM users WHERE last_login < '2023-01-01'"
Human: [Approve] / [Reject] / [Modify]
```

**Pattern 2: Escalation**
The agent handles routine queries autonomously but escalates complex/ambiguous ones to a human operator. LangGraph implements this via `interrupt_before` on specific nodes.

**Pattern 3: Confidence-Based Routing**
```python
if agent_confidence > 0.9:
    execute_autonomously()
elif agent_confidence > 0.7:
    execute_with_logging()  # Auto-execute but flag for review
else:
    escalate_to_human()
```

**Pattern 4: Edit-and-Continue**
The agent generates a full plan, presents it to the human. The human can edit individual steps before the agent executes. (LangGraph's `Command` pattern supports this).

---

### Q10: Explain Multi-Agent Communication Protocols.

**Answer:**

When multiple agents collaborate, they need structured communication. Different frameworks use different paradigms:

**1. Shared State (LangGraph):**
All agents read from and write to a single shared state object. Communication is implicit — Agent A modifies the state, Agent B reads the updated state.
- **Pros:** Deterministic, debuggable, easy to track data flow.
- **Cons:** State can grow large; no "private" communication between specific agents.

**2. Message Passing (AutoGen, CrewAI):**
Agents send direct messages to each other in a conversation thread. Each message has a sender, recipient, and content.
- **Pros:** Natural for debate/discussion patterns (Coder → Reviewer → Coder).
- **Cons:** Hard to control, conversations can go off-track.

**3. Blackboard Architecture:**
A central "blackboard" stores all partial solutions. Agents independently monitor the blackboard and contribute when they can.
- **Pros:** Highly scalable, agents can be added/removed without changing the system.
- **Cons:** Non-deterministic execution order.

**4. Supervisor Pattern:**
A central "Supervisor" agent routes tasks to specialized worker agents and aggregates their outputs.
- **Pros:** Simple to reason about. The supervisor controls the flow.
- **Cons:** Supervisor becomes a bottleneck; single point of failure.

---

### Q11: How do you add Guardrails to Agent systems?

**Answer:**

Agents have more failure modes than standard LLM applications because they can take actions (modify databases, send emails, execute code).

**Layer 1: Tool-Level Guardrails**
```python
@tool
def execute_sql(query: str) -> str:
    # Block destructive operations
    if any(keyword in query.upper() for keyword in ["DROP", "DELETE", "TRUNCATE"]):
        raise ToolError("Destructive SQL operations are blocked")
    # Rate limiting
    if rate_limiter.exceeded():
        raise ToolError("Tool call rate limit exceeded")
    return db.execute(query)
```

**Layer 2: Agent-Level Guardrails**
- **Max iterations:** Kill the agent after N steps to prevent infinite loops.
- **Budget limits:** Track token usage and stop if cost exceeds threshold.
- **Visited state tracking:** If the agent is in the same state as 3 iterations ago, break the loop.

**Layer 3: Output Guardrails**
- Run the agent's final output through a safety classifier before returning to the user.
- Check for PII leakage, toxic content, or off-topic responses.

**Layer 4: Sandbox Execution**
- Execute code-generating agents in sandboxed environments (Docker containers, E2B, Modal).
- Never let an agent execute code on the production host.

---

### Q12: How do you evaluate and benchmark Agent systems?

**Answer:**

Agent evaluation is fundamentally harder than LLM evaluation because you must evaluate the *trajectory* (sequence of actions), not just the final output.

**Metrics:**
1. **Task Success Rate:** Did the agent complete the task correctly? (Binary: pass/fail).
2. **Step Efficiency:** How many steps did it take? (Fewer = better).
3. **Tool Use Accuracy:** Did it call the right tools with correct arguments?
4. **Cost Efficiency:** Total tokens consumed (prompt + completion) per task.
5. **Error Recovery:** When a tool fails, does the agent recover or crash?

**Benchmarks:**
| Benchmark | What it tests |
|-----------|--------------|
| **SWE-Bench** | Can the agent fix real GitHub issues in open-source repos? |
| **WebArena** | Can the agent navigate websites and complete tasks? |
| **GAIA** | Can the agent answer questions requiring multi-step reasoning + tool use? |
| **HumanEval-Agent** | Can the agent write and debug code end-to-end? |

**Production Evaluation Pattern:**
1. Create a golden test suite of 50+ tasks with known correct trajectories.
2. Run the agent on each task.
3. Compare its trajectory to the golden trajectory (or just check if the final answer matches).
4. Track success rate over time — any prompt/model change that drops success rate below threshold blocks deployment.

---

*End of Agentic AI Theory — 12 comprehensive questions covering ReAct, Plan-and-Execute, Memory, Tool Calling, Multi-Agent systems, Reflection, Failure Modes, Human-in-the-Loop, Communication Protocols, Guardrails, and Evaluation.*

