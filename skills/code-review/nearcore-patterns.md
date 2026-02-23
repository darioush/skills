# nearcore Domain Knowledge Reference

This document captures domain-specific patterns, conventions, and knowledge required for effective code review of nearcore.

---

## Protocol Versioning & Feature Gating

Protocol features control the activation of new consensus-visible behavior. Misuse can cause consensus-breaking bugs or allow malicious exploitation.

**How it works:**
- Each protocol feature is defined in `ProtocolFeature` enum with an associated protocol version number.
- At runtime, `ProtocolFeature::X.enabled(protocol_version)` gates new behavior based on the epoch's active protocol version.
- `latest_protocol_version` is a node's vote for the next protocol version -- it is NOT the active version. Confusing them causes consensus-breaking bugs (PR #14906).

**Checklist:**
- [ ] New consensus-visible behavior gated behind `ProtocolFeature::X.enabled(protocol_version)` at runtime.
- [ ] Compile-time `cfg!(feature = ...)` used ONLY for excluding code from production builds entirely (GC, tooling, test-only). Never for consensus behavior (PR #14913).
- [ ] Old code path preserved intact alongside new code. The caller selects based on protocol version (PR #13418).
- [ ] Blocks/messages using unreleased features are rejected by validators (PR #13970).
- [ ] No accidental protocol stabilization (setting a feature to `false` at a version instead of removing it) (PR #13602).
- [ ] Long-since-enabled protocol features cleaned up: remove the else branch, deprecate the flag.
- [ ] Archival nodes can still replay blocks from old protocol versions (PR #13435).

---

## Borsh Serialization & Backward Compatibility

Borsh uses positional encoding. Field/variant ordering is part of the protocol. Mistakes here break all deserialization.

**Rules:**
- [ ] New enum variants appended at the END only. Never insert in the middle (PR #12737, PR #13440).
- [ ] Deprecated variants prefixed with `_Unused` or `_Deprecated`. Never removed (PR #13340, PR #13835).
- [ ] Old versioned structs (V1, V2) never modified. Create a new version instead (PR #14013).
- [ ] Protocol-visible structs derive `ProtocolSchema` (PR #12730).
- [ ] Use `#[borsh(use_discriminants)]` and `#[repr(u8)]` on protocol enums (PR #13867).
- [ ] Computed fields (like `hash`) marked `#[borsh(skip)]` (PR #12730).
- [ ] Hash/signature fields recomputed on deserialization via `#[borsh(init=init)]` -- receivers must never trust the sender's values (PR #12730).
- [ ] Signed structs include `SignatureDifferentiator` (placed at end) to prevent cross-type signature reuse (PR #12730).
- [ ] Field renames are safe (Borsh uses positional encoding), but type or ordering changes are not (PR #13185).
- [ ] Removing fields from persisted structs breaks deserialization of existing data on disk or cloud (PR #14966).

**Serde (JSON configs and RPC views):**
- [ ] New optional config fields have `#[serde(default)]` (PR #14760).
- [ ] New optional fields use `#[serde(default, skip_serializing_if = "Option::is_none")]` (PR #12877).
- [ ] External API views (`AccountView`, etc.) remain backward compatible.
- [ ] Avoid `#[serde(untagged)]` on enums -- causes ambiguity and issues with OpenAPI clients (PR #14307).
- [ ] Use serde aliases for renamed config fields to maintain backward compatibility (PR #13219).

---

## Resharding & Shard Layout

Resharding is the most review-intensive domain area. Shard identity concepts are frequently confused.

**Key distinctions:**
- `shard_id` -- logical shard identifier.
- `shard_uid` -- unique shard identifier (includes version).
- `shard_index` -- position in the current shard layout.
- During resharding, a chunk_header's `shard_id` may refer to the parent shard, not the child shard at the current index (PR #13783).

**Epoch gap pattern:** E0 tracks parent, E1 tracks nothing (gap epoch), E2 tracks child (PR #13695).

**Checklist:**
- [ ] Correct epoch's shard layout used (current vs previous vs next). Using `get_epoch_id` when `get_next_epoch_id` is needed at boundary blocks is a common bug (PR #13604, PR #13525).
- [ ] `shard_id` from chunk header vs shard layout correctly distinguished. These differ during resharding (PR #13783).
- [ ] Old chunks (from previous heights) handled separately from new chunks -- they carry data from potentially different protocol versions (PR #13783).
- [ ] Parent-child shard relationships correct during first block after resharding.
- [ ] Iterator/chain ordering consistent with parent-buffer-first processing (PR #12728).
- [ ] Behavior correct at `resharding_block +/- 1` (PR #14185).
- [ ] Only one resharding per epoch; `max_number_of_shards` takes precedence (PR #14856).
- [ ] `min_epochs_between_resharding` must be less than `gc_num_epochs_to_keep` (PR #14906).
- [ ] Use `ShardLayout::multi_shard` instead of deprecated `ShardLayout::v0` (PR #13000).
- [ ] Resharding-specific naming: `parent_shard_uid` / `child_shard_uid` prefixes (PR #13577).

---

## Epoch Management & Boundaries

Epoch boundary logic is pervasive and error-prone. The `next`/`next_next` naming convention creates confusion.

**Key concepts:**
- `next_next_epoch_id` is the hash of the last block of the current epoch, which may not exist during block production.
- "final" means "finalized by consensus", never "last" (PR #14504).
- Use `last_final` not `last_finalized` (PR #14906).
- `block_height` not just `height` for block height parameters (PR #14906).

**Checklist:**
- [ ] `get_next_epoch_id` vs `get_next_next_epoch_id` called correctly. Calling `get_next_epoch_id` once when `get_next_next_epoch_id` is needed is a common bug (PR #14906).
- [ ] `prev_block_hash` and `block_hash` map to different epochs at boundary blocks.
- [ ] Code handles missing blocks at epoch boundaries.
- [ ] `cares_about_shard` semantics preserved: the `is_me` flag controls whether tracked_config is checked.
- [ ] Activation timing clear: when does the feature take effect relative to protocol upgrade epoch (N, N+1, N+2)?

---

## Gas Accounting & Compute Costs

Gas values are protocol-critical. Silent overflow or incorrect accounting can break consensus.

**Rules:**
- [ ] Gas values use the `Gas` type, not raw `u64` (PR #13964).
- [ ] Use highest appropriate unit constructors: `Gas::from_tgas(100)` not `Gas::from_gas(10u64.pow(14))`.
- [ ] Compute and Gas are distinct units (PR #13964).
- [ ] Overflow handling: `checked` for protocol paths, `saturating` only where explicitly safe, never `wrapping` for gas.
- [ ] No `as u64`/`as u128`/`as u32` casts on gas values -- use `u128::from()`, `try_into().unwrap()`, or `.into()`.
- [ ] New action costs need proper send/exec fee calculations. Set to safe defaults with `TODO(feature-tag)` (PR #14532).
- [ ] Placeholder costs for unstabilized features set prohibitively high (e.g., `999_999_999_999_999_999_999_999_999`) (PR #12882).
- [ ] Gas parameter changes validated with real mainnet transaction data (PR #14046).
- [ ] Use `Balance::ZERO` and `Balance::is_zero()` (PR #14235).

---

## State Witnesses & Stateless Validation

State witnesses enable chunk validators to validate execution without local state. Missing recordings break validation.

**Checklist:**
- [ ] Any trie read in the runtime uses recording-capable methods (`get_ref`) so reads are captured in the state witness (PR #12886).
- [ ] Changes do not add to state witness size without justification (PR #12797).
- [ ] If tests still pass after removing recording, that may indicate insufficient test coverage (PR #13535).

---

## Storage Layer (Hot/Cold/Archival)

**Checklist:**
- [ ] Storage growth from new header/chunk fields is a first-class concern (PR #14777).
- [ ] DB migrations handle archival vs non-archival nodes, hot vs cold storage (PR #14749, PR #14654).
- [ ] DB migration performance considered: archival migrations can delay startup by hours (PR #14790).
- [ ] GC updated for new DB columns. New code does not depend on data that may have been garbage collected (PR #14906).
- [ ] Read-modify-write patterns in database operations are thread-safe (PR #13978).
- [ ] Archival nodes may not have memtries enabled (PR #13395).
- [ ] Panicking on old enum variants or old protocol paths may crash archival nodes (PR #13435).
- [ ] Database tooling changes consider cold storage (PR #13082).

---

## Runtime Error Classification

Three distinct error levels exist. Confusing them is a critical bug (PR #12886).

| Level | Meaning | How to signal |
|-------|---------|---------------|
| `ActionErrorKind` | User-facing action failure (transaction processed, action fails) | Set `result.result = Err(...)` and `return Ok(())` |
| `RuntimeError` | Internal error (fails entire chunk application) | Return as `Err(RuntimeError::...)` |
| `RuntimeError::StorageError` | Storage inconsistency (should never happen) | Return as `Err(RuntimeError::StorageError(...))` |

**Key questions:**
- "If this occurs on every validator simultaneously, what happens to the network?"
- "If this panics, does the shard stop making progress?"

---

## Actor Model & Async Patterns

nearcore is migrating from actix to tokio. Review patterns:

- [ ] Cleanup is robust against panics -- use Drop guards, not manual cleanup (PR #14127).
- [ ] The task owns the runtime/threadpool, not the handles (PR #14064).
- [ ] Prefer cancellation tokens over `AtomicBool` flags for shutdown (PR #14064).
- [ ] Migration toward `Send` futures; eliminate `boxed_local` (PR #14126).
- [ ] Messages sent AFTER store updates commit (PR #14001).
- [ ] Async actors writing data: readers must not depend on data being available synchronously (PR #14953).

---

## Structured Tracing & Logging Conventions

Follow `docs/practices/style.html#tracing`.

**Checklist:**
- [ ] All tracing calls have explicit `target:` for filtering (PR #13950, PR #14037).
- [ ] Dynamic data uses structured key-value fields (`?var`, `%var`), not format string interpolation (PR #13950).
- [ ] Fully qualified `tracing::debug!` rather than importing the macro (PR #14037).
- [ ] Use `#[instrument]` over manual `debug_span!` (PR #14105).
- [ ] Error formatting uses `?err` shorthand (PR #14255).
- [ ] Repeated fields across log lines captured in a `tracing::span` (PR #13887).
- [ ] Log messages lowercase (but preserve type names/acronyms: `ChunkStateWitness`, `S3`, `STUN`) (PR #14599).
- [ ] Present participle for ongoing actions: "processing block" not "process block" (PR #14599).
- [ ] No redundant text when structured field already conveys info (PR #14606).
- [ ] Log levels appropriate: `warn` only if operators can act on it; per-block events use `debug!` not `info!` (PR #13079, PR #13706).
- [ ] No log level changes in refactoring PRs -- that is a functional change (PR #14603).
- [ ] No expensive formatting in trace/debug messages; use `tracing::field::debug` or lazy formatting (PR #14095).
- [ ] Use `%shard_id` (Display) not `?shard_id` (Debug) (PR #13658).
- [ ] `humantime::Duration` for duration formatting (PR #14599).
- [ ] Errors: `err = &err as &dyn std::error::Error` (PR #13258).
- [ ] Do not log large arrays in span fields; use `.len()` instead.

---

## Configuration Management

**Checklist:**
- [ ] Config field names describe the concept, not implementation (`state_request_throttle_period` not `state_request_actor_throttle_period`) (PR #13974).
- [ ] Deprecated config fields have a deprecation plan: log warning, then panic, then remove (PR #13154).
- [ ] Add `#[deprecated]` attributes and CHANGELOG entries for deprecated fields (PR #13722).
- [ ] Backward-compatibility parsing test for old config format (PR #13154).
- [ ] Use serde aliases for renamed fields (PR #13219).
- [ ] Config validation rejects invalid combinations.
- [ ] New config fields wired up and accessible from config.json.
- [ ] Derived values (e.g., GC period from block time) automatically computed (PR #13825).
- [ ] Test configs remain intentionally different from production configs where needed (PR #13575).
- [ ] Ask: "Can we simplify? Do we even need this config field?" (PR #14239).
- [ ] CHANGELOG entries use user-friendly wording: "No action necessary" vs "Operators should update" (PR #13575).

---

## Rust Idioms in nearcore

### Early Returns & Flat Control Flow
- Prefer `let Some(x) = expr else { continue/return };` over nested `if let Some(x) = expr { ... }`.
- Handle error/None cases first, then proceed with main logic unindented.
- Invert conditions to `continue`/`return` early and reduce nesting depth.

### Iterator Patterns
- Prefer iterators over index-based loops when equivalent.
- Use `is_some_and()`, `is_none_or()` over `map_or()` chains.
- Merge `map` + `fold` into a single `fold`.
- Use `repeat_with().take(n)` over `[(); N].map(|_| ...)`.
- Use `collect::<Result<Vec<_>, _>>()` to avoid premature collection.
- Use `itertools`: `exactly_one()`, `collect_vec()`, `Itertools::sorted()`.

### Type Safety
- Enums over booleans for parameters where meaning is ambiguous at call site.
- Named variables for boolean arguments: `let is_forwarded = true;` instead of passing bare `true`.
- Domain type aliases: `BlockHeight`, `ShardId`, `Gas`, `Balance` over raw integers.
- `Result<(), Error>` over `bool` for fallible operations.
- Exhaustive match arms on enums (no `_ =>` wildcards on growing enums).
- Enum size: hot-path enums should be `Box`ed to keep size small. Use static size assertions.

### Safe Arithmetic
- `checked_sub`, `saturating_sub`, etc. No bare arithmetic operations, even with guards.
- Prefer `checked_mul`/`checked_add` over `saturating_*` in gas calculations.
- No `as T` casts -- use `u128::from()`, `try_into().unwrap()`, or `.into()`.

### Naming
- `to_*` for borrowing conversions, `into_*` for consuming conversions.
- `maybe_X` or `try_X` for fallible operations returning `Option`/`Result`.
- Enum variants should not repeat the enum name: `AccountContract::Local` not `AccountContract::LocalContract`.
- Use positive boolean names: `is_new_chunk` not `is_chunk_missing`.
- Avoid mixing abbreviations in the same module (`transaction` vs `tx`).
- Error variant names should not have redundant `Error` suffix.

### Other Conventions
- `expect("reason")` over bare `unwrap()` to document safety assumptions.
- Named constants over magic numbers with type annotations.
- `parking_lot::Mutex`/`RwLock` over `std::sync` equivalents.
- `borsh::object_length` over `to_vec(..).len()` (avoid unnecessary allocation).
- Prefer `impl Iterator` over `Box<dyn Iterator>`.
- Prefer `&dyn Trait` over `Arc<dyn Trait>` in function signatures unless the callee needs to clone the `Arc`.
- Avoid `use X::*` -- prefer explicit imports.
- Import from canonical crate, not re-exports from unrelated crates.
- `pub(crate)` when possible; no `pub` unless needed externally.
- Remove unused parameters rather than prefixing with `_`.
- Context arguments first in function signatures.
- Use `ok_or_else(|| ...)` over `ok_or(...)` when the error value requires allocation.

---

## Testing Patterns

### Framework
- [ ] Test-loop (`TestLoopBuilder`, `TestLoopNode`, `TestLoopEnv`) is mandatory for new end-to-end tests. Integration tests (`integration-tests/`, `TestEnv`) are deprecated.
- [ ] Use existing utilities: `create_account_ids`, `create_validators_spec`, `epoch_config_store_from_genesis()`, `TestLoopNode::view_account_query`, `run_until_head_height`, `run_until_new_epoch`, `rpc_node.run_tx`, `rpc_node.runtime_query`.
- [ ] Set configuration in genesis, not by overriding epoch config directly.
- [ ] Use RPC nodes for querying state (not producers tracking all shards).
- [ ] Reference existing test-loop tests when writing new ones.

### Structure
- [ ] One test function per test case for easier debugging.
- [ ] Minimal setup: fewest accounts, shards, validators needed. Simplest shard layout possible (e.g., `ShardLayout::single_shard()` for unit tests).
- [ ] Do not initialize subsystems (flat storage, memtries) unless the test actually requires them.
- [ ] Large test files split into dedicated files under `src/tests/`.
- [ ] Test utilities at end of `mod tests`; actual tests at top.
- [ ] Use builder patterns for test data (e.g., `TestWitnessBuilder`).

### Assertions
- [ ] `assert_eq!` over `assert!(x.is_empty())` for better failure messages.
- [ ] `assert_matches!` for pattern-matching assertions (PR #14184).
- [ ] Assert specific error types, not just `is_err()`.
- [ ] Assert preconditions hold before testing postconditions.
- [ ] Assertion messages explain what the condition means for an external reader.
- [ ] Use expressions (`txs.len()`) not hardcoded magic numbers.
- [ ] Every test must have meaningful assertions.
- [ ] Regression tests assert the specific invariant being tested.
- [ ] Tests with `|| unreachable!()` need companion tests exercising the reachable path.

### Quality
- [ ] Deterministic: hardcoded seeds, not `thread_rng()`.
- [ ] Semantic stop conditions over fixed block counts.
- [ ] Block-count progression over time-based progression.
- [ ] Tests use same APIs as production code, not internal trie access.
- [ ] Use `ProtocolFeature` constants, not hardcoded version numbers.
- [ ] Document magic numbers with derivation math.
- [ ] Bug fix tests verified to fail before the fix (red-green testing).
- [ ] Protocol upgrade tests use `sequential` schedule.
- [ ] Resharding tests verify all shards after resharding, not just before.
- [ ] Fork tests verify actual forks occurred, not just skipped blocks.
- [ ] Test infrastructure not deleted just because temporarily unused -- use `#[allow(unused)]`.
- [ ] Reduce iteration counts to the minimum needed (a few epochs, not 1000 blocks).
- [ ] Do not add public API (`Default` impls, constructors) solely for test convenience.
- [ ] Test configs remain distinct from production configs where intentionally needed.
- [ ] Prefer unit tests over heavy integration tests when functionality can be tested simply.
