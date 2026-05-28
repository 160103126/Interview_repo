# LangChain — Coding Patterns

> Practical implementation of the LangChain framework using LCEL (LangChain Expression Language), RAG pipelines, conversational memory architectures, Agents, and Output Parsers.

---

## Table of Contents

- [Q1: Basic LCEL Chain](#q1-basic-lcel-chain)
- [Q2: Conversational Retrieval Chain (RAG with Memory)](#q2-conversational-retrieval-chain-rag-with-memory)
- [Q3: Structured Pydantic Output Extraction](#q3-structured-pydantic-output-extraction)
- [Q4: Self-Healing Pydantic Extraction (Retries)](#q4-self-healing-pydantic-extraction-retries)
- [Q5: Routing Chains dynamically](#q5-routing-chains-dynamically)
- [Q6: Custom Tools for Agents](#q6-custom-tools-for-agents)

---

### Q1: Basic LCEL Chain

**Problem:** Build a chain that takes a topic, formats a prompt, queries an OpenAI model, and parses the output into a string.

```python
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

# 1. Define the components
prompt = PromptTemplate.from_template("Write a haiku about {topic}")
model = ChatOpenAI(model="gpt-3.5-turbo")
output_parser = StrOutputParser()

# 2. Build the LCEL Chain (Pipe syntax)
chain = prompt | model | output_parser

# 3. Invoke
result = chain.invoke({"topic": "artificial intelligence"})
print(result)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **LCEL (LangChain Expression Language):** The `|` (pipe) operator overloads standard Python syntax to feed the output of the left object directly into the input of the right object, much like Unix shell pipes.
- **Why LCEL?** It provides automatic support for asynchronous streaming (`chain.astream`), parallel batching (`chain.batch`), and built-in tracing (LangSmith) without changing a single line of your logic.

#### ⏱️ Complexity
- **Time Complexity:** Dictated by LLM API latency ($O(\text{Output Tokens})$).

---

### Q2: Conversational Retrieval Chain (RAG with Memory)

**Problem:** Build a RAG pipeline that remembers chat history and uses it to retrieve relevant documents.

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_community.chat_message_histories import ChatMessageHistory
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain.chains import create_history_aware_retriever, create_retrieval_chain
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_texts(["The company policy allows 20 days of PTO."], embeddings)
retriever = vectorstore.as_retriever()
llm = ChatOpenAI()

# 1. Contextualize the Question
contextualize_q_prompt = ChatPromptTemplate.from_messages([
    ("system", "Given a chat history and the latest user question, formulate a standalone question. Do NOT answer it."),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])
history_aware_retriever = create_history_aware_retriever(llm, retriever, contextualize_q_prompt)

# 2. Answer the Question
qa_prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer the user's question using the following context:\n\n{context}"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])
question_answer_chain = create_stuff_documents_chain(llm, qa_prompt)

# 3. Combine into final RAG Chain
rag_chain = create_retrieval_chain(history_aware_retriever, question_answer_chain)

# 4. State Management
store = {}
def get_session_history(session_id: str):
    if session_id not in store: store[session_id] = ChatMessageHistory()
    return store[session_id]

conversational_rag_chain = RunnableWithMessageHistory(
    rag_chain, get_session_history,
    input_messages_key="input", history_messages_key="chat_history", output_messages_key="answer",
)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The Conversational RAG Problem:** If the user asks "How much PTO do I get?", the DB retrieves chunks about PTO. If the user then says, "Can I roll *it* over?", embedding that string fails because "it" has no semantic meaning.
- **The Fix:** The `history_aware_retriever` uses a fast LLM to rewrite "Can I roll it over?" into "Can I roll over PTO?".
- **`RunnableWithMessageHistory`:** This wrapper automatically injects and extracts chat history into your dictionary `store`, removing the need for manual state loops.

#### ⏱️ Complexity
- **Time Complexity:** $O(N)$ network requests (2 LLM calls per turn + 1 DB lookup).

---

### Q3: Structured Pydantic Output Extraction

**Problem:** Extract specific entities from text into a strict Pydantic JSON schema.

```python
from langchain_core.pydantic_v1 import BaseModel, Field
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain.output_parsers import PydanticOutputParser

class PatientProfile(BaseModel):
    name: str = Field(description="The full name of the patient")
    age: int = Field(description="The age of the patient in years")
    symptoms: list[str] = Field(description="A list of symptoms described")

parser = PydanticOutputParser(pydantic_object=PatientProfile)

prompt = PromptTemplate(
    template="Extract patient data from this note.\n{format_instructions}\nNote: {note}\n",
    input_variables=["note"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)

chain = prompt | ChatOpenAI(temperature=0) | parser
result = chain.invoke({"note": "John Doe is a 42 year old male complaining of headaches."})
print(result.age) # 42
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Pydantic Alignment:** `parser.get_format_instructions()` translates the Python schema into explicit textual instructions for the LLM. The parser then intercepts the string output and strictly validates it.

---

### Q4: Self-Healing Pydantic Extraction (Retries)

**Problem:** If the LLM hallucinates a string `"forty two"` for the integer `age`, Pydantic crashes. Make the chain self-heal.

```python
from langchain_core.runnables import RunnableRetry

# Assuming `chain` from Q3 is defined:
retry_chain = chain.with_retry(
    retry_if_exception_type=(Exception,),
    wait_exponential_jitter=True,
    stop_after_attempt=3
)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Self-Healing Pipelines:** Instead of crashing the entire script when JSON parsing fails, `.with_retry` intercepts the `ValidationError`, applies exponential backoff, and re-invokes the chain, improving extraction reliability in production.

---

### Q5: Routing Chains dynamically

**Problem:** Route a user's question to a Math Chain or a Physics Chain based on the query.

```python
from langchain_core.runnables import RunnableBranch
from langchain_openai import ChatOpenAI

llm = ChatOpenAI()

branch = RunnableBranch(
    (lambda x: "math" in x["topic"].lower(), llm.bind(stop=["\n"])),
    (lambda x: "physics" in x["topic"].lower(), llm.bind(temperature=0.9)),
    llm # Default branch
)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **`RunnableBranch`:** Evaluates a list of (condition, runnable) tuples. It executes the runnable corresponding to the first true condition, providing an elegant `if/else` control flow natively within LCEL.

---

### Q6: Custom Tools for Agents

**Problem:** Create a Python function and bind it to an Agent as a callable tool.

```python
from langchain_core.tools import tool

@tool
def calculate_weather(location: str, unit: str = "celsius") -> str:
    """Gets the weather for a given location. Always provide the location."""
    if "san francisco" in location.lower():
        return f"It's 15 degrees {unit}."
    return f"It's 20 degrees {unit}."

# The docstring and type hints become the prompt for the LLM!
print(calculate_weather.name)
print(calculate_weather.description)
print(calculate_weather.args_schema.schema())
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The `@tool` Decorator:** Automatically inspects your Python function. It uses the `__doc__` string as the instructions for the LLM on *when* to use it, and uses the `type hints` to construct a Pydantic schema enforcing exactly *how* the LLM should format the arguments.

---

*End of LangChain Coding — All 6 fundamental patterns restored.*
