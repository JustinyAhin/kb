# Git Workflow

## Commit authorization

- Never commit unless explicitly asked. Do the work, run the checks, then stop
  and report until the user says to commit or gives equivalent approval.
- Do not push unless explicitly asked.

## Commit messages

- Do not add `Co-Authored-By` lines to commit messages.
- Keep commit descriptions short. Add a body only when it carries meaningful
  context, such as a non-obvious reason, tricky tradeoff, or follow-up note.
- Prefix every commit with a concise topic tag in square brackets so history
  is easy to search, for example:

  - `[feat]` — new features
  - `[fix]` — bug fixes
  - `[ui]` — styling, layout, and components
  - `[db]` — schema, migrations, and queries
  - `[infra]` — deployment, build, CI, and scripts
  - `[chore]` — dependencies, lockfiles, and internal tooling
  - `[docs]` — documentation and project instructions

Use the project's existing topic taxonomy when one is defined. Projects may
add more specific tags for their own domains, such as `[seo]`, `[content]`,
`[ai]`, `[audio]`, or `[speech]`.
