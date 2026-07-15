# Engineering workflow

## Working principles

- Understand the repository before editing. Read the nearest `AGENTS.md` or
  `CLAUDE.md`, the relevant project `kb/` notes, and any structure script the
  project provides.
- State important assumptions and success criteria before making a non-trivial
  change. Prefer the smallest change that solves the requested problem.
- Keep changes surgical: preserve established patterns and avoid unrelated
  refactors or formatting churn.

## Environment and secrets

- Never read `.env`, `.env.keys`, or other secret files.
- Describe required variables and let the user set them.
- Keep an `.env.example` up to date when the project uses one.
- Never commit secrets or machine-specific credentials.

## Editing and verification

After all edits, and once per task rather than after every file:

1. Format the changed files.
2. Run the project’s lint command, fixing only issues caused by the change.
3. Run the project’s type check and relevant tests/build checks.
4. Report unrelated pre-existing failures instead of modifying unrelated files.
5. Stop temporary development servers before handing work back.

Use the repository’s package scripts as the source of truth. Most current
projects use Bun, but do not replace a repository’s declared package manager
with Bun automatically.

## Git and issue tracking

- Do not commit unless the user explicitly asks for a commit.
- Do not add `Co-Authored-By` lines.
- Where the repository uses tagged commits, prefix messages with a short topic
  tag such as `[feat]`, `[fix]`, `[ui]`, `[db]`, `[infra]`, or `[chore]`.
- If the repository uses Beads (`bd`), inspect `bd prime`/the local workflow,
  create or update issues only when the task calls for it, and do not close an
  issue merely because the implementation is finished.

