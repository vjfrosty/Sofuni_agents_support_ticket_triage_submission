# Support Ticket Triage Agent — Final Blueprint v3.4 (Memory-Complete, Build-Ready)

## Changelog: v3.1 → v3.4

| Version | Change |
|---|---|
| v3.1 | Base blueprint — LangGraph workflow, RAG, SQLite checkpointer, HITL, 6 tests |
| v3.3 | Memory architecture added — short-term (thread), long-term (SQLite namespaces), read-before-act, write-back |
| v3.4 | Full merge — v3.1 + v3.3 integrated into single build-ready document; memory woven into state, nodes, build order, and all checklists |

---

## 1. Purpose

This document is the **full submission-ready technical blueprint** for a Google Colab project that implements a **LangGraph multi-agent support ticket triage workflow** using **LangChain**, **OpenAI**, **runtime-built RAG**, **SQLite persistence**, **complete short-term and long-term memory**, and **Human-in-the-Loop (HITL)** review.

It is designed to be:

- detailed enough to guide direct implementation,
- aligned with the assignment constraints,
- conservative for Colab execution,
- explicit about grounding, guardrails, and reliability,
- complete on memory architecture (short-term, long-term, read-before-act, write-back),
- structured for vibe-coding in small self-solvable steps.

---

## 2. Assignment Fit

This blueprint is designed to satisfy the project requirements:

- **Python in Google Colab**
- **LangGraph** stateful graph
- **At least two distinct agents**
- **At least two tools**
- **Conversation memory / persistence**
- **Human-in-the-loop interruption**
- Core function: **`execute_workflow(user_request)`**
- **At least 5 test cases** — this blueprint includes **6**
- **No hardcoded API keys**

### Compliance mapping

| Requirement | How this blueprint satisfies it |
|---|---|
| LangGraph stateful workflow | Explicit `StateGraph(TicketState)` |
| Multiple agents | Triage, Retrieval, Resolution, Drafting, Review |
| Tools | Retriever, account lookup, route policy lookup, risk/priority tools |
| Memory (short-term) | `messages` field in `TicketState` with append semantics + `trim_messages` |
| Memory (long-term) | SQLite `memory_store` table with namespaces; read-before-act; write-back in finalize |
| Memory (RAG) | Runtime FAISS vector store from KB articles |
| HITL | Interrupt before finalization |
| Core function | `execute_workflow(user_request)` |
| Tests | 6 structured scenarios |
| Secret safety | `OPENAI_API_KEY` loaded from Colab Secret or env |

---

## 3. Problem Definition

### Business problem

Support teams waste time doing repetitive first-line work:

- classifying the issue,
- checking internal knowledge,
- checking account/ticket context,
- deciding urgency and routing,
- writing an initial reply.

### Project goal

Build a **multi-agent workflow** that takes a raw customer support ticket and produces:

- issue classification,
- urgency/priority,
- escalation risk,
- internal route/team,
- grounding evidence from the KB,
- internal notes,
- customer reply draft,
- a human-reviewed final response.

The system also **remembers** user interaction patterns across sessions using long-term memory, adapting tone and routing based on prior history.

### Why this is a strong course-aligned scenario

It naturally demonstrates:

- specialized roles,
- tool use,
- retrieval grounding,
- explicit state passing,
- short-term message memory with trimming,
- long-term user memory with namespaced SQLite,
- branching,
- HITL review,
- structured outputs,
- safe agent behavior.

---

## 4. Final Architectural Position

This project uses the following fixed design choices:

### Mandatory core stack

- Python
- Google Colab / Jupyter Notebook
- LangChain
- LangGraph
- OpenAI as LLM provider
- local JSON data file
- runtime chunking + embeddings + local FAISS vector store
- SQLite — dual purpose:
  - **checkpointer** for LangGraph thread state
  - **memory_store** table for long-term user memory

### Memory architecture summary

| Layer | Mechanism | Scope |
|---|---|---|
| Short-term | `messages` in `TicketState` + `trim_messages` | Per thread |
| Long-term | SQLite `memory_store` with namespaces | Per user / global |
| RAG | FAISS vector store from KB JSON | Static KB |

### Explicit exclusions

- no LangSmith in the baseline submission
- no Google Drive dependency in the baseline
- no external CRM dependency
- no real email sending
- no hosted vector DB

### Submission model

The ZIP should contain:

- `support_ticket_triage.ipynb`
- `project_data.json`
- optionally `README.md`

---

## 5. Multi-Agent Pattern Coverage

### 5.1 Introduction to LangGraph & Multi-Agent Patterns

Covered by:

- graph-based orchestration,
- specialized nodes,
- explicit state,
- conditional routing,
- interrupt/resume behavior.

### 5.2 Agent Nodes, Tools & Routing: Planners, Workers and Evaluators

Covered by:

- **Triage Agent** → classifier / pre-processor
- **Retrieval Agent** → knowledge worker (RAG)
- **Resolution Agent** → planner / decision maker
- **Drafting Agent** → response generator
- **Review Node / Human Reviewer** → evaluator

### 5.3 Coordination & State: Message Passing, Shared Memory and Parallel Branches

Covered by:

- shared `TicketState` with `messages` append field,
- node-to-node state updates,
- thread-based persistence via SQLite checkpointer,
- long-term memory via `memory_store` table,
- optional parallel retrieval branch design.

### 5.4 Reliability & Safety: Retries, Timeouts, Conflict Resolution and Guardrails

Covered by:

- retry wrappers,
- fallback behavior,
- tool-over-model conflict precedence,
- explicit grounding rules,
- HITL checkpoint,
- safe finalization rules.

---

## 6. High-Level Workflow

### Grounded execution flow

1. User submits a raw ticket
2. Long-term memory is loaded for the user (preferences, escalation history)
3. Triage Agent converts ticket into structured metadata; messages appended to thread
4. Retrieval Agent gathers relevant KB evidence and account context
5. Resolution Agent determines route and recommended action (informed by long-term memory)
6. Drafting Agent writes a customer-safe reply grounded in state
7. Review Node interrupts execution for human approval
8. Workflow resumes with: approve / revise / manual escalation
9. Finalize Node writes back updated facts to long-term memory
10. Final state is returned

### Visual flow

```text
START
  ↓
[Load Long-Term Memory]
  ↓
Triage
  ├─→ Clarify (if too incomplete) → END
  └─→ Retrieve
           ↓
        Resolve  ← (informed by long-term memory)
           ↓
         Draft
           ↓
         Review (interrupt)
           ├─→ Finalize → [Write-Back Memory] → END
           ├─→ Revise → Review
           └─→ Manual Escalation → END
```

---

## 7. Data Strategy

### 7.1 Submission-safe data design

Use **one file**: `project_data.json`

This keeps the submission simple, reproducible, Colab-safe, and easy to inspect.

### Recommended top-level structure

```json
{
  "kb_articles": [],
  "account_context": {},
  "routing_rules": [],
  "test_cases": []
}
```

### 7.2 Why JSON over YAML

JSON is preferred because:

- native Python support,
- no extra parser dependency,
- easier schema validation,
- easier transformation into documents and dicts.

---

## 8. Example `project_data.json` Schema

### 8.1 KB articles

```json
{
  "id": "kb_billing_duplicate_charge",
  "title": "Duplicate Billing Charge Policy",
  "category": "billing",
  "tags": ["billing", "refund", "duplicate charge"],
  "text": "If a customer reports a duplicate billing event, apologize, verify the billing period, check if a refund is already pending, and route to billing support when account-specific action is required."
}
```

### 8.2 Account context

```json
{
  "acct_1001": {
    "open_ticket": true,
    "last_ticket_summary": "Duplicate billing complaint from last week.",
    "refund_pending": true,
    "known_outage": false,
    "service_tier": "business_plus"
  }
}
```

### 8.3 Routing rules

```json
{
  "trigger_category": "billing",
  "priority": "high",
  "route_to_team": "billing_support",
  "recommended_action": "review_payment_activity"
}
```

### 8.4 Test cases

```json
{
  "id": "test_01",
  "input": "I was charged twice this month and nobody replied to my email.",
  "expected_category": "billing",
  "expected_priority": "high"
}
```

---

## 9. Runtime RAG Pipeline

This project uses **Option B** explicitly:

> Store KB articles in JSON, then at runtime: create chunks, generate embeddings, build a local vector store.

### Runtime pipeline

1. Load `project_data.json`
2. Extract `kb_articles`
3. Convert articles to LangChain `Document` objects
4. Split documents into chunks
5. Generate embeddings with OpenAI
6. Build a local FAISS vector store
7. Create a retriever
8. Use the retriever inside the Retrieval Agent / retrieval tool

### Operational rules

- keep the KB small,
- chunk size moderate (500 tokens, 100 overlap),
- build the vector store at notebook startup,
- optionally persist the vector index under `/content/artifacts/`.

---

## 10. Package Plan

Keep dependencies narrow and pinned.

```python
%pip install -q -U \
  "langchain==0.3.*" \
  "langgraph==0.2.*" \
  "langchain-openai==0.2.*" \
  "langchain-community==0.3.*" \
  "faiss-cpu==1.8.*" \
  "pydantic==2.*" \
  "tiktoken==0.7.*"
```

> Note: exact versions may need a final freeze after real Colab testing. The important rule is: **pin versions**.

---

## 11. Colab Setup

### 11.1 Secret loading

Preferred pattern:

```python
import os

try:
    from google.colab import userdata
    if not os.environ.get("OPENAI_API_KEY"):
        os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
except Exception:
    pass

if not os.environ.get("OPENAI_API_KEY"):
    raise ValueError("OPENAI_API_KEY is required. Add it in Colab Secrets or environment variables.")
```

---

## 12. Imports Skeleton

```python
import os
import json
import uuid
import time
import sqlite3
from typing import TypedDict, List, Dict, Optional, Literal, Any

from pydantic import BaseModel, Field
from langchain_core.messages import SystemMessage, HumanMessage, trim_messages
from langchain_core.documents import Document
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import FAISS
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command

# Depending on installed version, this import may vary slightly:
from langgraph.checkpoint.sqlite import SqliteSaver
```

> **v3.4 addition:** `sqlite3` is now imported for the long-term `memory_store` table, separate from the LangGraph checkpointer connection.

---

## 13. Data Loading and Preprocessing

### 13.1 Load JSON

```python
def load_project_data(path: str = "project_data.json") -> dict:
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)

PROJECT_DATA = load_project_data()
```

### 13.2 Convert KB to documents

```python
def kb_to_documents(kb_articles: List[dict]) -> List[Document]:
    docs = []
    for article in kb_articles:
        docs.append(
            Document(
                page_content=article["text"],
                metadata={
                    "id": article["id"],
                    "title": article["title"],
                    "category": article.get("category", ""),
                    "tags": article.get("tags", []),
                },
            )
        )
    return docs
```

### 13.3 Chunk documents

```python
def build_chunks(documents: List[Document]) -> List[Document]:
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,
        chunk_overlap=100,
    )
    return splitter.split_documents(documents)
```

### 13.4 Build vector store and retriever

```python
def build_retriever(kb_articles: List[dict]):
    docs = kb_to_documents(kb_articles)
    chunks = build_chunks(docs)
    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
    vectorstore = FAISS.from_documents(chunks, embeddings)
    retriever = vectorstore.as_retriever(search_kwargs={"k": 4})
    return retriever, vectorstore, chunks

RETRIEVER, VECTORSTORE, KB_CHUNKS = build_retriever(PROJECT_DATA["kb_articles"])
```

> If embeddings fail, the notebook should fail early with a clear message. Do not silently degrade.

---

## 14. Account Context and Routing Data

```python
ACCOUNT_CONTEXT = PROJECT_DATA.get("account_context", {})
ROUTING_RULES   = PROJECT_DATA.get("routing_rules", [])
TEST_CASES      = PROJECT_DATA.get("test_cases", [])
```

---

## 15. Complete Memory Architecture (v3.4)

This section covers all three memory layers. They are **distinct and non-overlapping**.

| Layer | Storage | Scope | Purpose |
|---|---|---|---|
| Short-term | `messages` in `TicketState` | Per thread | Active conversation continuity |
| Long-term | SQLite `memory_store` table | Per user / global | Persistent preferences, escalation history |
| RAG | FAISS vector store | Static | KB grounding |

---

### 15.1 Short-Term Memory — Thread Messages

Short-term memory lives in the `messages` field of `TicketState`.

**Append semantics — every node appends, never overwrites.**

```python
class TicketState(TypedDict, total=False):
    messages: List[dict]  # [{"role": "user" | "assistant", "content": "..."}]
    # ... all other fields below
```

#### Memory Limit Control (MANDATORY)

Before **every LLM call**, trim the message list to prevent context window overflow:

```python
from langchain_core.messages import trim_messages

def prepare_messages(messages: List[dict]) -> List[dict]:
    return trim_messages(
        messages,
        max_tokens=1500,
        strategy="last",
    )
```

Usage pattern in every node that calls an LLM:

```python
msgs = prepare_messages(state.get("messages", []))
response = base_llm.invoke(msgs + [SystemMessage(content=PROMPT), HumanMessage(content=prompt)])
```

Alternatives (optional, not required for baseline):

- `ContextEditingMiddleware`
- `SummarizationMiddleware`

---

### 15.2 Long-Term Memory — SQLite `memory_store`

Long-term memory uses a dedicated table in the **same SQLite database** as the checkpointer, or a separate connection. It stores structured facts keyed by namespace.

#### Schema

```sql
CREATE TABLE IF NOT EXISTS memory_store (
    namespace TEXT NOT NULL,
    key       TEXT NOT NULL,
    value     JSON NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (namespace, key)
);
```

#### Namespace examples

| Namespace | Purpose |
|---|---|
| `user:C001:preferences` | Tone preference, contact style |
| `user:C001:history` | Escalation flag, prior category |
| `global:routing_rules` | Shared routing overrides |

#### Setup — create the table at notebook startup

```python
MEMORY_DB_PATH = "memory_store.sqlite"

def init_memory_db(path: str = MEMORY_DB_PATH):
    conn = sqlite3.connect(path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS memory_store (
            namespace  TEXT NOT NULL,
            key        TEXT NOT NULL,
            value      JSON NOT NULL,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (namespace, key)
        )
    """)
    conn.commit()
    return conn

MEMORY_CONN = init_memory_db()
```

---

### 15.3 Read-Before-Act — Node Injection Pattern

Every important node **must read long-term memory before invoking the LLM**.
This makes the system aware of prior user history and preferences without relying on re-sending raw transcript.

```python
def load_memory(conn: sqlite3.Connection, namespace: str) -> dict:
    cursor = conn.execute(
        "SELECT key, value FROM memory_store WHERE namespace = ?",
        (namespace,)
    )
    return {row[0]: json.loads(row[1]) for row in cursor.fetchall()}
```

#### Example — resolution node reads user preferences

```python
def resolution_node(state: TicketState):
    user_id = state.get("extracted_entities", {}).get("account_id", "anonymous")
    user_prefs = load_memory(MEMORY_CONN, f"user:{user_id}:preferences")
    user_history = load_memory(MEMORY_CONN, f"user:{user_id}:history")

    memory_context = f"""
User preferences: {user_prefs}
User history: {user_history}
""".strip()

    prompt = f"""
{memory_context}

User request: {state['user_request']}
Category: {state.get('category')}
...
""".strip()

    msgs = prepare_messages(state.get("messages", []))
    # ... rest of node logic
```

---

### 15.4 Write-Back — Profile Extraction After Finalize

After the finalize node confirms a completed interaction, extract structured facts and persist them to long-term memory.

```python
def save_memory(conn: sqlite3.Connection, namespace: str, key: str, value: Any):
    conn.execute(
        """INSERT OR REPLACE INTO memory_store (namespace, key, value, updated_at)
           VALUES (?, ?, ?, CURRENT_TIMESTAMP)""",
        (namespace, key, json.dumps(value))
    )
    conn.commit()
```

#### Write-back in finalize node

```python
def finalize_node(state: TicketState):
    user_id = state.get("extracted_entities", {}).get("account_id", "anonymous")

    # Persist tone preference based on sentiment
    if state.get("sentiment") in ("angry", "frustrated"):
        save_memory(MEMORY_CONN, f"user:{user_id}:preferences", "tone", "empathetic")

    # Persist escalation flag for future routing decisions
    if state.get("requires_escalation"):
        save_memory(MEMORY_CONN, f"user:{user_id}:history", "escalation_flag", True)

    # Persist last seen category for context continuity
    if state.get("category"):
        save_memory(MEMORY_CONN, f"user:{user_id}:history", "last_category", state["category"])

    return {
        "final_reply": state.get("reply_draft"),
        "human_approved": True,
        "status": "approved",
    }
```

---

### 15.5 Clear Separation of Memory Layers

```
Short-term  →  messages field in TicketState  →  trimmed per LLM call
Long-term   →  SQLite memory_store table      →  namespaced, persists across sessions
RAG         →  FAISS vector store             →  KB articles only, static per run
```

Do not mix these layers. Never store raw KB chunks in `memory_store`. Never store long-term facts in `messages`.

---

## 16. State Design

### 16.1 State principles

- store only serializable values,
- never store model objects in state,
- never store tool/retriever instances in state,
- state is the single source of truth,
- nodes read state and return partial updates,
- `messages` uses append semantics — never overwrite.

### 16.2 TicketState (v3.4 — memory-complete)

```python
class TicketState(TypedDict, total=False):
    # --- identity / threading ---
    thread_id: str
    user_request: str

    # --- SHORT-TERM MEMORY ---
    # Append only. Trimmed before every LLM call via prepare_messages().
    messages: List[dict]  # [{"role": "user"|"assistant", "content": "..."}]

    # --- triage ---
    category: str
    subcategory: str
    priority: str
    sentiment: str
    requires_escalation: bool
    missing_info: List[str]
    extracted_entities: Dict[str, str]  # includes "account_id" for long-term memory lookup

    # --- evidence / retrieval ---
    kb_results: List[Dict[str, Any]]
    kb_summary: str
    account_context: Dict[str, Any]

    # --- planning ---
    route_to_team: str
    recommended_action: str
    internal_notes: str
    should_ask_for_clarification: bool

    # --- drafting ---
    reply_draft: str
    final_reply: str

    # --- review / HITL ---
    review_action: Optional[str]
    human_feedback: Optional[str]
    human_approved: bool

    # --- reliability / meta ---
    status: str
    errors: List[str]
```

> **Note:** Long-term memory (`memory_store`) is **not** stored in `TicketState`. It is read and written via `MEMORY_CONN` at the start and end of key nodes. State stays clean and serializable.

---

## 17. Structured Schemas

### 17.1 Triage schema

```python
class TriageOutput(BaseModel):
    category: str = Field(description="Primary support category")
    subcategory: str = Field(description="Specific issue type")
    priority: str = Field(description="low, medium, high, or critical")
    sentiment: str = Field(description="neutral, frustrated, or angry")
    requires_escalation: bool = Field(description="Whether the ticket needs escalation")
    missing_info: List[str] = Field(default_factory=list)
    extracted_entities: Dict[str, str] = Field(default_factory=dict)
```

### 17.2 Resolution schema

```python
class ResolutionPlan(BaseModel):
    route_to_team: str
    recommended_action: str
    internal_notes: str
    should_ask_for_clarification: bool
```

---

## 18. Model Setup

### 18.1 Base model

```python
base_llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0,
)
```

### 18.2 Structured variants

```python
triage_llm = base_llm.with_structured_output(TriageOutput, method="json_schema")
plan_llm   = base_llm.with_structured_output(ResolutionPlan, method="json_schema")
```

---

## 19. Prompt Library

### 19.1 Triage agent prompt

```python
TRIAGE_SYSTEM = """
You are a support triage specialist.
Your task is to convert a raw customer ticket into structured metadata:
- category, subcategory, priority, sentiment, escalation need, missing information, extracted entities.
Be conservative. Do not invent facts.
If the ticket is incomplete, list the missing information clearly.
""".strip()
```

### 19.2 Retrieval agent prompt

```python
RETRIEVAL_SYSTEM = """
You are a support knowledge retrieval agent.
Collect only the most relevant internal evidence needed to resolve the ticket.
Use retrieval results and account context.
Do not make policy claims that are not supported by retrieved evidence.
Return grounded evidence only.
""".strip()
```

### 19.3 Resolution planner prompt

```python
RESOLUTION_SYSTEM = """
You are a support operations planner.
Based only on the ticket, retrieved evidence, account context, and user memory:
- choose the correct route_to_team,
- choose the recommended action,
- write concise internal notes,
- decide whether clarification is required.
Prefer clarification over false certainty.
Prefer escalation over unsupported commitments.
""".strip()
```

### 19.4 Drafting prompt

```python
DRAFTING_SYSTEM = """
You are a customer support response writer.
Write a concise, empathetic, professional, policy-safe reply.
Use only the grounded evidence from the provided state.
Do not promise refunds, credits, deadlines, or technical actions
unless they are explicitly supported by the evidence.
If information is missing, ask for it clearly.
If escalation is required, communicate next steps without overpromising.
""".strip()
```

### 19.5 Revision prompt

```python
REVISION_SYSTEM = """
You are revising a customer support response based on reviewer feedback.
Preserve factual correctness.
Apply the reviewer feedback exactly where possible.
Do not introduce unsupported claims.
""".strip()
```

---

## 20. Guardrails and Grounding Rules

### 20.1 Grounding constraints

Drafting Agent must use only:

- `user_request`, `kb_results`, `account_context`,
- `route_to_team`, `recommended_action`, `internal_notes`.

If evidence is insufficient: ask for clarification, escalate, or say further review is needed.

### 20.2 No hallucination rule

If the KB does not support the answer, the system must not invent refunds, policy exceptions, outage timelines, ticket resolutions, or account-specific facts.

### 20.3 Tool-over-model precedence

If a tool or retriever result conflicts with the model's intuition: **tool/retrieval result wins**.

### 20.4 Escalation override rule

Force escalation when:

- priority = `critical`
- explicit legal/regulatory threat
- severe customer risk language
- known outage + business impact
- insufficient evidence for a confident resolution

### 20.5 Tone guardrails

- Angry/frustrated users must receive empathetic tone.
- No defensive phrasing. No blame. No unsupported reassurance.
- **v3.4 addition:** If long-term memory contains `tone: "empathetic"` for a user, apply this unconditionally regardless of detected sentiment.

### 20.6 Finalization checks

Before finalization:

- `reply_draft` must exist,
- `route_to_team` must be valid or intentionally omitted,
- `final_reply` must not contain unsupported action promises,
- if `requires_escalation == True`, the reply must acknowledge next-step handling.

---

## 21. Reliability Rules

### 21.1 Retry policy

```python
def with_retry(fn, retries=2, delay=1.0):
    for attempt in range(retries + 1):
        try:
            return fn()
        except Exception:
            if attempt == retries:
                raise
            time.sleep(delay)
```

### 21.2 Fallback behavior

| Failure point | Fallback |
|---|---|
| Retrieval fails | Raise clear error at setup time |
| Planning fails | Route to manual escalation |
| Drafting fails | Return minimal safe draft |
| Memory read fails | Proceed with empty preferences (log warning) |
| Memory write fails | Log warning; do not crash finalize |

### 21.3 Conflict resolution

1. account/tool data wins over free reasoning
2. retrieved evidence wins over unsupported claims
3. long-term memory preferences inform but do not override safety rules
4. human reviewer wins over previous draft

---

## 22. Tool Layer

### 22.1 Tool A — Search KB via retriever

```python
@tool
def search_kb(query: str) -> List[dict]:
    """Search the internal support knowledge base and return top relevant chunks."""
    docs = RETRIEVER.invoke(query)
    return [{"text": d.page_content, "metadata": d.metadata} for d in docs]
```

### 22.2 Tool B — Lookup account context

```python
@tool
def lookup_account_context(account_id: str) -> Dict[str, Any]:
    """Look up mock account/ticket context by account id."""
    return ACCOUNT_CONTEXT.get(account_id, {})
```

### 22.3 Tool C — Detect escalation risk

```python
@tool
def detect_escalation_risk(text: str) -> bool:
    """Detect whether a ticket contains strong escalation or risk signals."""
    text_l = text.lower()
    risky = ["cancel", "lawyer", "regulator", "complaint", "unacceptable",
             "still not fixed", "nobody replied", "escalate this"]
    return any(k in text_l for k in risky)
```

### 22.4 Tool D — Priority scorer

```python
@tool
def priority_score(text: str) -> str:
    """Estimate support priority from deterministic urgency signals."""
    text_l = text.lower()
    if any(k in text_l for k in ["down", "outage", "cannot work", "all users affected"]):
        return "critical"
    if any(k in text_l for k in ["charged twice", "cancel", "lawyer", "urgent"]):
        return "high"
    if any(k in text_l for k in ["not working", "issue", "refund", "login"]):
        return "medium"
    return "low"
```

### 22.5 Tool E — Route policy lookup

```python
@tool
def route_policy_lookup(category: str, priority: str) -> Dict[str, Any]:
    """Return a route policy rule for a category/priority combination."""
    for rule in ROUTING_RULES:
        if rule.get("trigger_category") == category and rule.get("priority") == priority:
            return rule
    for rule in ROUTING_RULES:
        if rule.get("trigger_category") == category:
            return rule
    return {}
```

---

## 23. Node Design (v3.4 — memory-integrated)

### 23.1 Triage node

```python
def triage_node(state: TicketState):
    msgs = prepare_messages(state.get("messages", []))

    def run():
        return triage_llm.invoke([
            SystemMessage(content=TRIAGE_SYSTEM),
            HumanMessage(content=state["user_request"]),
        ])

    result = with_retry(run)

    new_message = {"role": "user", "content": state["user_request"]}

    return {
        "messages": state.get("messages", []) + [new_message],
        "category": result.category,
        "subcategory": result.subcategory,
        "priority": result.priority,
        "sentiment": result.sentiment,
        "requires_escalation": result.requires_escalation,
        "missing_info": result.missing_info,
        "extracted_entities": result.extracted_entities,
        "status": "triaged",
    }
```

### 23.2 Conditional route after triage

```python
def after_triage_route(state: TicketState) -> Literal["clarify", "retrieve"]:
    missing = state.get("missing_info", [])
    if missing and len(missing) >= 2:
        return "clarify"
    return "retrieve"
```

### 23.3 Clarification node

```python
def clarify_node(state: TicketState):
    missing = ", ".join(state.get("missing_info", [])) or "a few more details"
    reply = f"To help you faster, please provide the following details: {missing}."
    return {
        "messages": state.get("messages", []) + [{"role": "assistant", "content": reply}],
        "reply_draft": reply,
        "final_reply": reply,
        "status": "needs_clarification",
    }
```

### 23.4 Retrieval node

```python
def retrieval_node(state: TicketState):
    account_id = state.get("extracted_entities", {}).get("account_id", "")
    kb_query = f"{state.get('category', '')} {state['user_request']}"

    kb_results        = search_kb.invoke({"query": kb_query})
    account_ctx       = lookup_account_context.invoke({"account_id": account_id}) if account_id else {}
    escalation_signal = detect_escalation_risk.invoke({"text": state["user_request"]})
    priority_signal   = priority_score.invoke({"text": state["user_request"]})

    kb_summary = " | ".join([r["text"][:180] for r in kb_results[:3]]) if kb_results else ""

    return {
        "kb_results": kb_results,
        "kb_summary": kb_summary,
        "account_context": account_ctx,
        "requires_escalation": bool(state.get("requires_escalation", False) or escalation_signal),
        "priority": state.get("priority") or priority_signal,
        "status": "retrieved",
    }
```

### 23.5 Resolution node (with long-term memory read)

```python
def resolution_node(state: TicketState):
    # --- READ LONG-TERM MEMORY ---
    user_id      = state.get("extracted_entities", {}).get("account_id", "anonymous")
    user_prefs   = load_memory(MEMORY_CONN, f"user:{user_id}:preferences")
    user_history = load_memory(MEMORY_CONN, f"user:{user_id}:history")

    policy_rule = route_policy_lookup.invoke({
        "category": state.get("category", ""),
        "priority": state.get("priority", ""),
    })

    prompt = f"""
User request: {state['user_request']}
Category: {state.get('category')}
Subcategory: {state.get('subcategory')}
Priority: {state.get('priority')}
Sentiment: {state.get('sentiment')}
Requires escalation: {state.get('requires_escalation')}
Missing info: {state.get('missing_info')}
Retrieved KB summary: {state.get('kb_summary')}
Full KB results: {state.get('kb_results')}
Account context: {state.get('account_context')}
Policy rule: {policy_rule}

Long-term user preferences: {user_prefs}
Long-term user history: {user_history}
""".strip()

    msgs = prepare_messages(state.get("messages", []))

    def run():
        return plan_llm.invoke([
            SystemMessage(content=RESOLUTION_SYSTEM),
            HumanMessage(content=prompt),
        ])

    try:
        result = with_retry(run)
        route_to_team      = result.route_to_team
        recommended_action = result.recommended_action
        internal_notes     = result.internal_notes
        clarification      = result.should_ask_for_clarification
    except Exception:
        route_to_team      = policy_rule.get("route_to_team", "human_specialist")
        recommended_action = policy_rule.get("recommended_action", "manual_review")
        internal_notes     = "Automatic planning failed. Routed for manual review."
        clarification      = False

    if state.get("requires_escalation") or state.get("priority") == "critical":
        if not route_to_team:
            route_to_team = "human_specialist"

    return {
        "route_to_team": route_to_team,
        "recommended_action": recommended_action,
        "internal_notes": internal_notes,
        "should_ask_for_clarification": clarification,
        "status": "planned",
    }
```

### 23.6 Draft node

```python
def draft_node(state: TicketState):
    prompt = f"""
Write a customer-facing support reply.

User request: {state['user_request']}
Category: {state.get('category')}
Priority: {state.get('priority')}
Sentiment: {state.get('sentiment')}
Requires escalation: {state.get('requires_escalation')}
Missing info: {state.get('missing_info')}
Route to team: {state.get('route_to_team')}
Recommended action: {state.get('recommended_action')}
Internal notes: {state.get('internal_notes')}
KB evidence: {state.get('kb_results')}
Account context: {state.get('account_context')}

Rules:
- Ground the reply only in the provided evidence.
- Do not promise unsupported actions.
- If the ticket needs clarification, ask for missing information.
- If escalation is required, explain the next step conservatively.
""".strip()

    msgs = prepare_messages(state.get("messages", []))

    def run():
        return base_llm.invoke([
            SystemMessage(content=DRAFTING_SYSTEM),
            HumanMessage(content=prompt),
        ])

    try:
        response = with_retry(run)
        reply = response.content
    except Exception:
        reply = (
            "Thank you for contacting support. We are reviewing your case and will route it "
            "to the appropriate team. If needed, please reply with any additional details."
        )

    return {
        "messages": state.get("messages", []) + [{"role": "assistant", "content": reply}],
        "reply_draft": reply,
        "status": "drafted",
    }
```

### 23.7 Review node (HITL interrupt)

```python
def review_node(state: TicketState) -> Command[Literal["finalize", "revise", "manual_escalation"]]:
    review_payload = {
        "question": "Review the proposed support response.",
        "category": state.get("category"),
        "priority": state.get("priority"),
        "route_to_team": state.get("route_to_team"),
        "requires_escalation": state.get("requires_escalation"),
        "reply_draft": state.get("reply_draft"),
        "instructions": "Reply with one of: approve | revise: <feedback> | escalate_manually",
    }

    decision = interrupt(review_payload)

    if isinstance(decision, str):
        d = decision.strip().lower()
        if d == "approve":
            return Command(goto="finalize")
        if d.startswith("revise:"):
            return Command(update={"human_feedback": decision}, goto="revise")
        if d == "escalate_manually":
            return Command(update={"human_feedback": decision}, goto="manual_escalation")

    return Command(update={"human_feedback": "approve"}, goto="finalize")
```

### 23.8 Revise node

```python
def revise_node(state: TicketState):
    prompt = f"""
Current draft:
{state.get('reply_draft')}

Reviewer feedback:
{state.get('human_feedback')}
""".strip()

    def run():
        return base_llm.invoke([
            SystemMessage(content=REVISION_SYSTEM),
            HumanMessage(content=prompt),
        ])

    response = with_retry(run)
    revised  = response.content

    return {
        "messages": state.get("messages", []) + [{"role": "assistant", "content": revised}],
        "reply_draft": revised,
        "status": "revised",
    }
```

### 23.9 Finalize node (with long-term memory write-back)

```python
def finalize_node(state: TicketState):
    user_id = state.get("extracted_entities", {}).get("account_id", "anonymous")

    # --- WRITE-BACK TO LONG-TERM MEMORY ---
    try:
        if state.get("sentiment") in ("angry", "frustrated"):
            save_memory(MEMORY_CONN, f"user:{user_id}:preferences", "tone", "empathetic")

        if state.get("requires_escalation"):
            save_memory(MEMORY_CONN, f"user:{user_id}:history", "escalation_flag", True)

        if state.get("category"):
            save_memory(MEMORY_CONN, f"user:{user_id}:history", "last_category", state["category"])
    except Exception as e:
        print(f"[WARNING] Memory write-back failed: {e}")  # Non-fatal

    return {
        "final_reply": state.get("reply_draft"),
        "human_approved": True,
        "status": "approved",
    }
```

### 23.10 Manual escalation node

```python
def manual_escalation_node(state: TicketState):
    reply = (
        "Your case has been escalated for manual handling by a support specialist. "
        "We will continue the review and follow up through the appropriate channel."
    )
    return {
        "final_reply": reply,
        "human_approved": False,
        "status": "manual_escalation",
        "route_to_team": "human_specialist",
    }
```

---

## 24. Optional Parallel Retrieval Design

For Colab reliability, sequential retrieval is the baseline.
A parallel upgrade can be noted conceptually: run KB search, account lookup, and urgency scoring simultaneously, then merge into shared state.

---

## 25. Graph Construction

### 25.1 Dual SQLite setup

```python
# LangGraph checkpointer — thread state persistence
checkpointer = SqliteSaver.from_conn_string("workflow_state.sqlite")

# Long-term memory — separate table (initialized in Section 15.2)
# MEMORY_CONN = init_memory_db("memory_store.sqlite")
```

> Both can point to the same `.sqlite` file or separate files.
> The important rule: **use SQLite, not InMemorySaver, in the final notebook**.

### 25.2 Build graph

```python
builder = StateGraph(TicketState)

builder.add_node("triage",            triage_node)
builder.add_node("clarify",           clarify_node)
builder.add_node("retrieve",          retrieval_node)
builder.add_node("resolve",           resolution_node)
builder.add_node("draft",             draft_node)
builder.add_node("review",            review_node)
builder.add_node("revise",            revise_node)
builder.add_node("finalize",          finalize_node)
builder.add_node("manual_escalation", manual_escalation_node)

builder.add_edge(START, "triage")
builder.add_conditional_edges("triage", after_triage_route, ["clarify", "retrieve"])
builder.add_edge("clarify",           END)
builder.add_edge("retrieve",          "resolve")
builder.add_edge("resolve",           "draft")
builder.add_edge("draft",             "review")
builder.add_edge("revise",            "review")
builder.add_edge("finalize",          END)
builder.add_edge("manual_escalation", END)

graph = builder.compile(checkpointer=checkpointer)
```

---

## 26. Core Function Required by the Assignment

```python
def execute_workflow(user_request: str):
    thread_id = str(uuid.uuid4())
    config    = {"configurable": {"thread_id": thread_id}}

    result = graph.invoke(
        {
            "thread_id": thread_id,
            "user_request": user_request,
            "messages": [{"role": "user", "content": user_request}],
            "human_approved": False,
            "status": "started",
            "errors": [],
        },
        config=config,
    )

    return {
        "thread_id": thread_id,
        "result": result,
    }
```

> **v3.4 change:** `messages` is now initialized with the first user message instead of the old `conversation_history` list.

---

## 27. Resume Helper for HITL

```python
def resume_workflow(thread_id: str, decision: str):
    config = {"configurable": {"thread_id": thread_id}}
    return graph.invoke(Command(resume=decision), config=config)
```

### Example usage

```python
run       = execute_workflow("I was charged twice this month and nobody replied to my last email.")
thread_id = run["thread_id"]

interrupt_payload = run["result"]["__interrupt__"]
print(interrupt_payload)

# Option A — approve
approved = resume_workflow(thread_id, "approve")

# Option B — revise
revised  = resume_workflow(thread_id, "revise: keep the apology stronger and under 90 words")

# Option C — escalate manually
manual   = resume_workflow(thread_id, "escalate_manually")
```

---

## 28. Notebook Demonstration Pattern

For each test case, show:

1. raw input
2. first `execute_workflow(...)`
3. interrupt payload from review node
4. human decision
5. resumed result
6. final summary (use `print_summary`)
7. *(optional)* long-term memory state after run (use `load_memory`)

This makes both HITL and memory write-back visible for grading.

---

## 29. Test Plan (6 tests)

### Test 1 — Duplicate billing charge

**Input:** `"I was charged twice this month and nobody replied to my last email."`

**Expected:** category = billing, priority = high, escalation risk likely, route = billing_support, grounded draft produced.

### Test 2 — Business outage

**Input:** `"Our office internet has been down since this morning and we cannot work."`

**Expected:** category = technical, priority = critical, escalation required, technical route selected, manual escalation is a valid review outcome.

### Test 3 — Login/access issue

**Input:** `"I cannot log into my account after changing my password yesterday."`

**Expected:** category = account_access, priority = medium, KB evidence includes reset guidance, response suggests safe next steps.

### Test 4 — Cancellation threat

**Input:** `"If this is not fixed today, cancel everything. I am done with your service."`

**Expected:** angry sentiment, high escalation risk, retention/senior route or manual escalation, conservative reply. Long-term memory should record `tone: "empathetic"` after finalize.

### Test 5 — Incomplete ticket

**Input:** `"My account is broken."`

**Expected:** missing info populated, routed to clarification, no normal resolution flow.

### Test 6 — Revision loop

**Input:** `"Your app keeps logging me out every hour."`

**Expected:** normal draft generated, reviewer requests revision, graph resumes successfully, revised reply returned. Message history should contain both the original draft and the revised version.

---

## 30. Output Inspection Helpers

### 30.1 Compact summary printer

```python
def print_summary(state: dict):
    fields = [
        "status", "category", "subcategory", "priority", "sentiment",
        "requires_escalation", "route_to_team", "recommended_action",
        "human_approved", "final_reply",
    ]
    for f in fields:
        if f in state:
            print(f"{f}: {state[f]}")
```

### 30.2 Interrupt inspector

```python
def print_interrupt(run_result: dict):
    if "__interrupt__" in run_result:
        print(run_result["__interrupt__"])
    else:
        print("No interrupt found.")
```

### 30.3 Long-term memory inspector (v3.4 addition)

```python
def print_user_memory(user_id: str):
    for ns in ["preferences", "history"]:
        mem = load_memory(MEMORY_CONN, f"user:{user_id}:{ns}")
        print(f"[{ns}] {mem}")
```

---

## 31. Build Order (v3.4 — memory-complete)

Build in this order:

1. package install cell
2. secret-loading cell
3. JSON data loading
4. KB → documents → chunks → vector store
5. account/routing data
6. **`init_memory_db()` — create `memory_store` table**
7. **`load_memory()` / `save_memory()` helpers**
8. **`prepare_messages()` trim helper**
9. state and schemas (`TicketState` with `messages`)
10. LLM setup
11. prompts
12. tools
13. triage node (appends to `messages`)
14. retrieval node
15. resolution node (**reads long-term memory**)
16. draft node (appends to `messages`)
17. review / revise / finalize nodes (**finalize writes back memory**)
18. graph construction (dual SQLite)
19. execute/resume helpers
20. tests
21. polish + `print_user_memory` demos

---

## 32. Colab Risk Assessment and Mitigation

### 32.1 Main risks

- package version mismatch
- embedding/vector setup issues
- SQLite saver import variation
- non-serializable state
- incomplete HITL resume logic
- memory write-back failure crashing finalize

### 32.2 Mitigations

- pin versions
- keep the KB small
- keep state serializable (long-term memory lives outside state)
- keep vector store local
- build artifacts at runtime
- test from a fresh runtime
- wrap `save_memory` calls in try/except — write failures are non-fatal
- use SQLite for both checkpointer and memory store

### 32.3 Specific rules

- do not add Google Drive to the baseline
- do not add LangSmith to the baseline
- do not add real email sending
- do not add extra secrets unless necessary

---

## 33. Reliability + Safety Checklist (v3.4)

- [ ] prompts are role-specific
- [ ] retrieval uses actual retriever, not keyword-only stub
- [ ] account context is a tool, not hallucinated
- [ ] escalation override is enforced
- [ ] tool data wins over unsupported reasoning
- [ ] final draft does not promise unsupported actions
- [ ] human review interrupt is visible
- [ ] revise path works
- [ ] manual escalation path works
- [ ] state is serializable
- [ ] `messages` uses append semantics — never overwritten
- [ ] `prepare_messages()` called before every LLM invocation
- [ ] `memory_store` table created at startup
- [ ] resolution node reads long-term memory before LLM call
- [ ] finalize node writes back sentiment/escalation/category
- [ ] memory write-back is wrapped in try/except (non-fatal)
- [ ] long-term memory not stored inside `TicketState`

---

## 34. Submission Checklist (v3.4)

- [ ] notebook runs in Colab
- [ ] `project_data.json` included
- [ ] keys are not hardcoded
- [ ] at least 2 agents implemented
- [ ] at least 2 tools implemented
- [ ] LangGraph used
- [ ] short-term memory: `messages` field with `trim_messages`
- [ ] long-term memory: SQLite `memory_store` with read-before-act and write-back
- [ ] RAG: runtime FAISS vector store from KB JSON
- [ ] HITL interrupt used
- [ ] `execute_workflow(user_request)` exists and initializes `messages`
- [ ] 6 tests demonstrated
- [ ] memory write-back visible in at least one test (recommended: Test 4 or Test 6)

---

## 35. Minimal ZIP Structure

```text
project.zip
├─ support_ticket_triage.ipynb
├─ project_data.json
└─ README.md   # optional
```

---

## 36. What to Tell the Grader

> This notebook implements a stateful multi-agent support ticket triage workflow using LangGraph and LangChain. The workflow uses a Triage Agent, Retrieval Agent, Resolution Agent, and Drafting Agent. Knowledge grounding is handled through a runtime-built RAG pipeline using FAISS and OpenAI embeddings over local JSON knowledge-base articles. Account and routing context are accessed through custom tools.
>
> Memory is implemented at three distinct layers: short-term thread memory using a `messages` field with `trim_messages` to prevent context overflow; long-term user memory using a namespaced SQLite `memory_store` table with read-before-act injection in the resolution node and write-back in the finalize node; and a static RAG vector store for KB grounding.
>
> Before finalization, the graph pauses using LangGraph interrupts for human review, and can resume on approval, revision feedback, or manual escalation. The workflow uses SQLite persistence for both graph checkpointing and long-term memory, avoids hardcoded secrets, and is demonstrated with six test cases.

---

## 37. Final Recommendation

Do **not** turn this into a giant autonomous agent with complex dynamic tool-calling everywhere.

For this assignment, the strongest design is:

- graph-first orchestration,
- structured LLM calls,
- deterministic tools,
- explicit retrieval grounding,
- explicit three-layer memory (short-term, long-term, RAG),
- visible human review,
- stable Colab execution.

That gives you the best balance of correctness, explainability, safety, memory coverage, and implementation reliability.
