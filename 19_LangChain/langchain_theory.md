# LangChain — Theory & Concepts

> Orchestrating LLMs, RAG pipelines, LCEL (LangChain Expression Language), Memory, and Tool calling.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is LangChain? Why use it instead of just calling the OpenAI API?

**Answer:**

LangChain is a framework for developing applications powered by LLMs.

If you just want to generate text, the OpenAI API is enough. But building a real-world app requires orchestration. LangChain provides abstractions for:
1. **Model I/O:** Swapping out OpenAI for Anthropic or an open-source local model with just one line of code change.
2. **Retrieval (RAG):** Built-in document loaders (PDFs, Web), text splitters, and vector store integrations (Pinecone, Chroma).
3. **Memory:** Storing chat history.
4. **Agents & Tools:** Allowing the LLM to browse the web or run SQL.

---

### Q2: What is a PromptTemplate?

**Answer:**

A class that handles injecting dynamic variables into a static prompt string.

```python
from langchain.prompts import PromptTemplate

template = "Translate the following English text to {language}: {text}"
prompt = PromptTemplate(
    input_variables=["language", "text"],
    template=template,
)

# Returns: "Translate the following English text to French: Hello"
print(prompt.format(language="French", text="Hello"))
```

*Note: In modern LangChain, we often use `ChatPromptTemplate` which natively supports `SystemMessage`, `HumanMessage`, and `AIMessage` structures required by Chat Models.*

---

## 🟡 Medium (Intermediate)

### Q3: What is LCEL (LangChain Expression Language)?

**Answer:**

LCEL is a declarative way to easily compose chains together using the pipe `|` operator (similar to Unix pipes). It replaces the old, bulky `LLMChain` classes.

**Benefits:**
It natively supports streaming, asynchronous execution, and parallel execution out of the box.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("Tell me a joke about {topic}")
model = ChatOpenAI()
parser = StrOutputParser()

# LCEL Pipe: Prompt -> Model -> String Parser
chain = prompt | model | parser

# Execution
result = chain.invoke({"topic": "bears"})
print(result) # "Why did the bear..."
```

---

### Q4: How does LangChain handle Memory in a chat application?

**Answer:**

LLMs are stateless. To have a conversation, you must pass the entire chat history back to the LLM on every single request.

LangChain provides Memory classes to manage this context window:
1. **ConversationBufferMemory:** Stores the entire raw conversation. (Danger: quickly exhausts the token limit and costs a lot of money).
2. **ConversationBufferWindowMemory:** Keeps only the last `K` interactions (e.g., last 5 messages).
3. **ConversationSummaryMemory:** Uses an LLM in the background to constantly summarize the old conversation, keeping the context window small while preserving the core facts.

---

### Q5: Explain Document Loaders and Text Splitters in LangChain.

**Answer:**

These are the first two steps of any RAG pipeline.

- **Document Loaders:** Connectors that fetch data from sources (e.g., `PyPDFLoader`, `WebBaseLoader`, `NotionDirectoryLoader`) and convert them into LangChain `Document` objects (which contain `page_content` and `metadata`).
- **Text Splitters:** LLMs have token limits. You can't feed a 500-page PDF into an embedding model.
  - `RecursiveCharacterTextSplitter`: The standard choice. It tries to split on paragraphs `\n\n`, then sentences `\n`, then words, ensuring that semantically related text stays together in the same chunk.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q6: How does Tool Calling (Function Calling) work under the hood in LangChain?

**Answer:**

Tool calling allows an LLM to interact with external systems (like a weather API or a SQL database).

1. **Definition:** You define a Python function (the Tool) and provide a strict Pydantic schema or docstring describing its inputs.
2. **Binding:** LangChain converts this schema into JSON and sends it to the LLM API along with the prompt (e.g., `model.bind_tools([get_weather])`).
3. **LLM Decision:** The LLM reads the user prompt ("What's the weather in Tokyo?"). It realizes it can't answer this, but sees the `get_weather` tool. It outputs a special JSON payload: `{"name": "get_weather", "arguments": {"location": "Tokyo"}}`.
4. **Execution:** LangChain intercepts this JSON, executes your actual Python `get_weather("Tokyo")` function, and passes the result back to the LLM.
5. **Final Answer:** The LLM reads the result ("25C, Sunny") and generates a natural language response for the user.

---

### Q7: Explain the difference between standard RetrievalQA and a Conversational Retrieval Chain.

**Answer:**

- **RetrievalQA (Simple RAG):** Takes a single user query, embeds it, searches the Vector DB, and passes the chunks + query to the LLM. 
  - *Flaw:* If the user asks a follow-up question ("Tell me more about *that*"), the embedding model embeds the word "that". The Vector DB finds no documents matching "that", and the system fails.
  
- **Conversational Retrieval Chain (Chat RAG):** Solves the follow-up problem using a Two-Step LLM process.
  1. **Standalone Question Generator:** Takes the Chat History + the vague new query ("Tell me more about that") and uses a cheap LLM to rewrite it into a standalone query ("Tell me more about the company's remote work policy").
  2. **Retrieval:** Embeds this *rewritten* standalone query to search the Vector DB.
  3. **Generation:** Passes the chunks + rewritten query to the main LLM to generate the answer.

---

### Q8: What are the main criticisms of LangChain? Why do some engineers prefer not to use it?

**Answer:**

In senior/staff interviews, you must know the drawbacks of frameworks.

1. **Over-Abstraction:** LangChain wraps simple API calls in layers of complex, undocumented classes. When something breaks, debugging the heavily nested OOP architecture is notoriously difficult.
2. **Rapid Breaking Changes:** The framework evolved so fast that code written 6 months ago often breaks due to deprecated modules (e.g., the massive shift from legacy Chains to LCEL).
3. **Lock-in:** It forces you to write code "the LangChain way," making it hard to integrate highly custom logic or optimize specific bottlenecks.

*Industry Trend:* Many MAANG teams use LangChain for prototyping, but write custom orchestration code (using just the raw OpenAI/Anthropic SDKs) for production systems to maintain control and debuggability.

---

*End of LangChain Theory — 8 questions covering LCEL, Memory, RAG pipelines, Tool Calling, and architectural criticisms.*
