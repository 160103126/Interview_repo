# Agentic AI — Real-World Scenarios & Troubleshooting

> Deep dives into the failure modes of autonomous agents and how to engineer robust systems capable of recovering from errors.

---

## Table of Contents

- [Scenario 1: The Infinite Loop Problem](#scenario-1-the-infinite-loop-problem)
- [Scenario 2: PII / HIPAA Redaction in Tool Calls](#scenario-2-pii--hipaa-redaction-in-tool-calls)
- [Scenario 3: The Self-Healing Reflection Loop](#scenario-3-the-self-healing-reflection-loop)
- [Scenario 4: Tool Hallucination (Fake APIs)](#scenario-4-tool-hallucination-fake-apis)
- [Scenario 5: Multi-Agent Argumentation (Deadlocks)](#scenario-5-multi-agent-argumentation-deadlocks)
- [Scenario 6: Context Degradation (Amnesia)](#scenario-6-context-degradation-amnesia)
- [Scenario 7: State Machine Checkpoint Corruption](#scenario-7-state-machine-checkpoint-corruption)
- [Scenario 8: API Rate Limiting & Exponential Backoff](#scenario-8-api-rate-limiting--exponential-backoff)

---

## Scenario 1: The Infinite Loop Problem

**The Scenario:**
Your ReAct agent has a `query_sql_db` tool. The agent tries to run `SELECT * FROM users;`. The database returns an error: `Table 'users' does not exist`. 
Instead of adjusting, the agent just calls the exact same tool with the exact same SQL query. It gets the same error. It does this 50 times until your API bill spikes and the context window crashes.

**Question:** How do you prevent autonomous agents from getting stuck in infinite failure loops?

**Answer:**

**The Fix:**
1. **Hard Limits (The Circuit Breaker):** The easiest and most mandatory fix. In frameworks like LangGraph, you must enforce a `max_iterations` counter in the state. If `iterations > 5`, forcefully route to an `END` node or fallback to a human.
2. **Forced Reflection Prompts:** When returning the error to the LLM, do not just return the raw text `Table 'users' does not exist`. Append a strict systemic instruction: 
   *"Error executing tool. Do NOT repeat the same input. You must use the `list_tables` tool to verify the schema before trying again."*
3. **State History Checking:** Add a middleware function to your Tool Node that hashes the tool arguments. If the exact same arguments are submitted twice in a row, the middleware intercepts it and returns: *"You already tried this and it failed. Try a different approach."*

---

## Scenario 2: PII / HIPAA Redaction in Tool Calls

**The Scenario:**
You built a Medical Agent that summarizes patient records. The agent decides to use its `search_web` tool to look up a rare symptom. Unfortunately, it hallucinates and includes the patient's real name and Social Security Number in the Google Search query string. You have just committed a massive HIPAA violation by sending PII to a public API.

**Question:** How do you secure tool executions against PII leaks?

**Answer:**

**The Fix (Tool Middleware & Data Masking):**
You must decouple the LLM from direct execution of external tools.
1. **Presidio Integration:** Microsoft Presidio is an NLP engine designed to detect PII. 
2. **The Middleware Pipeline:**
   - The LLM outputs a tool call: `search_web(query="Does John Doe (SSN: 123-45) have Lupus?")`.
   - Before executing the HTTP request, the Tool Node routes the arguments through Presidio.
   - Presidio detects the name and SSN, and redacts them: `search_web(query="Does [PERSON] (SSN: [ID]) have Lupus?")`.
3. **Execution:** The safely redacted string is sent to the Google Search API, ensuring compliance.

---

## Scenario 3: The Self-Healing Reflection Loop

**The Scenario:**
You have a Coding Agent writing Python scripts. It frequently writes scripts that look correct but have subtle `IndentationErrors` or missing `import` statements when actually executed on the server.

**Question:** How do you build a Self-Healing loop?

**Answer:**

**The Fix:**
1. **The Sandbox Tool:** Give the agent a `run_python` tool. This tool executes the code in a secure, sandboxed Docker container and returns the `stdout` and `stderr`.
2. **The Reflection Node:** When the `run_python` tool returns a traceback (e.g., `ModuleNotFoundError: No module named 'requests'`), route the graph to a dedicated `Reflector` agent.
3. **The Critique:** The Reflector agent's system prompt is: *"You are a senior debugging engineer. Look at the original code and the error trace. Explain exactly why the error occurred and output the exact line of code to fix it."*
4. **The Loop:** The Reflector passes the critique back to the original Coding Agent, which rewrites the code and calls the `run_python` tool again. This loop continues until `stderr` is empty.

---

## Scenario 4: Tool Hallucination (Fake APIs)

**The Scenario:**
Your agent has two tools: `get_weather(city)` and `get_stock(ticker)`. 
A user asks: "Who won the Superbowl?"
Because the LLM is eager to help, it hallucinates a tool that doesn't exist. It outputs: `Action: search_sports(query="Superbowl winner")`. The system crashes because `search_sports` is not defined in your python code.

**Question:** How do you prevent agents from hallucinating tools or parameters?

**Answer:**

**The Fix:**
1. **Native Tool Calling API:** Stop using zero-shot Prompt Engineering (e.g., ReAct prompts telling the LLM to output "Action: X"). Use OpenAI/Anthropic's native Function Calling API, where the model is explicitly fine-tuned to only select from the provided JSON schemas.
2. **Strict Pydantic Validation:** Wrap your python tools in Pydantic models. If the LLM tries to call `get_weather(city="NYC", date="tomorrow")` but your tool only accepts `city`, Pydantic will throw a `ValidationError`. 
3. **Graceful Degradation:** Catch the `ValidationError` and feed it directly back to the LLM: *"Error: 'date' is not a valid parameter for get_weather. Valid parameters are: ['city']."*. The LLM will self-correct on the next turn.

---

## Scenario 5: Multi-Agent Argumentation (Deadlocks)

**The Scenario:**
In your LangGraph application, a Coder Agent submits a PR to a Reviewer Agent. 
The Reviewer says: *"You must use a while loop here."*
The Coder replies: *"No, a for loop is more Pythonic."*
The Reviewer replies: *"I reject this, use a while loop."*
They argue in an infinite loop, never reaching a consensus.

**Question:** How do you resolve deadlocks in Multi-Agent systems?

**Answer:**

**The Fix (The Supervisor / Tie-Breaker pattern):**
1. **Iteration Tracking:** Track the number of back-and-forth exchanges in the shared State.
2. **The Escalation Edge:** If `exchanges > 3`, the routing edge forcefully diverts the graph away from the Coder/Reviewer loop and sends the entire chat history to a third agent: the **Supervisor Agent**.
3. **The Executive Decision:** The Supervisor Agent (prompted to act as a CTO) reads the argument, makes a final executive decision, and generates the final code, routing directly to the `END` node.

---

## Scenario 6: Context Degradation (Amnesia)

**The Scenario:**
An agent is tasked with researching 5 different competitors. It uses the `search_web` tool 15 times, scraping massive HTML pages. By the time it is researching the 4th competitor, the LLM's context window is filled with 100,000 tokens of raw HTML garbage. The agent completely forgets what it was originally asked to do and just outputs a random summary.

**Question:** How do you manage context bloat in long-horizon agentic tasks?

**Answer:**

**The Fix (Memory Compaction / Summarization Nodes):**
Never keep raw tool outputs in the main conversational context window indefinitely.
1. **The Summarizer Tool:** When the agent calls `search_web`, do NOT return the raw HTML to the main agent's scratchpad. Instead, pass the HTML to a small, cheap sub-agent (like Haiku or GPT-3.5) with the prompt: *"Extract only the facts relevant to the user's main query."*
2. **State Compaction:** In LangGraph, if the `messages` list exceeds 10,000 tokens, trigger a background Node that reads the oldest 20 messages, summarizes them into a single dense paragraph, deletes the 20 messages, and prepends the summary to the state.

---

## Scenario 7: State Machine Checkpoint Corruption

**The Scenario:**
You are using LangGraph's `SqliteSaver` to checkpoint agent states. An agent pauses at a Human-in-the-Loop breakpoint. The human developer uses `app.update_state()` to inject a manual fix, but accidentally provides a dict with missing required keys. When the graph resumes, it crashes with a `KeyError`, and that specific chat thread is permanently corrupted.

**Question:** How do you safely handle state modification in production graphs?

**Answer:**

**The Fix:**
1. **Strict TypedDicts with Defaults:** Ensure the global AgentState uses `typing.TypedDict` or Pydantic schemas with fallback defaults, so partial updates don't destroy the schema.
2. **Time Travel (Forking):** A massive benefit of LangGraph is that checkpoints are immutable. If a thread gets corrupted at Step 5, you don't lose the thread. You can query the database for the checkpoint at Step 4, assign it a new `thread_id` (effectively forking the timeline), and resume execution safely from the known-good state.

---

## Scenario 8: API Rate Limiting & Exponential Backoff

**The Scenario:**
Your agent is parsing a CSV of 1,000 URLs and calling the `fetch_url` tool for each one. The target server throws a `429 Too Many Requests` error. The agent receives the error, instantly retries, gets blocked again, and eventually crashes the graph.

**Question:** How should agents interact with third-party APIs?

**Answer:**

**The Fix:**
Agents operate much faster than humans, meaning they trigger rate limits instantly.
1. **Hide the Complexity from the LLM:** Do not rely on the LLM to say "Oh, I got a 429, I will wait 5 seconds." The LLM doesn't have a clock. 
2. **Tool-Level Backoff:** Implement `tenacity` (a Python retry library) directly inside the python function for the tool.
   ```python
   @retry(wait=wait_exponential(multiplier=1, min=2, max=10), stop=stop_after_attempt(5))
   def fetch_url(url: str):
       # HTTP request logic
   ```
3. By handling the backoff at the python level, the LLM's execution is simply paused transparently. If it fails 5 times, *then* you return a clean error string to the LLM to let it know the server is permanently down.

---

*End of Agentic AI Scenarios — Deep dives into Infinite Loops, Presidio PII Redaction, Self-Healing Code execution, Tool Hallucination, and Multi-Agent Deadlocks.*
