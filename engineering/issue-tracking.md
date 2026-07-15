# Beads Workflow and Backup Runbook

## Project workflow

This project uses **bd (beads)** for issue tracking.
Run `bd prime` for workflow context, or install hooks (`bd hooks install`) for auto-injection.

**Quick reference:**

- `bd ready` — find unblocked work
- `bd create "Title" --type task --priority 2` — create issue
- `bd close <id>` — complete work
- `bd dolt push` — push beads to remote

For full workflow details: `bd prime`

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
# Restore with: bd init --prefix <project-prefix> --skip-hooks && bd backup restore .beads-backup/ --force

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

Record the policy and restore command in the project-level agent instructions.
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
bd init --prefix <project-prefix> --skip-hooks
bd backup restore .beads-backup/ --force
bd doctor
```

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
