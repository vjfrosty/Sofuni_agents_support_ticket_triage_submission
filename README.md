# Cortex Support Triage — Intelligent Multi-Agent Workflow Engine

A production-inspired AI support triage system built on **LangGraph**, **LangChain**, and **OpenAI gpt-4o-mini**.
It demonstrates how autonomous agents, deterministic business policies, and Human-in-the-Loop (HITL)
supervision can be composed into a reliable, auditable, and personalisable support solution.

**Notebook:** `support_ticket_triage.ipynb`  
**Data:** `project_data.json` — loaded at runtime from GitHub (KB articles · account contexts · routing rules · test cases)  
**Persistence:** Two SQLite databases — `workflow_state.sqlite` (graph checkpoints) · `memory_store.sqlite` (long-term memory)

---

## 🌟 What Makes This System Different

| Feature | Detail |
|---|---|
| **Stateful graph** | LangGraph `StateGraph` with SQLite checkpointer — graph state survives kernel restarts |
| **API-forced tool call** | `routing_llm.bind_tools([route_policy_lookup], tool_choice="required")` — the model *must* call the routing tool; enforced at the OpenAI API level, not just a prompt hint |
| **3-layer memory** | Short-term (tiktoken-trimmed messages) · Long-term (namespaced SQLite) · RAG (FAISS over 17 KB articles) |
| **Model-as-Judge** | After the scorer, `gpt-4o-mini` grades each reply on groundedness · tone · completeness (1–5) with a pass/partial/fail verdict |
| **Unit tests** | 10 isolated `assert`-based tests on memory functions + all 5 tools — zero side effects |
| **Guardrails** | PII redaction (email, phone, SSN, card) + forbidden-topic blocking before graph invocation |

---

## 🏗️ Architecture

```
user_request
     │
     ▼
[apply_guardrails]  ← PII redact + forbidden-topic block
     │
     ▼
[load_memory_node]  ← pre-load SQLite preferences/history into state
     │
     ▼
[triage_node]  ← gpt-4o-mini structured output → TriageOutput
     │
     ├─ ≥2 missing fields ──► [clarify_node] ──► END
     │
     ▼
[retrieval_node]  ← 4 deterministic tools
     │              search_kb · lookup_account_context
     │              detect_escalation_risk · priority_score
     ▼
[resolution_node]  ← Step 1: routing_llm.bind_tools (API-forced tool call)
     │               Step 2: plan_llm structured output → ResolutionPlan
     ▼
[draft_node]  ← gpt-4o-mini, tone from long-term memory
     │
     ▼
[review_node]  ◄──────── HITL interrupt ──────────► human decision
     │                                                    │
     ├── approve ──────────────────────────────► [finalize_node]
     │                                                    │
     ├── revise: <feedback> ──► [revise_node] ──► back to review
     │
     └── escalate_manually ──► [manual_escalation_node] ──► END
```

---

## 🧠 Three-Layer Memory Architecture

### 1. Short-Term Memory (Thread State)
- **Mechanism:** `messages` field in `TicketState` — a list of `{role, content}` dicts.
- **Trimming:** `prepare_messages()` uses **tiktoken** (`cl100k_base`) to count tokens and keep only the most recent messages within `MSG_TRIM_MAX_TOKENS` (default 1800) before every LLM call.
- **Scope:** One graph thread — not shared across conversations.

### 2. Long-Term Memory (Persistent Profiles)
- **Mechanism:** Namespaced `memory_store` SQLite table — `PRIMARY KEY (namespace, key)`.
- **Namespaces:** `user:<account_id>:preferences` · `user:<account_id>:history` · `global:routing_preferences` · `global:tone_rules`
- **Read:** `resolution_node` and `draft_node` read preferences/history before acting.
- **Write:** `finalize_node` writes back updated tone, escalation flag, last category, and last known route.
- **Seed:** `seed_memory_defaults()` pre-loads defaults from `project_data.json` on startup using `INSERT OR IGNORE`.

### 3. RAG Grounding Memory (FAISS)
- **Mechanism:** 17 KB articles from `project_data.json` → `Document` chunks (500 chars, 100 overlap) → FAISS vector store with `text-embedding-3-small`.
- **Retrieval:** Top-4 chunks per query (`RETRIEVER_TOP_K = 4`).
- **Enforcement:** Agents may not promise refunds, credits, or actions unless the KB evidence supports them.

---

## 🤖 Agent Nodes (10)

| # | Node | Role | LLM |
|---|---|---|---|
| 1 | `load_memory_node` | Extracts `account_id` via regex, pre-loads 4 namespaces into `memory_context` | — |
| 2 | `triage_node` | Classifies ticket → `TriageOutput` (category · subcategory · priority · sentiment · escalation · missing_info · entities) | `triage_llm` (structured output) |
| 3 | `clarify_node` | Generates a clarification request when ≥2 fields are missing | — |
| 4 | `retrieval_node` | Calls 4 tools; reconciles LLM vs tool priority with `PRIORITY_RANK` | — |
| 5 | `resolution_node` | **Two-step**: forced tool call via `routing_llm.bind_tools` → structured plan via `plan_llm` | `routing_llm` + `plan_llm` |
| 6 | `draft_node` | Writes tone-aware, grounded customer reply; reads tone from long-term memory | `base_llm` |
| 7 | `review_node` | HITL interrupt — pauses for approve / revise / escalate_manually | — |
| 8 | `revise_node` | Refines draft per reviewer feedback | `base_llm` |
| 9 | `finalize_node` | Approves draft; writes 4 facts back to long-term memory | — |
| 10 | `manual_escalation_node` | Generates escalation message; marks `route_to_team=human_specialist` | — |

---

## 🛠️ Deterministic Tools (5)

| Tool | Type | Description |
|---|---|---|
| `search_kb` | FAISS retriever | Top-K vector search over 17 approved KB articles |
| `lookup_account_context` | Dict lookup | CRM-style fetch of subscription tier, billing status, open tickets |
| `detect_escalation_risk` | Keyword scanner | Heuristic: cancel · lawyer · regulator · ombudsman · "nobody replied" etc. |
| `priority_score` | Keyword scorer | Returns `critical` / `high` / `medium` / `low` from urgency keywords |
| `route_policy_lookup` | Rule table | Translates `(category, priority)` to `{route_to_team, recommended_action}` from `ROUTING_RULES` |

### API-Forced Tool Call (`routing_llm`)
`route_policy_lookup` is called with API-level enforcement in `resolution_node`:
```python
routing_llm = base_llm.bind_tools([route_policy_lookup], tool_choice="required")
```
The model **must** emit a `tool_call` — not just a text suggestion.
If the response contains no `tool_calls` (model defied the API), the node falls back to direct invocation using state values.

---

## 🚀 Core API

| Function | Description |
|---|---|
| `execute_workflow(user_request)` | Entry point. Applies guardrails, generates a UUID thread, invokes graph. Returns `{thread_id, result}`. |
| `resume_workflow(thread_id, decision)` | Resumes a paused HITL thread. Decision: `"approve"` · `"revise: <feedback>"` · `"escalate_manually"`. |
| `get_interrupt_payload(run_result)` | Safely unpacks the interrupt value dict from the raw graph result. |
| `print_summary(state)` | Compact state printout for grader/debug visibility. |
| `print_interrupt(run_result)` | Displays the HITL payload (draft + metadata) in readable format. |
| `print_agent_trace(state, thread_id)` | Narrates the agent handoff sequence from `state["status"]`. |
| `print_user_memory(user_id)` | Dumps preferences + history namespaces for a given account. |

---

## 🧪 Test Suite

### 7 End-to-End Tests

| # | Scenario | Account | HITL Decision | Key Assertion |
|---|---|---|---|---|
| 1 | Duplicate billing charge | acct_1001 | approve | `category=billing`, `route=billing_support` |
| 2 | Business office outage | acct_1002 | escalate_manually | `priority=critical`, `route=technical_support` |
| 3 | Login / MFA failure | acct_1008 | approve | `category=account_access`, `route=account_access_support` |
| 4 | Cancellation threat | acct_1004 | revise | `category=cancellation`, `route=retention_team`, memory write-back |
| 5 | Vague / incomplete ticket | — | clarify (no HITL) | `missing_info ≥ 2` → `clarify_node` |
| 6 | App logout revision loop | acct_1007 | revise | revision + message history contains both drafts |
| 7 | Legal / regulatory threat | acct_1009 | escalate_manually | `priority=critical`, `route=human_specialist` |

### HITL Showcase (2 isolated sub-runs)
- **Sub-run A**: approve path — asserts `human_approved=True`, `status=approved`
- **Sub-run B**: revise → re-review → approve loop — asserts `status ∈ {approved, revised}`

### 10 Unit Tests (Cell 41)
Isolated `assert`-based tests using `conn = init_memory_db(":memory:")` — no side effects on `MEMORY_CONN`:

| # | Test |
|---|---|
| 1 | `save_memory` + `load_memory` round-trip |
| 2 | `load_memory` on unknown namespace → `{}` |
| 3 | `save_memory` upsert — second write wins |
| 4 | `search_kb` returns `list` with `text` key |
| 5 | `lookup_account_context("acct_1001")` returns non-empty dict |
| 6–7 | `detect_escalation_risk` — True (cancel) + False (neutral) |
| 8–9 | `priority_score` — `critical` (outage) + `medium` (login) |
| 10 | `route_policy_lookup` returns dict with `route_to_team` key |

---

## 📊 Evaluation

### Automated Scorer (Cell 64)
Checks each test result against expected values from `project_data.json`:
- `category` · `priority` · `route_to_team` · `requires_escalation` · `has_final_reply`
- Reports per-test breakdown with `✓/✗/–` symbols and an overall `n/N checks passed (%)` summary.

### Model-as-Judge (Cell 65)
After the deterministic scorer, `gpt-4o-mini` re-evaluates each final reply:

| Dimension | Scale | Description |
|---|---|---|
| **groundedness** | 1–5 | Does the reply avoid claims not supported by the ticket or KB? |
| **tone** | 1–5 | Is the tone appropriate for the customer's sentiment and priority? |
| **completeness** | 1–5 | Does it address the core issue and explain next steps? |
| **verdict** | pass/partial/fail | `pass` = all ≥ 4 · `partial` = any = 3 · `fail` = any ≤ 2 |

Results are printed as a table with justifications. Errors are caught and logged as `ERR` — never crash the cell.

---

## 🛡️ Guardrails

Applied inside `execute_workflow()` before graph invocation:
- **PII redaction** — regex patterns for email, phone (US), SSN, credit card → `[EMAIL]`, `[PHONE]`, `[SSN]`, `[CARD]`
- **Forbidden topics** — blocks "guaranteed refund", "100% money back", "sue", "class action", "illegal", "fraud lawsuit", "criminal charges", "sex"
- Blocked requests return `status=blocked_by_guardrail` without invoking the graph.

---

## 💻 Getting Started

### Requirements
```
langchain>=0.3.16
langchain-openai>=0.2.11
langchain-community
langgraph>=0.2.76
faiss-cpu
tiktoken
pydantic>=2.0
```

### Quick Run (Google Colab or local Jupyter)
1. Add your `OPENAI_API_KEY` to Colab Secrets (key icon in sidebar) **or** set it as an environment variable:
   ```bash
   export OPENAI_API_KEY="sk-..."
   ```
2. Open `support_ticket_triage.ipynb`.
3. **Run cells top-to-bottom** — cells 2–38 are one-time setup.
4. After setup, run any test cell (43–61) or the full run below.
5. Start the interactive demo: run cell 69 (`interactive_session()`).

### Full Test Run (batch)
```
Run Cell 3  → install dependencies (Colab: may trigger restart — rerun from top)
Run Cell 6  → set API key
Run Cells 8–38 → all setup + graph compile
Run Cell 43 → Test 1
Run Cell 45 → Test 2
...
Run Cell 64 → automated scorer
Run Cell 65 → model-as-judge
```

### Project Data (`project_data.json`)
Loaded from GitHub at startup. Contains:
- `kb_articles` — 17 knowledge-base articles with `id`, `title`, `text`, `category`, `tags`
- `account_context` — per-account CRM data (subscription tier, billing status, open tickets, known outages)
- `routing_rules` — `(trigger_category, priority)` → `{route_to_team, recommended_action}`
- `test_cases` — 7 test scenarios with `input`, `expected_*` fields, `simulate_human_decision`, `notes`
- `memory_defaults` — pre-seeded long-term memory entries (preferences and history per account)
- `settings` — configurable constants (`embedding_model`, `retriever_top_k`, `message_trim_max_tokens`, `default_tone`)

---

## 📁 File Map

| File | Purpose |
|---|---|
| `support_ticket_triage.ipynb` | Main submission notebook — all 69 cells |
| `README.md` | This file — full project documentation |
| `support_ticket_triage_blueprint.md` | Design blueprint — architecture decisions and phase plan |
| `ai_agents_course_blueprint.md` | Course-level AI agents blueprint |
| `Course_materials.md` | Reference notes from course lectures |
| `memory_store.sqlite` | Long-term memory DB (created at first run) |
| `workflow_state.sqlite` | LangGraph checkpoint DB (created at first run) |

---

*Built for the SoftUni "AI Agents and Workflows for Developers" course · May 2026*
