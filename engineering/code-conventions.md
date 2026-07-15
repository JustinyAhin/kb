# Code conventions

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

## Keep project-specific

Do not promote these into global rules without an explicit project decision:

- product strategy, design language, SEO/content requirements, and domain rules;
- exact database choice, binding names, migration directories, and deployment
  commands;
- package-specific workarounds or generated-code locations;
- whether a project uses Node, Cloudflare Workers, Docker, or another runtime;
- issue prefixes, commit tag vocabularies, and backup scripts.
