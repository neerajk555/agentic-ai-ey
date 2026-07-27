# Exercise 4 — Agent Observability and Debugging

**Difficulty:** Intermediate–Advanced | **Time:** 15–20 min | **Theory link:** 5.8 Agent Observability, Tracing, Debugging, and State Management

---

## 🎯 Goal

When an agent fails, "it didn't work" isn't a diagnosis — you need to know exactly WHICH step failed, WHAT arguments caused it, and WHY, before you can fix anything. This exercise instruments an agent with full step-by-step tracing, deliberately injects a realistic failure, and uses the trace to pinpoint the exact root cause — then verifies the fix with a clean re-run.

By the end, you will be able to:
- Explain what information a useful agent trace needs to capture at each step
- Use a trace to pinpoint exactly where and why an agent run failed
- Distinguish a genuine root-cause fix from a re-run that happens to succeed
- Implement a basic token-budget alert as a simple cost-control observability signal

**Real-world context:** Any team running agents in production needs to answer "why did this specific run fail" quickly — without a trace, debugging an agent means guessing based on the final output alone, which is often nowhere near enough information to find the actual problem.

---

## 🧠 Concept Primer

### What does a trace actually need to record?

At minimum, for every tool call: which tool, what arguments, what came back (or what error was raised), and how long it took. For every LLM call: how many tokens were used and how long it took. This exercise's `TraceLogger` class records exactly these fields — nothing more exotic is needed for the debugging workflow this exercise demonstrates.

### Why inject a failure DELIBERATELY, rather than waiting for one to happen naturally?

A deliberately staged failure is fully reproducible — you know exactly what's wrong and can verify your debugging process actually finds it, rather than hoping a live model happens to make an interesting mistake during a live demo. This exercise's fault (a string passed where `calculate_emi` expects a number) is realistic — it's exactly the kind of error that CAN happen for real, from a model occasionally generating a malformed argument, or from a bug in argument-mapping code — just triggered on purpose here so the debugging technique itself is what's being taught.

### What does "pinpointing" actually look like in code?

```python
def find_first_failure(self):
    for entry in self.steps:
        if entry["type"] == "tool_call" and not entry["success"]:
            return entry
    return None
```

This is deliberately simple: scan the trace in order, return the FIRST failed step. In a real multi-step agent run, the first failure is usually the actual root cause — everything after it may just be downstream confusion caused by that original problem, not independent issues worth chasing separately.

### Why track cumulative tokens, specifically?

Token usage is the most direct proxy for cost in an LLM-based system. A "token budget alert" — flagging when a single agent run's cumulative usage crosses a threshold — is a simple, real observability signal that can catch a runaway or unexpectedly expensive agent run WHILE it's happening, not just after the bill arrives.

---

## Step 1 — The dataset

`data/product_knowledge_base.csv` — same knowledge base as Exercise 1.

---

## Step 2 — Set up

Stage 1 of this exercise (the deliberate failure demo) needs **no Azure OpenAI at all**. Stage 2 (a real traced agent run) needs the same single deployment as Exercise 1 — see the package `README.md`.

---

## Step 3 — Code walkthrough

### Stage 1 — The tracer itself

```python
class TraceLogger:
    def __init__(self):
        self.steps = []
        self.total_tokens = 0

    def log_tool_call(self, tool_name, args, result=None, error=None, latency_ms=None):
        entry = {"step": len(self.steps) + 1, "type": "tool_call", "tool_name": tool_name,
                 "args": args, "result": result, "error": error, "latency_ms": latency_ms,
                 "success": error is None}
        self.steps.append(entry)
        return entry
```

**What this does:** every call is recorded as a structured dict, appended to a simple in-memory list — no database, no external service. `"success": error is None` derives a clean boolean from whether an error was passed, so later code (like `find_first_failure`) can check success/failure with a simple attribute lookup instead of re-deriving it each time.

### Stage 2 — Injecting the deliberate failure

```python
execute_tool_traced("calculate_emi", {"principal": 40000, "annual_rate": "nine point five", "tenure_months": 36}, kb_df, tracer)
```

**What this does:** `annual_rate` is passed as the STRING `"nine point five"` instead of the number `9.5` — deliberately, to simulate exactly the kind of malformed-argument failure a real agent could produce. Because `calculate_emi()` has no type-checking (also deliberate — we want a REAL Python exception, not a sanitised hypothetical one), this raises a genuine `TypeError` when the arithmetic tries to divide a string by an integer. `execute_tool_traced()` catches it and logs it as a failed step rather than crashing the whole script.

### Stage 3 — Pinpointing and fixing

```python
failure = tracer.find_first_failure()
print(f"First failure found at step {failure['step']}: {failure['tool_name']}({failure['args']})")
print(f"Error was: {failure['error']}")
```

**What this does:** this is the actual debugging step — instead of scrolling through raw logs or guessing, `find_first_failure()` returns exactly the failed entry, with its exact arguments and exact error message immediately visible. **Verified running this exercise directly:** it correctly identifies step 2, `calculate_emi`, with the exact bad argument and the exact `TypeError` message, every time.

### Stage 4 — Verifying the fix with a clean re-run

```python
tracer2 = TraceLogger()  # a FRESH tracer, not reusing the failed one
execute_tool_traced("calculate_emi", {"principal": 40000, "annual_rate": 9.5, "tenure_months": 36}, kb_df, tracer2)  # FIXED
clean_failure = tracer2.find_first_failure()
print(f"Clean re-run: {'still broken' if clean_failure else '✅ fix verified'}")
```

**What this does:** using a NEW `TraceLogger` for the fixed re-run (rather than continuing to append to the failed one) means the verification is unambiguous — `find_first_failure()` on the fresh trace returning `None` is direct evidence the specific fix worked, not just that "something ran without crashing this time."

---

## Expected Output (verified — this is real, directly-tested output)

```
--- FULL TRACE ---
  [1] TOOL_CALL ✅ search_knowledge_base({'query': 'Flexi Personal Loan rate'}) [0.8ms]
        -> {'doc_id': 'PROD001', 'text': 'The Flexi Personal Loan offers amounts...'}
  [2] TOOL_CALL ❌ FAILED calculate_emi({'principal': 40000, 'annual_rate': 'nine point five', 'tenure_months': 36}) [0.0ms]
        -> ERROR: unsupported operand type(s) for /: 'str' and 'int'
  [3] TOOL_CALL ✅ get_current_date({}) [0.0ms]
        -> {'current_date': '2026-07-19', 'quarter': 'Q3', 'year': 2026}

-- Pinpointing the failure using the trace --
First failure found at step 2: calculate_emi({'principal': 40000, 'annual_rate': 'nine point five', 'tenure_months': 36})
Error was: unsupported operand type(s) for /: 'str' and 'int'
Root cause: 'annual_rate' was passed as a string, but calculate_emi expects a number.

-- Fixing and re-running cleanly --
--- FULL TRACE ---
  [1] TOOL_CALL ✅ search_knowledge_base(...) [0.6ms]
  [2] TOOL_CALL ✅ calculate_emi({'principal': 40000, 'annual_rate': 9.5, 'tenure_months': 36}) [0.0ms]
        -> {'monthly_emi': 1281.32, ...}
  [3] TOOL_CALL ✅ get_current_date({}) [0.0ms]

Clean re-run failure check: ✅ no failures — fix verified
```

---

## 🛠 Common Pitfalls

- **Fixing the SYMPTOM instead of the root cause:** if the real fix here had just been "catch the error and skip that tool call," the agent would still be missing the EMI it needed to answer the user — always trace back to WHY the bad value occurred, not just how to survive it.
- **Not using a fresh tracer for the verification run:** reusing the same trace object for both the broken and fixed runs would muddy the "is this genuinely fixed" signal — always verify with a clean slate.
- **Ignoring latency data in the trace:** this exercise's trace captures `latency_ms` for every step — in a real system, a sudden latency spike on one specific tool is often as valuable a debugging signal as an outright failure.

---

## 🏠 Homework Exercise

1. Inject a DIFFERENT deliberate failure — try calling `search_knowledge_base` with a missing `query` key entirely (an empty dict `{}` instead of `{"query": "..."}`), and trace what error results.
2. Extend `TraceLogger` with a new method, `total_latency_ms()`, that sums the `latency_ms` across every step in the trace — useful for answering "how much of this agent run's wall-clock time was spent in tools vs. waiting on the LLM?"
3. Lower `TOKEN_BUDGET_ALERT_THRESHOLD` to a very small number (e.g., 50) and re-run Stage 2 against live Azure — confirm the alert fires, then write 1-2 sentences on what a REAL token budget alert in production should probably DO beyond just printing a warning (e.g., halt the run? notify an on-call engineer? both?).

**Hints:**
- For question 1, `args["query"]` inside `search_knowledge_base()` will raise a `KeyError` when the key is missing — a different, equally realistic failure mode from the type-mismatch one already in the script.
- For question 3, there's no single right answer — but "just log it and keep going" is rarely the right production behaviour for a genuine budget overrun; this is worth a real discussion about acceptable automated responses vs. ones that need a human in the loop (connecting back to Exercise 3).
