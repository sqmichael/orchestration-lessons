# Orchestration Lessons

Hard-won lessons from multi-agent AI coding workflows. Extracted from production experience building a voice AI system (Boardy) using Claude Code with sub-agent delegation, external model reviews (Codex/GPT-5.2), and automated quality gates.

## The Core Problem

When you delegate code tasks to AI sub-agents, each agent operates with partial context. Bugs cluster at the seams between agents — where one agent's output becomes another's input. These bugs are invisible to unit tests because each component works correctly in isolation.

---

## Lesson 1: Agents Write Against Imagined Interfaces

**What happened:** A sub-agent was told to "fetch match suggestions using PersonService." It wrote `person.get("person_id")` — treating a dataclass as a dict and using a key name that didn't exist.

**Why:** The agent never read the `PersonProfile` dataclass definition. It inferred the interface from the task description and guessed wrong on both the access pattern and the field name.

**Fix:** Every agent prompt that touches shared data must include:
1. The actual model/dataclass definitions (copy-pasted, not described)
2. The dict shapes for any shared structures (exact keys)
3. The function signatures of any existing functions to call
4. Explicit instruction: "Read all referenced model files before writing code"

**Rule: Never delegate code that touches shared data models without providing the model definitions.**

---

## Lesson 2: Review Gates Must Require Proof

**What happened:** A pre-commit hook checked for a `.review-done` marker file. The workflow was: run code review, then `touch .review-done` to unblock the commit. In practice, the `touch` command was run without any review — the marker proved nothing.

**Why:** The gate was trivially bypassable. A boolean file (`exists` / `doesn't exist`) can't distinguish "review happened" from "developer skipped review."

**Fix:** Changed the gate to require `.review-output-{DIFF_HASH}` — a non-empty file containing actual review output. The review step must write findings to the file. The hook checks `-s` (non-empty) instead of `-f` (exists).

**Rule: Quality gates should require evidence, not just acknowledgment.**

---

## Lesson 3: Silent Exception Swallowing Hides Integration Bugs

**What happened:** A `try/except Exception: pass` block caught an `AttributeError` from the dataclass bug (Lesson 1). The function returned an empty list instead of crashing. The feature appeared to work — it just never returned any data.

**Why:** The catch-all was written as a graceful degradation pattern. But it masked the real bug. The function silently failed on every call.

**Fix:** Always log in catch blocks, even for expected failures:
```python
# Bad
except Exception:
    pass

# Good
except Exception as e:
    logger.warning("match_suggestion_phone_lookup_error", error=str(e))
```

**Rule: Graceful degradation without logging is silent data loss.**

---

## Lesson 4: Type Checking Catches Integration Bugs at Lint Time

**What happened:** Three HIGH-severity bugs (wrong dict key, dataclass `.get()`, removed Pydantic fields) all involved type mismatches across module boundaries. None were caught by unit tests because each module was tested with its own mock data.

**Why:** Python's dynamic typing means `person.get("person_id")` is syntactically valid on any object. The error only surfaces at runtime — specifically, only when that exact code path is exercised with real data.

**Fix:** Added `mypy` with `check_untyped_defs = True`. This catches type errors even in functions without type annotations. All three bugs would have been flagged at lint time.

**Rule: Add static type checking early. It's the cheapest integration test you'll ever write.**

---

## Lesson 5: Contract Tests Beat Unit Tests for Shared Data

**What happened:** `main.py` constructed a dict with key `"id"`. `conversation.py` read it with `.get("person_id")`. Both had passing unit tests. The integration was broken.

**Why:** Unit tests use locally-defined test data. They test "does my code work with my data?" — not "does my data match what the other module produces?"

**Fix:** Added contract tests that verify:
- Producer keys match consumer expectations (exact key set)
- Shared model attributes exist (dataclass field checks)
- Serialization output has required keys (`.to_dict()` contracts)

```python
def test_conversation_reads_id_not_person_id():
    """Regression: conversation.py must use 'id', not 'person_id'."""
    profile = {"id": "uuid-123", "full_name": "Test"}
    assert profile.get("id") == "uuid-123"
    assert profile.get("person_id") is None  # This was the bug
```

**Rule: When one module produces data and another consumes it, add a test that verifies they agree on the shape.**

---

## Lesson 6: Parallel Agent Tracks Need Convergence Reviews

**What happened:** Two agent tracks ran in parallel — Track A rewrote prompts, Track B added graph queries. Both passed their own tests. When merged, Track A's prompt builders didn't pass `match_suggestions` to the new stage context that Track B created.

**Why:** Neither track knew about the other's changes. They modified different files, so there were no merge conflicts — just a silent integration gap.

**Fix:** After parallel tracks converge, always review the integration points:
- Does Track A's output include the fields Track B expects?
- Do function signatures match across tracks?
- Are there new parameters that one track added that the other needs to pass through?

**Rule: No merge conflicts ≠ no integration bugs. Always review convergence points.**

---

## Lesson 7: External Model Review Catches What Self-Review Misses

**What happened:** Codex (GPT-5.2) review on a 488-line diff found 3 HIGH bugs in under 2 minutes. The authoring agent (Claude) had written and self-tested the code without noticing these issues.

**Why:** The authoring agent has a coherent mental model of what the code *should* do. It reads its own code through that lens. An external reviewer reads the code for what it *actually* does.

**Specific catches:**
- `returning_user_profile.get("person_id")` — wrong key (should be `"id"`)
- `person.get("person_id")` — PersonProfile is a dataclass, not a dict
- `POST /api/intros` — no authentication on a sensitive endpoint

**Rule: Self-review is necessary but insufficient. External model review before every commit.**

---

## Implementation Checklist

When setting up a multi-agent coding workflow:

- [ ] **Agent prompts include model definitions** — not descriptions, actual code
- [ ] **Review gates require non-empty output** — not just a boolean marker
- [ ] **Exception handlers always log** — no bare `except: pass`
- [ ] **Static type checking enabled** — mypy/pyright with `check_untyped_defs`
- [ ] **Contract tests for shared data shapes** — producer keys match consumer reads
- [ ] **Convergence review after parallel tracks** — check integration points explicitly
- [ ] **External model review before commit** — different model than the author

---

## Cost of Not Doing This

From one 6-hour session on a medium-complexity feature:

| Bug | Severity | Root Cause | Would Have Caught |
|-----|----------|-----------|-------------------|
| Wrong dict key (`person_id` vs `id`) | HIGH | Agent guessed key name | Contract test, mypy, external review |
| Dataclass treated as dict (`.get()`) | HIGH | Agent didn't read model | mypy, contract test, external review |
| Removed Pydantic fields still referenced | HIGH | Incomplete cleanup | mypy, external review |
| Missing auth on endpoint | HIGH | Agent didn't check patterns | External review |
| Silent exception swallowing | MEDIUM | Defensive coding without logging | External review |
| Conflicting instruction rules | MEDIUM | Prompt rewrite inconsistency | External review |

All 6 bugs were caught before reaching production. Without these gates, they would have shipped as silent failures — features that "work" but never return data.
