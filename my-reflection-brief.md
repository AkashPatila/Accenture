# Reflection Brief — Harness Engineering Capstone

**Name:*Akash Malagoud Patil*
**Date:*2026-10-30*

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s):
- OS / Python:
- Approx. API spend:

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   →  In claims_intake/loop.py, the stop_reason property from the Anthropic API response is checked to determine whether to continue the agentic loop or stop for clarifications.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   →  test_antipatterns.py checks for unbounded recursive tool execution without termination checks; if used, an infinite loop would consume tokens indefinitely.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   →  Specialized schema tools with strict parameter typing prevent invalid tool inputs, whereas a generic string prompt would allow unstructured hallucinated arguments.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   →  For claim claim_01_kitchen_fire, turn count was 2 clarifications = 0, tokens = 6267/418, estimated cost = $0.0084 (matching summary.md trace outputs).

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   →  From budget.json, conversation history and bulky raw transcripts dominate the assembled context; keeping recent turns verbatim preserves immediate user intent while older turns are summarized.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → System messages and active customer parameters are kept byte-exact, whereas resolved historical dialog turns are summarized to fit token windows.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → Comparing eval.jsonl and eval.control.jsonl, strict card-verification questions regressed under compression when specific trailing digits were truncated.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → Path-scoped frontmatter rules (path: "api/*/.py") target specific subsystems cleanly without cluttering global context like a directory-level CLAUDE.md does.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → context: fork and allowed-tools: [View, Grep] run tasks in an isolated subprocess with read-only safety, preventing accidental modifications to production assets.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → Project-level scope is defined in .claude/ rules, whereas user-level scope resides in global configuration settings.
    System 4 (Orchestration & Shift Monitor)

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The SQLite warm-tier queries filter raw defects via indexed SQL (warm.sqlite), ensuring Claude only sees synthesized anomaly summaries rather than millions of raw log rows.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → The staleness threshold (recovery.py) triggers a fresh state reload when checkpoint files exceed time-to-live limits, preventing cascading failures from corrupted state deltas.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → hot_state.json byte size stays minimal (~643 bytes), ensuring predictable memory limits across indefinite recurring shift executions.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model:Anthropic API client
    → Harness:Pydantic validation loops & state guards
    → Orchestration:Multi-shift warm store SQLite database.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → * Code-guaranteed: The use of restricted tool definitions and read-only allowlists in System 3 (.claude/skills/deploy-check/SKILL.md) guarantees safety by programmatically blocking unauthorized shell commands.
     * Prompt-guided: Shaping the agent's reasoning style or persona instructions (e.g., tone or decomposition style in System 1's dynamic prompt).
      * When each is right: Use code guarantees for security boundaries, data integrity, and strict API constraints where failure is unacceptable; use prompt guidance for creative synthesis, formatting styles, and flexible contextual reasoning.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → * System 2 (Intra-session): Manages local memory growth within a single conversation thread using window trimming and token budgeting (budget.json), ensuring historical turns fit context limits without losing immediate parameters.
     * System 4 (Cross-session): Manages long-term persistence across independent runs using an external SQLite warm store and lightweight state files (hot_state.json ~643 bytes).
      * The mechanism: Both preserve core context under tight limits, but System 2 compresses dialogue history, whereas System 4 offloads state into external structured storage to prevent cross-session memory bloat.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → * Guaranteed behavior: Automated test suites (such as the 33 passing tests in System 4's test_crash_recovery.py) verify edge cases like atomic write failures, missing state recovery, and malformed JSON payloads.
     * Why it matters: A single manual run might succeed under "happy path" conditions, but only a comprehensive test suite ensures the system gracefully handles concurrency, network drops, or corrupted state files before production deployment.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    →  * System chosen: System 1 (Claims Intake Agent).
     * Blast radius: If the agent loop misbehaves or enters an infinite recursive tool-call loop, it could flood the Anthropic API with unnecessary tokens, exhaust rate limits, and incur excessive financial costs.
      * Kill switch & enforcement: Controlled programmatically via explicit loop bounds, token cost monitoring (summary.md total cost caps), and explicit stop_reason termination checks in loop.py.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The initial httpx/proxies compatibility error due to version mismatches, fixed by pinning httpx==0.27.2 and anthropic==0.39.0.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → Architectural change: Instead of relying on manual file-copying scripts to export configuration files across systems, I would implement a unified CLI utility or automated setup script that automatically synchronizes .claude/ rules and packages evidence.
    Grounding: During execution, manual directory navigation (cd) and separate virtual environment activations across all four systems introduced dependency version conflicts (such as the httpx/anthropic client parameter mismatch). Centralizing environment orchestration and artifact bundling under a single runner would make the harness far more resilient and repeatable before production.
