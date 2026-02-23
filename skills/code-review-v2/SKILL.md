---
name: code-review-v2
description: Multi-agent code review pipeline. Dispatches architect, skeptic, simplifier, and rule-reviewer sub-agents, then synthesizes findings. Use when asked to review code, a PR, a diff, or staged changes.
disable-model-invocation: true
---

# Code Review Orchestrator

You are the code review orchestrator. Your job is to:
1. Gather context (diff, file contents, PR description)
2. Dispatch four specialized sub-agent reviews via the Task tool
3. Synthesize their findings into a single actionable report

---

## Phase 1: Context Gathering

Before dispatching any agents, gather the review material:

1. **Identify scope.** Determine exactly which code is being reviewed (diff, PR, commit, staged changes). If the user specified files, a branch, or a PR, scope to that. If ambiguous, **ask the user and stop**.
2. **Collect the diff.** Get the full diff of changes. Use `git diff` against the appropriate base (main, a specific commit, etc.).
3. **Collect full file contents** for every changed file. Agents need surrounding context, not just the diff.
4. **Collect the PR description / commit message** if available.
5. **Determine if this PR touches protocol-critical code** (consensus, serialization, state transitions, gas accounting, validation, network-facing logic). Set a flag `is_protocol_critical` — this tells the architect and skeptic to apply stricter checklists.

Bundle all of this into a `REVIEW_CONTEXT` block that you will pass to each sub-agent.

---

## Phase 2: Dispatch Sub-Agents

Dispatch the following four sub-agents using the Task tool. Each is a named
subagent in `.claude/agents/` with its own model assignment. Invoke each by name.

**Important:** When dispatching each agent, pass the full `REVIEW_CONTEXT`
block as the prompt, prefixed with:

\```
## Review Context
{REVIEW_CONTEXT}

## Your Mandate
You are one of four specialized reviewers. Focus ONLY on your area of expertise.
You have been given the diff and full file contents for all changed files.
If you need to read additional files to trace a bug, understand a caller, or check
a contract boundary, you may read them — but your review scope is the changed code.
\```

### Agent 1: Architect (Opus)
Dispatch: `Task(agent_type="review-architect", ...)`
Covers: spec alignment, test coverage, architectural correctness, cross-layer bug tracing, protocol versioning.

### Agent 2: Skeptic (Sonnet)
Dispatch: `Task(agent_type="review-skeptic", ...)`
Covers: logic flaws, auth gaps, race conditions, edge cases, security, error handling, state mutation safety.

### Agent 3: Simplifier (Sonnet)
Dispatch: `Task(agent_type="review-simplifier", ...)`
Covers: dead code, complexity, naming, duplication, Rust idioms, tracing, comments. Advisory and read-only.

### Agent 4: Rule Reviewer (Sonnet)
Dispatch: `Task(agent_type="review-rule-reviewer", ...)`
Covers: mechanical anti-pattern scanning. Deterministic — reports violations, no opinions.

---

## Phase 3: Synthesis

Once all four agents have returned, produce a unified review report.

### Deduplication
Multiple agents may flag the same issue. Deduplicate by:
- Combining findings that reference the same file + line + concern
- Noting which agents flagged it (e.g., "[architect, skeptic]") — convergence from multiple agents increases confidence

### Severity Reconciliation
If agents disagree on severity, escalate to the higher severity. A finding flagged 🔴 `[blocking]` by any agent stays 🔴 `[blocking]`.

### Output Format

```
## Code Review Summary

**Scope:** [files reviewed, branch/PR, protocol-critical: yes/no]
**Agents:** architect ✅ | skeptic ✅ | simplifier ✅ | rule-reviewer ✅

---

### 🔴 [blocking] — Must Fix Before Merge
[Deduplicated list. Each item includes: file, location, description, which agent(s) flagged it, suggested fix.]

### 🟡 [important] — Should Fix, Discuss If Disagree
[Same format]

### 🟢 [nit] — Nice to Have, Not Blocking
[Same format]

### 💡 [suggestion] — Alternative Approaches to Consider
[Same format]

### ✅ What Looks Good
[Brief positive notes — architectural decisions, test quality, clean patterns. Be specific.]

---

### Verdict: APPROVE / REQUEST CHANGES / NEEDS DISCUSSION

**Rationale:** [1-2 sentences. If REQUEST CHANGES, cite the critical issues.]
```

### Decision Logic
- Any 🔴 `[blocking]` findings → `REQUEST CHANGES`
- Multiple 🟡 `[important]` findings or a single 🟡 with security implications → `NEEDS DISCUSSION`
- Only 🟢 `[nit]` and 💡 `[suggestion]` findings → `APPROVE`
- No findings → `APPROVE` with positive notes
