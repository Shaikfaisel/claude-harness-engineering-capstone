# Reflection Brief — Harness Engineering Capstone

**Name:** Shaik Faisel Ahmed

**Date:** September 23, 2026

---

## System 1 — Claims Intake Agent

### 1. **Loop control.**

Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

→ From the `claim_01_kitchen_fire` trace:

```text
turn 1 stop_reason=tool_use
  lookup_policy(POL-1001)
  record_claim_fact(incident_type, incident_date, location, description, injury_party)

turn 2 stop_reason=end_turn
  no tool calls
```

The loop is implemented in `claims_intake/loop.py`. It uses Claude's `stop_reason` to decide whether to continue or stop: `tool_use` means execute the requested tool(s) and continue the loop; `end_turn` means return the final response and stop. Unexpected stop reasons raise an error.

The loop tests confirmed this behavior:

* `end_turn` returns immediately.
* `tool_use` executes the tool and loops again.
* Multiple tool blocks are handled.
* Unexpected stop reasons raise an error.

### 2. **Anti-pattern.**

Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

→ One checked anti-pattern is using an **integer-limited iteration loop** instead of making `stop_reason` the termination condition.

The tests also check that the loop does not use string-membership logic to decide whether to continue and does not branch on claim types outside the appropriate tool/pricing logic.

If the loop used a fixed iteration cap instead of `stop_reason`, a valid claim could stop before Claude finished its required tool calls, or the loop could continue unnecessarily after Claude had already returned `end_turn`. The harness specifically requires the model's `stop_reason` to control the loop.

### 3. **Tool design.**

Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

→ `route_to_adjuster` and `escalate_to_human` both operate on claims and can receive a reason/context for the action, so their descriptions need to make the intended terminal outcome explicit.

The tool descriptions distinguish the normal adjuster-routing path from the human-escalation path. The agent is instructed to use a terminal routing tool when there is enough information and sufficient confidence, while escalation is used when the claim cannot be safely resolved.

Structured tool errors are useful because they identify the failure in a machine-readable way. Instead of receiving an ambiguous message such as `"something went wrong"`, the agent can see what failed and respond according to the tool contract rather than inventing a successful result.

### 4. **Your numbers.**

Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

→ For `claim_01_kitchen_fire`:

```text
turns = 2
input tokens = 6,388
output tokens = 490
estimated cost = $0.0088
result = incomplete
```

The README describes the expected full end-to-end behavior for the eight-claim run, with approximately **$0.05 estimated cost** for the sample run.

My actual eight-claim run used approximately **$0.1025 total**, and several claims did not reach their expected terminal outcome. For example:

```text
claim_05_auto_collision
expected routed -> routed
turns=5
estimated cost=$0.0240

claim_08_minor_porch_damage
expected routed -> routed
turns=4
estimated cost=$0.0188
```

The difference is because the live model behavior in my run did not always reach the terminal routing/escalation tool. For `claim_01`, Claude stopped with `end_turn` after asking for an estimated damage amount, even though the fixture had no clarification response for that question.

### 5. **Evidence of validation.**

What objective evidence shows that the harness implementation itself passed its required tests?

→ System 1 passed the complete test suite:

```text
29 passed in 0.12s
```

The tests covered loop behavior and anti-patterns, including:

* `stop_reason` controls continue-vs-stop behavior.
* `tool_use` causes tool execution and another loop iteration.
* `end_turn` stops the loop.
* Multiple tool calls are handled.
* Unexpected stop reasons are rejected.
* Integer iteration caps are not used.
* String-membership shortcuts are not used.
* Claim-type branching is not improperly placed in the loop.

The live run was then performed separately to observe actual Claude behavior against the eight fixtures.

---

## System 2 — Context strategy

### 6. **The reduction.**

From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

→ Baseline: **38,708 tokens** → Assembled: **16,828 tokens** (**56.53% reduction**).

Active segment dominates at **15,789 tokens (93.8% of assembled)**. It is kept verbatim because it is the current live conversation — only resolved segments get compressed.

### 7. **Summarize vs preserve.**

State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

→ Resolved segments summarized:

* Refund: **12,334 → 419 tokens (96.6% reduction)**
* Subscription: **11,475 → 434 tokens (96.2% reduction)**

Case facts (**204 tokens**) are kept byte-exact as the golden record. The active segment (**15,789 tokens**) is kept verbatim as the live exchange.

### 8. **Facts block.**

Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

→ From `eval_control.jsonl`:

* **Q1 PASSES** — the refund amount `$22.14` appears in the active conversation text.
* **Q6 FAILS** — the `"in_progress"` status token cannot be found without the `case_facts` block.

This proves that the **204-token `case_facts` block is load-bearing**. Compressing or removing it can lose structured decision data even when the conversational text remains available.

---

## System 3 — Claude Code config

### 9. **Path-scoped rules.**

Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

→ From `.claude/rules/react.md`:

```yaml
---
description: Conventions for React components and pages
paths:
  - "src/components/**/*"
  - "src/pages/**/*"
---
```

Path-scoped rules apply enforcement only where it matters — React source — avoiding `CLAUDE.md` bloat in unrelated directories such as `data/` and `scripts/`.

A single directory-level `CLAUDE.md` would force all rules on all files in that tree, mixing unrelated concerns. The glob pattern keeps the convention scoped to the files where it applies.

### 10. **Forked skill.**

Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

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

Running forked isolates the skill's session from the main conversation. The validation work can therefore use its own context without polluting the main working state.

The read-only allowlist also prevents accidental modification of files or premature pushes. Without the fork, validation could pollute the main context; without read-only enforcement, a validation skill could potentially modify or delete project files.

### 11. **Scope.**

From the validator output: project-level vs user-level scope. Give one example of each from this config.

→ Validator output:

```text
OK — 35 tests passed
```

Project-level example: the `/review` command in `.claude/commands/review.md`, which is configured as part of the project and can be used consistently across the monorepo.

User-level example: standards in `.claude/standards/`, such as naming conventions and error-handling patterns, which developers can reference and follow during their own sessions.

---

## System 4 — Orchestration

### 12. **Push work down.**

Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

→ Test output included:

```text
test_defects_since_uses_index_and_does_not_load_full_table PASSED
test_defects_since_respects_limit PASSED
```

Shift C output reported:

```text
"0 new defects"
```

The indexed query is:

```text
defects_since(warm_db, staleness_threshold=30_minutes)
```

It returns only defects within the relevant staleness window instead of loading the entire history.

The model never sees the full **143-defect fixture table** because SQL performs the filtering first. Only the relevant slice is passed to the model, keeping context smaller and the selection deterministic.

### 13. **Crash recovery.**

The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

→ Test:

```text
test_threshold_constant_is_30_minutes PASSED
```

From `shift_monitor/recovery.py`: if the last recorded state is **more than 30 minutes old**, the system starts fresh and injects a summary of recent defects instead of resuming the old analysis.

A fresh start can be more reliable because an old model state may no longer represent current reality. New defects or alerts may have appeared while the old state was sitting idle.

A new session with a compressed summary therefore gives the model current relevant information without depending on potentially stale reasoning from the previous session.

### 14. **Small state.**

Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

→ My `hot_state.json` was **643 bytes**, with a budget of **under 5 KB**.

The budget matters because hot state persists across shifts. If full historical information were continuously appended to it, the state could grow from a small file into megabytes over weeks or months.

A fixed byte budget forces the system to retain only essential state such as hashes, counts, and flags, while detailed information is kept in warm/cold storage.

---

# Part 2 — Synthesis

**Graded on connecting two or more systems. Cite a named file/artifact from each.**

### 15. **Three layers.**

Point to a file/artifact for each layer and justify.

→ **Model layer:** `shift_monitor/pipeline.py` (System 4) — invokes the Claude API with a carefully assembled prompt containing the relevant defect slice and schema. The model is the decision point.

→ **Harness layer:** `.claude/CLAUDE.md` + `.claude/skills/deploy-check/SKILL.md` (System 3) — configuration that structures Claude Code behavior, path rules, and task-specific skills.

→ **Orchestration layer:** `shift_monitor/warm.py` + `shift_monitor/recovery.py` (System 4) — handles tiered state, crash recovery, and session management. Orchestration determines when the model runs, what state it sees, and how it recovers from failure.

### 16. **Deterministic vs prompt.**

Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?

→ **Deterministic:** `.claude/skills/deploy-check/SKILL.md` uses an `allowed-tools` read-only allowlist. The allowed tools are explicitly constrained, so the skill cannot simply decide through prompting that it is allowed to perform an unrelated write operation.

→ **Prompt-guided:** `shift_monitor/pipeline.py` contains the defect-analysis prompt. The model uses that prompt to reason about which defects are related, whether escalation is appropriate, and whether supervisor review is needed.

→ **When each is right:** deterministic enforcement is appropriate for safety and reliability invariants such as read-only access, byte limits, and atomic state updates. Prompt guidance is appropriate for judgment-heavy tasks such as classification, summarization, and reasoning.

### 17. **Context, two faces.**

Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?

→ **System 2 — intra-session:** compresses a single long conversation within one session.

Actual run:

```text
baseline = 38,708 tokens
assembled = 16,828 tokens
reduction = 56.53%
```

The system compresses resolved conversation segments while preserving the active conversation and critical case facts.

→ **System 4 — cross-session:** keeps hot state at **643 bytes** and retrieves only relevant recent defects from the warm tier.

→ **Same principle, different scope:** both systems prune unnecessary history to stay within a controlled context budget.

System 2 manages context **inside one conversation**, while System 4 manages state **across separate shift sessions**.

### 18. **Reliability you can't see in one run.**

Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

→ Test:

```text
test_fork_for_hypothesis_copies_state_without_mutating_base PASSED
```

A normal successful run may never exercise the fork path. The test guarantees that when a sub-agent investigation does fork, its scratch state remains isolated from the base state.

This matters before shipping because a fork-state bug could corrupt the main state even though normal runs appear successful.

### 19. **Blast radius.**

Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.

→ **System 3 — Claude Code config.**

If a path-scoped rule has an incorrect glob pattern, the rule may silently fail to apply to the intended files. Developers could then work without the intended convention being applied.

The validator provides an important safety check:

```text
35 tests passed
```

The configuration tests verify rule parsing and glob behavior before the configuration is relied upon. The `.claude/rules/` structure also limits the scope of each rule to its intended files.

---

# Part 3 — Honest assessment

### 20. **What broke?**

One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)

→ **System 1 dependency issue:** the initial environment had an incompatible Anthropic SDK version. The environment had `anthropic 0.69.0`, while the project requires `anthropic==0.39.0`.

The initial error was:

```text
TypeError: Client.__init__() got an unexpected keyword argument 'proxies'
```

The fix was to create and activate the System 1 virtual environment and install the project's pinned dependency:

```text
anthropic==0.39.0
```

After the environment was corrected, the complete System 1 test suite passed:

```text
29 passed in 0.12s
```

This demonstrated that the failure was an environment/dependency compatibility issue rather than a failure of the loop tests themselves.

### 21. **What you'd change.**

One architectural decision you'd make differently, grounded in what you observed.

→ **System 4 staleness threshold:** the 30-minute threshold for crash recovery works as a deterministic rule, but I would make it configurable per deployment.

For example, a slower-moving defect stream might use a longer threshold, while a high-velocity environment might require a shorter one.

Observed behavior included Shift C reporting:

```text
"0 new defects"
```

A configurable threshold would allow operations teams to tune recovery behavior without changing application code while preserving the same deterministic rule:

```text
resume if state is fresh enough
otherwise start fresh with a summary
```
