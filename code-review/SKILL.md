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
For each file, check:
- **Logic & Correctness** -- Edge cases, off-by-one, null checks, race conditions, use of protocol features.
- **Naming Precision** -- Names must be self-documenting and domain-precise. See [nearcore Patterns](nearcore-patterns.md) for conventions.
  - [ ] Variable/field names encode meaning, not abbreviation (`duration` not `dur`, `tokens_burnt` not `amount_burnt`).
  - [ ] Method names reflect computational complexity (`find_split` not `current_split` for non-trivial logic).
  - [ ] Boolean config fields use verb prefixes (`enable_early_prepare_transactions` not `early_prepare_transactions`).
  - [ ] Domain terminology is precise ("final" means "finalized by consensus", not "last").
  - [ ] Stale names updated after refactoring -- method names must reflect current behavior, not historical behavior (PR #13359).
- **Error Handling** -- nearcore uses a tiered strategy:
  - [ ] `debug_assert` + error return for "should never happen" (crashes canaries, production survives).
  - [ ] `tracing::error!` + graceful degradation for bugs where crashing is worse than continuing.
  - [ ] `panic!` only for truly unrecoverable conditions or test code. Never in protocol-critical paths.
  - [ ] `Result<(), Error>` over `bool` for fallible operations (compiler warns on ignored Results).
  - [ ] No `as T` casts on numeric types -- use `u128::from()`, `try_into().unwrap()`, or `.into()`.
  - [ ] Overflow handling intentional: `checked` for protocol paths, `saturating` only where explicitly safe.
  - [ ] Runtime error classification correct: `ActionErrorKind` for user-facing failures, `RuntimeError` for internal errors (PR #12886).
- **Determinism** -- See [Security Checklist](security-checklist.md).
  - [ ] No `HashMap` in consensus-adjacent or state-processing code (use `BTreeMap` or `Vec`).
  - [ ] Transaction/receipt ordering unchanged or explicitly gated behind protocol version.
- **Serialization Safety** -- See [Security Checklist](security-checklist.md).
  - [ ] Borsh enum variants added at the END only.
  - [ ] Old versioned structs (V1, V2) not modified.
  - [ ] New serde config fields have `#[serde(default)]`.
- **Security** -- For protocol-critical code, apply the full [Security Checklist](security-checklist.md).
- **Performance** -- Unnecessary loops, memory leaks, tracing overhead in hot paths.
- **Maintainability** -- Clear names, single responsibility, comments.
- **Partially mutated state and side effects on early return** -- If there is a return statement before the end of the function, check if there are any mutations to state that could cause confusion or bugs. Especially if the function returns an error and the caller could assume no changes. If so, suggest refactoring to avoid this pattern.

### Phase 4: Minimal Diff Review
For each file and diff chunk, check:
- [ ] The change compared to the prior version is minimal and necessary. If not, suggest adherence to the original.
- [ ] The diff does not remove original comments, documentation, or tests without good reason. If it does, suggest restoring them.
- [ ] No leftover debug code: `dbg!`, `println!`, `//println!`, hardcoded debug shard IDs, debug markers (PR #14586: `hierwasik`), commented-out imports (PR #14782).
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
- [ ] Test-loop (`TestLoopBuilder`, `TestLoopNode`, `TestLoopEnv`) is used for new end-to-end tests. Integration tests (`integration-tests/`, `TestEnv`) are deprecated.
- [ ] Test setup is minimal: fewest accounts, shards, validators needed. Use existing helpers: `create_account_ids`, `create_validators_spec`, `epoch_config_store_from_genesis()`.
- [ ] Set configuration in genesis, then use `TestEpochConfigBuilder::build_store_from_genesis`. Do not override epoch config directly.
- [ ] Use RPC nodes for querying state (not producers tracking all shards).
- [ ] One test function per test case for easier debugging (PR #14377).
- [ ] Assertions are strong and descriptive:
  - [ ] `assert_eq!` over `assert!(x.is_empty())` for better failure messages.
  - [ ] `assert_matches!` for pattern-matching assertions.
  - [ ] Assert specific error types, not just `is_err()`.
  - [ ] Assert preconditions hold before testing postconditions.
  - [ ] Use expressions (`txs.len()`) not hardcoded magic numbers in assertions.
- [ ] Tests are deterministic: hardcoded seeds, not `thread_rng()` (PR #14883).
- [ ] Semantic stop conditions preferred over fixed block counts (PR #14474).
- [ ] Block-count progression preferred over time-based progression (PR #13886).
- [ ] Tests use same APIs as production code, not internal trie access.
- [ ] Use `ProtocolFeature` constants, not hardcoded version numbers (PR #13881).
- [ ] Bug fixes include regression tests that fail without the fix (red-green testing).
- [ ] Protocol upgrade tests use `sequential` schedule (PR #13392).
- [ ] Document magic numbers with derivation math.
- [ ] Test infrastructure not deleted just because temporarily unused -- use `#[allow(unused)]` (PR #13776).

### Phase 6: Code Consistency Review
- Check if the code is consistent with the surrounding code and the overall codebase. Flag any inconsistencies and suggest making the code consistent.
- [ ] `BTreeMap` over `HashMap` in consensus-adjacent code (determinism requirement).
- [ ] Early returns with `let Some(x) = ... else { continue/return };` over nested `if let` (PR #14997, PR #13527).
- [ ] Structured tracing: explicit `target:`, key-value fields (`?var`, `%var`), not format string interpolation. Fully qualified `tracing::debug!` rather than importing the macro.
- [ ] Log messages lowercase (but preserve type names/acronyms like `ChunkStateWitness`, `S3`).
- [ ] Present participle for ongoing actions: "processing block" not "process block".
- [ ] `#[instrument]` over manual `debug_span!` (PR #14105).
- [ ] Enums over booleans for parameters where meaning is ambiguous at call site (PR #13577).
- [ ] Exhaustive match arms on enums (no `_ =>` wildcards on growing enums).
- [ ] `expect("reason")` over bare `unwrap()` to document safety assumptions.
- [ ] Named constants for magic numbers with type annotations.
- [ ] Named variables for boolean arguments at call sites.
- [ ] Domain type aliases used (`BlockHeight`, `ShardId`, `Gas`, `Balance`).
- [ ] `parking_lot::Mutex`/`RwLock` over `std::sync` equivalents.
- [ ] `is_some_and()`, `is_none_or()` over `map_or()` chains.
- [ ] `itertools` usage where appropriate: `exactly_one()`, `collect_vec()`.
- [ ] Minimize visibility: `pub(crate)` when possible, no `pub` unless needed externally.
- [ ] Import from canonical crate, not re-exports from unrelated crates.

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

**Protocol Feature Gating:**
- [ ] New protocol behavior gated behind `ProtocolFeature::X.enabled(protocol_version)` at runtime?
- [ ] Compile-time `cfg!(feature = ...)` used ONLY for non-consensus code (GC, tooling)?
- [ ] Blocks/messages using unreleased features rejected when feature is not enabled?
- [ ] `latest_protocol_version` (vote) not confused with current epoch protocol version?
- [ ] Old code path preserved intact alongside new code for protocol changes?

**Serialization & Backward Compatibility:**
- [ ] Borsh enum variants added at END (before unstable/nightly variants)?
- [ ] Old versioned structs (V1, V2) not modified?
- [ ] Deprecated enum variants use `_UnusedXxx` naming?
- [ ] Protocol-visible structs derive `ProtocolSchema`?
- [ ] Signed structs use `SignatureDifferentiator` and `#[borsh(init=init)]`?
- [ ] New serde config fields have `#[serde(default)]`?

**Storage & GC:**
- [ ] New header/chunk fields minimize storage overhead?
- [ ] DB migrations handle archival vs non-archival, hot vs cold storage?
- [ ] New code does not depend on data that may have been garbage collected?
- [ ] Read-modify-write patterns in DB are thread-safe?

**Fork Safety & Edge Cases:**
- [ ] Code handles blockchain forks correctly (non-final blocks may be abandoned)?
- [ ] Behavior correct at resharding_block +/- 1?
- [ ] Correct epoch's shard layout used (current vs previous vs next)?
- [ ] `last_final_block` vs `head` vs `prev_hash` used correctly?
- [ ] Missing block/chunk cases handled (`height_created` vs `height_included`)?
- [ ] All execution paths (chunk production, state witness validation, state sync, replay) behave consistently?

**Gas & Compute:**
- [ ] Gas values use `Gas` type, not raw `u64`?
- [ ] Overflow handling intentional: `checked` for protocol, `saturating` only where safe?
- [ ] No `as u64`/`as u128` casts on gas values?
- [ ] Placeholder costs for unstabilized features set prohibitively high?

**Deployment Safety:**
- [ ] Activation timing: when does the feature take effect relative to protocol upgrade epoch?
- [ ] Canary safety: could errors crash canary nodes or leak to production?
- [ ] GC compatibility: does new code depend on data that may have been garbage collected?
- [ ] Indexer impact: breaking changes documented in main CHANGELOG?

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
