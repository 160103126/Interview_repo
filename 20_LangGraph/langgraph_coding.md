# LangGraph — Coding Patterns

> Practical implementation of stateful, multi-actor applications with LLMs. Focusing on State Graphs, Nodes, Edges, and cyclic ReAct agent architectures.

---

## Table of Contents

- [Q1: Basic ReAct Agent Graph](#q1-basic-react-agent-graph)
- [Q2: Multi-Agent System (Coder & Reviewer)](#q2-multi-agent-system-coder--reviewer)
- [Q3: Checkpointing & Time Travel (Human-in-the-Loop)](#q3-checkpointing--time-travel-human-in-the-loop)
- [Q4: Branching and Parallel Execution](#q4-branching-and-parallel-execution)

---

### Q1: Basic ReAct Agent Graph

**Problem:** Build an agent that can access a "Search" tool, decide whether to use it based on the user's query, and loop until it has the answer.

```python
from typing import TypedDict, Annotated, Sequence
import operator
from langchain_openai import ChatOpenAI
from langchain_core.messages import BaseMessage, HumanMessage
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode

class AgentState(TypedDict):
    # operator.add ensures messages are appended, not overwritten
    messages: Annotated[Sequence[BaseMessage], operator.add]

def search_weather(location: str) -> str:
    """Gets the weather for a location."""
    return f"The weather in {location} is 72 degrees and sunny."

tools = [search_weather]
model = ChatOpenAI(temperature=0).bind_tools(tools)

def call_model(state: AgentState):
    response = model.invoke(state["messages"])
    return {"messages": [response]}

def should_continue(state: AgentState) -> str:
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "continue"
    return "end"

workflow = StateGraph(AgentState)
workflow.add_node("agent", call_model)
workflow.add_node("action", ToolNode(tools)) 

workflow.set_entry_point("agent")
workflow.add_conditional_edges("agent", should_continue, {"continue": "action", "end": END})
workflow.add_edge("action", "agent")

app = workflow.compile()
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The State (`TypedDict`):** Unlike standard LangChain where data flows in a pipe, LangGraph uses a globally shared `State` object. Every node receives the current state and returns a dict containing *updates* to that state. 
- **`operator.add`:** If we didn't annotate `messages` with `add`, returning `{"messages": [response]}` would overwrite the entire chat history. `add` tells LangGraph to append it.
- **Cyclic Execution:** DAGs cannot loop. LangGraph allows cycles. The `action` node explicitly loops back to the `agent` node.

---

### Q2: Multi-Agent System (Coder & Reviewer)

**Problem:** Build a system where a Coder writes code, and a Reviewer critiques it. If rejected, it loops back to the Coder.

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

class CodeState(TypedDict):
    task: str; code: str; feedback: str; iterations: int

llm = ChatOpenAI(temperature=0)

def coder_node(state: CodeState):
    prompt = ChatPromptTemplate.from_messages([
        ("system", "Write code for the task. If there is feedback, fix the code."),
        ("human", "Task: {task}\nCurrent Code: {code}\nFeedback: {feedback}")
    ])
    res = (prompt | llm).invoke(state)
    return {"code": res.content, "iterations": state.get("iterations", 0) + 1, "feedback": ""}

def reviewer_node(state: CodeState):
    prompt = ChatPromptTemplate.from_messages([
        ("system", "Output 'APPROVE' if code is perfect. Otherwise, output feedback."),
        ("human", "Task: {task}\nCode: {code}")
    ])
    res = (prompt | llm).invoke(state)
    return {"feedback": res.content}

def route_after_review(state: CodeState) -> str:
    if "APPROVE" in state["feedback"] or state["iterations"] > 3:
        return "end"
    return "rewrite"

builder = StateGraph(CodeState)
builder.add_node("coder", coder_node)
builder.add_node("reviewer", reviewer_node)
builder.set_entry_point("coder")
builder.add_edge("coder", "reviewer")
builder.add_conditional_edges("reviewer", route_after_review, {"end": END, "rewrite": "coder"})
app = builder.compile()
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Separation of Concerns:** Instead of a single massive prompt, we create two specialized nodes with isolated System Prompts.
- **The Infinite Loop Breaker:** The `iterations` counter in the state, checked in `route_after_review`, acts as a hard circuit breaker to prevent massive API bills.

---

### Q3: Checkpointing & Time Travel (Human-in-the-Loop)

**Problem:** Add a checkpoint to pause the graph, ask a human for approval, and then resume execution.

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

# Define the graph nodes (agent, action)
builder = StateGraph(AgentState)
# ... add nodes ...

# Setup Sqlite Checkpointer
conn = sqlite3.connect("checkpoints.sqlite", check_same_thread=False)
memory = SqliteSaver(conn)

# Compile the graph WITH memory and an interrupt
app = builder.compile(
    checkpointer=memory,
    interrupt_before=["action"]  # Pause BEFORE the action node executes
)

# 1. Start the run
config = {"configurable": {"thread_id": "trade_1"}}
inputs = {"messages": [HumanMessage(content="Buy 100 shares of AAPL.")]}

for event in app.stream(inputs, config=config):
    pass # Reaches interrupt and pauses

# 2. Human updates the state
app.update_state(config, {"messages": [HumanMessage(content="Actually, buy 50 shares instead.")]})

# 3. Resume the graph from where it paused
for event in app.stream(None, config=config):
    pass
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **`interrupt_before`:** Tells the compiler to execute normally until it reaches the edge pointing to `"action"`. It then suspends execution, saves the state to the DB, and yields control back to the main thread.
- **State Mutation (Time Travel):** A human operator can use `app.update_state()` to inject new instructions or modify variables before calling `app.stream(None)` to resume execution from the exact point it paused.

---

### Q4: Branching and Parallel Execution

**Problem:** Execute two nodes in parallel and aggregate their results.

```python
def fan_out(state: AgentState):
    # This node just serves as a starting point for parallel branches
    return {}

def aggregate(state: AgentState):
    # This node receives the merged state from both parallel branches
    return {}

builder = StateGraph(AgentState)
builder.add_node("start", fan_out)
builder.add_node("branch_a", run_task_a)
builder.add_node("branch_b", run_task_b)
builder.add_node("aggregate", aggregate)

builder.set_entry_point("start")
# Fan out
builder.add_edge("start", "branch_a")
builder.add_edge("start", "branch_b")
# Fan in
builder.add_edge("branch_a", "aggregate")
builder.add_edge("branch_b", "aggregate")
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Parallelism:** LangGraph automatically executes `branch_a` and `branch_b` simultaneously in different threads. 
- **Aggregation:** The `aggregate` node waits for both to complete. The `AgentState` automatically merges the results (using `operator.add` for lists or replacing dict keys) before passing them to the final node.

---

*End of LangGraph Coding — Restored patterns covering cyclic graphs, Multi-Agent systems, HITL Checkpoints, and Parallel Branching.*
