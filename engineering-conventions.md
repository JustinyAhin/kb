# Reusable engineering conventions

## Working principles

- Understand the repository before editing. Read the nearest `AGENTS.md` or
  `CLAUDE.md`, the relevant project `kb/` notes, and any structure script the
  project provides.
- State important assumptions and success criteria before making a non-trivial
  change. Prefer the smallest change that solves the requested problem.
- Keep changes surgical: preserve established patterns and avoid unrelated
  refactors or formatting churn.
- Never read `.env`, `.env.keys`, or other secret files. Describe required
  variables and let the user set them. Keep an `.env.example` up to date when
  the project uses one.

## TypeScript

The most repeated convention across the projects is:

- Prefer `type` over `interface`.
- Use arrow functions rather than function declarations.
- When a function has multiple parameters, accept one named object argument.
- Keep reusable types in `types.ts`; keep a type local when it has one consumer.
- Define implementation symbols first and export them at the end of the file.

SvelteKit route exports (`load`, `entries`, `prerender`, and similar framework
hooks) are the intentional exception: use the direct `export const` form the
framework expects.

```ts
type SendEmailOptions = {
  to: string;
  subject: string;
  body: string;
};

const sendEmail = ({ to, subject, body }: SendEmailOptions) => {
  // ...
};

export { sendEmail, type SendEmailOptions };
```

## Files and components

- Use kebab-case for component and utility filenames.
- Put reusable UI in the project’s shared component directory, commonly
  `src/lib/components/`, and import it through the project alias such as
  `$lib/components/...`.
- If the project uses shadcn-svelte, follow its composition API and existing
  local component style. Do not introduce a second component system casually.
- If the project uses Lucide, import icons from `@lucide/svelte`.
- Add `cursor-pointer` to clickable elements when that is the project’s
  established UI convention.

## Editing workflow

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

## What stays project-specific

Do not promote these into global rules without an explicit project decision:

- product strategy, design language, SEO/content requirements, and domain rules;
- exact database choice, binding names, migration directories, and deployment
  commands;
- package-specific workarounds or generated-code locations;
- whether a project uses Node, Cloudflare Workers, Docker, or another runtime;
- issue prefixes, commit tag vocabularies, and backup scripts.
