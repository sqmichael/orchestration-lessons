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

## Lesson 8: Seam Audits Are Not Optional After Multi-File Changes

**What happened:** A 6-file feature implementation (v8 state machine) passed all 414 unit tests and Codex diff review. Three bugs survived:
1. A sync function couldn't `await` an async dependency — MATCHING state never fetched match suggestions
2. A `.format()` template used `{{{{` (quadruple braces) producing `{{` instead of `{` — LLM received malformed JSON examples
3. A type annotation said `DialogStage` but callers passed `ConversationStage` — no crash, but misleading for tools and reviewers

**Why:** Unit tests test each module with mock data. Diff reviews check each hunk in isolation. Neither traces a call chain end-to-end across module boundaries. The bugs live *between* the modules — at the seams.

**Fix:** After any change touching >3 files, perform a manual seam audit:
1. List every function call that crosses a module boundary
2. For each cross-module call, verify: argument types match parameter types, return types match what the caller expects
3. For each shared data structure, verify: producer keys match consumer reads
4. For async code, verify: the full call chain supports `await` if any leaf function is async
5. For string templates, verify: rendered output has balanced braces/brackets (test the actual `.format()` result)

**Rule: Tests check components. Reviews check diffs. Seam audits check integration. All three are needed.**

---

## Lesson 9: Lessons Applied Locally Get Replicated Globally

**What happened:** Lesson 2 fixed `review-gate.sh` to require non-empty proof (`.review-output-{DIFF_HASH}` with `-s` check). Then three more gates were written afterward — `push-gate.sh`, `core-gate.sh`, and the PASS paths in `codex-review.sh` and `deepseek-arch-review.sh` — all using `touch` (empty boolean markers). The exact anti-pattern Lesson 2 warned about was replicated in every subsequent gate.

**Why:** The lesson was applied to the specific file where the bug occurred (`review-gate.sh`) but not encoded as a principle that applies to all future gates. New hooks were written by different agents in different sessions — none knew about Lesson 2.

**Fix:** When you learn a lesson, encode it in two places:
1. The specific fix (patch the broken code)
2. A general rule in your instructions/CLAUDE.md (prevent recurrence in new code)

Audit existing code for the same pattern. If `review-gate.sh` was fixed but `push-gate.sh` wasn't, the lesson wasn't learned — it was just patched.

**Rule: A lesson applied to one file but not generalized is a patch, not a lesson.**

---

## Lesson 10: Async/Sync Boundary Bugs Are Silent

**What happened:** `_check_v8_stage_transition` was a sync function called from an async context. It worked — Python allows this. But when the MATCHING state needed to fetch match suggestions via `await _fetch_match_suggestions()`, it couldn't. The `await` keyword is invalid in a sync function. The match suggestions were simply never fetched.

**Why:** Python doesn't warn when you call a sync function from an async context. The sync function runs, returns, and nothing breaks — unless that function needs to call other async code. The bug is invisible until you trace the full dependency chain.

**Specific trap:**
```python
# This works fine
async def caller():
    result = sync_function()  # No error

# But sync_function can't do this:
def sync_function():
    data = await async_dependency()  # SyntaxError
    # So it either skips the call or was never written to make it
```

**Fix:** When writing or reviewing functions in an async codebase:
- If a function calls ANY async dependency (directly or transitionally), it must be `async def`
- When adding new async calls to existing sync functions, you must convert them to async and update all callers
- Add a seam-audit check: "Does this sync function need to call anything async?"

**Rule: In async codebases, sync functions are dependency dead-ends. Verify the full call chain supports async before assuming a sync function can do its job.**

---

## Lesson 11: CLI Tools Silently Corrupt Large Inputs

**What happened:** The Gemini seam audit hook passed prompts via `-p "$VARIABLE"`. At ~20KB, the model returned garbled nonsense — random words, hallucinated bash scripts, single-character outputs. At ~140KB, the OS rejected the command with `Argument list too long`. Exit code was 0 in both cases. No error message.

**Why:** Two separate failure modes compound:
1. **Shell variable escaping** — When a shell variable containing code (braces, quotes, backslashes) is passed as a CLI argument, the shell interprets special characters. At small sizes this works; at large sizes the corruption accumulates.
2. **OS argument length limit** — `MAX_ARG_STRLEN` on Linux x86_64 is 128KB per argument. `$(cat bigfile)` as a command argument hits this hard limit.

**Specific symptoms by size:**
- <10KB: Works fine
- 20-50KB: Returns but response is garbled (model received corrupted input)
- 50-128KB: Sometimes works, sometimes garbled (depends on content)
- >128KB: `Argument list too long` (OS rejection)

**Fix:** Split instruction from content:
```bash
# BAD — breaks at scale
gemini -p "$LARGE_PROMPT" -o text

# BAD — hits ARG_MAX
gemini -p "$(cat bigfile)" -o text

# GOOD — stdin for content, -p for instruction
cat content.txt | gemini -p "Short instruction here" -o text
```

The gemini CLI docs say `-p` is "appended to input on stdin" — this is the intended pattern. Content on stdin, instruction via `-p`.

**Rule: Always test CLI tools at production-scale input sizes. Silent corruption is worse than a crash.**

---

## Lesson 12: Shell Helper Functions Have Hidden Input Limits

**What happened:** The `deepseek()` shell function used `read -r` to accept stdin when no positional argument was given. The DeepSeek architecture review hook piped a multiline prompt via `echo "$PROMPT" | deepseek`. `read -r` only reads ONE line — the review model received "Review this diff to architecture-sensitive files in a voice AI system. Check for:" and nothing else. The actual diff, instructions, and output format were silently dropped.

**Why:** `read -r` is designed for single-line input. It stops at the first newline. For multiline content, `cat` or `mapfile` is needed. The function worked in quick tests (single-line prompts) but broke on real multi-line review prompts.

**Specific pattern:**
```bash
# BAD — read -r drops everything after first newline
my_model() {
  local prompt="$1"
  [[ -z "$prompt" ]] && { read -r prompt; }  # ← only gets line 1
  call_api "$prompt"
}

# GOOD — cat reads all stdin
my_model() {
  local prompt="$1"
  [[ -z "$prompt" ]] && { prompt=$(cat); }  # ← gets everything
  call_api "$prompt"
}
```

**Broader lesson:** Each tool in the chain has its own input method with its own limits:
- Shell args → 128KB (ARG_MAX)
- `read -r` → first line only
- gemini `-p` flag → garbles at >20KB
- Codex CLI → can read files from sandbox
- curl JSON body → no practical limit (HTTP POST)

**Rule: Document and test the input method for every external model call. "It works in quick tests" ≠ "it works in production."**

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
- [ ] **Seam audit after multi-file changes** — trace call chains across module boundaries
- [ ] **Lessons generalized, not just patched** — audit existing code for same anti-pattern
- [ ] **Async chain verification** — if any leaf is async, the full call chain must be async
- [ ] **CLI tools tested at scale** — test with production-sized inputs, not toy examples

---

## Cost of Not Doing This

### Session 1: Package F (6-hour session, medium-complexity feature)

| Bug | Severity | Root Cause | Would Have Caught |
|-----|----------|-----------|-------------------|
| Wrong dict key (`person_id` vs `id`) | HIGH | Agent guessed key name | Contract test, mypy, external review |
| Dataclass treated as dict (`.get()`) | HIGH | Agent didn't read model | mypy, contract test, external review |
| Removed Pydantic fields still referenced | HIGH | Incomplete cleanup | mypy, external review |
| Missing auth on endpoint | HIGH | Agent didn't check patterns | External review |
| Silent exception swallowing | MEDIUM | Defensive coding without logging | External review |
| Conflicting instruction rules | MEDIUM | Prompt rewrite inconsistency | External review |

### Session 2: v8 State Machine (single session, 6-file feature)

| Bug | Severity | Root Cause | Would Have Caught |
|-----|----------|-----------|-------------------|
| Sync function can't await async dep | HIGH | Function wasn't made async | Seam audit |
| `.format()` template malformed JSON | HIGH | Quadruple braces instead of double | Seam audit, template render test |
| Missing match_suggestions fetch | HIGH | v8 path didn't call existing fetch function | Seam audit |
| Regex case sensitivity (3 instances) | MEDIUM | Patterns had uppercase `I`, input was lowercased | External review (Codex caught 2) |
| Type annotation mismatch | LOW | `DialogStage` annotation, `ConversationStage` passed | mypy, seam audit |

**Key finding:** Session 2 had all the gates from Session 1 in place (Codex review, pytest, DeepSeek arch review). The 3 HIGH bugs still survived — they required a manual seam audit to find. External review is necessary but insufficient for multi-file changes.

All bugs were caught before reaching production.
