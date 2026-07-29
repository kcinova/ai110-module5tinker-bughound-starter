# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

BugHound runs a small agent loop with five steps: PLAN (decide to scan and propose a fix), ANALYZE (find issues), ACT (propose a fix), TEST (score the risk), and REFLECT (decide whether to auto-apply or defer to a human). In heuristic mode it uses simple pattern rules to find issues like print statements, bare except clauses, and TODOs, and applies rule-based fixes. In Gemini mode it sends the code to the Gemini model to analyze and propose fixes. Either way, the risk assessor (a local rule-based layer) always runs before deciding whether the fix is safe to auto-apply. If the AI fails, the agent falls back to heuristics.

---

## 3) Inputs and outputs

Inputs: I tested short Python snippets from the sample_code folder, including a file with print statements (print_spam.py), a file with a bare try/except (flaky_try_except.py), a file with multiple mixed issues (mixed_issues.py), and edge cases like empty input and a comments-only file.

Outputs: Detected issues were labeled by type and severity (e.g. "Reliability | High" for a bare except, "Code Quality | Low" for print statements). Fixes were either heuristic (e.g. replacing print with logging.info) or AI-generated. The risk report showed a level (low/medium/high), a score (0-100), an auto-fix decision, and a list of reasons.

---

## 4) Reliability and safety rules

Rule 1: High-severity penalty. The rule subtracts a large amount from the score when a High-severity issue is detected. This matters because high-severity issues (like a bare except) are the most likely to hide real bugs, so the agent should be cautious. False positive: it could over-penalize a bare except that was actually intentional. False negative: it relies on the analyzer correctly labeling severity, so if severity is mislabeled as Low, this rule won't trigger.

Rule 2: "Code much shorter than original" penalty. The rule subtracts points if the fixed code is significantly shorter than the input, since that often means the fix deleted logic. This matters because a fix that silently removes code can change behavior. False positive: a legitimate fix that removes genuinely dead code would look risky. False negative: a fix that adds a subtle bug without changing length would not be caught.
---

## 5) Observed failure modes

1. Deprecated model failure: When I first ran Gemini mode, the API returned a 404 because the code requested a model (gemini-2.5-flash) that Google no longer offers to new users. The agent did not crash — it caught the error, logged the real cause, and fell back to heuristics. This showed the reliability layer working, but it also showed the system can silently degrade to rule-based analysis if the model name is out of date.

2. Over-confident on trivial input: When I gave BugHound a comments-only file (no real code), it found zero issues, scored 100, and reported Auto-fix: YES. This is wrong — the agent claimed it was safe to auto-apply a fix to something with nothing to fix. I added a guardrail so the agent will not auto-fix when there is no real executable code.

---

## 6) Heuristic vs Gemini comparison

On the same print-statement file, heuristic mode labeled the issue "Code Quality | Low" while Gemini labeled it "readability | Low" with a slightly different explanation. On the bare-except file, Gemini caught an extra issue the heuristics missed entirely — a file being opened without a "with" statement (a maintainability risk) — showing the AI reasons about the code more broadly than fixed rules. Heuristics were consistent and predictable but limited to their hardcoded patterns. The risk scorer generally agreed with my intuition: it was most cautious about the bare-except (high severity) and least cautious about print statements.

---

## 7) Human-in-the-loop decision

BugHound should refuse to auto-fix when a High-severity issue is present, even if the overall score looks low. I implemented this trigger in risk_assessor.py by requiring both that the risk level is "low" AND that no High-severity issue exists before allowing auto-fix. I put it in the risk assessor because that is the single place where the auto-fix decision is made, so the guardrail applies no matter how the fix was generated. The tool should show the user a message like: "This fix touches a high-severity issue and needs human review before it can be applied."

---

## 8) Improvement idea

Add a "minimal diff" guardrail: if a proposed fix changes more than a set percentage of the original lines, automatically lower the risk score and require human review. This is low-complexity (a line-count comparison plus one rule) but directly targets the most common failure I saw, which was fixes that over-edited or shortened the code. It would be backed by a test using the MockClient so it runs offline.
