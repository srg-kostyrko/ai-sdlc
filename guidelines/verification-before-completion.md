# Verification Before Completion

**Iron law: no completion claim without fresh verification evidence.**

Claiming work is complete without actually verifying is dishonesty, not efficiency. Confidence is not evidence; "should pass" is not "passes"; agent self-reports are not verification.

## The gate

Before any success claim, satisfaction expression, status update, commit, or task tick:

1. **Identify** — what command proves this claim?
2. **Run** — execute the full command, fresh.
3. **Read** — the output, including exit code and failure count.
4. **Verify** — does the output confirm the claim?
5. **Then claim** — state the result with the evidence inline.

Skip any step → not verified.

## Mapping claims to commands

| Claim | Required evidence | Not sufficient |
|-------|-------------------|----------------|
| Tests pass | Test runner output: 0 failures | "Should pass now" |
| Build succeeds | Build command: exit 0 | Linter passing |
| Linter clean | Linter output: 0 errors | Partial check |
| Bug fixed | Test against original symptom: passes | Code changed, assumed fixed |
| Regression test works | Red → green cycle verified (test fails before fix, passes after) | Test passes once |
| Requirements met | Per-requirement checklist with evidence | Tests passing alone |
| Subagent completed | Diff inspected, changes confirmed | Subagent reports "success" |

## Red flags — stop

- Words like "should", "probably", "seems to", "looks correct"
- Premature satisfaction language ("Great!", "Done!", "Perfect!")
- Trusting subagent success reports without inspecting the diff
- Partial verification accepted as complete
- Tests that pass only once (no red-green cycle for regression tests)
- Any claim of success preceded by no verification command in this turn

## Patterns

**Tests:**
- ✅ Run test command → see "34/34 pass" → "All tests pass."
- ❌ "Should pass now" / "Looks correct"

**Regression test (TDD red-green):**
- ✅ Write test → run (fails) → apply fix → run (passes) → revert fix → run (fails) → restore fix → run (passes).
- ❌ "I've added a regression test" without red-green verification.

**Requirements:**
- ✅ Re-read deltas → check each scenario/criterion → mark PASS/GAP with evidence → report.
- ❌ "Tests pass, requirements met."

**Subagent delegation:**
- ✅ Subagent reports success → run `git diff` or read modified files → confirm changes → report actual state.
- ❌ Trust the subagent's summary.

## Bottom line

Run the command. Read the output. Then claim the result. No exceptions.
