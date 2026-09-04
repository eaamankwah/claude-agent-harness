# Reflection Brief — Harness Engineering Capstone

**Name:**   Edward Amankwah

**Date:** August 11, 2026

**Environment**

- Model(s): claude-haiku-4-5-20251001 (default, Systems 1 & 2), claude-sonnet-4-5-20250929 (System 1 two-model comparison)
- OS / Python: Linux (Ubuntu), Python 3.13 (student workspace)
- Working directory root: `/workspace/` — all four systems' `solution/` directories, and their `runs/`, `data/`, and `.claude/` artifacts, are cited below with full paths rooted at `/workspace/` to match this environment (e.g. `/workspace/Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20260810_024505/`).
- Approx. API spend: System 1: $0.1202 (Haiku run `runs/20260810_024505`) + $0.3832 (Sonnet run `runs/20260810_025046`) = $0.5034. System 2: compression API calls ~12,334 + 11,475 input tokens (refund + subscription compressions, run `runs/20260810-030459`) plus 6 eval questions x 2 (main + control) ≈ under $0.10. Total capstone spend well under $1.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

**Response 1**

   → From `/workspace/Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20260810_024505/traces/claim_03_water_damage.jsonl`, the 7-turn sequence is: `tool_use, tool_use, tool_use, tool_use, tool_use, tool_use, end_turn` — six tool-calling turns (`lookup_policy` → `record_claim_fact` x5 → `request_clarification` → `record_claim_fact` → `classify_claim`+`assess_severity` → `route_to_adjuster`) followed by a clean `end_turn`. The decision lives in `claims_intake/loop.py::run()`: the `while True:` loop (line 68) checks `response.stop_reason` after every API call — `if response.stop_reason == "end_turn":` returns a `FinalState` and exits (line 103); `if response.stop_reason == "tool_use":` executes the requested tools, appends the results, and `continue`s (line 113); any other value raises `UnexpectedStopReason` (line 130). The function's own docstring states this directly: "The loop condition is `response.stop_reason == "tool_use"` — nothing else. There is no `while turn < N` cap. The Budget object is the safety mechanism."

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

**Response 2**

   → `test_no_integer_literal_iteration_cap_in_loop` statically AST-walks `loop.py` to confirm there is no `for _ in range(<int literal>)` or `while <var> < <int literal>` construct capping iterations. If the loop instead used a hard-coded cap like `while turn < 5`, claim_03_water_damage which legitimately needed 7 turns to route correctly in my run — would have been cut off mid-investigation at turn 5, right after the `request_clarification` call and before the classification/routing tools ever ran, silently producing an incomplete claim instead of a correctly routed one.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

**Response 3**

   → `lookup_policy` and `record_claim_fact` both take string inputs, but their descriptions steer the model to different points in the workflow: `lookup_policy`'s description says to "Use this early in the conversation to confirm the policy exists," while `record_claim_fact`'s covers "one normalized fact extracted from the claimant's statements (e.g., incident_date, location, description...)." In practice this ordering held exactly in my trace, turn 1 was `lookup_policy`, turns 2 and 4 were `record_claim_fact`. Every tool error is returned as structured JSON via `_err(category, retryable, message)` E.g. `{"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "..."}` rather than a free-text string. This gave the agent a machine-readable `error_category`/`is_retryable` pair to condition its next action on (retry vs. try a different tool vs. surface the problem to the claimant), instead of having to parse an ambiguous natural-language error message to infer the same thing.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

**Response 4**

   → `claim_03_water_damage` took 7 turns and cost $0.0325 on Haiku (run 20260810_024505) vs. only 4 turns and $0.0536 on Sonnet (run 20260810_025046) for the same fixture Sonnet actually took fewer turns but cost more per turn (larger per-token price) and, notably, its outcome was `incomplete` rather than `routed`, meaning it asked one clarification and stopped short of the six-tool sequence Haiku completed. This shows turn count alone doesn't predict cost or outcome quality. The model choice changed both the path taken and where the claim landed.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

**Response 5**

   → From `/workspace/Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/20260810-030459/budget.json`: `baseline_tokens: 38708`, `assembled_tokens: 16871`, `reduction_pct: 56.41`. The active section dominates at 15,789 of the 16,871 assembled tokens (93.6% of the assembled total), versus `case_facts: 204`, `resolved_refund: 385`, and `resolved_subscription: 511` combined at just 1,100 tokens. The active segment is kept verbatim because it's the in-flight issue (the payment-method update, per the eval answers below) the agent needs the claimant's exact recent wording to keep responding coherently, and summarizing it risks losing details mid-negotiation that haven't been resolved yet and so can't be safely compressed into a stable summary.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

**Response 6**

   → The rule, enforced in code by `test_summarize_segment_refuses_to_compress_the_active_segment`, is: resolved issues get compressed into short summaries under a token budget (in my run, the refund issue compressed to 385 tokens and the subscription cancellation to 511 tokens, using compression API calls of 12,334 and 11,475 input tokens respectively), while the active, unresolved issue is preserved byte-exact at its full 15,789 tokens with zero compression applied. The 204-token case-facts block is also never compressed and it is generated once via structured extraction, not summarization.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

**Response 7**

   → All 6 questions pass in `eval.jsonl` (with the case-facts block present). In `eval_control.jsonl` (case-facts block removed), Q6 — "What is the structured status of the payment-method update issue (use the exact status token from the case record, not a paraphrase)?" flips from `passed: true` to `passed: false`. The control run's model answer states explicitly: *"there is no structured status token or case record explicitly documented... The agent has discussed potential next steps... but has not yet formally opened a ticket or assigned a status code... Answer: Unknown."* This proves the case-facts block is load-bearing, not redundant with the compressed transcript: the exact status token (`in_progress`) exists as a structured field in the case-facts block but is not recoverable from the compressed/active transcript text alone, even though the same underlying conversation is present in both runs.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

**Response 8**

   → From `.claude/rules/react.md`:
   ```yaml
   ---
   description: Conventions for React components and pages
   paths:
     - "src/components/**/*"
     - "src/pages/**/*"
   ---
   ```
   A directory-level `CLAUDE.md` only loads for files physically nested under that one directory. This rule needs to apply to two separate top-level trees (`src/components/**` and `src/pages/**`) that share a convention but aren't siblings under a common folder. A glob-scoped rule activates by pattern match regardless of directory structure, covering both trees from one file instead of duplicating the convention across two separate `CLAUDE.md`s that could drift apart and verified by `tests/test_us02_path_scoped_rules.py::test_ac_02_04_react_file_matches_only_react_rule`, confirmed passing in my test run (35/35 passed).

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

**Response 9**

   → From `.claude/skills/deploy-check/SKILL.md`:
   ```yaml
   context: fork
   allowed-tools:
     - Read
     - Grep
     - Glob
     - Bash(git status:*)
     - Bash(git diff:*)
     - Bash(git log:*)
     - Bash(git rev-parse:*)
     - Bash(git ls-files:*)
     - Bash(gh pr view:*)
     - Bash(gh pr checks:*)
   ```
   The skill's own rationale states it "produces verbose intermediate output (file enumeration, diff parsing, check execution traces) that has no value once the summary is known". Running forked keeps that noise out of the main session's context window and returns only the pass/fail summary. The `allowed-tools` list contains no write-capable git or gh command (no `push`, `commit`, `merge`), the skill cannot modify or deploy anything even if instructed to, because those tools aren't in its allowlist. Without the fork, every deploy check would consume main-session context proportional to repo size; without the read-only restriction, a compromised or hallucinating invocation could push or merge instead of merely reporting.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.

**Response 10**

    → The validator ran clean: `python -m ecommerce_team_config .` → `OK`, exit 0. Per `CLAUDE.md`'s own scope table: 
    
    **project-level** config lives in `./CLAUDE.md`, `.claude/standards/`, `.claude/rules/`; e.g. `.claude/rules/react.md`, versioned and shared with the whole team. 
    
    **User-level** config lives in `~/.claude/CLAUDE.md`, `~/.claude/commands/`, `~/.claude/skills/`; e.g. a personal `/morning` summary command or individual commit-message style, which CLAUDE.md states explicitly "stay on your laptop and never reach teammates."

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

**Response 11**

    → My shift run (`shift_monitor run-shift --shift C --warm-db data/warm.sqlite --recorded-response fixtures/recorded_responses/shift_C_2026-04-30.json`) returned 0 new defects for the analyzed window, against whatever larger total sits in the seeded `warm.sqlite` store. The indexed query is `WarmStore.defects_since()` in `shift_monitor/warm.py`: `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, whose docstring states "This is SQL-side filtering only — no Python-side filtering of severity, component, or time," enforced by `test_gather_new_defects_has_no_python_side_filtering`. 
    
    The model never sees the full defect history because filtering happens in SQLite before any row reaches the prompt, keeping prompt size bounded by the shift window regardless of total accumulated history.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

**Response 12**

    → `shift_monitor/recovery.py` sets `STALE_RESUME_THRESHOLD_MINUTES = 30`, confirmed by the passing test `test_threshold_constant_is_30_minutes`. 
    
    The module's reasoning: 30 minutes is roughly 1/16th of an 8-hour shift, so a resume inside that window is still operating on the same shift's working set, while anything older is treated as stale and restarted fresh with an injected summary of whatever the manifest already captured. A fresh start is more reliable past that window because resuming a genuinely stale partial risks continuing reasoning threads (open hypotheses, in-flight investigations) the model has no memory of forming a fresh, current summary avoids that risk entirely.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

**Response 13**

    → My `hot_state.json` is 643 bytes, holding `recent_defect_hashes: []`, a `current_shift_summary`, three `active_alerts`, and `threshold_statuses` (two `"ALARM"`, one `"OK"`). `test_hotstate_rejects_more_than_20_hashes` enforces a hard cap in code, not convention. Because this system runs once per shift indefinitely, unbounded hot-state growth would raise both prompt cost and read/write latency without limit over months of operation; a fixed, code-enforced budget keeps per-invocation cost constant no matter how long the system has been running.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.

**Response 14**

    → **Model:** the single `client.messages.create()` call inside `claims_intake/loop.py::run()` (System 1) — one call, one turn, no memory of its own; every trace line in `/workspace/Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20260810_024505/traces/claim_03_water_damage.jsonl` is the record of exactly one such call.

    → **Harness:** `.claude/skills/deploy-check/SKILL.md` (System 3) — `context: fork` plus the read-only `allowed-tools` list decides *how* the model is allowed to run, independent of what any single call to it says.

    → **Orchestration:** `shift_monitor/recovery.py`'s 30-minute threshold and `warm.py`'s `defects_since()` SQL filter (System 4) state and recovery that persist across independent invocations, unrelated to any one model call's content.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?

**Response 15**

    → Deterministic: System 4's `hot_state.json` hash cap (`test_hotstate_rejects_more_than_20_hashes`) and System 3's read-only `allowed-tools` on `/deploy-check`, both hold regardless of model behavior. 
    
    Prompt-guided: System 2's compression step, which asks the model to summarize the resolved refund issue faithfully into 385 tokens — there's no code check that the summary is *accurate*, only that a summary under budget is produced. Code-level guarantees belong wherever a violation is unrecoverable or unsafe; prompt guidance belongs wherever the outcome is a matter of quality or judgment code can't directly verify.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?

 **Response 16**

    → System 2 compresses within a single, continuous conversation: 38,708 baseline tokens down to 16,871 assembled (56.41% reduction), with the active segment (15,789 tokens) preserved byte-exact and only the two resolved issues (385 + 511 tokens) actually compressed. 
    
    System 4 compresses *across* discrete, independent invocations: `hot_state.json` at 643 bytes is the entire carry-over from one shift's analysis to the next, with no continuous window at all, just a small, durable, capped handoff file. Same underlying principle (never carry more than the next decision needs), solved by summarization-within-a-window in System 2 versus a structured, size-capped handoff file in System 4.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

**Response 17**

    → `tests/test_us04_fork_scratchpad.py::test_merge_findings_appends_without_rewriting_existing` (System 4) guarantees merges never disturb prior scratchpad entries. My own fork-standout run demonstrated this directly: the main scratchpad grew from 2 pre-existing lines to 4 lines after merging both forks' findings (`H1-lot-defect` and `H2-line-calibration`, one entry each), with the original two lines byte-identical and positioned first. A single successful run only shows the merge worked *once*; it doesn't prove the merge is append-only under repeated use over months of shifts, which is what the test and the fact that my real run's pre-existing lines survived intact actually confirms.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.

**Response 18**

    → System 3's `/deploy-check` skill: its `allowed-tools` list is scoped entirely to read-only git/gh commands (`git status`, `git diff`, `git log`, `git rev-parse`, `git ls-files`, `gh pr view`, `gh pr checks`). There is no write-capable command is reachable at all. Even a fully compromised or hallucinating invocation is capped at "produces a wrong report," never "damages the repo or ships bad code." 
    The kill switch is structural: because write-capable tools were never in the allowlist, there's nothing to disable mid-run and the tool surface itself is the enforcement point.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it.(If nothing, what you checked to be sure.)

**Response 19**

    → `pip install -e ".[dev]"` for both System 1 and System 2 initially failed at the live-run step with `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'`. Cause: `pyproject.toml` pins `anthropic==0.39.0` (System 1) / `anthropic==0.69.0` (System 2) but doesn't cap `httpx`, so a fresh install pulled `httpx==0.28.1`, which removed the `proxies` kwarg the older SDK still passes internally. 
    
    Fix: `pip install "httpx<0.28"` after the normal install — confirmed working, both systems' live runs (Haiku, Sonnet, and the System 2 case-facts control) completed successfully afterward.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.

**Response 20**

    → System 2's compressor refuses to touch the active segment at all (`test_summarize_segment_refuses_to_compress_the_active_segment`), which is safe but means the reduction ceiling is capped by how much of the transcript counts as "active". In my run the active segment alone was 15,789 of 16,871 assembled tokens (93.6%), so nearly all the assembled budget goes to one still-open issue. If the active segment itself grows very long (many back-and-forth turns on the same unresolved issue), the 56.41% reduction achieved here would shrink toward zero, since compression can only ever touch the resolved issues. 
    
    I'd consider letting the *early* turns of a long-running active issue be lightly summarized (keeping only the most recent N active turns byte-exact) rather than treating the entire active segment as untouchable, provided a test could guarantee the structured facts (like the Q6 status token) are pulled out into the case-facts block before any such trimming — which the current case-facts extraction step already does independently of the active segment, so this would be a safe extension rather than a redesign.

---

## Standout Work

### 1. Two-model comparison — System 1

Ran `python -m claims_intake.run --all` twice against the same 8 fixtures: once on the default `claude-haiku-4-5-20251001` (`/workspace/Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20260810_024505`, total cost **$0.1202**) and once on `claude-sonnet-4-5-20250929` (`/workspace/Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20260810_025046`, total cost **$0.3832** — roughly 3.2x Haiku's cost for the same fixture set).

| claim_id | Haiku outcome / turns | Sonnet outcome / turns |
|---|---|---|
| claim_01_kitchen_fire | incomplete / 2 | incomplete / 2 |
| claim_02_stolen_bike | incomplete / 2 | incomplete / 2 |
| claim_03_water_damage | **routed** / 7 | **incomplete** / 4 |
| claim_04_neighbor_injury | incomplete / 2 | **routed (liability)** / 4 |
| claim_05_auto_collision | routed / 4 | routed / 4 |
| claim_06_low_confidence_escalation | incomplete / 3 | **escalated** / 5 |
| claim_07_tree_falls_on_car | incomplete / 2 | incomplete / 2 |
| claim_08_minor_porch_damage | routed / 4 | routed / 4 |

**Where routing changed:** three claims flipped outcome between models. `claim_06_low_confidence_escalation`: the fixture purpose-built to be ambiguous, only actually triggered a formal escalation under Sonnet; Haiku stopped after 3 turns without escalating or routing. Sonnet's `/workspace/Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20260810_025046/escalations.jsonl` shows the full structured escalation record: reason `unresolved_ambiguity`, root cause "multi-asset storm damage affecting both auto (parked car) and property (garage roof, fence)... requires coordinated auto and property adjusters," with `confidence: 0.5`. This is a case where the stronger model recognized the ambiguity was severe enough to warrant a formal escalation rather than either guessing or stalling and Haiku's loop simply went `incomplete` on the same input.

Conversely, `claim_03_water_damage`, which Haiku routed cleanly in 7 turns but went `incomplete` under Sonnet in only 4 turns; Sonnet asked a clarification (1) and did not reach a routing tool call at all in this run. `claim_04_neighbor_injury` flipped the other way, from `incomplete` under Haiku to a clean `routed (liability)` under Sonnet. This shows the stronger model does not uniformly do "more" work per claim, Rather, it changes *which* claims it treats as complete enough to route, in both directions, not just how confidently it escalates ambiguous ones.

### 2. Breaking each system on purpose

**System 1 — ambiguous claim.** `claim_06_low_confidence_escalation` is the fixture purpose-built for this. Under Haiku (`/workspace/Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20260810_024505`) it stopped after 3 turns with outcome `incomplete` and 0 clarifications asked — the loop simply ran out of useful tool calls without reaching a terminal decision. Under Sonnet (`/workspace/Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20260810_025046`) the same fixture took 5 turns, asked 1 clarification, and produced a full structured escalation (see above) with an explicit `confidence: 0.5` and a `case_facts_at_escalation` snapshot capturing exactly what was and wasn't known at the point of giving up. This is a concrete degradation-under-ambiguity comparison: the weaker model silently stalled, while the stronger model surfaced the ambiguity explicitly and handed off a structured record an adjuster could act on.

**System 2 — case-facts block stripped.** Built into the solution's own `--all` command, which produces both `eval.jsonl` and `eval_control.jsonl` in one invocation (`/workspace/Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/04-assemble-and-locate/solution/runs/20260810-030459`). Q6 asking for the exact structured status token of the in-progress payment-method update, passes with the case-facts block present (`"in_progress"`, quoted directly) and fails without it, with the model answering *"Answer: Unknown with the payment-method update issue does not currently have a documented status token in the provided context."* This demonstrates concretely that the case-facts block isn't redundant with the compressed transcript: the same conversation is present in both runs, but only the structured extraction reliably preserves an exact status token a downstream system might need to match against (e.g., a support-ticket state machine).

### 3. Fork path exercised end-to-end — System 4

Forked the real shift-C hot state (`current_shift_summary` covering the capacitor-bank-C-7 defect cluster, `/workspace/Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/data/hot_state.json`, 643 bytes) into two competing-hypothesis branches under `/workspace/Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/data/forks/`:

- **H1-lot-defect**: `"All 5 defects trace to lot 2026-0430-B; DAR 0.39 vs spec >=0.85"` → `"Lot 2026-0430-B out of spec -- recommend quarantine"`
- **H2-line-calibration**: `"No calibration-drift events in line-C logs for shift date"` → `"No evidence supporting calibration-drift theory"`

**Isolation, verified directly against real files:** `diff /workspace/Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/data/hot_state.json /workspace/Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/data/forks/H1-lot-defect/hot_state.json` returns identical — the fork copied the base state without mutating it. Each fork's `scratchpad.jsonl` contains only its own hypothesis's entry; neither fork's file contains the other's evidence or conclusion text.

**Merge back to main, verified:** `/workspace/Build a Multi-Shift Quality Monitoring System with Claude Orchestration/04-fork-scratchpad/solution/data/shift_scratchpad.jsonl` grew from 2 pre-existing lines (from the earlier `shift-C` runs, timestamped `2026-08-10T02:28:56Z` and `2026-08-10T03:16:39Z`) to 4 lines after merging.  The 2 original lines remain first and byte-identical, followed by one H1 entry and one H2 entry appended in order. This confirms `merge_findings()` is append-only against real, previously-generated data, not only under a synthetic test fixture, which matters because the main scratchpad is the durable, cross-shift record a human reviewer reads; silently losing or reordering prior entries would be a data-loss bug in a system meant to run unattended indefinitely.

### 4. Mapping a system to a new use case

I'd map System 1 (the stop_reason-driven claims intake loop) onto a B2B SaaS support-ticket escalation system, the kind that triages incoming customer tickets, gathers account and usage context, and either resolves the issue directly or routes it to the right specialist team (billing, security, infrastructure). The structural fit is close to one-to-one: both problems are "gather facts via tools → classify → terminate in a routed or escalated state," and both need the same guarantee System 1 provides a loop that keeps working until it reaches a real terminal decision, without a hard-coded turn cap that might cut off a legitimately complex ticket (as `claim_03_water_damage` needed 7 turns to route correctly in my run, see Q1/Q4), and without silently looping forever on an ambiguous one. The structured tool-error pattern (Q3's `_err(category, retryable, message)` helper) would carry over directly too: a ticket-classification tool returning a typed `{"error_category": "missing_field"}` instead of a vague string lets the agent recover from an incomplete ticket the same way the claims agent recovers from an incomplete claim.

I would explicitly not reach for System 2's compression strategy here unless tickets routinely ran long, most support tickets resolve within a handful of exchanges, so the token-budget problem System 2 solves (which only became necessary in my run once the transcript hit 38,708 baseline tokens — see Q5) doesn't really exist at that scale. I'd only bring it in for a subset of tickets that turn into extended, multi-day threads.

### 5. Personalizing one system for a vertical

I'd re-frame System 4 (multi-shift orchestration) for a clinical laboratory running quality-control checks once per shift instead of monitoring manufacturing defects. Three concrete things would change from what I verified in my own run (Q11–13):

- Tool set: `defects_since()` — the SQL-side query filtered by timestamp over a defects table that returned 0 new rows in my shift-C run (Q11) — would become a query over QC control-chart violations, keyed by `(assay, instrument, timestamp)` instead of `(component, lot, timestamp)`. The underlying principle (filter server-side, never load full history into the prompt) stays identical; only the schema changes.
- Context/state design: my `hot_state.json` (643 bytes, Q13) would need an added compliance dimension the manufacturing version doesn't need since tracking which violations have already been reported to a regulatory body (e.g., CLIA or a state health authority). In manufacturing, a defect that ages out of the `recent_defect_hashes` cache is just no longer "recent." In a clinical lab, failing to track whether a violation was already reported is a compliance failure, not just a loss of context — so that field can't be allowed to silently expire the way the defect hashes do.
- Crash-recovery threshold: the 30-minute staleness window I confirmed in `recovery.py` (Q12, `STALE_RESUME_THRESHOLD_MINUTES = 30`) — roughly 1/16th of an 8-hour manufacturing shift would likely need to shrink. Clinical lab operations run more continuously, and a QC violation that's 30 minutes stale might already be behind a second, conflicting run on the same instrument. So the "safe to resume" window would need to tighten to match how frequently the underlying process actually changes state.
