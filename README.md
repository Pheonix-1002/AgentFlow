# AgentFlow

**AgentFlow** is a multi-utility AI chatbot built with **LangGraph** and **Streamlit**, designed around modular, stateful workflows. It combines retrieval-augmented generation, dynamic tool use, and persistent conversation memory into a single agent that can reason, search, retrieve, and act on a user's behalf.

## Highlights

- **Modular, stateful agent workflows** — the agent is modeled as a LangGraph `StateGraph`, cleanly separating the reasoning step (`chat_node`) from tool execution (`ToolNode`), with conditional routing (`tools_condition`) deciding whether to call a tool or respond directly.
- **Human-in-the-Loop (HITL) capable** — the graph-based architecture supports interrupting execution for human review/approval at defined checkpoints before an action is finalized, rather than forcing every step to run autonomously end-to-end.
- **Retrieval-Augmented Generation (RAG)** — documents are chunked, embedded using an open-source **Hugging Face** sentence-transformer model, and indexed in a **FAISS** vector store per conversation thread, enabling fast similarity search and grounded, context-aware answers over uploaded knowledge sources.
- **Dynamic MCP tool integration** — the agent works with both pre-built and custom **MCP (Model Context Protocol)** tools, letting it discover, select, and invoke the right tool at runtime based on inferred user intent rather than hardcoded logic.
- **Persistent conversation state** — every thread's state is checkpointed to **SQLite** via `SqliteSaver`, so conversations can be paused, resumed, and revisited across sessions.
- **Observability with LangSmith** — LLM calls, tool invocations, and full run traces are integrated with **LangSmith** for tracing, debugging, and evaluation of agent behavior over time.

## Architecture

```
┌─────────────────────┐        ┌──────────────────────────┐
│  Streamlit Frontend  │ <----> │   LangGraph Agent Core    │
│  (chat UI, sidebar,  │        │   (StateGraph, HITL       │
│   document upload)   │        │    checkpoints, tool      │
└─────────────────────┘        │    routing)               │
                                └──────────────┬────────────┘
                                                │
                ┌───────────────┬──────────────┼───────────────┬───────────────┐
                ▼               ▼              ▼               ▼               ▼
          FAISS Vector    Pre-built &    Open-Source HF     SQLite         LangSmith
          Store (RAG)     Custom MCP     LLM + Embedding    Checkpointer   Tracing &
                           Tools         Models              (persisted     Evaluation
                                                            state)
```

The core graph flows as:

```
START → chat_node → (tools_condition) → tools → chat_node → ... → END
                                     └──(HITL checkpoint, optional)──┘
```

1. **`chat_node`** — invokes the LLM (bound to all available tools) with the current thread's message history and system context.
2. **`tools_condition`** — routes to the `tools` node when the LLM requests a tool call, or ends the turn when it has a final answer.
3. **`tools`** — executes the selected MCP/pre-built tool (retrieval, search, computation, etc.) and returns results to `chat_node` for the next reasoning step.
4. **HITL checkpoints** — the graph can pause at designated nodes so a human can review, approve, or modify agent actions before they proceed, using LangGraph's interrupt/resume mechanics.

## Tech Stack

| Layer | Technology |
|---|---|
| UI | Streamlit |
| Agent orchestration | LangGraph (`StateGraph`, HITL interrupts, conditional routing) |
| Tool integration | Model Context Protocol (MCP) — pre-built and custom tools |
| LLM | Open-source Hugging Face model (e.g. `meta-llama/Llama-3.1-8B-Instruct`, served via `langchain-huggingface` / HF Inference) |
| Embeddings | Hugging Face sentence-transformer model (e.g. `sentence-transformers/all-MiniLM-L6-v2`) via `langchain-huggingface` |
| Retrieval | FAISS vector similarity search |
| Persistence | SQLite checkpointing (`langgraph-checkpoint-sqlite`) |
| Observability | LangSmith (tracing, debugging, evaluation) |

## Project Structure

```
.
├── langraph_backend.py      # LangGraph agent: state, nodes, tools, checkpointer
├── streamlit_frontend.py    # Streamlit chat UI, sidebar, document upload
├── requirements.txt         # Python dependencies
└── chatbot.db                # SQLite checkpoint DB (created at runtime)
```

## Getting Started

### Prerequisites

- Python 3.10+
- A Hugging Face account/token (required for gated models, and recommended for higher rate limits on HF Inference API); not required at all if running models fully locally
- A LangSmith API key (for tracing/observability)
- API keys for any additional tools you enable (e.g. a stock/finance API)

### Installation

```bash
git clone https://github.com/Pheonix-1002/AgentFlow
cd AgentFlow
python -m venv venv
source venv/Scripts/activate   # on Linux / macOS.: venv\bin\activate
pip install -r requirements.txt
```

### Environment Variables

Create a `.env` file in the project root:

```env
HUGGINGFACEHUB_API_TOKEN=your-huggingface-token   # required for gated models / HF Inference API
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your-langsmith-api-key
LANGCHAIN_PROJECT=AgentFlow
```

### Running the App

```bash
streamlit run streamlit_frontend.py
```

Then open the local URL Streamlit prints (typically `http://localhost:8501`).

## Usage

1. **Chat naturally** — AgentFlow decides on its own whether to answer directly or invoke a tool based on your intent.
2. **Upload a document** — attach a file to the current thread; it's chunked and embedded into FAISS so the agent can ground its answers in your content.
3. **Let the agent use tools** — pre-built and custom MCP tools are discovered and invoked automatically as needed.
4. **Review sensitive actions (HITL)** — where configured, the agent pauses for your approval before completing an action.
5. **Resume any thread** — pick up a past conversation from the sidebar; full state is restored from the SQLite checkpoint.
6. **Trace and debug runs** — inspect any conversation or tool call in LangSmith for full observability into the agent's reasoning.

## License

Pheonix-1002
