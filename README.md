# Cortex Support Triage: Intelligent Multi-Agent Workflow Engine

Welcome to the **Cortex Support Triage** project—a highly modular, context-aware, and production-inspired AI support triage system built on **LangGraph**. This project demonstrates how autonomous agents, deterministic business policies, and human-in-the-loop (HITL) supervision can be composed into a reliable support solution. 

By offloading repetitive tasks—categorization, document retrieval, and drafting—while enforcing strict policy guardrails, this system drastically reduces resolution times without compromising on quality or safety.

---

## 🌟 The Architecture Highlights

Our system is structured around an explicit state machine `StateGraph`, seamlessly passing a robust `TicketState` through varied specialized nodes. 

### 🧠 The Three-Layer Memory Architecture
To achieve truly contextual behavior, the system relies on three distinct memory layers avoiding context pollution while enabling deep personalization:

1. **Short-Term Memory (Thread State)**  
   - **Mechanism:** LangGraph state carrying conversation `messages`.
   - **Advantage:** Preserves immediate continuity between agent steps. Messages are aggressively managed using `trim_messages` to ensure token limits and context clarity are strictly maintained before every LLM invocation.

2. **Long-Term Memory (Persistent Profiles)**  
   - **Mechanism:** Namespaced SQLite database table (`memory_store`).
   - **Advantage:** Remembers a user's past escalations, preferences, and tone requirements across different sessions and threads. The system reads this before making resolution decisions, ensuring a uniquely tailored customer experience, and writes back updated facts upon ticket finalization.

3. **Grounding Memory (Runtime RAG)**  
   - **Mechanism:** Dynamic FAISS vector store created from JSON KB articles (`project_data.json`).
   - **Advantage:** Strict factual grounding. The agents are forbidden to promise refunds, exceptions, or technical capabilities unless specifically extracted from the company's approved Knowledge Base.

---

## 🤖 The Agent Ecosystem (Nodes)

This graph consists of highly specialized agents, each single-minded in its goal and orchestrated conditionally.

- **`triage_node` (The Classifier):** Leverages `gpt-4o-mini` with structured output capabilities to parse raw, messy user input into clean metadata—determining category, subcategory, base priority, sentiment, and missing information. It tells the system if the ticket is actionable or unresolvably incomplete.
- **`retrieval_node` (The Researcher):** Gathers empirical facts. It invokes deterministic tools to pull account metadata, looks up precise routing policies, assesses risk heuristically, and conducts vector search on the Knowledge Base.
- **`resolution_node` (The Planner):** This agent connects the dots. It reviews the long-term user profile, the RAG evidence, and the Triaged metadata to produce a structured `ResolutionPlan`. Should we escalate? Should we ask for clarity? It decides the `route_to_team` and the `recommended_action`.
- **`draft_node` (The Writer):** Drafts a professional, empathetic, policy-safe reply grounded strictly in the RAG outputs. It enforces tone compliance based on the customer's historical preferences.
- **`review_node` (Human-In-The-Loop):** The safeguard. Halts execution entirely and waits for a human manager to approve, revise, or manually escalate the draft.
- **`revise_node` / `finalize_node`:** Acts on human feedback. `revise_node` refines the draft to human specifications, while `finalize_node` records the resolution and updates the customer's Long-Term Memory profile for future interactions.

---

## 🛠️ Deterministic Tools: Bridging AI and Business Rules

LLMs are brilliant reasoners but poor determinists. We enforce safety using explicitly coded business logic:
- `search_kb`: Vector search over our approved article repository.
- `lookup_account_context`: Simulates a CRM fetch, pulling exact subscription statuses, billing tiers, or current user context.
- `detect_escalation_risk`: A heuristic text parser looking for explicit legal or churn risk terminology to force human review.
- `priority_score`: Algorithmically identifies critical terms (like "outage" or "breach").
- `route_policy_lookup`: Translates category/priority matrix directly into target internal support teams.

---

## 🚀 Core API Components

The system is easily embedded into external user-interfaces or automated runners using three clean wrapper functions:

- **`execute_workflow(user_request: str) -> dict`**  
  The main entry point. Generates a unique thread, invokes the graph, and runs until completion or until it hits the `review_node` interrupt. Returns the `thread_id` and the current state payloads.
  
- **`get_interrupt_payload(run_result: dict) -> dict`**  
  A secure accessor that reads the checkpoint to show the human operator the proposed draft, internal notes, and route destination waiting for their approval.
  
- **`resume_workflow(thread_id: str, decision: str, feedback: str) -> dict`**  
  Restarts the frozen graph thread with human directives. `decision` can securely route the graph to Finalize, Revise, or Manual Escalation logic branches.

---

## 🧪 Bulletproof Automated Testing

We don't just build; we verify. The notebook ships with an automated test harness evaluating multiple distinct support scenarios—ensuring routing precision, RAG effectiveness, clarification request behaviors, and explicit HITL overrides function flawlessly under varied stressors. 

Run the built-in test suite loop inside the notebook to generate a categorical report of pass/fail grading on system decisions versus expected truths.

---

## 💻 Getting Started (Quick Run)

1. Set your `OPENAI_API_KEY` in Colab Secrets or as an environment variable.
2. Open `support_ticket_triage.ipynb` and run the initial setup cells to install dependencies (LangGraph, FAISS, OpenAI).
3. Execute the workflow setup cells to compile the StateGraph and initialize the SQLite checkpoints.
4. Run the interactive test cells or the batch test harness at the bottom of the notebook to witness the agents in action.

*Welcome to the future of customer support resolution!*
