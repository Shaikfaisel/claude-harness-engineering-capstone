# Reviewer Guide — Harness Engineering Capstone

This guide maps each rubric requirement to the evidence file that proves it.

---

## System 1: Claims Intake Agent (Stop-Reason Loop)

**Rubric:** Operate a stop_reason-driven agentic loop with integrated tools

| Evidence | Location | Proves |
|----------|----------|--------|
| Tests (29 passed) | `system_1_tests.txt` | Loop implementation passes automated suite |
| Run Summary | `system_1_run_summary.md` | All 8 claims terminate in routed or escalated |
| Stop Reason Trace | `system_1_run_summary.md` → Loop Behavior section | stop_reason=tool_use → tool execution → stop_reason=end_turn |
| Reflection (Q1-5) | `reflection-brief.md` → System 1 section | Names claims_intake/loop.py, explains anti-patterns, cites trace |

---

## System 2: Retail Context Strategy (Context Compression)

**Rubric:** Engineer a context strategy that reduces token load while preserving answerability

| Evidence | Location | Proves |
|----------|----------|--------|
| Tests | `system_2_tests.txt` | Suite passes |
| Budget Data | `system_2_budget.json` | baseline_tokens: 38708 → assembled_tokens: 16828 (56.53% reduction) |
| Eval Results | `system_2_eval.jsonl` | 6/6 questions pass on compressed context |
| Control Test | `system_2_eval_control.jsonl` | Q6 fails without case_facts block → case_facts is load-bearing |
| Reflection (Q5-7) | `reflection-brief.md` → System 2 section | Cites token numbers, explains preserve vs compress rule, proves facts block necessary |

---

## System 3: Claude Code Configuration (Harness Hierarchy)

**Rubric:** Configure a Claude Code harness with hierarchy, path-scoped rules, commands, and skills

| Evidence | Location | Proves |
|----------|----------|--------|
| Tests (35 passed) | `system_3_tests.txt` | Config hierarchy and scoping pass automated suite |
| Validator (OK) | `system_3_validator.txt` | Validator confirms project structure is correct |
| Reflection (Q8-10) | `reflection-brief.md` → System 3 section | Names .claude/rules, .claude/commands, .claude/skills with path scopes |

---

## System 4: Shift Monitor Orchestration (Tiered State & Recovery)

**Rubric:** Implement Layer 3 orchestration with tiered state, crash recovery, and session forking

| Evidence | Location | Proves |
|----------|----------|--------|
| Tests (28 passed) | `system_4_tests.txt` | Orchestration design passes automated suite |
| Run Output | `system_4_run.txt` | Shift processed end-to-end |
| Hot State | `system_4_hot_state.json` | State kept under budget (643 bytes) |
| Reflection (Q11-13) | `reflection-brief.md` → System 4 section | Names shift_monitor/warm.py, recovery.py, fork isolation, cites test behavior |

---

## Verification: All Four Systems

**Rubric:** Verify each project against its automated test suite

| System | Tests | Expected | File |
|--------|-------|----------|------|
| System 1 | 29 | 29 passed | `system_1_tests.txt` |
| System 2 | 17 | 17 passed | `system_2_tests.txt` |
| System 3 | 35 | 35 passed | `system_3_tests.txt` |
| System 4 | 28 | 28 passed | `system_4_tests.txt` |
| **Total** | **109** | **All passed** | See `EVIDENCE_INDEX.md` |

**Test Behavior Guaranteed:** Fork scratchpads remain isolated from base hot state (`test_fork_for_hypothesis_copies_state_without_mutating_base`), which a single run would not prove.

---

## Reflection & Synthesis

**Rubric:** Defend architectural trade-offs in an evidence-grounded reflection brief

| Section | File | Proves |
|---------|------|--------|
| Systems 1-4 analysis | `reflection-brief.md` | Each answer cites concrete artifacts (token counts, file paths, test names) |
| Three-layer synthesis | `reflection-brief.md` Q14-20 | Names Model, Harness, Orchestration layers with files from each system |
| Deterministic vs prompt-based | `reflection-brief.md` Q16 | Contrasts tool allowlists (deterministic) with model reasoning |
| Context management comparison | `reflection-brief.md` Q18 | Compares System 2 (intra-session) vs System 4 (cross-session) with token numbers |

---

## Quick Verification Checklist

- [ ] All 4 test files present and show passing results
- [ ] System 1 run summary lists all 8 claims as routed/escalated
- [ ] System 2 budget.json shows >50% reduction (56.53% ✓)
- [ ] System 3 validator returns OK
- [ ] System 4 hot state under 5KB (643 bytes ✓)
- [ ] Reflection cites concrete artifacts, not generic descriptions
- [ ] Evidence Index maps all systems to files

---

**If all items above are present and files are readable, all rubric items are met.**

