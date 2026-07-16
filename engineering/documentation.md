# Documentation ownership

Documentation stays useful when each fact has one canonical owner and every
other location links to it. Prefer a short route to the source over a second
copy that can drift.

## Where information belongs

| Location                   | Purpose                                                                                                                      |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Repository `README.md`     | A brief entry point: what the repository is, the minimum local start, and links to deeper documentation.                     |
| `AGENTS.md` or `CLAUDE.md` | Routing and enforceable agent instructions. Link to conventions and product decisions instead of restating them.             |
| Shared KB                  | Reusable engineering conventions, review practices, stack guidance, and cross-project runbooks.                              |
| Project KB                 | Product requirements, domain contracts, accepted decisions, design rules, and project-specific operations.                   |
| ADR                        | Why one consequential technical choice was made, its alternatives, and its consequences.                                     |
| Runbook                    | Commands, prerequisites, verification, rollback, and recovery for one operated system.                                       |
| Code and configuration     | Exact scripts, bindings, defaults, schemas, and behavior that can be derived reliably from the implementation.               |
| Issue tracker              | Temporary work state, acceptance criteria, and implementation history. It is not the long-term home of an accepted decision. |

A README should not become an architecture document or an operations manual.
Keep quick-start commands there only when they help a new contributor reach the
canonical tooling. Put deployment, smoke checks, recovery, and rollback in a
runbook under the project KB.

## Avoiding duplication

- Choose one normative document for each decision or invariant.
- Link to the normative section from summaries, instructions, and runbooks.
- A summary may explain why a linked document matters, but should not repeat
  change-prone values, command sequences, state machines, or checklists.
- Keep product summaries intentionally broad. Detailed workflow behavior belongs
  in contracts; implementation rationale belongs in an ADR; operator actions
  belong in a runbook.
- Keep exact commands in `package.json` or another executable configuration and
  refer to the script name from documentation.
- When moving content, remove or replace the old copy in the same change.

Some repetition is deliberate: a safety boundary may appear as a one-line
invariant in a contract and as an actionable check in a runbook. The wording
should point to the same owner and serve a different reader, not create two
independent definitions.

## Documentation review

Before adding a document or section:

1. Search the shared and project KBs for an existing owner.
2. Decide whether the information is reusable engineering guidance or a
   project-specific product/operations decision.
3. Extend the owner when the subject already exists; create a document only
   when it has a distinct audience and lifecycle.
4. Replace duplicated text with links.
5. Check inbound links and the relevant KB index after moving content.

Review documentation when the behavior changes, not as a separate cleanup at
the end. A code change is incomplete when its canonical contract or runbook is
now false.
