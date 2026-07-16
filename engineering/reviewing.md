# Change and pull-request review

Review asks whether a change satisfies its product contract safely in the
current repository, not merely whether its new tests pass.

## Establish the contract

- Read the issue, PR description, applicable project KB, and repository
  instructions before reviewing implementation details.
- Identify the acceptance criteria and the existing modules that already own
  the behavior.
- Prefer extending established services, components, schemas, and operations to
  introducing a parallel abstraction.

## Trace state and identity

- Treat related writes as one state transition. If partial success would leave
  contradictory or orphaned records, require an atomic transaction, batch, or
  conditional write.
- Follow identifiers across persistence, domain models, render inputs, and
  navigation. A model should carry the identifier of the resource it links to;
  do not substitute a nearby role, report, delivery, or run identifier because
  it has the same primitive type.
- Test the final observable result, such as the complete URL or persisted record
  set, rather than only checking that a link or row exists.
- For optional user-controlled values, review the whole lifecycle: create,
  update, clear or disable, stale/concurrent updates, and the downstream effect
  of each transition.

## Review tests as product contracts

- Include positive evidence, gaps, uncertainty, invalid references, duplicate
  mappings, stale state, and partial-failure cases where they matter.
- No-invention tests should verify relationships between claims and their
  allowed evidence. A blacklist of a few exact phrases is not a grounding
  guarantee; paraphrases and unrelated verified evidence must not evade it.
- Keep candidate-controlled or otherwise sensitive fields explicit. Test that a
  duplicate field or alternate representation cannot bypass the boundary.
- Prefer deterministic fixtures with no network or model dependency for quality
  gates, while retaining narrowly scoped integration checks for real adapters.

## Verify the integrated branch

1. Run focused tests while reviewing the changed behavior.
2. Integrate the current base branch.
3. Run the repository's complete required checks on that exact head.
4. Wait for required remote checks before marking a draft ready or merging.
5. Update the PR validation record with the final results and distinguish
   pre-existing failures from change-specific failures.

When a test fails unexpectedly, rerun the case in isolation, its containing
file, and the full suite. This evidence can distinguish a deterministic defect
from shared-resource or lifecycle contention. A repeated flake is still work:
record it and fix the isolation instead of normalizing intermittent failure.

## Preserve local work

When the contributor's local branch contains unpublished commits or unrelated
changes, review the remote PR head in a separate clean worktree. Push only the
reviewed history to the PR branch. Do not reset, discard, or silently absorb the
contributor's local work.

## Review the record

Before merge, confirm that the title, description, linked issue, validation
results, migration or rollout notes, and known limitations describe the final
head. The PR should be useful evidence after the branch is gone.
