# Environment variables with Varlock and 1Password

This is one optional approach for projects and CLI tools that use Varlock with 1Password.
Each project can choose another secret management workflow.
Use 1Password to store values. Use Varlock to load and check them when a command starts.
Project scripts and CLI launchers keep normal commands such as `bun run dev` and `example-cli`.

Reviewed against the linked official documentation on 2026-10-12.
All project names, paths, variable names, and identifiers below are examples.
Keep real account details and values outside this public repository.

## Terms and access boundaries

| Term | Purpose |
| --- | --- |
| 1Password vault | Stores password-manager items. A service account receives access at vault scope. |
| 1Password Environment | Stores named environment variables under **Developer → Environments**. It is separate from ordinary vaults. |
| Varlock schema | Defines expected variable names, types, sources, and sensitivity in `.env.schema`. |
| Service account | Gives a machine access to selected vaults or Environments. Environment access is read only. |
| Bootstrap token | Authenticates the loader before it can fetch application secrets. This is also called *secret zero*. |

If using Environments, create separate Environments for separate access needs:

```text
1Password
  shared-creds        -> shared CLI tools
  example-project-dev -> one project's development commands
  other-project-dev   -> another project's development commands

Mac desktop authentication or scoped server token
  -> Varlock schema
  -> selected command and its child processes

Cloudflare production secrets
  -> deployed Worker
```

Do not grant a server access to ordinary password vaults when it only needs Environments.
Prefer a separate service account for each machine and access scope.
An unrestricted agent under the same Unix account can read token files and injected application secrets.
File permissions and output redaction do not isolate that agent.
Use a sandbox and a separate credential broker when the agent must not receive secret values.

Varlock does not intercept every command after installation.
A repository script, framework integration, or executable CLI launcher must perform the loading step.

## Prerequisites

- Permission to create the chosen 1Password vaults or Environments and service accounts.
- 1Password CLI (`op`) on each machine.
- For Environment loading through the CLI, an Environment-capable beta, version `2.33.0-beta.02` or later.
- Varlock on each machine, or a pinned project dependency.
- A normal Unix account for server development and agents.
- Access to the original application binary before installing a CLI launcher.

Install system tools as root when necessary. Run configuration and application commands as the development user.
The server does not need a root 1Password sign-in.

Check versions without loading secrets:

```bash
op --version
op environment --help
varlock --version
```

For a standalone Mac installation:

```bash
brew tap dmno-dev/tap
brew install dmno-dev/tap/varlock
```

For Linux, install into the development user's command directory:

```bash
curl -fsSL https://varlock.dev/install.sh | sh -s -- --dir="$HOME/.local/bin"
```

The official installer checks the downloaded release checksum.
See [Varlock installation](https://varlock.dev/getting-started/installation/) for release verification and supported installation methods.
See [1Password Environment access](https://www.1password.dev/environments/read-environment-variables) for CLI versions and authentication requirements.

Ensure `$HOME/.local/bin` precedes other application installations in `PATH`.
Configure the agent runner's `PATH` too. Noninteractive processes do not necessarily read interactive shell files.
Restart a runner only if its service configuration changed.

## 1. Create the 1Password scope

This procedure uses selected Environments. The dedicated-vault alternative follows below.

1. In the desktop app, open **Developer → Environments**.
2. Create `shared-creds` for CLI credentials, or `example-project-dev` for a project.
3. Add the variable names required by the tool or project.
4. Enter values directly in 1Password through an authorized private workflow.
5. Use **Manage environment → Copy environment ID**.
6. Keep the ID in machine configuration when public documentation must not identify private infrastructure.

For server access, use the [service-account web wizard](https://www.1password.dev/service-accounts/get-started):

1. Disable creation of new vaults.
2. Leave ordinary vault access empty.
3. In **Environment access**, select only the required Environment.
4. Save the token in a private password-manager item.

The token is shown once. Do not paste it into chat, command arguments, shell history, or public documentation.
Service-account permissions, selected vaults, and selected Environments cannot change after creation.
Create a replacement account when another scope is needed.
The existing shared CLI account cannot automatically access a new project Environment.
Adding another vault also requires a replacement account.
See [service-account grant restrictions](https://www.1password.dev/service-accounts/get-started#create-a-service-account).

On the Mac, enable the [desktop app CLI integration](https://www.1password.dev/cli/app-integration).
Keep the service token unset when desktop authentication is intended.
Desktop authentication uses the signed-in person's access, which can exceed the selected schema's scope.

### Alternative: reusable access through a dedicated vault

Selected Environments provide separate project access scopes.
A dedicated vault is an optional way to reuse an account for credentials with the same access requirements.
Group credentials by their access requirements. Choose vault names and item structure for the project or workflow.
Choose the model before creating the account. This runbook does not require switching an existing setup.

| Model | Adding a project | Access boundary |
| --- | --- | --- |
| Selected Environments | Create a replacement account if the new Environment was not selected at creation. | The account reads the selected Environments. |
| Dedicated vault | Add an item to the already selected vault and reference its fields. | The account reads every item in that vault. |

The vault grant stays fixed, but the vault's contents can change.
The same account can read future items added to that vault without a new account.
It cannot access a newly created vault without replacement.
See the [vault read-access boundary](https://www.1password.dev/service-accounts/security#security-model).

If choosing this model:

1. Create a dedicated vault, such as `example-vault`.
2. Include only credentials with the same access requirements.
3. Add items with named fields. For project credentials, `example-project` with an `api-key` field is one possible structure.
4. Create a service account with read-only access to the selected vault.
5. Leave vault creation, write access, and unrelated vault or Environment access disabled.
6. Save the token through the protected-token procedure below.

Use Varlock's `op()` resolver for each required field instead of `opLoadEnvironment()`:

```dotenv
# @plugin(@varlock/1password-plugin@2.0.5)
# @import(~/.config/varlock/example-project/.env.local, pick=[OP_TOKEN], allowMissing=true)
# @initOp(token=$OP_TOKEN, allowAppAuth=true, useCliWithServiceAccount=true)
# @defaultSensitive=true
# ---

# @type=opServiceAccountToken @internal @optional
OP_TOKEN=

# @internal @optional
OP_SERVICE_ACCOUNT_TOKEN=

# @required @type=string
EXAMPLE_API_KEY=op("op://example-vault/example-project/api-key")
```

An `op://` reference identifies a vault, item, and field. It contains a lookup path rather than the secret value.
See [1Password secret references](https://www.1password.dev/cli/secret-references) for reference syntax and section paths.
Copy the field's secret reference from 1Password. Keep private names and IDs outside public configuration when necessary.
This model does not need `OP_ENVIRONMENT_ID` or the bulk Environment resolver.
The existing startup commands and automatic CLI launchers can remain.
See [Varlock field references](https://varlock.dev/plugins/1password/#pulling-data-from-1password) for the resolver and authentication options.

The schema limits what Varlock injects into the application. It does not limit the account's vault-wide read access.
A stolen token or unrestricted agent can read other projects' items in the same vault.
Use separate vaults and accounts when that exposure is unacceptable.

Where access requirements differ, use separate scopes.
For example, signing or publishing credentials can need tighter access than development credentials.
A separate vault and account can provide that scope without prescribing a fixed number of vaults.
Separate vaults do not isolate credentials if the same unrestricted Unix user can read both bootstrap token files.
If a token must be unavailable to an agent, keep it outside that agent's Unix account.
Use a controlled session, a different operating-system user, or an isolated credential broker where that boundary is needed.
Read access in 1Password does not reduce a retrieved credential's signing, publishing, or other application powers.
Cloudflare production secrets remain separate unless the deployment workflow deliberately integrates them.

Before replacing an Environment-based setup, verify the new field references and token scope with the existing safe checks.
Keep the working account available until verification passes. Revoke unused accounts after migration.

## 2. Store one server bootstrap token

Use a platform secret manager when available.
For this user-level server pattern, one protected token file supplies the loader.
This file contains an unencrypted credential. Its owner, root, and accessible backups can read it.
It avoids separate local copies of each application secret.

In an interactive **Bash** session as the development user, run:

```bash
set +x
umask 077
install -d -m 700 "$HOME/.config/varlock/tokens"
token_tmp=$(mktemp "$HOME/.config/varlock/tokens/.shared-creds.XXXXXX")
read -r -s -p 'Service account token: ' service_token
printf '\n'
if [ -z "$service_token" ]; then
  rm -f "$token_tmp"
  unset service_token token_tmp
else
  printf '%s\n' "$service_token" > "$token_tmp"
  unset service_token
  chmod 600 "$token_tmp"
  mv -f "$token_tmp" "$HOME/.config/varlock/tokens/shared-creds"
  unset token_tmp
fi
```

Paste the token only at the hidden prompt.
For a separate project account, change the file name to `example-project-dev`.
For a reusable vault account, use a name that identifies its access scope.
Do not export the token in global shell startup files.
Keep token directories at mode `700` and files at mode `600`, owned by the development user.
An existing protected token file can supply the resolver. Do not make another copy without a need.

## 3. Define a shared CLI schema

Create the user configuration directory:

```bash
install -d -m 700 "$HOME/.config/varlock/shared-creds"
```

Create `$HOME/.config/varlock/shared-creds/.env.schema` with this example:

```dotenv
# @plugin(@varlock/1password-plugin@2.0.5)
# @import(~/.config/varlock/shared-creds/.env.local, pick=[OP_TOKEN, OP_ENVIRONMENT_ID], allowMissing=true)
# @initOp(token=$OP_TOKEN, allowAppAuth=true, useCliWithServiceAccount=true)
# @setValuesBulk(opLoadEnvironment($OP_ENVIRONMENT_ID))
# @defaultSensitive=true
# ---

# @type=opServiceAccountToken @internal @optional
OP_TOKEN=

# @internal @optional
OP_SERVICE_ACCOUNT_TOKEN=

# @internal @required
OP_ENVIRONMENT_ID=

# @required
EXAMPLE_API_KEY=

# @public @optional
EXAMPLE_REGION=
```

Declare each expected application variable. Keep `createMissing` disabled.
Mark credentials sensitive. Mark only intended nonsecret settings public.
Use types and `@required` for values that must exist before startup.
Use `@optional` only when the application has a defined fallback.
Pin the plugin version when a standalone schema loads it directly.

Create `$HOME/.config/varlock/shared-creds/.env.local`.
On the Mac, supply only the Environment ID:

```dotenv
OP_ENVIRONMENT_ID=replace-with-environment-id
```

On the server, also add the protected-token resolver:

```dotenv
OP_ENVIRONMENT_ID=replace-with-environment-id
OP_TOKEN=exec("cat /absolute/path/to/.config/varlock/tokens/shared-creds")
```

Replace the path with the development user's actual absolute path.
Keep this configuration file outside repositories. Set its mode to `600`.
The `exec()` output is consumed by Varlock. Do not run the underlying `cat` command in a logged terminal.

`OP_TOKEN` is internal and is removed from the child environment.
The loader uses `op` for desktop or service-account authentication in this configuration.
See the [1Password plugin](https://varlock.dev/plugins/1password/) and [internal items](https://varlock.dev/reference/item-decorators/#internal).

Existing shell exports and higher-precedence files can override fetched values.
Remove obsolete exports through a private workflow after verification.
See [bulk-value precedence](https://varlock.dev/reference/root-decorators/#setvaluesbulk).

## 4. Load shared values automatically for a CLI

Install a real executable launcher instead of a shell alias.
This also works for agents that use noninteractive shells.

1. Install the verified application binary at `$HOME/.local/lib/example-cli/example-cli`.
2. Keep its original basename, especially when its updater uses the executable path.
3. If moving an existing binary, first confirm that it is the real binary rather than an existing launcher.
4. Keep the original binary available until the launcher passes verification.

Publish the launcher with an atomic rename:

```bash
install -d -m 755 "$HOME/.local/bin" "$HOME/.local/lib/example-cli"
launcher_tmp=$(mktemp "$HOME/.local/bin/.example-cli.XXXXXX")
cat > "$launcher_tmp" <<'SH'
#!/bin/sh
set -eu
exec varlock run \
  --path "$HOME/.config/varlock/shared-creds/.env.schema" \
  --inject vars -- \
  "$HOME/.local/lib/example-cli/example-cli" "$@"
SH
chmod 755 "$launcher_tmp"
mv -f "$launcher_tmp" "$HOME/.local/bin/example-cli"
unset launcher_tmp
```

Run these commands as the development user. Ensure all installed files have that user as owner.
The result is a normal command:

```bash
example-cli some-command
```

Varlock resolves values before the real binary starts.
`--inject vars` omits the serialized configuration blob.
It still gives the application its declared secrets.
The schema also marks the CLI's ambient service-account token internal.
This prevents an old token export from reaching the application.

For several tools, use a narrower schema or injection filter for each tool when their credential needs differ.
Do not make every tool receive every shared credential without a reason.

For verified commands that need no secrets, a launcher can bypass Varlock:

```sh
case "${1-}" in
  --help|--version|update|completion)
    exec "$HOME/.local/lib/example-cli/example-cli" "$@"
    ;;
esac
```

Insert that block before `exec varlock run` only after checking the tool's behavior.
Confirm that self-update modifies the real binary and preserves the launcher.
External installers can replace `$HOME/.local/bin/example-cli`.
Reapply the launcher after such installations and repeat verification.

## 5. Put project loading in repository scripts

Give each project its own schema.
Use its selected Environment or dedicated-vault field references, according to the chosen access model.
Keep the secret-loading step in the project's existing startup command.
For SvelteKit setup and script consumers, follow the [stack guide](../stacks/sveltekit/environment-variables-with-varlock-and-1password.md).
For a Bun and Vite project, an example `package.json` entry is:

```json
{
  "scripts": {
    "dev": "varlock run --path .env.schema --inject vars -- vite dev",
    "env:check": "varlock load --path .env.schema --agent"
  }
}
```

Install pinned `varlock` and `@varlock/1password-plugin` dependencies with the project's package manager.
Commit the lockfile. Use `@plugin(@varlock/1password-plugin)` when the plugin is installed as a project dependency.
Keep the project's existing dev-server arguments after `--`.

Adapt the shared schema for project use:

- Import `~/.config/varlock/example-project/.env.local` with only `OP_TOKEN` and `OP_ENVIRONMENT_ID` selected.
- Declare only the project's application variables.
- Restrict bulk loading with `pick=[EXAMPLE_API_KEY, EXAMPLE_REGION]` when the Environment has additional keys.
- Keep bootstrap settings internal, including the legacy `OP_SERVICE_ACCOUNT_TOKEN` name.

Create the external machine configuration on each development machine.
Use desktop authentication on the Mac and the chosen scoped token on the server.
The ordinary startup command stays:

```bash
bun run dev
```

Only commands wrapped by a script or integration load those values.
A direct `vite dev` or unrelated shell command can bypass the configured path.
The explicit `--path .env.schema` selects the schema instead of Varlock's automatic discovery of project environment files.
The limited import accepts only the two bootstrap settings from machine configuration.
Other tools, such as Bun or Vite, can still load their own environment files. Review their behavior during migration.
For Bun, put this in the package's `bunfig.toml` to disable automatic dotenv loading:

```toml
env = false
```

Use `bun --no-env-file` for child Bun commands when necessary.
See [Bun environment loading](https://bun.com/docs/runtime/environment-variables) for these settings and explicit-file behavior.
Remove obsolete exported application values through a private migration workflow.
Intentional process exports remain standard Varlock overrides. The provider does not always take precedence over them.
Wrap other commands only when they need the same configuration.
Keep production deployment scripts separate from development secret loading.

Framework integrations are another option.
Use the [SvelteKit integration](https://varlock.dev/integrations/sveltekit/) when the project needs Varlock's code-level features.
Do not add duplicate loaders or change application environment access without a defined need.
Scripts that already read `process.env` need only startup configuration.
Scripts that explicitly read an environment file need a small lookup adapter or an automatically generated file.
Generated files need private permissions, cleanup, and a defined refresh lifecycle.

## Cloudflare Workers

Local Varlock loading does not update deployed Worker secrets.
Manage production secrets with the project's Wrangler or deployment workflow.
See the [Worker binding contract](../stacks/cloudflare-workers/conventions.md#runtime-and-bindings).

For local Worker development, prefer an explicit `secrets.required` list in Wrangler configuration.
Wrangler can load those names from the command's process environment.
Verify the selected Wrangler version and the project's adapter behavior.
An existing `.dev.vars` file can change which values are loaded.

Without `secrets.required`, `CLOUDFLARE_INCLUDE_PROCESS_ENV=true` can include process variables when no `.dev.vars` file exists.
Prefer the explicit list to exposing the whole process environment.
See [Cloudflare local environment variables](https://developers.cloudflare.com/workers/local-development/environment-variables/).

An intentional production integration needs separate scope, credentials, deployment checks, and rotation rules.
Do not upload development values automatically.

## Verification

Run checks as the same user and through the same launch method as the application or agent.
Use [Varlock's load and run commands](https://varlock.dev/reference/cli/load-and-run/).

```bash
command -v example-cli
varlock load --path "$HOME/.config/varlock/shared-creds/.env.schema" --agent
```

The command path must select the launcher.
The schema check must succeed without exposing sensitive values.
Public values can appear in `--agent` output, so do not mark private metadata public.

Check injection with presence-only output:

```bash
varlock run \
  --path "$HOME/.config/varlock/shared-creds/.env.schema" \
  --inject vars -- python3 -c '
import os
assert os.environ.get("EXAMPLE_API_KEY"), "required key missing"
assert "OP_TOKEN" not in os.environ, "bootstrap token was injected"
assert "OP_SERVICE_ACCOUNT_TOKEN" not in os.environ, "ambient bootstrap token was inherited"
assert "OP_ENVIRONMENT_ID" not in os.environ, "internal ID was injected"
assert "__VARLOCK_ENV" not in os.environ, "configuration blob was injected"
print("required values present; internal values absent")
'
```

Then run one read-only application command through the normal CLI or project script.
Test a noninteractive agent command separately.
Check that the application exits before startup when a required value is missing.
Confirm optional features follow their documented fallback.

Do not print `env`, `printenv`, raw `op environment read`, or unredacted Varlock output in logs or agent transcripts.
Avoid `varlock load --format env`, `shell`, `json`, or `json-full` without an appropriate private consumer.
Do not enable shell tracing around credentials.
Varlock redacts piped output by default. Interactive terminals use terminal passthrough.
An application can still disclose a secret, and an unrestricted agent can deliberately retrieve it.

## Rotation and recovery

Changing a value in 1Password affects the next load unless caching is configured.
Restart existing processes to fetch changed values.
For a cache-enabled configuration, use `--skip-cache` during verification or clear the relevant cache.

To rotate a bootstrap token:

1. Create a replacement service account with the required scope.
2. Save its token privately.
3. Replace the protected token file through the hidden-input procedure.
4. Run schema and application checks on each consuming machine.
5. Restart long-running consumers.
6. Revoke the old service account after the checks pass.

| Failure | Recovery |
| --- | --- |
| Access denied | Check the selected Environment ID and service-account scope. Replace the account if its scope is wrong. |
| Desktop prompt on a headless server | Check the token file, its owner, and the `OP_TOKEN` resolver. |
| Old values | Check shell exports, imported overrides, caches, and process restart. |
| Plain command bypasses loading | Check `PATH` and whether another installer replaced the launcher. |
| Updater damages the launcher | Restore the verified real binary and reinstall the launcher. |
| Required value missing | Add it in the selected Environment or correct the schema name. |
| 1Password unavailable or rate limited | Leave required commands stopped. Restore access, then retry. Do not write emergency secrets into source. |

To undo CLI automatic loading, restore the original binary at `$HOME/.local/bin/example-cli` through an atomic replacement.
To undo project loading, restore the prior script and its documented configuration path.
Keep the loader configuration until the replacement workflow passes verification.
Remove obsolete token files and revoke unused accounts afterward.

## Official documentation

- [1Password Environments](https://www.1password.dev/environments)
- [1Password Environment reading and authentication](https://www.1password.dev/environments/read-environment-variables)
- [1Password service-account creation and scope](https://www.1password.dev/service-accounts/get-started)
- [1Password service-account security and vault boundaries](https://www.1password.dev/service-accounts/security)
- [1Password secret references](https://www.1password.dev/cli/secret-references)
- [1Password CLI desktop app integration](https://www.1password.dev/cli/app-integration)
- [1Password service-account rate limits](https://www.1password.dev/service-accounts/rate-limits)
- [Varlock installation](https://varlock.dev/getting-started/installation/)
- [Varlock 1Password plugin](https://varlock.dev/plugins/1password/)
- [Varlock load and run](https://varlock.dev/reference/cli/load-and-run/)
- [Varlock root decorators and bulk loading](https://varlock.dev/reference/root-decorators/)
- [Varlock sensitivity and internal items](https://varlock.dev/reference/item-decorators/)
- [Varlock resolver functions](https://varlock.dev/reference/functions/)
- [Varlock SvelteKit integration](https://varlock.dev/integrations/sveltekit/)
- [Bun environment loading](https://bun.com/docs/runtime/environment-variables)
- [Cloudflare local environment variables](https://developers.cloudflare.com/workers/local-development/environment-variables/)
