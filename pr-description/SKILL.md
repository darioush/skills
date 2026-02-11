---
name: pr-description
description: In this skill, the agent suggests a pull request description based on the changes in the current branch, paying attention to the base branch and any relevant context.
disable-model-invocation: true
---
- **Context Awareness** - The agent should analyze the changes in the current branch, understand the base branch, and consider any relevant context such as linked issues, commit messages, and code comments to generate an accurate and informative PR description.
- **Conciseness** - The PR description should not use more words than necessary or long sentences.
- **Single Section** - The PR description should be a single section without separate subsections such as "Details", "Testing", "Documentation", etc. The description should be a cohesive summary of the changes, not a structured report.
- **Description, then bullet points** - The PR description should start with a concise summary of the changed ins 1-2 sentences, followed by bullet points that highlight the key changes, motivations, and any important context. This format helps reviewers quickly grasp the essence of the PR while providing additional details in an easily scannable format. If the description is clear from the commit message / PR title, the description can be omitted and only bullet points can be provided.
- **MD escape**: Ensure special characters like ` are visible to user for pasting into github.
