# 🤖 Autonomous Data Operations & Incident Triage Agent

**An agentic AI system that replaces hardcoded ITSM triage rules with a reasoning LLM — built on n8n's LangChain integration.**

> Turns a static incident spreadsheet into a conversational operations analyst: it plans, calls tools, cross-references live data, and reports risk — with zero hardcoded routing logic.

---

## 🎯 The Problem

Traditional ITSM (IT Service Management) triage relies on **hardcoded rules, keyword-matching routers, or manual sorting** by support staff. This creates three compounding failures:

| Failure Mode | Operational Cost |
|---|---|
| Hardcoded routing logic | Brittle, doesn't adapt to novel incident patterns |
| Manual severity assignment | Introduces human bias and inconsistency |
| Delayed operational visibility | Missed SLAs go unnoticed until post-mortem, not in real time |

## 💡 The Use Case

An **Autonomous Data Operations & Incident Triage Agent** — a cognitive overlay sitting on top of live support queues. Rather than routing text through fixed if/else logic, the agent:

- **Dynamically interrogates** live spreadsheet data through natural-language requests
- **Handles multi-turn follow-ups** from operations managers without losing context
- **Calculates complex performance metrics on the fly** (SLA breach rates, resolution time distributions, reopen rates) — no manual Excel pivot tables
- **Surfaces hidden operational risk**, such as isolating disproportionate SLA failure rates by priority tier, that a static dashboard would bury

This is agentic AI applied to a genuinely painful, recognizable enterprise problem — not a toy demo.

---

## 🧠 Why This Is an Agentic System (Not a Chatbot Wrapper)

The core engineering bet here is **prompt engineering as the control layer**, replacing what would traditionally be hardcoded conditional logic in the workflow itself. The agent's `systemMessage` is the actual "program":

```xml
<role>
You are the n8n Data Analyst AI Agent, an expert data operations and 
governance analyst. You help users load, clean, and inspect spreadsheet 
data by intelligently calling the appropriate tools. You guide users 
step-by-step, narrate your actions transparently, flag anomalies, and 
turn raw data rows into clear business insights.
</role>

<instructions>
<goal>
Assist users in loading, previewing, and analyzing dataset records. 
When a user requests data processing, select the correct tool, execute 
it with the exact JSON parameters required, and explain the data 
results in plain, professional language.
</goal>

<rules>
1. Always state what action you are about to take before executing a tool.
2. If the data contains anomalies, major drops, or unexpected duplicates, 
   explicitly flag them for the user.
3. Keep summaries structured, clear, and focused on operational impact.
4. If an execution step fails or missing parameters are detected, 
   explain the issue clearly to the user rather than guessing.
</rules>
</instructions>
```

**Deliberate prompt engineering decisions worth noting:**

- **XML-structured prompting** (`<role>`, `<instructions>`, `<goal>`, `<rules>`) — gives the model unambiguous section boundaries, which measurably improves rule-following in longer system prompts compared to flat prose.
- **Rule 1 forces transparent planning** ("state what action you are about to take") — this converts the agent from a black box into an auditable actor, critical for anything touching operational/compliance data.
- **Rule 2 hardcodes an anomaly-detection *disposition*, not anomaly-detection *logic*** — the model is instructed to actively look for and surface drops/duplicates rather than passively reporting whatever it's asked. This is prompt-level behavior shaping standing in for what would otherwise require explicit statistical thresholding code.
- **Rule 4 constrains hallucination under failure** — explicitly forbids guessing on missing parameters, forcing graceful degradation instead of confidently wrong output.

The result: tool selection, sequencing, and error handling all emerge from prompt design — not from n8n's visual branching logic.

---

## 🏗️ Architecture

```
[Chat Trigger] → [AI Agent] ──┬─→ [OpenAI Chat Model — gpt-4o]   (reasoning engine)
                               ├─→ [Buffer Window Memory, k=10]    (multi-turn context)
                               ├─→ [Google Sheets Tool]            (live data retrieval)
                               └─→ [Gmail Tool]                    (report delivery)
```

The **AI Agent** node is the sole orchestrator. There is no manual IF/Switch branching in the workflow — every routing decision (fetch data? send email? ask a clarifying question?) is made by the LLM at inference time, based on the system prompt above plus the live conversation state.

---

## ⚙️ Tech Stack

| Layer | Technology | Role |
|---|---|---|
| **LLM Cognitive Provider** | OpenAI `gpt-4o` | Reasoning, tool selection, natural-language synthesis |
| **Orchestration Framework** | n8n (Advanced AI Agent + Memory nodes, LangChain-powered) | Agent runtime, tool binding, execution graph |
| **Vector Store / Database** | *None* | Deliberately excluded — demonstrates pure agentic tool-use over structured data rather than RAG-dependent retrieval |
| **External API — Google Sheets** | OAuth 2.0 | Dynamic fetch/read/cross-reference of live incident tracking data |
| **External API — Gmail** | OAuth 2.0 | Automated report delivery endpoint |

> **Note on the vector store omission:** this is intentional, not a gap. The dataset (53 structured incident rows) is small enough that direct in-context row filtering outperforms embedding-based retrieval on both latency and accuracy — a deliberate architecture decision, not a missing feature.

---

## 📈 Demonstrated Output

Given the prompt *"Analyse the dataset and create a summary report also highlighting the outliers"*, the agent autonomously:

1. States its plan before acting
2. Pulls the filtered incident set from Google Sheets
3. Computes: mean/median/std-dev resolution time, SLA breach count & rate, reopen rate, priority distribution
4. **Flags a P2 SLA breach rate of ~85.7%** as operationally significant — unprompted

This is the core proof point: the agent isn't just summarizing, it's **exercising judgment about what matters**.

---

## 🚀 Deployment & Execution

**Environment:** n8n Cloud or self-hosted (**v1.0.0+** required for Advanced AI Agent nodes)

**Required credentials:**

| Credential | Purpose |
|---|---|
| `OPENAI_API_KEY` | Provisions the cognitive engine (`gpt-4o`) |
| `GOOGLE_OAUTH_CREDENTIALS` (Client ID + Secret) | Authorizes Google Sheets + Gmail node connections |

**Setup steps:**

1. Create a project in **Google Cloud Console**; enable the **Gmail API** and **Google Sheets API**
2. Configure the OAuth consent screen and generate credentials
3. Import `workflow_template.json` directly into the n8n canvas
4. Connect the OpenAI and Google OAuth credentials in n8n's credential manager
5. Link the Google Sheets node to your target incident log spreadsheet ID
6. Activate the workflow and open the chat panel (or share the public chat URL)

---

## 🗂️ Dataset

Prototype tested against a 53-row ITSM incident governance dataset with fields spanning SLA tracking, priority tiers, business unit, resolution time, reopen status, and change-request linkage — representative of a real service-desk export.

---

## 👤 Repository Intent & Audience

This repository is a **premium portfolio showcase**, structured to demonstrate production-level competence in:

- **Agentic AI system design** — tool-use orchestration without hardcoded control flow
- **Prompt engineering as an architectural discipline** — treating the system prompt as the primary specification surface, not an afterthought
- **Enterprise operations logic** — modeling a real ITSM pain point, not a generic demo

It's built for technical recruiters and engineering hiring managers evaluating hands-on agentic AI and LLM orchestration capability.

---

## ⚠️ Known Limitations

- Currently scoped to a single-team filter (`Assigned_Team = Network Operations`) — generalize via the `filtersUI` parameter for multi-team triage
- Memory resets per session (10-message window, no persistent long-term store)
- Email body composition is LLM-generated at runtime (`$fromAI`) — recommend a human review step before use in compliance-sensitive reporting contexts
- No vector store means no semantic search over historical incident archives (see architecture note above for why this is by design at current scale)

---

## 🔮 Roadmap

- [ ] Add a vector store (Pinecone/Weaviate) for semantic search once incident history scales beyond in-context limits
- [ ] Scheduled trigger for autonomous daily/weekly reporting without a user prompt
- [ ] Slack/Teams delivery channel alongside Gmail
- [ ] Write-back tool so the agent can update ticket status directly in Google Sheets

---

