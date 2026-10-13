# Environment variables with Infisical

Use this optional workflow to store project and CLI credentials in Infisical.
Each command fetches its selected values when it starts.
Package scripts and CLI launchers can keep ordinary commands such as `bun run dev` and `example-cli`.

This guide describes reusable setup steps.
Keep exact variable names, release requirements, and commands in the project's maintained configuration and runbook.
All names, paths, and identifiers below are examples.
Keep real credentials and private infrastructure details outside this public repository.
Reviewed against the linked official documentation on 2026-10-13.

## Structure and access

| Term                 | Purpose                                                                               |
| -------------------- | ------------------------------------------------------------------------------------- |
| Organization         | Contains projects, users, and organization machine identities.                        |
| Project              | Groups credentials with related access requirements.                                  |
| Environment          | Selects a named set, such as `dev`, `staging`, or `prod`.                             |
| Folder               | Groups related values within an environment, such as `/web`, `/build`, or `/release`. |
| Machine identity     | Authenticates a server or workload without a person's account.                        |
| Universal Auth       | Exchanges a client ID and client secret for an access token.                          |
| Bootstrap credential | Lets the loader authenticate before it can fetch application secrets.                 |

```text
Infisical organization
  application project
    dev:/web        -> local application commands
    prod:/web       -> selected production workflow
  maintainer project
    prod:/build     -> build inputs
    prod:/release   -> signing and publishing inputs
  shared-tools project
    dev:/cli        -> installed CLI credentials

Mac user login or server machine identity
  -> infisical run
  -> selected application and child processes

Cloudflare deployed Worker secrets
  -> deployed Worker
```

Choose scopes by access requirements.
Folders organize data. A folder does not restrict access by itself.
The built-in **Viewer** project role reads all resources in that project, including every environment and folder.
Use separate projects when one identity must not read another set of credentials.
Use scoped roles or privileges when the selected plan supports them.
See [Infisical access controls](https://infisical.com/docs/documentation/platform/access-controls/role-based-access-controls).

Create an organization machine identity when one server needs several projects.
Give it **No Access** at organization level.
Add it to each required project with read access.
You can add another project later without creating another identity or client secret.
See [machine identity scope](https://infisical.com/docs/documentation/platform/identities/machine-identities).

Read access in Infisical does not reduce a retrieved credential's publishing or signing powers.
An unrestricted agent under the same Unix user can read protected files and injected values.
Separate Unix users or an isolated credential broker are necessary when those values must remain unavailable to the agent.

## 1. Install and select the project

Install the CLI from the [official installation guide](https://infisical.com/docs/cli/install).
Use compatible CLI versions on the Mac, server, and CI.
Pin production installations to a tested release.

On the Mac:

```bash
brew install infisical/get-cli/infisical
infisical --version
infisical login
```

Select the correct Cloud region or self-hosted instance during setup.
Use its documented `--domain` setting for login and secret commands.
Use HTTPS. Review a repository's configured domain before sending credentials to it.

In Infisical:

1. Create the project.
2. Select the environment.
3. Create folders for distinct command inputs.
4. Add the application's existing variable names.
5. Enter values through a private workflow.

In the repository root:

```bash
infisical init
```

The project configuration contains selection metadata, not application secret values.
Current CLI releases use `.infisical.yaml`.
Older repositories can use `.infisical.json`.
Keep the existing format until every consuming CLI supports its replacement.
See [project configuration](https://infisical.com/docs/cli/project-config) and [configuration migration](https://infisical.com/docs/cli/project-config-migration).

Machine identity commands need `--projectId` or `INFISICAL_PROJECT_ID`.
Do not depend on the project configuration alone for machine authentication.
Keep actual IDs in private machine configuration when they reveal private infrastructure.

## 2. Put loading in project scripts

Wrap commands that need secrets in the repository's normal package scripts.
For example:

```json
{
  "scripts": {
    "dev": "infisical run --projectId=replace-with-project-id --env=dev --path=/web --silent -- env -u INFISICAL_TOKEN -u INFISICAL_UNIVERSAL_AUTH_ACCESS_TOKEN vite dev",
    "build": "infisical run --projectId=replace-with-project-id --env=prod --path=/build --secret-overriding=false --silent -- env -u INFISICAL_TOKEN -u INFISICAL_UNIVERSAL_AUTH_ACCESS_TOKEN bun --no-env-file scripts/build.ts"
  }
}
```

Replace the example ID during private project setup.
Use the project's package manager and existing arguments.
The fixed `env -u` arguments remove ambient access tokens before application startup.
Remove other bootstrap names if the runner exports them.
Application variable names do not need a separate list in these commands.
The normal command stays:

```bash
bun run dev
```

Scripts that already read `process.env` can keep that access.
SvelteKit applications can keep their existing `$env` imports.
Startup injection does not require a new environment accessor in application code.
Code that explicitly reads a file needs a separate migration decision.

Only wrapped commands fetch values.
A direct `vite dev` or another shell command can bypass this workflow.
Infisical does not add every project's secrets to every shell automatically.

### Avoid competing environment files

Check every tool that starts before and after Infisical.
Remove obsolete shell exports and secret files only after the new workflow passes verification.
Keep a private recovery copy until migration is complete.

For Bun package scripts, disable automatic file loading in `bunfig.toml`:

```toml
env = false
```

Use `bun --no-env-file` for child Bun commands when necessary.
Explicit `--env-file` arguments still load files.
See [Bun environment loading](https://bun.com/docs/runtime/environment-variables#disabling-automatic-env-loading).

For Vite, `envDir: false` disables its environment file loading.
SvelteKit also has `kit.env.dir`.
For versions that load files through this setting, select a dedicated directory that contains no secret files.
Setting Vite's option alone does not establish SvelteKit's behavior.
Check the installed framework version and run a presence-only test.
See [Vite `envDir`](https://vite.dev/config/shared-options.html#envdir) and [SvelteKit environment configuration](https://svelte.dev/docs/kit/configuration#env).

Keep public prefixes such as `PUBLIC_` and `VITE_` limited to values intended for browsers.
Loading a credential from Infisical does not make browser-exposed variables private.

### References and overrides

Use [secret references](https://infisical.com/docs/documentation/platform/secret-reference) to reuse a value within a project.
For example, `${prod.build.EXAMPLE_CLIENT_SECRET}` references a value in `prod:/build`.
The fetching identity needs read access to every referenced scope.
Check reference resolution without printing the value.

Keep expansion enabled when references must resolve.
The current CLI reference describes `--expand=false` as injecting values exactly as stored.
Expansion behavior can differ between CLI versions and authentication paths.
Test both user and machine authentication after upgrades.
For literal multiline signing input, verify exact bytes through a private consumer.
Do not print keys or their content.

`--secret-overriding=false` selects shared values instead of a person's same-name personal overrides.
Use this setting for maintained builds and releases when every machine must use the shared inputs.
Cross-project sharing is a separate feature with plan and grant requirements.
Do not assume a shared-tools project automatically supplies other projects.
See [CLI `run`](https://infisical.com/docs/cli/reference#run) and [cross-project sharing](https://infisical.com/docs/documentation/platform/secret-reference#cross-project-secret-sharing).

### Optional Varlock validation

Infisical can supply values while Varlock validates and filters them.
For example:

```json
{
  "scripts": {
    "build": "infisical run --projectId=replace-with-project-id --env=prod --path=/build --secret-overriding=false --silent -- varlock run --path env/build --inject vars -- bun --no-env-file scripts/build.ts"
  }
}
```

Use a schema with ambient values, required names, types, and sensitivity settings.
Remove provider imports from that schema when Infisical replaces the provider.
Declare bootstrap credentials and unused loaded names as `@internal`.
Include `INFISICAL_TOKEN` and any client credential names used by the authentication setup.
Include the native `INFISICAL_UNIVERSAL_AUTH_CLIENT_ID`, `INFISICAL_UNIVERSAL_AUTH_CLIENT_SECRET`, and `INFISICAL_UNIVERSAL_AUTH_ACCESS_TOKEN` names when applicable.
Include legacy provider bootstrap names during migration.
Verify that they are absent from the final application environment.

Varlock filters declared names. It is not a general process sandbox.
Pin its dependency and keep validation rules in the project's schema.
See [Varlock internal values](https://varlock.dev/reference/item-decorators/) and [load and run](https://varlock.dev/reference/cli/load-and-run/).

## 3. Authenticate the server user

Install system tools as root when necessary.
Run authentication and application commands as the normal development user.
The server does not need a root personal login.

1. Open **Access Control → Machine Identities** in the Infisical organization.
2. Create or select the server's organization identity.
3. Confirm its project grants and organization role.
4. Enable Universal Auth.
5. Create a client secret.
6. Save the client ID and client secret through a protected private workflow.

The Universal Auth client ID differs from the machine identity ID.
Do not put the client secret in Git, chat, shell history, or global shell startup files.
Keep user credential directories at mode `700` and credential files at mode `600`.
The file owner, root, and accessible backups can read these unencrypted credentials.

For an initial interactive **Bash** session, use hidden input:

```bash
set +x
read -r -p 'Universal Auth client ID: ' INFISICAL_UNIVERSAL_AUTH_CLIENT_ID
read -r -s -p 'Universal Auth client secret: ' INFISICAL_UNIVERSAL_AUTH_CLIENT_SECRET
printf '\n'
export INFISICAL_UNIVERSAL_AUTH_CLIENT_ID INFISICAL_UNIVERSAL_AUTH_CLIENT_SECRET
if identity_token="$(
  infisical login --method=universal-auth --silent --plain
)" && [ -n "$identity_token" ]; then
  export INFISICAL_TOKEN="$identity_token"
else
  unset INFISICAL_TOKEN
  printf '%s\n' 'Infisical authentication failed.' >&2
fi
unset INFISICAL_UNIVERSAL_AUTH_CLIENT_ID INFISICAL_UNIVERSAL_AUTH_CLIENT_SECRET identity_token
```

Paste the secret only at the hidden prompt.
This avoids a literal secret in shell history.
The native client variables supply login without putting the secret in process arguments.
They remain available to the login process and privileged local processes.
The generic `INFISICAL_CLIENT_ID` and `INFISICAL_CLIENT_SECRET` names do not perform this native login automatically.

Machine login outputs an access token. It does not create a persistent user login.
The exported token applies only to that shell and its child processes.
Run project commands in that session, with an explicit project selection.
Remove bootstrap variables before the final application starts, through the launcher or Varlock schema.

Access tokens expire.
Universal Auth renewal cannot exceed the configured maximum lifetime unless periodic tokens are configured.
Periodic tokens also need renewal before expiry.
Client secret expiry and use limits are separate from access token lifetime.
Do not save a token and assume it remains valid indefinitely.
See [Universal Auth](https://infisical.com/docs/documentation/platform/identities/universal-auth) and [CLI login](https://infisical.com/docs/cli/reference#login).

### Unattended authentication

The [Infisical Agent](https://infisical.com/docs/integrations/platforms/infisical-agent) can authenticate and renew tokens automatically.
It can write each renewed token to a protected file, called a token sink.
Application secret templates are optional.
Use token management alone when `infisical run` will fetch application values at command startup.

For example, configure a user-level agent with private paths:

```yaml
infisical:
  address: https://app.infisical.com
auth:
  type: universal-auth
  config:
    client-id: /home/developer/.config/infisical/client-id
    client-secret: /home/developer/.config/infisical/client-secret
sinks:
  - type: file
    config:
      path: /home/developer/.config/infisical/access-token
```

The two client fields are file paths, not credential values.
Create their contents through hidden input or a protected credential transfer.
Restrict all credential files and the token sink to the development user.
Use absolute paths that match that user.

Start the agent with its configuration:

```bash
infisical agent --config /home/developer/.config/infisical/agent.yaml
```

Manage this process through the server's service supervisor for persistent operation.
Set its Unix user and restrictive file creation permissions explicitly.
Check token file permissions after initial creation and renewal.
Define failure handling, restart behavior, and credential rotation in the runner's private runbook.

Each job or CLI launcher must read the latest token sink immediately before calling Infisical.
For example, inside the launcher's process:

```bash
set +x
if ! INFISICAL_TOKEN="$(cat "$HOME/.config/infisical/access-token")"; then
  exit 1
fi
[ -n "$INFISICAL_TOKEN" ] || exit 1
export INFISICAL_TOKEN
```

Check that the file exists and contains a token before launching the application.
Use the project's normal script after this step.
Strip the token before the final application starts.
Reading the file once at shell startup keeps an old token after the agent replaces the file.
Replacing a token file does not update an existing process environment.

Do not assume an interactive shell file configures an agent service.
Check the service's Unix user, `PATH`, working directory, and bootstrap access.

For a containerized Statamic deployment, use the [Dokploy runbook](statamic-infisical-dokploy.md).

## 4. Load shared CLI credentials automatically

Installed tools have no repository package script to perform loading.
Use a user-owned launcher when their normal command must fetch shared credentials.

1. Keep the real executable at a distinct, stable path.
2. Put the launcher earlier in `PATH`.
3. Select the shared-tools project, environment, and folder explicitly.
4. Authenticate through the runner's bootstrap setup.
5. Strip bootstrap credentials before starting the real executable.
6. Pass arguments and exit status through unchanged.

Use `exec` in the final launcher step.
Avoid calling the launcher itself recursively.
Pass harmless help or version commands directly only when the tool supports that behavior.
Check self-update behavior. An installer can overwrite a launcher.
Use the same `PATH` for interactive commands and the agent runner.

A launcher selects runtime CLI credentials.
A maintainer build or release script selects its separate maintainer scope.
Installing a CLI does not give it the signing or publishing credentials needed to maintain that CLI.

## Cloudflare Workers and hosted builds

Local injection does not change deployed Worker secrets.
Keep production values in the project's Wrangler or deployment workflow.
An explicit [Infisical Worker sync](https://infisical.com/docs/integrations/secret-syncs/cloudflare-workers) is a separate integration.
Check its direction, permissions, and deletion behavior before enabling it.

For local Workers, review Wrangler's required secret names and process environment support.
Existing `.dev.vars` files can affect loading.
See [Cloudflare local environment variables](https://developers.cloudflare.com/workers/local-development/environment-variables/).

A hosted build can use platform-provided environment variables without a second Infisical login.
Keep an explicit build script for that path.
Check the configured dashboard or CI command when changing local package scripts.
Do not upload development values automatically.

## Verification, changes, and recovery

Run checks through the normal script or launcher as the application user.
Use presence-only assertions, such as:

```bash
infisical run --projectId=replace-with-project-id --env=dev --path=/web --silent -- \
  python3 -c 'import os; assert os.environ.get("EXAMPLE_API_KEY"); print("required value present")'
```

For a Varlock or launcher check, also assert that bootstrap variables are absent from the final child environment.
Check unresolved references without printing values.
Run a read-only application command and a separate noninteractive runner command.
Confirm startup fails when a required value is missing.

Do not print `env`, `printenv`, secret listings, raw exports, or plain secret reads in agent transcripts.
`--silent` removes informational messages. It does not prevent an application from printing a secret.
Do not enable shell tracing around credentials.

Changed values apply to the next fetch.
Restart running applications to load them.
CLI `--watch` can restart development commands when values change.
It does not alter an existing process's environment in place.

To rotate Universal Auth credentials:

1. Create a replacement client secret.
2. Update the protected bootstrap source.
3. Obtain a new access token.
4. Check every consuming script and runner.
5. Restart long-running consumers.
6. Revoke the old client secret after verification.

| Failure                         | Action                                                                   |
| ------------------------------- | ------------------------------------------------------------------------ |
| Expired access token            | Authenticate again or repair the configured renewal process.             |
| Access denied                   | Check the identity's project role and every referenced scope.            |
| Project not found               | Check explicit project selection, instance, and region.                  |
| Unresolved reference            | Check expansion settings and reference permissions.                      |
| Old values                      | Check personal overrides, competing files, and process restart.          |
| Plain CLI bypasses loading      | Check command resolution and whether an installer replaced the launcher. |
| Runner works only in a terminal | Check service bootstrap configuration and `PATH`.                        |
| Infisical unavailable           | Keep required commands stopped. Restore service access and retry.        |

To undo the migration, restore the previous scripts or launcher and their verified private configuration.
Keep the previous provider available until both Mac and server checks pass.
Remove unused credential files and revoke unused accounts after all consumers migrate.
The [Varlock and 1Password workflow](environment-variables-with-varlock-and-1password.md) remains another optional method.
