# Reflection Brief — Harness Engineering Capstone

**Name:** Ruslan Marchanka
**Date:** 2026-10-27

Replace each `→` with your answer. Every answer cites artifacts from these runs. The S1 run completed, but its outcome distribution differed from the course example; that deviation is recorded rather than normalized away.

**Environment**

- Model(s): Claude Haiku 4.5 (`claude-haiku-4-5-20251001`)
- OS / Python: Windows 11 / Python 3.14.6
- Approx. API spend: ~$0.31 estimated from the S1/S2 generated-call usage; S3 and the recorded-response S4 run made no model-generation calls.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In `system1/runs/20261027_151442/traces/claim_04_neighbor_injury.jsonl`, the sequence is `tool_use → tool_use → tool_use → tool_use → end_turn`. `claims_intake/loop.py:run` continues after `tool_use`, returns on `end_turn`, and raises `UnexpectedStopReason` for any other value. The final turn count was five, including four tool-use turns.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `test_no_integer_literal_iteration_cap_in_loop` in `tests/test_antipatterns.py` rejects a fixed integer loop cap. Claim 04 needed four tool-use turns before it routed, as shown in its trace; a low hard-coded cap could stop the loop before `route_to_adjuster` ran. The S1 test log records 30 passing tests, including the anti-pattern checks (`system1/pytest.log`).

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `classify_claim` and `route_to_adjuster` both refer to the claim type, but the first commits to `claim_type` plus confidence and rationale, while the terminal tool accepts a queue only after classification and severity. The descriptions distinguish a decision from the subsequent terminal routing action (`claims_intake/tools.py`). The executor returns structured errors with `error_category` and `is_retryable`, allowing a caller to distinguish a transient failure worth retrying from a permanent precondition error; the graceful-error behavior is covered in `system1/pytest.log`.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → Claim 04 routed in 5 turns at an estimated `$0.0221`; the whole eight-claim run cost an estimated `$0.1049` (`system1/runs/20261027_151442/summary.md`). The exercise README's example describes 7 routed and 1 escalated at roughly `$0.05`; this run instead had 3 routed and 5 escalated. Five claims ended with `end_turn` without a terminal tool, so the deterministic backstop escalated them; claim 03 also did not ask for clarification.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → `system2/runs/20261027-151722/budget.json` records a 38,708-token baseline, 16,775 assembled tokens, and a 56.66% reduction. The active section dominates at 15,789 tokens, about 94% of the assembled context. It remains verbatim because the payment-method issue is unresolved and the exact dialogue may still affect the next decision.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The two resolved sections were summarized: refund is 370 tokens and subscription is 430; the active issue stays byte-exact at 15,789 tokens (`system2/runs/20261027-151722/budget.json`). The 204-token case-facts block preserves IDs, amounts, and status tokens needed after compression. The rule is implemented by `retail_context/compressor.py:summarize_segment` refusing to summarize an active segment, then `assemble.py:build` places the active dialogue after the resolved summaries.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → The full context passed 6/6 questions (`system2/runs/20261027-151722/eval.jsonl`). In the control variant, Q1 still passed because the refund amount also appeared in the resolved summary, but Q6 failed because the exact `in_progress` token was absent (`system2/runs/20261027-151722/eval_control.jsonl`). That regression shows the case-facts block carries structured state that the narrative alone did not preserve.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → `system3/config/.claude/rules/tests.md` declares `paths: ["**/*.test.tsx", "**/*.test.ts"]`. Those patterns apply to matching tests across the repository rather than only files below one directory. The copied configuration plus the successful validator output are in `system3/config/` and `system3/validator.log`.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → `system3/config/.claude/skills/deploy-check/SKILL.md` declares `context: fork` and an `allowed-tools` list limited to `Read`, `Grep`, `Glob`, and read-oriented Git/GitHub commands. A fork keeps the lengthy deployment-check investigation out of the main conversation, and the read-only allowlist prevents the check from changing the working tree. Without those boundaries, an inspection workflow could pollute the main session or mutate code while it is supposed to be reporting status; `system3/validator.log` records `OK`.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → `system3/validator.log` reports `OK`, and `system3/config/CLAUDE.md` documents the scopes. The project-level example is the committed `./CLAUDE.md` with imports from `.claude/standards/`; a user-level example is a personal `~/.claude/CLAUDE.md`, which is not shared with teammates through this repository.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → `system4/seed.log` records 40 rows in the warm tier and 17 returned for the requested time window; SQLite's plan uses `idx_defects_ts`. `WarmStore.defects_since` runs `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` (`shift_monitor/warm.py`). The pipeline passes only those 17 rows with hot state to the model, so SQL performs temporal filtering before context construction.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `shift_monitor/recovery.py:decide` resumes an incomplete manifest when its last step is no more than 30 minutes old; an empty or complete manifest, or an older incomplete one, starts fresh. A fresh invocation with captured findings summarized into its prompt avoids depending on stale or partially applied state while retaining known results. The 30-minute boundary and resumed-prompt behavior are covered by the 33 passing S4 tests (`system4/pytest.log`). The CLI run logged in `system4/run_shift.log` takes the normal fresh shift path; recovery and forking are exercised in the test suite rather than wired into that CLI command.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → `system4/state_metrics.txt` records a 965-byte `hot_state.json`, under the 5,120-byte budget enforced by `HotState.write_atomic` in `shift_monitor/state.py`. A bounded hot tier prevents prompt size from growing with every shift and leaves the historical record in warm/cold storage. The offline shift produced one scratchpad entry and the 17-defect summary (`system4/data/shift_scratchpad.jsonl`, `system4/run_shift.log`).

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `claims_intake/loop.py` sends messages to Claude and dispatches the returned tool calls; the observed turn sequence is in `system1/runs/20261027_151442/traces/claim_04_neighbor_injury.jsonl`. Harness: `system3/config/CLAUDE.md` and `.claude/rules/` provide project instructions and path-scoped conventions, validated by `system3/validator.log`. Orchestration: `shift_monitor/pipeline.py:run_shift` combines persisted hot state with a SQL-filtered warm-tier slice and writes the next state; its observed run is in `system4/run_shift.log`.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → The S1 tools enforce terminal preconditions and reject a second terminal action in `claims_intake/tools.py`; `system1/pytest.log` records those tests passing. By contrast, `claims_intake/system_prompt.py` asks the model to route at confidence ≥0.6, but the prompt cannot force a terminal tool call: in my run, five claims ended in the deterministic backstop (`system1/runs/20261027_151442/summary.md`). Use code for invariants and side-effect boundaries; use prompts for judgment that depends on context.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → S2 reduced one conversation from 38,708 to 16,775 tokens (56.66%) by summarizing resolved issues while preserving the active issue (`system2/runs/20261027-151722/budget.json`). S4 selected 17 of 40 defects for a shift and retained a 965-byte hot state under a 5,120-byte budget (`system4/seed.log`, `system4/state_metrics.txt`). Both keep decision-relevant context small, but S2 compresses within a conversation while S4 filters and bounds state across shift sessions.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → `test_loop_raises_on_unexpected_stop_reason` verifies that an unrecognized model stop reason raises instead of silently continuing (`system1/pytest.log`). My successful trace only showed expected `tool_use` and `end_turn` values, so it could not establish that the exceptional path was safe. This guard matters because an unforeseen termination mode must not be mistaken for a completed claim.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In S1, a bad decision can put one claimant's record into the wrong queue or escalate it unnecessarily; the run summary shows five escalations in this run (`system1/runs/20261027_151442/summary.md`). `route_to_adjuster` validates classification and severity, terminal actions are single-use, and `Budget` caps input tokens and wall-clock time (`claims_intake/tools.py`, `claims_intake/budget.py`). There is no application-level emergency-stop command, so the operator's immediate kill switch is to stop the process and revoke the API credential; the budgets bound normal runaway work but do not replace that switch.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The first S1/S4 pytest attempts hit `PermissionError` while pytest tried to create its default directory under Windows `%TEMP%`. I reran them with a writable, isolated `--basetemp` under `%USERPROFILE%\.venvs`; the resulting logs show 30/30 S1 and 33/33 S4 tests passing (`system1/pytest.log`, `system4/pytest.log`). I also set `PYTHONUTF8=1` so the Windows console preserved Unicode output.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would replace S2's substring-only evaluation with a rubric that checks the requested fact and its polarity. Q5 was marked `PASS` because the answer contained “prorated,” yet it first said the customer had received the refund and later said it had not posted (`system2/runs/20261027-151722/eval.jsonl`). That contradiction shows the current evaluator can reward a semantically wrong answer, so I would require consistency checks around the actual question rather than treating one matching fragment as sufficient.
