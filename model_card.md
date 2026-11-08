# BugHound Mini Model Card (Reflection)

## 1) What is this system?

**Name:** BugHound
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before deciding whether the fix can be auto-applied or needs human review.
**Intended users:** Students learning agentic workflows and AI reliability. Not meant for production code.

---

## 2) How does it work?

BugHound runs a fixed 5-step agent loop and logs each decision in the Agent Trace:

1. **PLAN** – Logs that it will scan and fix. (Hard-coded; no real planning.)
2. **ANALYZE** (`analyze`) – Decides *what's wrong*. Asks Gemini for a JSON list of `{type, severity, msg}`. If the call fails or the output breaks the format, it falls back to 3 heuristics: `print(`, bare `except:`, `TODO`.
3. **ACT** (`propose_fix`) – Decides *how to change it*. Asks Gemini for a rewrite ("preserve behavior", "smallest changes"). If the output is empty, not valid Python, or deletes a function, it falls back to the heuristic fixer (print → logging.info, except: → except Exception as e:).
4. **TEST** (`risk_assessor.py`) – Decides *whether it's safe*. Starts at 100 and subtracts points for issue severity, much shorter code, removed returns, and a modified bare except.
5. **REFLECT** – Auto-fix only if risk is LOW **and** the fix changes ≤ 3 lines. Otherwise: human review.

**Heuristics vs Gemini:** Heuristics run when there is no API, when the API fails, when Gemini's output is invalid, or when Gemini's fix is rejected. Risk scoring is always local rules — the AI never grades its own fix.

**Note:** In the app, "Heuristic only" mode uses `MockClient` (a fake AI), not "no client". So the trace says "Using LLM analyzer" and then falls back.

---

## 3) Inputs and outputs

**Inputs:** The 4 `sample_code/` files (short functions with prints, bare excepts, TODOs, an unclosed file, and one clean file), plus a weird case: `except: pass` on one line.

**Outputs:**

| Input | Mode | Issues | Risk | Auto-fix |
|---|---|---|---|---|
| mixed_issues.py | Heuristic | print Low, bare except High, TODO Medium | HIGH (0) | NO |
| mixed_issues.py | Gemini | TODO Low, print Low, bare except High | MEDIUM (45) | NO |
| flaky_try_except.py | Gemini | Resource Leak Medium, Bare Except High | HIGH (35) | NO |
| print_spam.py | Gemini | print Low | LOW (95) | NO (after Part 3 rule; was YES) |
| cleanish.py | Heuristic | none | LOW (100) | YES (code unchanged) |
| `except: pass` (one line) | Heuristic | **none – missed** | LOW (100) | YES |

**Fixes proposed:** Heuristic fixes are tiny find-and-replace edits. Gemini fixes are smarter but bigger (e.g. `with open(...)`, catching `ZeroDivisionError`, adding type checks).

---

## 4) Reliability and safety rules

**Rule A – Severity deductions (High −40, Medium −20, Low −5)**
- *Checks:* How serious the detected issues are.
- *Why it matters:* Serious issues need bigger edits, which are riskier.
- *False positive:* A heuristic false alarm (e.g. `TODO_LIST = []` flagged as a TODO) lowers the score for nothing. Also, when Gemini found *more* real issues in flaky_try_except.py, the score got *worse* (55 → 35) even though its fix was good.
- *False negative:* It scores the *original code*, not the *fix*. A Low-only file starts at 95 no matter how much the AI rewrote. Before Part 2, a made-up severity like "Critical" cost **0 points**.

**Rule B – "Return statements may have been removed" (−30)**
- *Checks:* The word `return` is in the original but not in the fix.
- *Why it matters:* Dropping a return silently changes what a function gives back.
- *False positive:* The word "return" in a comment or string still counts; a valid refactor to `yield` gets penalized.
- *False negative:* Changing `return x / y` to `return 0`, or deleting 2 of 3 returns, passes — "return" still appears.

**Rule C (added in Part 3) – Auto-fix only if the fix changes ≤ 3 lines**
- *Checks:* Size of the diff.
- *Why it matters:* Behavior changes hide in bigger diffs.
- *False positive:* A safe rename across many lines goes to a human.
- *False negative:* A one-line change can still be harmful.

---

## 5) Observed failure modes

1. **Missed issue (false negative):** `except: pass` on one line. The heuristic only matches `except:` followed by a new line, so it found 0 issues, scored 100, and approved auto-fix on code that swallows every error.

2. **Destructive fix (over-editing):** In Heuristic mode, MockClient's fixer returned one comment line (`# MockClient: no rewrite available...`). The agent accepted it as the fix and deleted the whole `compute_ratio` function. *(Fixed by the Part 4 guardrail.)*

3. **Risky Gemini fix:** On mixed_issues.py, Gemini deleted the TODO comment marked "intentional", added an `isinstance` check, and changed behavior: `compute_ratio("a", 2)` used to return 0, now it raises `TypeError`. The risk report didn't notice.

4. **Incomplete + behavior-changing Gemini fix:** On print_spam.py, Gemini changed only 1 of 3 prints, and used `logging.debug`, which is hidden by default — so `verbose=True` stopped printing anything. The old scorer rated it LOW / auto-fix YES. *(Blocked by the Part 3 rule.)*

5. **Silent API failure:** When the Gemini model name was wrong (404) or the server was busy (503), `GeminiClient` hid the error and returned "". The trace said "not parseable JSON → falling back", so the user thought Gemini had analyzed the code when it never ran. The app's "API Request Failed" warning never showed.

---

## 6) Heuristic vs Gemini comparison

- **Gemini found what heuristics can't:** the unclosed file in flaky_try_except.py, and *why* a bare except is dangerous (hides NameError/TypeError).
- **Heuristics caught consistently:** print, multi-line bare `except:`, TODO — every time, same result. But they also misfire on strings and names and miss `except: pass`.
- **Fixes differed:** Heuristic fixes are tiny and mechanical. Gemini's are smarter but often bigger than asked and sometimes change behavior.
- **Did the scorer match my intuition?** Not always. It blocked bad fixes on high-severity files, but would have auto-applied Gemini's broken print_spam.py fix (score 95) until the Part 3 rule was added. It also punished Gemini for finding more real issues.

---

## 7) Human-in-the-loop decision

**Scenario:** The fix changes what a function returns or raises — like Gemini turning `except: return 0` into code that raises `TypeError`.

- **Trigger:** Compare the `return` and `raise` statements of each function (using `ast`) in the original vs the fix. If they differ, set `should_autofix = False`.
- **Where:** In `risk_assessor.py`, next to the other safety rules, so it can be unit-tested without the UI.
- **Message:** "⚠️ This fix changes what `compute_ratio` returns or raises when something goes wrong. Code that calls it may break. Please review before applying."

---

## 8) Improvement idea

**Make API failures visible.** Change `GeminiClient.complete` to raise the error instead of returning "". The agent already catches exceptions and logs "API Error: …", and the app already shows an "API Request Failed" warning for that — they just never trigger today. This is a ~2-line change, and it makes the agent honest: users will know when Gemini didn't actually run. Add one test with a fake client that raises an error and check that the log contains "API Error".