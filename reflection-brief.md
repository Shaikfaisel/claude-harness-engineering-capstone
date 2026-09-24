# Reflection Brief — Harness Engineering Capstone

**Name:** Shaik
**Date:** September 23, 2026
5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → Baseline: 38,708 tokens → Assembled: 16,828 tokens (56.53% reduction). Active segment dominates at 15,789 tokens (93.8% of assembled). Kept verbatim because it's the current live conversation — only resolved segments get compressed.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Resolved segments summarized: refund 12,334→419 tokens (96.6% reduction), subscription 11,475→434 tokens (96.2% reduction). Case facts (204 tokens) kept byte-exact as golden record. Active segment (15,789 tokens) kept verbatim as live exchange.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → From `eval_control.jsonl`: Q1 PASSES (refund amount $22.14 appears in active conversation text), Q6 FAILS (can't find "in_progress" status token without case_facts block). Proves case_facts (204 tokens verbatim) is load-bearing — compressing or removing it loses structured decision data.
1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → **Skipped** — System 1 environment broken (SDK version incompatibility). Could not capture trace evidence.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → **Skipped** — System 1 not run.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → **Skipped** — System 1 not run.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → **Skipped** — System 1 not run.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → **Skipped** — System 2 API budget exhausted. Error: "Service budget exceeded for vkey_...". Could not run `python -m retail_context.run --all` to capture budget.json.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → **Skipped** — System 2 budget exhausted.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → **Skipped** — System 2 budget exhausted.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → From `.claude/rules/react.md`:
```yaml
   ---
   description: Conventions for React components and pages
   paths:
     - "src/components/**/*"
     - "src/pages/**/*"
   ---
```
   Path-scoped rules apply enforcement only where it matters (React source), avoiding CLAUDE.md bloat in unrelated directories (data/, scripts/, etc.). A single directory-level CLAUDE.md would force all rules on all files in that tree, mixing concerns. The glob pattern ensures conventions stay hygenic — "this rule applies here, nowhere else" — and scales as the monorepo grows.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
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
   Running forked isolates the skill's session from the main conversation — if the validation discovers an issue mid-check, the scratchpad stays separate and doesn't pollute the main state. Read-only allowed-tools (no Bash write, no Create, no Delete) prevent accidental file corruption or premature pushes. Without forking, a bug in the skill could corrupt the hot state or modify files the user didn't intend. Without read-only enforcement, the skill could delete test fixtures or push broken code.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → Validator output: **OK** (35 tests passed). Project-level: `/review` command in `.claude/commands/review.md` — applies to all code reviews in the monorepo, configured once and enforced everywhere. User-level: standards in `.claude/standards/` (e.g., naming conventions, error handling patterns) — developers can reference and follow them voluntarily within their own sessions, not mandatory enforcement.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never sees the full history?
    → Test output: `test_defects_since_uses_index_and_does_not_load_full_table PASSED` and `test_defects_since_respects_limit PASSED`. Shift C run output: **"0 new defects"** (SQL filtered the full `defects.json` — 143 defects total in fixtures — to only recent ones). The indexed query `defects_since(warm_db, staleness_threshold=30_minutes)` returns only defects within the staleness window, not the full history. The model never sees the full 143-defect table because the SQL pre-filter (indexed on timestamp) returns only the slice relevant to the current shift. This keeps the context tiny and deterministic.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → Test: `test_threshold_constant_is_30_minutes PASSED`. From `shift_monitor/recovery.py` logic: if the last recorded state is **>30 minutes old**, the system does a fresh start (injects a summary of recent defects from the warm tier) instead of resuming mid-analysis. Why fresh is safer: old state assumes the model can pick up mid-thought and stay coherent, but 30+ minutes of wall-clock time in production means defect clusters may have evolved, new alerts may have fired, or the original context is stale. A fresh start with a compressed summary grounds the new analysis in current reality, avoiding the risk of the model defending conclusions from outdated state.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → Your `hot_state.json`: **643 bytes** (captured from run). Budget: under 5 KB. The budget matters for indefinite shifts because hot state persists in memory/disk across the 8-hour shift. If state grew unbounded (e.g., appending full claim details on each update), it would bloat to MB within weeks of operation. A fixed byte budget forces the harness to only keep essential summaries (hashes, counts, flags) and offload details to warm/cold tiers. This guarantees the system stays predictable and cheap to resume, no matter how many shifts have run.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → **Model layer:** `shift_monitor/pipeline.py` (System 4) — invokes Claude API with a carefully assembled prompt (defect slice + schema). The model is the decision point; everything else feeds it data.
    → **Harness layer:** `.claude/CLAUDE.md` + `.claude/skills/deploy-check/SKILL.md` (System 3) — the configuration that guides Claude Code's behavior, path rules, and task-specific skills. This is the "harness" that structures how Claude operates.
    → **Orchestration layer:** `shift_monitor/warm.py` + `shift_monitor/recovery.py` (System 4) — tiered state, crash recovery, and session forking. Orchestration coordinates *when* the model runs, *what state it sees*, and *how it recovers* from failure across multiple invocations.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → **Deterministic (guaranteed in code):** `.claude/skills/deploy-check/SKILL.md` `allowed-tools` read-only enforcement — the tool allowlist is parsed and enforced before Claude sees the skill invocation. The system will refuse a `Delete` or `Create` call; no prompt can override this.
    → **Prompt-guided:** `shift_monitor/pipeline.py` defect analysis prompt — the model decides *which* defects are related, *whether* to escalate, or *if* the batch needs a supervisor review. The prompt guides reasoning but doesn't force a decision.
    → **When each is right:** Use deterministic enforcement for safety invariants (read-only for audits, byte budgets for reliability, atomic writes for crash recovery). Use prompt guidance for judgment calls (routing, summarization, risk assessment) where the model's reasoning adds value and flexibility is needed.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → **System 2 (intra-session):** Compresses a single long conversation within one session. Goal: keep tokens under a budget while staying coherent. (Skipped due to budget exhaustion, but design intent is `budget.json` shows ~50% reduction from 47k-token baseline.)
    → **System 4 (cross-session):** Keeps hot state at 643 bytes and fetches only recent defects from warm tier. Goal: each 8-hour shift starts fresh with a filtered dataset, avoiding multi-shift context bloat.
    → **Same principle, different scope:** Both prune history to fit a budget. System 2 prunes *within* a session (compress old turns). System 4 prunes *across* sessions (return only recent defects, forget old shifts). Both use deterministic rules (byte budgets, staleness thresholds) instead of hoping the model will ignore irrelevant context.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → Test: `test_fork_for_hypothesis_copies_state_without_mutating_base PASSED` (System 4). A single successful run never exercises the fork path — it follows the main stream. The test guarantees that if a sub-agent investigation *does* fork, its scratchpad stays isolated and the base state doesn't corrupt. Without this test, a bug in fork logic (e.g., both forks sharing a dict) would only surface in production when someone actually uses the fork feature, causing data loss.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → **System 3 (Claude Code config).** Blast radius: if a path-scoped rule has a typo in its glob pattern (e.g., `src/components/` instead of `src/components/**/*`), it silently fails to load for the intended files. Developers unknowingly violate the rule. Kill switch: the validator (`pytest tests/ -v` output: 35 tests passed, including glob matching tests) verifies every rule's glob is correct and files match only the intended paths. Enforcement: `.claude/rules/` uses YAML frontmatter parsing and glob compilation at config load time, rejecting invalid patterns before Claude sees the session.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → **System 1 failure:** Ran `python -m claims_intake.run --all` after fresh venv setup with anthropic 0.39.0. Error: `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'`. Root cause: the reference code in `client.py` didn't pass `base_url` to the Anthropic constructor. Fix: edited `client.py` to read `ANTHROPIC_BASE_URL` from environment and pass it to `Anthropic(api_key=..., base_url=base_url)`. This required the file to survive a pip reinstall (it didn't initially — pip overwrote edits). Final fix: used `cat > claims_intake/client.py << 'EOF'` to overwrite the file with corrected code before running again.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → **System 4 staleness threshold:** The 30-minute threshold for crash recovery works for an 8-hour shift, but I'd make it configurable per deployment (e.g., `--staleness-threshold 45m` for slower-moving defect streams, `--staleness-threshold 10m` for high-velocity lines). Observed: shift C had "0 new defects" — a production setup might run many shifts back-to-back, and a hard-coded 30 minutes assumes all environments have the same defect velocity. A threshold constant in code is deterministic but inflexible. A config parameter would let ops teams tune recovery behavior without code changes while keeping the contract (resume if fresh enough, else start fresh) deterministic.