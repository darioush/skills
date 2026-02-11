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
❌ Bad: "This is wrong."
✅ Good: "This could cause a race condition when multiple users
         access simultaneously. Consider using a mutex here."

❌ Bad: "Why didn't you use X pattern?"
✅ Good: "Have you considered the Repository pattern? It would
         make this easier to test. Here's an example: [link]"

❌ Bad: "Rename this variable."
✅ Good: "[nit] Consider `userCount` instead of `uc` for
         clarity. Not blocking if you prefer to keep it."
```

### 3. Review Scope

**What to Review:**
- Logic correctness and edge cases
- Security vulnerabilities
- Performance implications
- Test coverage and quality
- Error handling
- Documentation and comments
- API design and naming
- Architectural fit

## Review Process

### Phase 1: Context Gathering 
Before diving into code, understand:
1. Ensure you understand which code is being reviewed (diff with master, specific PR, latest commit, staged changes, etc.) If there is any ambiguity here ASK AND ABORT.
2. Understand related documentation, requirements, and design decisions.
3. Note any relevant architectural decisions

### Phase 2: High-Level Review
1. **Architecture & Design** - Does the solution fit the problem?
   - For significant changes.
   - Check: SOLID principles, coupling/cohesion, anti-patterns
2. **Performance Assessment** - Are there performance concerns?
   - For performance-critical code.
   - Check: Algorithm complexity, N+1 queries, memory usage
3. **File Organization** - Are new files in the right places?
4. **Testing Strategy** - Are there tests covering edge cases?

### Phase 3: Line-by-Line Review
For each file, check:
- **Logic & Correctness** - Edge cases, off-by-one, null checks, race conditions, use of protocol features.
- **Security** - Using unverified data, unbounded allocations, non-deterministic execution in blockchain runtime context.
- **Performance** - N+1 queries, unnecessary loops, memory leaks
- **Maintainability** - Clear names, single responsibility, comments
- **Partially mutated state and side effect on early return** - If there is a return statement before the end of the function, check if there are any mutations to state that could cause confusion or bugs. Especially if the function returns an error and the caller could assume no changes. If so, suggest refactoring to avoid this pattern.

### Phase 4: Minimal diff review
For each file and diff chunk, check:
- Whether the change compared to the prior version is minimal and necessary. If not, suggest adherence to the original.
- Ensure the diff does not remove original comments, documentation, or tests without good reason. If it does, suggest restoring them.

### Phase 5: Testing Coherence Review.
- Test setups must be minimal and follow the recommended patterns for the codebase.
- When reviewing a test, especially with a complex setup, conduct broader research of the code base for better (=simpler) testing setup. If you find a better setup, suggest refactoring the test to use it.
- Sometimes a setup must be extended to cover a new edge case. In this case, the test should be extended with the minimal necessary additions to cover the new edge case, which helps adhere to the recommended patterns.
- In nearcore, integration-tests are deprecated.
- Writing test-loops is the preferred end to end testing pattern.

### Phase 6: Code consistency review
- Check if the code is consistent with the surrounding code and the overall codebase. Flag any inconsistencies and suggest making the code consistent with the surrounding code and overall codebase.

### Phase 7: Comment consistency review
- Check if the code comments are consistent with the code. If there are any inconsistencies, suggest updating the comments to match the code or vice versa. This helps ensure that comments remain accurate and useful for future readers.
- Check surrounding code, top of functions, file headers, for comments that may be affected by the change. If there are any changes that should be made to these comments, suggest them.

### Phase 8: Summary & Decision (2-3 minutes)
1. Summarize key concerns
2. Highlight what you liked
3. Make clear decision:
   - ✅ Approve
   - 💬 Comment (minor suggestions)
   - 🔄 Request Changes (must address)

   ## Review Techniques

### Technique 1: The Checklist Method

(TODO: Sorry, for now I don't have any checklists for you.)

### Technique 2: The Question Approach

Instead of stating problems, ask questions:

```markdown
❌ "This will fail if the list is empty."
✅ "What happens if `items` is an empty array?"

❌ "You need error handling here."
✅ "How should this behave if the API call fails?"
```

### Technique 3: Suggest, Don't Command

Use collaborative language:

```markdown
❌ "You must change this to use async/await"
✅ "Suggestion: async/await might make this more readable. What do you think?"

❌ "Extract this into a function"
✅ "This logic appears in 3 places. Would it make sense to extract it?"
```

### Technique 4: Differentiate Severity

Use labels to indicate priority:

- 🔴 `[blocking]` - Must fix before merge
- 🟡 `[important]` - Should fix, discuss if disagree
- 🟢 `[nit]` - Nice to have, not blocking
- 💡 `[suggestion]` - Alternative approach to consider

### Technique 5: Style guide

- Read [style guide](<project-root>/docs/practices/style.md) if available before reviewing, but adherence to this is somewhat loose. Check nearby code for examples of style. If there is a clear style in the codebase, follow it. If not, use your best judgment. Don't block on style issues that don't cause confusion or bugs.
