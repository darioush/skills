# nearcore Common Anti-patterns Reference

Each anti-pattern includes a brief description and real examples from PRs. These are the mistakes most frequently caught in code review.

---

## Naming & Semantics

### Misleading Names Causing Bugs
Names that do not reflect actual behavior lead to incorrect usage at call sites.

- PR #14906: `get_next_next_epoch_protocol_version` only called `get_next_epoch_id` once instead of twice. The name promised "next_next" but the implementation delivered "next".
- PR #13359: Method names reflecting historical behavior rather than current behavior after refactoring. A reviewer flagged the name as a blocking issue because it would mislead future callers.

### Stale Names After Refactoring
When behavior changes but names stay the same, the mismatch creates latent bugs.

- PR #12734: Function name no longer matched what it did after a refactor. The stale name was flagged as blocking.
- PR #14353: Comments and variable names not updated to match new code semantics after refactoring.

### Ambiguous Boolean Parameters
Bare `true`/`false` at call sites convey no meaning.

- PR #13577: Boolean parameter replaced with enum for clarity in resharding code.
- PR #14013: Reviewer requested enum over boolean for return types representing more than two states.

---

## Serialization & Protocol

### Breaking Borsh Enum Ordering
Adding a variant in the wrong position breaks all deserialization across the network.

- PR #12737: A variant inserted in the middle of a Borsh enum would have broken all serialization. Caught in review.
- PR #13440: New Borsh-serialized enum variant not appended at the end. Reviewer flagged as a protocol-breaking change.

### Removing Fields from Persisted Structs
Breaks deserialization of existing data on disk or in cloud storage.

- PR #14966: `global_actions_burnt_amount` removed from `BalanceStats`. Existing persisted data would fail to deserialize.

### Accidental Protocol Changes
Features accidentally stabilized or protocol behavior changed during cleanup/refactoring PRs.

- PR #13602: Feature accidentally stabilized by setting it to `false` at a version instead of removing it entirely.
- PR #13402: What appeared to be a cleanup PR silently changed protocol-visible behavior.

---

## Consensus & Determinism

### HashMap in Consensus-Adjacent Code
HashMap iteration order is non-deterministic. Using it in consensus or state-processing code risks validators diverging.

- PR #13395: HashMap used in code near state processing. Reviewer required BTreeMap for deterministic ordering.
- PR #13369: HashMap in consensus-adjacent code flagged. Replaced with BTreeMap.

### Fork-Unsafe Code
Code that assumes blocks are final or does not handle fork switching.

- PR #14880: Code reading historical chain data did not handle forks correctly. `last_final_block` does not cover all finalized blocks.
- PR #12720: Code did not distinguish between final and non-final blocks. Non-final blocks may be on forks that are eventually abandoned.

### Non-Deterministic Processing Order
Transaction/receipt ordering changes that break protocol compatibility.

- PR #12983: Transaction processing order changed during refactoring, which could break protocol compatibility with older nodes.

---

## Error Handling

### `debug_assert` for Production Invariants
`debug_assert` is compiled away in release mode. Using it for production invariants means production silently returns invalid data.

- PR #14395: `debug_assert` used with a dummy return value. In release builds, the assert disappears and the function silently returns wrong data.
- PR #13369: Reviewer noted that if a condition should never fail, use `assert!`; if genuinely expensive, use `debug_assert!`; if documentation-only, use a comment.

### Silent Error Drops
Errors swallowed during refactoring, changing `?` propagation to `unwrap_or_default()`.

- PR #14699: `get_ser(...)?` wrapped in a new function that silently defaulted on error. Storage errors should fail fast, not return defaults.
- PR #13137: `.ok()` and `let _ = ...` used without logging, silently discarding errors.

### Wrong Error Types Hiding Failures
Changing error types during refactoring can change how callers handle failures.

- PR #13350: `InvalidChunk` error changed to `IOError` during refactoring, altering retry behavior.
- PR #12886: `Err(RuntimeError)` used instead of `result.result = Err(ActionErrorKind)`, failing the entire chunk instead of just one action.

### `unreachable!()` in Reachable Code
Using `unreachable!()` in message handlers triggered by external messages crashes the node.

- PR #13637: `unreachable!()` placed in a handler for external messages. When the message arrived, the node crashed.

---

## Refactoring Hazards

### Removing Validation During Refactoring
Validation logic accidentally dropped during code restructuring.

- PR #13892: Pre-validation reordered during refactoring, allowing trust-dependent actions before validation completed. Caught as a security bug.
- PR #14142: Signature validation accidentally dropped during refactoring.

### Changing Semantics Silently
Refactoring PRs that silently alter behavior.

- PR #13362: Converting `fn -> impl Future` to `async fn` changed when side effects execute (eager vs lazy evaluation).
- PR #13783: Adding `Deref` implementation that bypassed intentional type-safety barriers.
- PR #13575: Configuration default values changed inside a refactoring PR without callout.

### Lost Metrics During Refactoring
Metrics accidentally dropped during code reorganization.

- PR #12983: Pre-existing metrics stopped being recorded after a refactoring PR. Not caught by tests because metric recording is typically not asserted.

### Breaking Defaults
Default values changed during refactoring without explicit documentation.

- PR #13575: Config values changed during a refactoring PR without explicit callout. Reviewers caught the default change that would have altered production behavior.

---

## Debug & Cleanup

### Leftover Debug Code
Debug statements, markers, and print calls left in production code.

- PR #14963: `//println!` commented-out debug print left in code.
- PR #14232: `if key.1 == ShardId::new(3) && key.2 == 0` -- hardcoded debug log with specific shard ID left in.

### Commented-Out Code
Code commented out instead of deleted. Creates confusion about whether it is intentional.

- PR #13708: Commented-out code left behind during refactoring.
- PR #14782: Commented-out imports left in the file.

### Debug Artifacts (`dbg!`, `print`)
`dbg!` macros and `println!` statements left in production code.

- PR #12627: `dbg!` macro shipped in production code.
- PR #12969: Print statements left in production paths.

---

## Copy-Paste & Duplication

### Wrong Variable After Paste
Copy-pasting code and failing to update all variable references.

- PR #14475: `block` used instead of `cur_block` for fetching chunks after a copy-paste. This was a real bug causing incorrect indexer data.

### Copy-Pasted Span Names / Metric Descriptions
Tracing span names or metric descriptions copied from another function without updating.

- PR #14213: Span name copied from another function, making traces misleading and hard to debug.

### Divergent Test Helpers
Test helpers that behave differently from production code, making tests unreliable.

- PR #15033: `set_tx_state_changes` test helper diverged from actual runtime behavior, meaning tests passed but production code would not work the same way.

### Duplicated Logic Across Files
Fee calculations, validation logic, or other business logic duplicated in multiple places.

- PR #12965: Fee/logic calculations duplicated across files. Changes to one copy would not propagate to the other.
- PR #12983: Duplicate test code for complex/critical code paths flagged as a blocking issue.

---

## Testing

### Weak Assertions
Tests that do not assert meaningful invariants.

- PR #13913: Test had no meaningful assertions -- just ran code without checking results.
- PR #13422: `assert!(x.is_empty())` used instead of `assert_eq!(x, vec![])`, giving poor diagnostic output on failure.
- PR #14715: `is_err()` asserted without checking the specific error type. The test would pass even if the error was for a different reason.

### Non-Deterministic Seeds
Using `thread_rng()` instead of deterministic RNG with hardcoded seed. Failures become unreproducible.

- PR #14883: Test used `thread_rng()`. When it failed, the failure could not be reproduced because the seed was random.
- PR #14545: Same pattern. Reviewer required hardcoded seed for reproducibility.

### Deprecated Integration-Test Usage
Using the deprecated `integration-tests` crate and `TestEnv` instead of the test-loop framework.

- All 20 batches of review data confirm that integration tests are deprecated. New end-to-end tests must use `TestLoopBuilder`, `TestLoopNode`, `TestLoopEnv`.

### Testing Only Happy Path
Tests that exercise only the success case without testing error conditions or edge cases.

- PR #15002: Tests with `|| unreachable!()` but no companion test exercising the reachable error path.
- PR #14474: Tests that did not assert preconditions before testing postconditions, making it unclear what was actually being validated.

### Using `pop` on LRU Cache Instead of `get`
Removes entries from cache on first access when multiple accesses are needed.

- PR #13637: `pop` used on LRU cache, removing the entry. Subsequent accesses found it missing.

---

## Configuration

### Hardcoded Values That Should Be Configurable
Values hardcoded in logic that should be configuration parameters.

- PR #14168: Non-obvious constants embedded in code without explanation or configurability.
- PR #14232: Hardcoded shard ID in a debug log that should have been parameterized or removed.

### Config Changes Buried in Refactors
Default value changes or new configuration fields hidden in larger refactoring PRs.

- PR #13575: Configuration default values changed inside a refactoring PR without explicit callout in the PR description.
- PR #13488: Config changes bundled with logic changes, making rollback difficult.

### Non-Functional Defaults
Default values that do not actually work or cause unexpected behavior.

- PR #14239: Reviewer asked "Can we simplify? Do we even need this config field?" -- unnecessary config complexity with defaults that added no value.
- PR #13154: Deprecated config fields without a deprecation plan (warn -> panic -> remove), leaving non-functional configuration in place indefinitely.

### Missing Backward Compatibility
Config changes that break existing node operator deployments.

- PR #13219: Config field renamed without serde alias, breaking existing config files.
- PR #13154: Missing backward-compatibility parsing test for old config format.
