# BugHound Mini Model Card (Reflection)

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.  
**Intended users:** Students learning agentic workflows and AI reliability concepts. Also useful as a lightweight code review aid for small Python scripts.

---

## 2) How does it work?

BugHound runs a five-step agentic loop:

1. **PLAN** — Logs intent to run a scan and fix-proposal workflow. No decisions are made here; it is a bookkeeping step.
2. **ANALYZE** — Detects issues in the code. In heuristic mode (no API key or `MockClient`), it pattern-matches for three signals: `print(` calls, bare `except:` blocks, and `TODO` comments. In Gemini mode it sends the code to the model using the structured prompts from `prompts/analyzer_system.txt` and `prompts/analyzer_user.txt`, then parses the JSON response. If the model returns non-JSON or raises an error, it falls back to heuristics automatically.
3. **ACT** — Proposes a fix. In heuristic mode it performs mechanical text replacements: bare `except:` → `except Exception as e:`, and `print(` → `logging.info(`. In Gemini mode it sends the issues and original code to the model using `prompts/fixer_system.txt` and `prompts/fixer_user.txt`, and strips any markdown code fences from the response. Falls back to heuristics on empty or error responses.
4. **TEST** — Calls `assess_risk()` to score the proposed fix from 0–100, penalizing for high/medium/low severity issues, code shrinkage, missing return statements, and removal of exception handling.
5. **REFLECT** — Decides `should_autofix` based on the risk level. Only a "low" risk score (≥75) allows auto-fix; everything else requires human review.

---

## 3) Inputs and outputs

**Inputs tested:**

| File | Shape | Notable patterns |
|---|---|---|
| `print_spam.py` | Single function, 4 lines | Multiple `print()` calls |
| `flaky_try_except.py` | Single function with `try/except` | Bare `except:`, file I/O |
| `mixed_issues.py` | Function with `print`, `try/except`, `TODO` | All three heuristic signals |
| `cleanish.py` | Single function using `logging` | No heuristic issues |

**Outputs observed:**

- *Issues detected:* `Code Quality / Low` (print statements), `Reliability / High` (bare except), `Maintainability / Medium` (TODO comments). Clean code correctly returns an empty list.
- *Fixes proposed:* Heuristic fixer prepends `import logging`, replaces `print(` with `logging.info(`, and rewrites bare `except:` to `except Exception as e:` with a placeholder comment. Gemini fixes are more contextual (e.g., catching `ZeroDivisionError` specifically, adding type hints).
- *Risk reports:* `mixed_issues.py` typically scores around 50–55 (medium, no autofix) due to a High severity issue deducting 40 points. `cleanish.py` scores 100 with no deductions.

---

## 4) Reliability and safety rules

**Rule 1 — Missing return statement (`-30 points`)**  
Checks whether `return` appears in the original code but not in the fixed code.  
*Why it matters:* Removing a return statement silently changes a function's contract — callers expecting a value receive `None`, which can cause downstream type errors or crashes with no obvious failure point.  
*False positive:* A refactor that intentionally converts a function to a procedure (e.g., moving a side-effectful function to return nothing). The scorer penalizes this even though the change may be correct.  
*False negative:* The check is string-based amd if the original uses `return` inside a nested helper defined inside the same snippet, and the fix removes the outer function's return but keeps the inner one, the rule won't fire.

**Rule 2 — All exception handling removed (`-25 points`, added in this session)**  
Checks whether a `try:` block present in the original is completely absent from the fixed code.  
*Why it matters:* A fix that strips all error handling may introduce unhandled exceptions at runtime. This is especially risky for I/O-heavy code where failures are expected.  
*False positive:* If the original `try:` block wrapped code that genuinely cannot raise (e.g., pure arithmetic), removing it is safe — but the rule still fires.  
*False negative:* The fix could retain `try:` but replace a specific `except ValueError` with a bare `except:`, which is arguably worse — but this rule would not trigger.

---

## 5) Observed failure modes

**Failure 1 — Logic bugs are invisible to heuristics**  
Input: `cleanish.py` — a function using `logging.info` that returns `a + b`. BugHound correctly finds no issues. However, if the function had an off-by-one error, a wrong operator, or a missing edge-case guard, BugHound would still report zero issues. Heuristics only check surface patterns, not program semantics.

**Failure 2 — Heuristic fixer over-edits print statements**  
Input: `mixed_issues.py` with `print("computing ratio...")`. The heuristic fix replaces this with `logging.info("computing ratio...")` — mechanically correct but semantically wrong, since `logging.info` expects a string message and will behave differently when given format arguments. More critically, it also prepends `import logging` even if the fix is applied to a script where logging was never configured, potentially producing silent output. The fix changes more than the issue strictly requires.

---

## 6) Heuristic vs. Gemini comparison

| Dimension | Heuristic mode | Gemini mode |
|---|---|---|
| Issues detected | Only `print`, bare `except`, `TODO` | Broader: naming, type hints, logic gaps, missing docstrings |
| Consistency | Perfectly deterministic | Varies by temperature; can return different issues on repeated runs |
| Fix quality | Mechanical text swap; always valid Python | More context-aware; may restructure logic or rename variables |
| Risk scorer alignment | Scores are calibrated to heuristic output (known severities) | Gemini may return edge-case severities that the scorer does not expect |
| Fallback behavior | Is the fallback | Falls back to heuristics on any parse failure |

Heuristics are more predictable but miss entire categories of bugs. Gemini detects richer issues but requires the agent to validate its output carefully , which is exactly what `_parse_json_array_of_issues` and `_normalize_issues` exist to do.

---

## 7) Human-in-the-loop decision

**Scenario:** A fix that rewrites a `try/except` block in code that performs file I/O or network calls. Even if the risk score is "low" (e.g., only a Low-severity issue was detected), silently auto-applying a change that alters error-handling behavior in production I/O code is high-stakes.

**Trigger:** Any fix where the structure of the `except` clause changes AND the original code performs I/O operations (detected by checking for `open(`, `requests.`, `socket.`, etc.)

**Where to implement:** In `risk_assessor.py` as a new deduction block — keeping the safety logic centralized and testable, rather than scattered across the UI or agent workflow.

**Message to show the user:**  
> "This fix modifies error-handling in code that may perform I/O. Auto-fix has been disabled. Please review the diff before applying."

---

## 8) Improvement idea

**Minimal-diff guardrail in `assess_risk`**

Add a penalty when the number of changed lines exceeds 50% of the original, even if the code did not shrink The existing shrinkage check (`len(fixed_lines) < len(original_lines) * 0.5`) only catches deletions — it misses the case where the LLM rewrites the entire file while keeping it the same length (a common Gemini behavior when it decides to "clean up" beyond the stated issues).

Implementation: compute a simple line-by-line diff count using `difflib.ndiff` and deduct 15 points if more than half the lines differ. This requires no new dependencies, is easy to test, and directly addresses the "over-editing" failure mode without restricting what kinds of fixes are allowed.
