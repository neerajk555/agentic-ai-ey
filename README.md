# Module 5 — Agentic AI
## Hands-On Lab Package

**Days 12–16 | Advanced | 5 core exercises + 1 capstone, ~15–30 min each**

This module builds agentic systems from the ground up: a single tool-using agent, a multi-agent supervisor-worker system, a human-in-the-loop approval gate, an observable/debuggable agent, and an agent hardened against indirect prompt injection.

```
module5_labs/
├── README.md
├── requirements.txt
├── .env.example
├── .gitignore
├── data/
│   ├── product_knowledge_base.csv     (Exercises 1, 4)
│   ├── loan_applications.csv          (Exercise 2)
│   ├── transfer_requests.csv          (Exercise 3)
│   ├── external_documents.csv         (Exercise 5)
│   └── capstone_applications.csv      (Exercise 6 — capstone)
└── exercises/
    ├── 01_ReAct_Agent_Tools.md              + 01_react_agent_tools.py
    ├── 02_Supervisor_Worker.md              + 02_supervisor_worker.py
    ├── 03_Human_In_The_Loop.md              + 03_human_in_the_loop.py
    ├── 04_Observability_Debugging.md        + 04_observability_debugging.py
    ├── 05_Guardrails_Injection_Defence.md   + 05_guardrails_injection_defence.py
    └── 06_Capstone_Loan_Agent.md            + 06_capstone_loan_agent.py
```

| # | Exercise | Theory Topic | Needs |
|---|----------|--------------|-------|
| 1 | Single ReAct Agent with Tools | 5.2 Single-Agent Architecture / 5.3 Tool Use & Function Calling | 1 Azure deployment |
| 2 | Multi-Agent Supervisor-Worker System | 5.2 Supervisor-Worker Architecture | 1 Azure deployment |
| 3 | Human-in-the-Loop Approval Gate | 5.7 Human-in-the-Loop Workflows and Approval Gates | Nothing — pure local logic, run interactively |
| 4 | Agent Observability and Debugging | 5.8 Agent Observability, Tracing, Debugging | 1 Azure deployment (Stage 2 only — Stage 1 runs offline) |
| 5 | Agent Guardrails and Injection Defence | 5.9 Agent Risks / 5.10 Reliability Patterns | 1 Azure deployment |
| 6 | **CAPSTONE:** End-to-End Loan Processing Agent | All of 5.2–5.10 combined | 1 Azure deployment, partially interactive |

**A note on frameworks:** the theory session names **LangGraph** (agent orchestration) and **LangSmith** (observability) as the production tools for this domain. Every exercise here builds the underlying mechanism by hand instead — a plain `while` loop calling Azure OpenAI's native tool-calling API, and a simple in-memory trace logger — using the exact same philosophy as earlier modules (hand-built vector store instead of ChromaDB in Module 4, hand-built RRF instead of Azure AI Search's hybrid mode, etc.). The goal is that when you DO pick up LangGraph or LangSmith later, they read as "a framework automating something I already understand," not a black box.

---

## Quick Start (TL;DR)

```bash
python -m venv module5-env && source module5-env/bin/activate
pip install -r requirements.txt
cp .env.example .env   # then fill in your Azure OpenAI credentials — see below
python exercises/03_human_in_the_loop.py   # safe first run — no Azure needed, fully interactive
```

---

## One-Time Environment Setup

### 1. Create and activate a virtual environment
```bash
python -m venv module5-env
source module5-env/bin/activate        # Windows: module5-env\Scripts\activate
```

### 2. Install dependencies
```bash
pip install -r requirements.txt
```

### 3. Set up Azure OpenAI (needed for Exercises 1, 2, 4's Stage 2, and 5)

If you already have a resource from an earlier module, reuse it — this module only needs **one** deployment. It does need to be a model that supports **tool/function calling** — every mainstream Azure OpenAI chat model does (including reasoning models), so this shouldn't require anything special.

**3.1 — Create the resource** (skip if you already have one):
- [Azure Portal](https://portal.azure.com) → search **"Azure OpenAI"** → **Create**
- Fill in Subscription / Resource group / Region / Name / Pricing tier (Standard S0)
- **Review + Create** → **Create** → **Go to resource**

**3.2 — Check what's actually deployable, then deploy it:**
> ⚠️ Azure's available models have been changing rapidly through 2026 — check live rather than trusting a specific name from any guide.
1. Resource → **Go to Azure AI Foundry portal** → **Deployments** → **+ Create new deployment**
2. Filter **Deployment type** to **Standard**, see what's actually selectable
3. Pick any available chat model — **Deployment name:** e.g. `agentic-ai-deploy`
4. Wait for **"Succeeded"**

Every script in this module that sets `temperature` has an automatic fallback if your deployment rejects it (reasoning models) — you don't need to worry about which type of model you end up with.

**3.3 — Get credentials:** resource → **Keys and Endpoint** → copy **KEY 1** and the **Endpoint**.

**3.4 — Configure `.env`:**
```bash
cp .env.example .env
```
```
AZURE_OPENAI_API_KEY=your-key-here
AZURE_OPENAI_ENDPOINT=https://your-resource-name.openai.azure.com/
AZURE_OPENAI_DEPLOYMENT=agentic-ai-deploy
AZURE_OPENAI_API_VERSION=2024-10-21
```

---

## Running the Exercises

Run from the **project root** (`module5_labs/`), not from inside `exercises/`:

```bash
python exercises/01_react_agent_tools.py
python exercises/02_supervisor_worker.py
python exercises/03_human_in_the_loop.py
python exercises/04_observability_debugging.py
python exercises/05_guardrails_injection_defence.py
python exercises/06_capstone_loan_agent.py
```

**Exercise 3 is interactive** — it will pause and prompt you to type `a` (approve), `r` (reject), or `m` (modify) for each transfer above the approval threshold. Run it in a real terminal, not a notebook cell that can't accept input.

If Azure OpenAI isn't configured yet, Exercises 1, 2, and 5 print a clear warning and exit cleanly; Exercise 4 runs its offline Stage 1 (the trace/debugging demo) regardless and only skips Stage 2.

---

## Editing the Datasets

Every script reads its data from `data/` — nothing is hardcoded in the Python files. Add a new product, loan application, transfer request, or news article, then re-run. No code changes needed.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `[WARNING] Azure OpenAI not configured — missing: ...` | Complete Step 3 above — `.env` isn't filled in yet. |
| Exercise 3 doesn't seem to respond to input | Make sure you're running it in an actual terminal (Command Prompt, PowerShell, Terminal.app), not an environment that can't pass through keyboard input. |
| `AuthenticationError` / `401` | Check `AZURE_OPENAI_API_KEY` / `AZURE_OPENAI_ENDPOINT` are copied exactly from **Keys and Endpoint**. |
| `ServiceModelDeprecating` when creating a deployment | The model you picked can't be newly deployed anymore. Filter to **Standard** in Azure AI Foundry and pick from what's actually selectable. |
| `BadRequestError` mentioning `temperature` | Should be handled automatically by every script that calls Azure OpenAI in this module — if you still see this, you may be running an older copy. |
| `BadRequestError` mentioning `tools` or function calling not supported | Rare, but some very old or specialised deployments don't support tool calling. Try a different, more mainstream chat model from your Standard deployment list. |
| Agent seems to loop without giving a final answer | Every agent loop in this module has a `MAX_ITERATIONS` circuit breaker (see Exercise 1's Concept Primer) — it will stop and print a message rather than loop forever. If you hit this often, check your tool schemas match what the functions actually expect. |
| `ModuleNotFoundError` for any package | Confirm your virtual environment is activated, then re-run `pip install -r requirements.txt`. |

---

## Suggested Pacing (fits across the Days 12–16 sessions)

| Time | Activity |
|------|----------|
| 0–5 min | Environment + `.env` check |
| 5–25 min | Exercise 1 — Single ReAct Agent with Tools |
| 25–45 min | Exercise 2 — Multi-Agent Supervisor-Worker System |
| 45–60 min | Exercise 3 — Human-in-the-Loop Approval Gate (interactive) |
| 60–80 min | Exercise 4 — Observability and Debugging |
| 80–100 min | Exercise 5 — Guardrails and Injection Defence |
| 100–130 min | Exercise 6 — CAPSTONE: End-to-End Loan Processing Agent (partially interactive) |

Instructors: the original curriculum notes "Select 2 options. Advanced teams may attempt 3 if time allows" for this module — with 5 core exercises plus a capstone now built out, consider running Exercises 1 and 3 as the core pair for most cohorts (they're the most foundational and the most demonstrable live), treating 2, 4, and 5 as extension material for advanced groups, and reserving the Exercise 6 capstone specifically for advanced cohorts who've completed all five — it assumes familiarity with every prior exercise's pattern and is the best single exercise to run if you only have time for ONE demonstration of "what a real agentic system looks like end-to-end."
