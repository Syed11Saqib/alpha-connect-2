# AI Agent Rules

- Read the repository before generating code.
- Prefer modifying existing files over creating duplicates.
- Never assume a feature exists without verifying it.
- Keep commits small and logically grouped.
- Follow the approved architecture documents exactly.
- If uncertain, stop and explain the issue instead of guessing.

Before making any change:

1. Read IMPLEMENTATION_GUIDE.md
2. Read every document inside docs/
3. Read SESSION_STATE.md
4. Analyze the current repository
5. Continue from the next unfinished task

Never:

- Change architecture
- Ignore Clean Architecture
- Skip tests
- Break module boundaries

Always:

- Update SESSION_STATE.md
- Explain architectural decisions
- Make small commits

# Before Finishing

Before ending any session:

1. Update SESSION_STATE.md
2. Summarize every file changed.
3. List architectural decisions made.
4. List any blockers.
5. Suggest the next checkpoint.

## Token Efficiency Rules

- Read SESSION_STATE.md before any other file.
- Read only the files required for the current checkpoint.
- Never reread the full repository unless explicitly instructed.
- Keep responses under 300 words unless asked otherwise.
- Do not explain concepts unless requested.
- Do not repeat architecture summaries.
- Implement exactly one checkpoint per request.

Never read every file in the repository.

Determine the minimum required files for the current checkpoint and read only those.