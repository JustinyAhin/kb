# Beads Workflow and Backup Runbook

## Project workflow

This project uses **bd (beads)** for issue tracking.
Run `bd prime` for workflow context. Keep project agent instructions to one
sentence linking to this runbook; install agent integrations or hooks only when
the project asks for them.

**Quick reference:**

- `bd ready` — find unblocked work
- `bd epic status` — inspect progress across epics
- `bd show <id>` — read scope, acceptance criteria, and dependencies
- `bd create "Title" --type task --priority 2` — create issue
- `bd close <id>` — complete work
- `bd dolt push` — only for a deliberately adopted remote and an authorized push

For full workflow details: `bd prime`

Use Beads for live status and dependencies, and the project KB for supporting
briefs, research, decisions, and verification evidence. Include acceptance
criteria and evidence links in tickets. Backfill completed work as closed only
when verified; keep proposed research and unauthorized publication deferred.
Creating a ticket does not authorize publishing, messaging, purchasing, or
deployment. Follow [Git authorization](git.md) for commits and pushes.

## Minimal initialization

For a new local tracker, first inspect `bd --version`, `bd init --help`, and the
repository's existing files and Git status. Do not reinitialize an existing
database to change preferences.

The following was checked with Beads 1.1.0 in a disposable Git repository on
September 16, 2026, including a configured Git origin:

```bash
bd init --prefix <project-prefix> --stealth --skip-agents --skip-hooks --remote= --non-interactive
bd config set dolt.local-only true
bd config set backup.enabled false
bd config set backup.git-push false
bd config set dolt.auto-push false
```

This created the embedded Dolt database without a Git commit, generated agent
files, installed Git hooks, or Dolt remote. `--remote=` explicitly selects no
remote. Stealth mode suppresses the initializer's automatic Git commit and adds
local ignore entries in `.git/info/exclude`; it does not prevent later deliberate
tracking of configuration or `.beads-backup/`.

When a Git commit is explicitly authorized, first create the native backup under
the policy below. For a new setup initialized in stealth mode, add the small
configuration files explicitly, without force-adding the database directory:

```bash
git add -f .beads/.gitignore .beads/config.yaml .beads/metadata.json
git add .beads-backup/ scripts/beads-backup.sh
```

Also include any intentional project docs or `.beads/PRIME.md` customization.
Already-tracked configuration stays tracked when stealth initialization is used
in a new clone. The native backup remains the portable recovery source.

**Version-specific behavior:** In 1.1.0, `--skip-hooks` alone does not skip
Claude/Codex setup; use `--skip-agents` too. Ordinary non-stealth initialization
can infer a Dolt remote from Git and automatically stage and commit setup files,
including existing agent/configuration paths. Neither `--skip-agents` nor
`--skip-hooks` disables that bootstrap commit. This was observed during setup
and corroborated in the [1.1.0 initializer source](https://github.com/gastownhall/beads/blob/v1.1.0/cmd/bd/init.go).
Recheck installed behavior before using these instructions with another version.

If a new initializer has already inferred an unwanted remote, first set
`bd config set no-git-ops true`, then inspect
`bd dolt remote list`, remove only that newly introduced remote with
`bd dolt remote remove <name>`, and clear its inferred `sync.remote` setting with
`bd config unset sync.remote`. Preserve pre-existing remotes and user files.
Remove generated agent scaffolding only after confirming which files were
created by the setup; do not overwrite the user's instructions. Report any
automatic setup/configuration commits rather than silently rewriting Git history.

### Embedded-mode diagnostics

In Beads 1.1.0, `bd doctor` reports that it is unsupported in embedded mode and
can still exit zero. That is not a passed health check. Use the available checks:

```bash
bd stats
bd list --all --limit 0
bd ready
bd dep cycles
bd lint --status all
```

Inspect representative `bd show <id>` results and verify parent/dependency
references. These checks do not replace all doctor diagnostics or prove backup
restore works. Do not reinitialize or migrate a healthy database merely to make
doctor available. For server-backed databases, use the doctor commands below.

## Beads backup and recovery runbook

Use this runbook for projects that use Beads with its default Dolt storage.
It preserves the full Beads database rather than only an issue export.

### Policy

- Use a git-tracked, Dolt-native `.beads-backup/` directory. It preserves
  tables, branches, history, and the Dolt working set.
- Run a backup only when explicitly requested, before a risky Beads upgrade or
  recovery, or before a Git commit that includes ticket changes. Do not make
  automatic backups after ordinary ticket commands.
- Commit changed backup files to the project repository when a portable
  recovery point is wanted. A local backup that is not committed is not an
  off-machine backup.
- Do not use JSONL exports as the recovery source for a Dolt database. They
  are useful for inspection/interchange, not full-fidelity restore.

### Per-project setup

Add `scripts/beads-backup.sh`:

```bash
#!/usr/bin/env bash
# Sync a Dolt-native Beads backup to a git-tracked directory.
# Restore: initialize using the minimal command above, then bd backup restore .beads-backup/ --force

set -euo pipefail

cd "$(git rev-parse --show-toplevel)"

BACKUP_DIR=".beads-backup"
BACKUP_PATH="$PWD/$BACKUP_DIR"

mkdir -p "$BACKUP_DIR"
bd backup init "$BACKUP_PATH" -q
bd backup sync

# Remove artifacts from the obsolete JSONL backup method.
rm -f \
  "$BACKUP_DIR/LOCK" \
  "$BACKUP_DIR/issues.jsonl" \
  "$BACKUP_DIR/dependencies.jsonl" \
  "$BACKUP_DIR/config.jsonl" \
  "$BACKUP_DIR/backup_state.json" \
  "$BACKUP_DIR/labels.jsonl" \
  "$BACKUP_DIR/events.jsonl" \
  "$BACKUP_DIR/comments.jsonl"

echo "Beads backup synced to $BACKUP_DIR/"
echo "Commit the changed files in $BACKUP_DIR/ to make the backup portable."
```

Keep project-level agent instructions to one sentence pointing to this runbook.
Record the prefix, storage mode, and any project-specific details in its KB.
Do not add a Dolt remote unless the project deliberately adopts one.

### Normal backup and restore

Create a backup only when the policy above permits it:

```bash
bash scripts/beads-backup.sh
git status --short .beads-backup
```

On a new machine, initialize the project database, then restore the tracked
backup:

```bash
bd init --prefix <project-prefix> --stealth --skip-agents --skip-hooks --remote= --non-interactive
bd config set dolt.local-only true
bd backup restore .beads-backup/ --force
bd stats
bd ready
bd dep cycles
```

Restore replaces the target database; protect existing work before using
`--force`. Run `bd doctor` when supported by the storage mode, and do not report
unperformed restore checks as passing.

### Upgrading Beads

Before upgrading the CLI:

1. Check the current version and health: `bd --version && bd doctor`.
2. Take an explicit Dolt-native backup and commit it if it must survive the
   machine.
3. Upgrade using the installed package manager, for example
   `brew upgrade beads`.
4. Inspect the upgraded state with `bd doctor --agent --json`.
5. Run any migration only after the diagnostic output is understood, then
   validate with `bd doctor`, `bd stats`, and a fresh backup.

For multi-clone or remote-backed databases, follow the Beads release-specific
migration instructions and nominate one migrator. Do not let every clone run
a schema migration independently.

### Schema migration recovery

Symptoms include errors such as:

```text
pending schema migrations alter pre-existing dirty tables
```

or an index/column error during a Beads schema migration.

1. Stop normal mutating Beads commands. Do not run `bd doctor --fix` as a
   first response and do not blindly retry the failing command.
2. Run `bd doctor --agent --json` and save its output with the incident notes.
   `bd dolt status` confirms whether the server is healthy and whether the
   working set is dirty.
3. Take the explicit Dolt-native backup above before changing migration state.
   If the backup itself cannot run because of this schema guard, do not
   substitute a JSONL export for a database backup. Continue with the recovery
   steps below.
4. Upgrade Beads if the installed version is behind a release that fixes the
   reported migration. Re-run the diagnostic.
5. Follow the upgraded CLI's exact recovery instruction. For the Beads 1.1
   dirty-working-set guard, that is:

   ```bash
   bd dolt commit
   bd migrate schema
   ```

   `bd dolt commit` checkpoints the local Dolt working set; it is not a Git
   commit. Run it only after the backup when possible and only when the CLI
   explicitly asks for it.

   If `bd dolt commit` is blocked by the same schema guard, use Dolt directly
   for the checkpoint, then rerun the migration:

   ```bash
   bd dolt show
   dolt --data-dir="$PWD/.beads/dolt" --use-db=<database-name> \
     sql -q "CALL DOLT_COMMIT('-Am', 'checkpoint before schema migration')"
   bd migrate schema
   ```

   Use the database name reported by `bd dolt show`. This direct Dolt command
   is a recovery-only workaround for the wrapper deadlock; do not use it for
   ordinary Beads changes.
6. Validate with `bd doctor`, `bd stats`, and representative read commands
   such as `bd ready` and `bd list --status=open`. Then take another backup.

### Temporary read-only inspection

When a schema guard blocks ordinary commands, this may allow limited
inspection:

```bash
bd --ignore-schema-skew --readonly list --status=open
```

Treat it as emergency read-only access. It is not a repair, and some commands
can still fail while the schema is inconsistent.

### Do not do these things

- Do not commit a partial migration just because a generic warning suggests
  `bd vc commit`; use the recovery command from the upgraded CLI after backup.
- Do not use `bd doctor --fix` until the migration issue is resolved or its
  diagnostic specifically directs that fix.
- Do not edit `.beads/config.yaml` or `.beads/metadata.json` merely to silence
  a local schema or port problem.
- Do not assume a healthy Dolt server means a healthy Beads schema.
