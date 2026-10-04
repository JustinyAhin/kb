# Documentation ownership

Give each fact one authoritative owner. Link to that owner instead of keeping another copy.

## Where information belongs

| Location               | Purpose                                                                    |
| ---------------------- | -------------------------------------------------------------------------- |
| Repository README      | Project purpose, minimum local start, and guide links                      |
| Agent instructions     | Enforceable rules and routes to their owners                               |
| Shared KB              | Reusable engineering, documentation, stack, and operations guidance        |
| Project KB             | Product scope, domain contracts, decisions, design, and project operations |
| Decision record        | A consequential choice, its reason, alternatives, and consequences         |
| Runbook                | Prerequisites, commands, verification, rollback, and recovery              |
| Code and configuration | Exact scripts, bindings, defaults, schemas, and implemented behavior       |
| Issue tracker          | Active work, dependencies, acceptance criteria, and completion status      |

## Writing policy

Use **ASD-STE100 Issue 9** for maintained English documentation and explanatory public text.
Use the [official standard](https://www.asd-ste100.org/assets/files/ASD-STE100_ISSUE9.pdf) for writing rules and the controlled dictionary.

- Answer the reader's question first. Give each document one clear purpose.
- Write at most 20 words per instructional sentence and 25 words per descriptive sentence.
- Give one instruction per sentence. Use active voice in procedures.
- Put a necessary condition before the action it controls.
- Use approved words with their approved meanings and parts of speech.
- Define necessary technical nouns and verbs in the owning domain guide. Use one term for one meaning.
- Use simple sentence structures. Replace semicolons with separate sentences.
- Remove filler, slogans, repeated summaries, process narration, and unsupported claims.
- Use lists for procedures and tables for comparisons. Do not add a word quota.
- Preserve literal code, commands, equations, URLs, quotations, and original source snapshots.
- Do a dictionary and rule review before claiming full STE compliance. A readability score or sentence-length scan is insufficient.

Keep documents readable. Leave blank lines between logical sections.
A procedure must state its prerequisites, actions, expected result, and necessary failure or recovery steps.

## Preventing duplication

- Keep current behavior in its topic owner. Keep performed checks in dated evidence and active tasks in the tracker.
- Keep quick-start commands in a README only when they help the reader reach the canonical tooling.
- Keep deployment and recovery steps in the operated system's runbook.
- Link to exact scripts in executable configuration. Do not maintain another script inventory without a distinct purpose.
- Do not repeat change-prone counts, command sequences, state machines, or checklists in summaries.
- Combine related briefs and reviews when separate files add no value.
- Preserve unique source evidence, methods, reviewer roles, corrections, and actual verification.
- Identify consumers before moving or removing a file. Update references in the same change.

A short safety boundary can appear in a contract and a runbook when it serves different readers.
The two references must point to the same owner.

## Documentation review

1. Identify the reader's question.
2. Find the existing topic owner.
3. Update that owner, or create a document only for a distinct audience and lifecycle.
4. Replace duplicate material with links.
5. Review the text against the writing policy.
6. Verify affected links, indexes, and applicable existing checks.

Update documentation when behavior changes. A change is incomplete while its authoritative contract or runbook remains false.
