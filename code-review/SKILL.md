---
name: code-review
description: This skill performs a comprehensive code review of the requested code.
disable-model-invocation: true
---
## Core Principles

### 1. The Review Mindset

**Goals of Code Review:**
- Catch bugs and edge cases
- Ensure code maintainability
- Share knowledge across team
- Enforce coding standards
- Improve design and architecture
- Build team culture

**Not the Goals:**
- Show off knowledge
- Nitpick formatting (use linters)
- Block progress unnecessarily
- Rewrite to your preference

### 2. Effective Feedback

**Good Feedback is:**
- Specific and actionable
- Educational, not judgmental
- Focused on the code, not the person
- Balanced (praise good work too)
- Prioritized (critical vs nice-to-have)

```markdown
Bad: "This is wrong."
Good: "This could cause a race condition when multiple users
       access simultaneously. Consider using a mutex here."

Bad: "Why didn't you use X pattern?"
Good: "Have you considered the Repository pattern? It would
       make this easier to test. Here's an example: [link]"

Bad: "Rename this variable."
Good: "[nit] Consider `userCount` instead of `uc` for
       clarity. Not blocking if you prefer to keep it."
```

### 3. Review Scope

**What to Review:**
- Logic correctness and edge cases
- Security vulnerabilities (see [Security Checklist](security-checklist.md))
- Performance implications
- Test coverage and quality
- Error handling
- Documentation and comments
- API design and naming
- Architectural fit

**Reference Guides:**
- [Security Checklist](security-checklist.md) -- critical security review items
- [nearcore Patterns](nearcore-patterns.md) -- domain knowledge reference
- [Common Anti-patterns](anti-patterns.md) -- mistakes to watch for

---

## Review Process

### Phase 1: Context Gathering
Before diving into code, understand:
1. Ensure you understand which code is being reviewed (diff with master, specific PR, latest commit, staged changes, etc.) If there is any ambiguity here ASK AND ABORT.
2. Understand related documentation, requirements, and design decisions.
3. Note any relevant architectural decisions.
4. Determine if this PR touches protocol-critical code (consensus, serialization, state transitions, gas accounting). If so, apply the [Security Checklist](security-checklist.md).

### Phase 2: High-Level Review
1. **Architecture & Design** -- Does the solution fit the problem?
   - For significant changes.
   - Check: SOLID principles, coupling/cohesion, anti-patterns.
   - Does the change belong in this crate? (`core/primitives` is for data structures; business logic goes elsewhere.)
2. **Performance Assessment** -- Are there performance concerns?
   - For performance-critical code.
   - Check: Algorithm complexity, unnecessary clones/allocations, tracing spans in hot paths (PR #14448), metric label pre-resolution.
3. **File Organization** -- Are new files in the right places?
4. **Testing Strategy** -- Are there tests covering edge cases?
5. **PR Scope** -- Is the PR focused on a single concern? Refactoring should not be mixed with functional changes. Config changes should be separate from logic changes for easy rollback.

### Phase 3: Line-by-Line Review
Review the code **as written** — is it correct, safe, and well-structured?
For each file, check:
- **Logic & Correctness** -- Edge cases, off-by-one, null checks, race conditions, use of protocol features.
- **Naming Precision** -- Names must be self-documenting and domain-precise. See [nearcore Patterns](nearcore-patterns.md) for conventions.
  - [ ] Variable/field names encode meaning, not abbreviation (`duration` not `dur`, `tokens_burnt` not `amount_burnt`).
  - [ ] Method names reflect computational complexity (`find_split` not `current_split` for non-trivial logic).
  - [ ] Boolean config fields use verb prefixes (`enable_early_prepare_transactions` not `early_prepare_transactions`).
  - [ ] Domain terminology is precise ("final" means "finalized by consensus", not "last").
  - [ ] Stale names updated after refactoring -- method names must reflect current behavior, not historical behavior (PR #13359).
- **Error Handling** -- Apply the error handling checklists in [Security Checklist](security-checklist.md) and [nearcore Patterns](nearcore-patterns.md). Key concern: nearcore uses a tiered strategy (`debug_assert` + error return for canary detection, `tracing::error!` + degradation, `panic!` only for unrecoverable).
- **Determinism** -- Apply the consensus & determinism checklist in [Security Checklist](security-checklist.md).
- **Serialization Safety** -- Apply the serialization checklists in [Security Checklist](security-checklist.md) and [nearcore Patterns](nearcore-patterns.md).
- **Security** -- For protocol-critical code, apply the full [Security Checklist](security-checklist.md).
- **Performance** -- Unnecessary loops, memory leaks, tracing overhead in hot paths.
- **Maintainability** -- Clear names, single responsibility, comments.
- **Partially mutated state and side effects on early return** -- If there is a return statement before the end of the function, check if there are any mutations to state that could cause confusion or bugs. Especially if the function returns an error and the caller could assume no changes. If so, suggest refactoring to avoid this pattern.

### Phase 4: Diff Hygiene Review
Review the **delta** — is the diff clean, minimal, and free of accidental side effects? This complements Phase 3 by focusing on what changed rather than the code's absolute quality.
For each file and diff chunk, check:
- [ ] The change compared to the prior version is minimal and necessary. If not, suggest adherence to the original.
- [ ] The diff does not remove original comments, documentation, or tests without good reason. If it does, suggest restoring them.
- [ ] No leftover debug code: `dbg!`, `println!`, `//println!`, hardcoded debug shard IDs, debug markers, commented-out imports (PR #14782).
- [ ] No configuration default value changes buried in refactors (PR #13575). Config changes belong in separate PRs.
- [ ] No removed validation logic during refactoring. Validation removal must be explicit and justified (PR #13892, PR #14142).
- [ ] Semantic preservation: refactoring PRs must not change behavior. Watch for:
  - Log level changes (separate PR) (PR #14603).
  - Error type/variant changes (e.g., `InvalidChunk` becoming `IOError`) (PR #13350).
  - `fn -> impl Future` to `async fn` conversion changing when side effects execute (PR #13362).
  - Adding `Deref` implementations that bypass intentional type-safety barriers (PR #13783).
- [ ] No metrics accidentally dropped during refactoring (PR #12983).
- [ ] No previously-removed clones reintroduced (PR #12983).

### Phase 5: Testing Coherence Review
Apply the testing checklist in [nearcore Patterns](nearcore-patterns.md). Key review questions:
- Is the test framework correct? (test-loop for new e2e tests, not deprecated `integration-tests`/`TestEnv`)
- Is the test setup minimal and configuration set through genesis?
- Are assertions strong, specific, and descriptive? (`assert_eq!` over `assert!`, specific error types over `is_err()`)
- Are tests deterministic? (hardcoded seeds, semantic stop conditions, block-count over time-based progression)
- Do bug fixes include regression tests that fail without the fix?

### Phase 6: Code Consistency Review
Check if the code is consistent with the surrounding code and the overall codebase. Flag any inconsistencies and suggest making the code consistent.
Apply the Rust idioms and tracing conventions in [nearcore Patterns](nearcore-patterns.md). Key areas:
- Control flow: early returns, flat structure, exhaustive match arms.
- Type safety: enums over booleans, domain type aliases, `expect()` over `unwrap()`.
- Tracing: explicit `target:`, structured key-value fields, `#[instrument]` over manual spans, lowercase messages.
- Visibility: `pub(crate)` by default, import from canonical crates.

### Phase 7: Comment Consistency Review
- Check if the code comments are consistent with the code. If there are any inconsistencies, suggest updating the comments to match the code or vice versa. This helps ensure that comments remain accurate and useful for future readers.
- Check surrounding code, top of functions, file headers, for comments that may be affected by the change. If there are any changes that should be made to these comments, suggest them.
- [ ] Stale comments updated after refactoring -- surrounding comments must be updated when code changes (PR #14353, PR #13926).
- [ ] TODOs tagged with feature/team identifier: `TODO(resharding)`, `TODO(gas-keys)`, `TODO(#14005)` (PR #14405).
- [ ] Non-obvious constants and configuration values have explanatory comments (PR #14168, PR #13532).
- [ ] Panics justified with comments explaining why they are safe.
- [ ] Comments explain "why" not "what" -- remove comments that merely restate the code.
- [ ] Doc comments on public items follow rustdoc conventions: one-sentence summary, blank line, then details.
- [ ] Counterintuitive test setups explained with comments (PR #13598).

### Phase 8: Summary & Decision
1. Summarize key concerns
2. Highlight what you liked
3. Make clear decision:
   - ✅ Approve
   - 💬 Comment (minor suggestions)
   - 🔄 Request Changes (must address)

---

## Review Techniques

### Technique 1: The nearcore Checklist

For protocol-critical PRs, apply the full checklists in the reference guides:

- [Security Checklist](security-checklist.md) -- protocol safety, consensus & determinism, serialization, validation, state mutation, error handling, gas & overflow safety.
- [nearcore Patterns](nearcore-patterns.md) -- protocol versioning, Borsh rules, resharding, epoch management, gas accounting, storage layer, runtime errors, actor patterns, tracing, configuration, testing.

Focus on the sections relevant to the PR's scope. Not every section applies to every PR.

### Technique 2: The Question Approach

Instead of stating problems, ask questions:

```markdown
Bad: "This will fail if the list is empty."
Good: "What happens if `items` is an empty array?"

Bad: "You need error handling here."
Good: "How should this behave if the API call fails?"
```

### Technique 3: Suggest, Don't Command

Use collaborative language:

```markdown
Bad: "You must change this to use async/await"
Good: "Suggestion: async/await might make this more readable. What do you think?"

Bad: "Extract this into a function"
Good: "This logic appears in 3 places. Would it make sense to extract it?"
```

### Technique 4: Differentiate Severity

Use labels to indicate priority:

- 🔴 `[blocking]` - Must fix before merge
- 🟡 `[important]` - Should fix, discuss if disagree
- 🟢 `[nit]` - Nice to have, not blocking
- 💡 `[suggestion]` - Alternative approach to consider

### Technique 5: Concrete Counter-Examples

When reviewing algorithms handling sequences (blocks, epochs, heights), construct concrete examples with edge cases:

- "Consider block sequence: B1, B2, B3(1), X4, B5(1), X6, B7(1), B8(1) where X=missing. With estimated epoch start at 9, is_next_epoch_after_n_blocks(B8, 2) would incorrectly return false."
- "What happens during protocol upgrade at epoch 150? Will the new shard layout become effective in 150 or 152?"
- "What if the node is an RPC node and not a validator?"
- "What if the block producer missed this height?"
- "Does this work for nodes that don't track this shard?"

Use "sanity check:" prefix to verify your understanding and invite the author to confirm or correct your mental model.

### Technique 6: Style Guide

- Read [style guide](<project-root>/docs/practices/style.md) if available before reviewing, but adherence to this is somewhat loose. Check nearby code for examples of style. If there is a clear style in the codebase, follow it. If not, use your best judgment. Do not block on style issues that don't cause confusion or bugs.
