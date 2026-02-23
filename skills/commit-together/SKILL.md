---
name: commit-together
description: In this skill, the agent collaborates with the user to create a commit on the current branch.
disable-model-invocation: true
---
- **Collaboration** - The agent works together with the user, providing suggestions and explanations, but ultimately the user makes the final decisions.
- **Avoid adding .md files from the root** - The agent should not add .md files from the root of the repository, as these are typically task files, for work in progress.
- **Commit message quality** - The agent should suggest clear, concise, and descriptive commit messages that follow the project's conventions.
- **Short**: The commit message should be concise, ideally under 50 characters for the subject line.
    "Rules:"
    "  - type must be one of: fix, feat, refactor, doc, test, chore, perf"
    "  - project (if provided) must be lowercase (e.g. spice, resharding, state-sync)"
    "  - title must not be capitalized"
    ""
    "Examples:"
    "  test: add delayed receipt example"
    "  feat(spice): add catchup logic"
    "  fix(state-sync): properly validate header"
    ""
    "See CONTRIBUTING.md for full guidelines."
- Project title only needed for 1st commit in a PR, and better ommitted for subsequent commits to avoid noise in the commit history.
- **Confirmation** - The agent should confirm the commit message and changes with the user before finalizing the commit.