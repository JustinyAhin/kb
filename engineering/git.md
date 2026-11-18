# Git Workflow

## Commit authorization

- Never commit unless explicitly asked. Do the work, run the checks, then stop
  and report until the user says to commit or gives equivalent approval.
- Do not push unless explicitly asked.

## Commit messages

- Do not add AI attribution or agent metadata to commit messages, including
  `Amp-Thread-ID` links and `Co-authored-by` trailers (regardless of casing).
- The subject line is the tag plus a short summary of one change. Aim for 50
  characters and never go over 72, tag included. GitHub cuts longer subjects
  at 72 characters and moves the rest into the body with "…".
- Do not put a list of changes in the subject (no "a; b; c").
- Put details in the body: a blank line after the subject, then short lines
  or `-` bullets wrapped at about 72 characters.
- Add a body only when it carries meaningful context, such as a non-obvious
  reason, tricky tradeoff, or follow-up note. For example:

  ```
  [fix] Keep the drawer under the main area below 1100px

  - The 230px side panel cut its content at 768px.
  - The drawer and the main area now scroll together.
  ```
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
