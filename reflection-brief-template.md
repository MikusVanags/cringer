# Reflection Brief — Harness Engineering Capstone

**Name:** Claude AI Agent
**Date:** 2026-11-23

**Every answer cites at least one artifact from these runs:**
- System 1 run: `20261123_085906`
- System 2 run: `20261123-090145`
- System 3: validator output `OK`
- System 4: `hot_state.json` (643 bytes)

**Environment**

- Model(s): claude-haiku-4-5-20251001 (Systems 1–2, 4), no API calls needed (System 3)
- OS / Python: Linux, Python 3.13.18
- Approx. API spend: $0.1149 (System 1) + ~$0.05 (System 2 estimated) + $0 (System 3) + $0 (System 4, used recorded response)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → From trace `claim_02_stolen_bike.jsonl` (run 20261123_085906): `tool_use, tool_use, tool_use, tool_use, end_turn`. The function `run()` in `loop.py:46–132` decides this via lines 103 and 113: if `response.stop_reason == "end_turn"` it returns, if `response.stop_reason == "tool_use"` it continues with `continue`, else it raises `UnexpectedStopReason`. This contract is documented in the module docstring: "Control flow is driven by `response.stop_reason`; loop continues iff stop_reason == 'tool_use'; loop returns iff stop_reason == 'end_turn'."

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → The test `test_no_string_membership_against_text_in_loop()` checks for string-membership tests (`"token" in response_text`) driving control flow. If the loop used this — e.g., `if "done" in final_response` — claims like `claim_02_stolen_bike` would route to the wrong queue if the agent's natural language happened to contain the magic keyword. The anti-pattern couples business logic to fragile NLP heuristics instead of the model's intentional structured stop_reason signal.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `request_clarification` and `route_to_adjuster` both end a phase of fact-gathering but with different intent. The `request_clarification` description in `tools.py:107–113` explicitly says "Use only when the claim type is genuinely ambiguous," while `route_to_adjuster` says "Call this exactly once when classification confidence is at least 0.6." A structured tool error from routing before classifying (line 294) returns `{"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "classify_claim must be called before routing"}` — the agent sees this is unrecoverable and must backtrack, not silently fail or retry with the same bad input.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → Claim `claim_02_stolen_bike`: 5 turns, $0.0211 estimated cost (from summary table, run 20261123_085906). The README doesn't provide sample numbers for routed claims, only the note that "every claim terminates (routed or escalated)." This run shows variance: `claim_01_kitchen_fire` took only 2 turns and escalated (possibly due to incomplete coverage), while `claim_02_stolen_bike` needed 5 turns but routed with higher confidence. The cost scales with turns and token volume, not claim complexity.

### System 2 — Context strategy

5. **The reduction.** From `budget.json` (run 20261123-090145): baseline 38,708 tokens, assembled 16,861 tokens, 56.44% reduction. The `active` section dominates at 15,789 tokens (93.6% of assembled), and it is kept byte-exact (verbatim from the transcript) because it is the live conversation with the customer — the model must see the exact words to reason about context switching, contradictions, and tone. The case_facts block (204 tokens) is extracted by the LLM and preserved in full; resolved segments are compressed from ~24k to 860 tokens combined.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Resolved (past) segments get summarized: `resolved_refund` compressed from 12,334 input to 408 output tokens; `resolved_subscription` from 11,475 to 452. The active segment stays byte-exact at 15,789 tokens (the customer's current context). The rule is: closed/historical conversation segments are summarized via the compression prompt to extract key facts; the ongoing conversation stays verbatim because the model needs to track discourse state, pronouns, and negotiation history in real time.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → Q6 regressed in `eval_control.jsonl` (run 20261123-090145): "What is the structured status of the payment-method update issue?" When case facts are stripped (control variant), the model cannot reliably answer; it returns "unknown" instead of the exact status token from the case record. This proves the case facts block is essential: it provides the structured metadata that the compressed active transcript alone cannot answer. Q1 passed in both (refund amount), suggesting simple facts survive context compression, but status codes don't — only the facts block preserves them reliably.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → From `.claude/rules/react.md` (System 3 solution, validator output `OK`): `paths: ["src/components/**/*", "src/pages/**/*"]`. This is better than a directory-level CLAUDE.md because (1) it activates only when editing matching files — the React rules do not clutter sessions focused on API or database work; (2) it unifies cross-cutting conventions (like "tests are co-located") across multiple directories with a single glob, whereas separate `src/components/CLAUDE.md` and `src/pages/CLAUDE.md` files would duplicate the same guidance; (3) the glob pattern is explicit and version-controlled, so the team knows exactly when each rule applies.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → From `.claude/skills/deploy-check/SKILL.md` (System 3, validator passed): `context: fork` and `allowed-tools: [Read, Grep, Glob, Bash(git *), Bash(gh pr *)]` (read-only git/gh commands only). Running forked + read-only keeps verbose discovery output (file enumerations, diffs, `git status` logs) out of the main session — only the structured pass/fail summary returns. Without forking, megabytes of intermediate analysis would pollute the caller's context window. Without the read-only allowlist, a future maintainer could accidentally add Write or Bash(deploy) and silently ship bad code.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → From `CLAUDE.md` (System 3, validator output `OK`): **Project-level** example: `@.claude/standards/frontend.md` applies to all teammates editing React, enforced via git; **User-level** example: `~/.claude/commands/my-team-morning-standup` (not in this repo, stays on your laptop) would be a personal command visible only in your session, never committed. The validator confirms this hierarchy is respected: shared standards go in `.claude/standards/` and `.claude/rules/`, while personal preferences belong in `~/.claude/`.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → System 4 (run with recorded response, `hot_state.json` 643 bytes): The indexed query is `defects_since(since_ts)` in `warm.py:75–86`, which runs `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT 50`, using the `idx_defects_ts` index (line 20). This returns only new defects *since the last run's timestamp*, not the full table. The model never sees the full history because (1) the query filters to recent data only, (2) it's capped at 50 rows, and (3) the warm tier intentionally de-prioritizes old defects — if something hasn't been mentioned in 30 minutes, the cold tier (monthly summaries) holds it instead, and the model only gets injected historical summaries in the prompt, not raw rows.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → From `recovery.py:16` and `decide()`: `STALE_RESUME_THRESHOLD_MINUTES = 30`. If the last step was ≤30 minutes ago, resume from the manifest; if >30 minutes or the manifest is complete, start fresh with injected summary. A fresh start is more reliable after staleness because the working memory in the model's context (tone, discourse state, partial hypotheses) becomes stale and misleading. A fresh start forces re-analysis with the same data plus an injected summary of prior findings, preventing the model from second-guessing obsolete context and re-investigating the same edge cases.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matters for a system run once per shift, indefinitely?
    → System 4 output (run timestamp 2026-11-23T09:03:52.602031Z): `hot_state.json` is 643 bytes, well under the ~5 KB target mentioned in the README. This budget matters for indefinite operation because the hot state is loaded into memory on every shift run and serialized to disk after each invocation. If it grew to megabytes over months of shifts, (1) startup latency would creep up, (2) the state tree would become difficult to reason about, (3) drift between hot and warm tiers would compound. By keeping hot_state tiny, the system remains predictable and efficient across years of continuous operation.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → **Model:** System 1 `tools.py:30–191` — the tool schemas and descriptions define the interface between agent and harness. The model's behavior is constrained by the enum values, required fields, and human-readable guidance; nothing in the loop forces a particular choice.
    → **Harness:** System 1 `loop.py:46–132` — the message loop, budget checks, tracer, tool dispatch. This layer enforces the stop_reason contract, catches errors, and ensures the agent never escapes control.
    → **Orchestration:** System 4 `recovery.py:21–29` and `warm.py:75–86` — the multi-tier state machine and indexed queries push work down to the SQL layer, freeing the model from full-history reasoning. The manifest (System 4 `manifest.py`) tracks invocation steps across crash boundaries.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → **Guaranteed in code:** System 1 tool validation at `tools.py:294–297` — if `classify_claim` has not been called, `route_to_adjuster` returns a permanent error. This boundary is non-negotiable and prevents bad states from ever reaching the backend queue. System 3 `allowed-tools: [Read, Grep, Glob, Bash(git *)]` in deploy-check ensures no Write tool is available, so a future maintainer cannot accidentally enable destructive operations. **Guided by prompt:** System 1's system prompt describes the claim classification strategy and when to escalate; the model learns the judgment call. When is each right? Code-enforced boundaries are for safety (data integrity, permission boundaries, state machine invariants); prompts are for judgment (strategy, tone, reasoning steps). A claim cannot be routed without classification — that's code. But *how* to classify a bike theft vs. liability — that's the model's job.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 compresses a single long conversation from 38,708 tokens to 16,861 (56.44% reduction, `budget.json` run 20261123-090145) by summarizing resolved segments and keeping active verbatim. System 4 reduces cross-session history from megabytes of monthly cold storage to a 643-byte hot state + ~50 recent defects from the warm tier. Both apply the same principle: *recent/active context stays intact for reasoning, historical context is summarized and indexed*. System 2 uses Claude's compression API to extract key facts; System 4 uses SQL indices to skip irrelevant rows. Same principle (recency + compression), different scale (single session vs. months).

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → System 1 `test_antipatterns.py:test_no_string_membership_against_text_in_loop()` guarantees the loop never uses `"done" in response_text` to decide control flow. A single successful run of `claim_02_stolen_bike` passes, so you'd assume the loop is correct. But if the agent's response happened to contain "done" accidentally, the loop would break. The test makes the absence of this brittle heuristic explicit, catching it before a future refactoring introduces it. System 4 `test_us03_crash_recovery.py:test_recovery_decide_truth_table()` tests all 6 combinations of (time_elapsed, is_complete) → decision. One run uses only one combination; the test ensures all are correct before a rare edge case (a crash at the 30-minute boundary) causes production failure.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → **System 1 — Insurance Claims Intake.** Blast radius: a misconfigured model could route claims to the wrong queue (e.g., auto claims → theft queue) or escalate low-confidence claims incorrectly, impacting claims processing SLAs and customer satisfaction. Kill switch: the `route_to_adjuster` tool (`tools.py:285–307`) is the terminal, and it validates state: if `classification is None`, it returns a permanent error (line 295). Enforcement: a claim enters the trace only if both `classify_claim` and `assess_severity` have been called; the dispatcher prevents the agent from skipping them. State: each claim session is isolated (no state leakage across claims), so even if one claim goes to the wrong queue, it doesn't corrupt the next one.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → Nothing broke in the actual system runs (all 4 systems passed tests and executed end-to-end). What I checked to be sure: (1) Python version compatibility — confirmed System 1 uses `anthropic==0.39.0` while System 2 uses a later version, so using the shared `py313-venv` with its single pinned `anthropic` version could cause issues, but each system's `pyproject.toml` specifies its own requirements and they all work with the shared environment; (2) API key availability — confirmed `ANTHROPIC_API_KEY` was set before running Systems 1, 2, and 4; (3) initial dependency conflicts (awswrangler, matplotlib, scipy version mismatches in the system Python), but these didn't affect the tests or runs.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I'd expand System 4's warm-tier `defects_since` query to include a `severity` and `component` filter at the SQL layer instead of returning 50 raw rows and expecting the model's prompt to ignore low-severity or already-known components. Observation: the query returns all defects after a timestamp, but the model's reasoning loop is expensive (each invocation calls Claude). A SQL-layer filter (`WHERE ts > ? AND severity IN ('high', 'medium')`) would reduce the prompt size and let the model focus on signal. The tradeoff: loss of generality (the query becomes domain-specific) vs. savings in token volume and latency. For a system running indefinitely, the per-shift savings would compound.
