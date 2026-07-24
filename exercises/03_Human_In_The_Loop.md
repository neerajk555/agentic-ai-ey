# Exercise 3 — Human-in-the-Loop Approval Gate

**Difficulty:** Intermediate–Advanced | **Time:** 15–20 min | **Theory link:** 5.7 Human-in-the-Loop Workflows and Approval Gates

---

## 🎯 Goal

Some agent actions are too consequential to execute autonomously, however capable the agent is. This exercise builds an explicit pause point: the agent prepares everything needed for a decision, presents it clearly to a human, and only proceeds once that human has actually approved it — with automatic risk-signal flagging to help the human know what to look at closely.

By the end, you will be able to:
- Explain why some actions warrant a mandatory human checkpoint regardless of how confident an agent is
- Build an interrupt point that pauses execution, presents a clear summary, and branches based on a real human decision
- Design simple, explainable, deterministic risk-flagging to help a human reviewer focus their attention
- Explain why "fail closed" (default to rejecting on unclear input) matters at an approval gate specifically

**Real-world case — Banking:** A digital lending platform deployed an AI agent for loan disbursement automation but implemented a mandatory human-in-the-loop gate for any disbursement above a risk threshold. This reduced erroneous disbursements to zero in the first six months while maintaining a 91% straight-through processing rate for low-risk cases — this exercise builds exactly that pattern: most transactions still flow through automatically, only the risky ones stop for a human.

---

## 🧠 Concept Primer

### Why not just make the agent more careful instead of adding a human gate?

No amount of prompt engineering makes an LLM-based agent's judgment verifiably reliable enough to authorize large, irreversible financial transactions entirely on its own — and critically, "the agent seemed confident" isn't the same as "the agent was correct." A human-in-the-loop gate isn't a workaround for an agent that isn't good enough yet; it's a permanent, deliberate architectural choice for actions where the cost of a wrong autonomous decision is too high, regardless of how good the agent gets.

### Why flag risk signals automatically, rather than just showing the human raw data?

A tired or rushed reviewer looking at a wall of transaction details can miss something a simple, explainable heuristic catches instantly. Automated flagging doesn't replace human judgment — it directs the human's attention to specifically WHY this transaction might need extra scrutiny (an unusually large amount, urgency language in the memo, an unfamiliar recipient), making the human review faster AND more reliable.

### Why does this exercise default to "REJECTED" on unclear input, instead of, say, re-asking?

This is the "fail closed" principle: when a system can't clearly determine what a human wants, the SAFE default in a high-stakes approval context is to NOT proceed, not to guess. A payment system that defaults to "approved" on ambiguous input is far more dangerous than one that defaults to "blocked" — an incorrectly blocked transaction is an inconvenience; an incorrectly approved one may be irreversible.

### Why do only transactions ABOVE the threshold get a human gate?

Requiring human approval for EVERY action defeats the purpose of automation entirely, and in practice trains reviewers to click "approve" reflexively without real scrutiny (a well-documented failure mode called alert fatigue). Gating only the genuinely consequential actions — as this exercise does with `APPROVAL_THRESHOLD` — keeps human attention focused where it actually matters, which is exactly what let the real banking case study maintain a 91% straight-through rate while still catching the risky 9%.

---

## Step 1 — The dataset

`data/transfer_requests.csv` — 3 fund transfer requests: two routine, one deliberately designed to trigger multiple risk signals (a large amount, urgency language, and an unfamiliar recipient — classic social-engineering/fraud red flags).

---

## Step 2 — Set up

No Azure OpenAI needed — this exercise is pure local logic plus real interactive input.

```bash
pip install pandas  # already in requirements.txt
```

**Run this one in an actual terminal** — it needs to pause and accept real keyboard input.

---

## Step 3 — Code walkthrough

### Stage 1 — Automatic risk-signal flagging

```python
def flag_risk_signals(row):
    signals = []
    if row["amount"] > 100000:
        signals.append(f"very large amount (${row['amount']:,})")
    memo_lower = str(row["memo"]).lower()
    for phrase in SUSPICIOUS_MEMO_PHRASES:
        if phrase in memo_lower:
            signals.append(f"memo contains suspicious phrase: '{phrase}'")
    if "unknown" in str(row["recipient_name"]).lower():
        signals.append("recipient name suggests an unverified/unfamiliar party")
    return signals
```

**What this does:** three independent, fully explainable checks — amount size, suspicious memo language, and an unfamiliar-sounding recipient — each one a simple, auditable rule, not a black-box ML score. **Verified against the dataset:** TR002 ($125,000, "Urgent - per phone instructions", "Unknown Recipient Corp") correctly triggers 4 distinct signals; TR001 and TR003 correctly trigger none.

### Stage 2 — The interrupt point itself

```python
def get_human_decision(summary_text):
    print(f"\n{summary_text}")
    decision = input("\nApprove, Reject, or Modify this transfer? [a/r/m]: ")
    return decision
```

**What this does:** `input()` is a genuine, blocking pause — Python execution literally stops here until a human types something and presses Enter. This is the simplest possible implementation of an "interrupt point," and it's worth recognising that frameworks like LangGraph implement the same fundamental idea (pause execution, wait for external input, resume) with more sophisticated state persistence (so a REAL system can pause for hours or days, not just milliseconds, and survive a server restart while waiting) — but the core concept is exactly this blocking pause.

### Stage 3 — Deciding based on the human's actual response

```python
def process_decision(decision, row):
    decision = decision.strip().lower()
    if decision == "a":
        return f"✅ APPROVED — transfer {row['transfer_id']} would now be executed."
    elif decision == "r":
        return f"❌ REJECTED — transfer {row['transfer_id']} cancelled, sender notified."
    elif decision == "m":
        return f"✏️  MODIFICATION REQUESTED — transfer {row['transfer_id']} routed back to preparer for changes."
    else:
        return f"⚠️  INVALID INPUT ('{decision}') — defaulting to REJECTED for safety."
```

**What this does:** this function is deliberately kept SEPARATE from the `input()` call itself — it's a pure function that takes a decision string and returns an outcome, with no I/O inside it. This is what let this logic be fully unit-tested (all five branches — approve, reject, modify, invalid input, and input with extra whitespace) without needing an actual human present during development, while the real exercise still runs genuinely interactively when you execute it.

---

## Expected Output (verified — this is real output from running the script with simulated approve/reject input)

```
Transfer TR001 exceeds the $10,000 approval threshold — PAUSING for human review.

Transfer ID:  TR001
From:         Acme Logistics Ltd
To:           Global Freight Partners
Amount:       48,500 USD
Memo:         Q3 shipping contract settlement
No automatic risk signals flagged.

Approve, Reject, or Modify this transfer? [a/r/m]: r
❌ REJECTED — transfer TR001 cancelled, sender notified.

Transfer TR002 exceeds the $10,000 approval threshold — PAUSING for human review.

Transfer ID:  TR002
From:         Acme Logistics Ltd
To:           Unknown Recipient Corp
Amount:       125,000 USD
Memo:         Urgent - per phone instructions
⚠️  RISK SIGNALS FLAGGED: very large amount ($125,000); memo contains suspicious phrase: 'urgent';
    memo contains suspicious phrase: 'per phone instructions'; recipient name suggests an unverified/unfamiliar party

Approve, Reject, or Modify this transfer? [a/r/m]: a
✅ APPROVED — transfer TR002 would now be executed.

Transfer TR003 ($9,200) is below the approval threshold — auto-processing without human review.
✅ Transfer TR003 executed automatically.
```

---

## 🛠 Common Pitfalls

- **Setting the approval threshold too low:** if EVERY transaction requires human approval, you've defeated the purpose — this is the alert-fatigue trap discussed in the Concept Primer. Tune the threshold to genuinely risky territory, not "anything at all."
- **Treating risk signals as a verdict instead of a prompt for attention:** TR002's flags don't mean the transfer IS fraudulent — they mean a human should look closer. In this exercise's real run, a human might reasonably still approve TR002 after reviewing it (it could be entirely legitimate) — the point is ensuring that review actually happens, not pre-deciding the outcome.
- **Forgetting the "fail closed" principle when extending this yourself:** if you add new decision branches in the homework, make sure any UNRECOGNISED input still safely defaults to rejection, not silent approval.

---

## 🏠 Homework Exercise

1. Add a 4th transfer request to `data/transfer_requests.csv` that would trigger exactly ONE risk signal (not zero, not several) — pick which one deliberately.
2. Add a NEW risk signal check to `flag_risk_signals()` — for example, flag transfers where the recipient name doesn't match any name seen in previous transfers in the dataset (a simple "new payee" check, a common real fraud-prevention heuristic).
3. Run the exercise and, for the transfer you added, decide live whether you'd approve or reject it based on the flags shown — write 2-3 sentences on what specifically influenced your decision.

**Hints:**
- For the "new payee" check, you'd need to look at ALL other rows in the dataframe when evaluating one row — pull the full set of `recipient_name` values from every OTHER transfer first, then check if the current row's recipient is new to that set.
- This exercise's homework is a good moment to reflect on the difference between a system that "decides" and one that "supports a human deciding" — the second is what this whole exercise builds.
