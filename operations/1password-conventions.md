# 1Password conventions for tokens

Use these rules for API tokens and other machine credentials in 1Password.

Written on 2026-11-23. All names and identifiers below are examples.

## One item for each token

Do not keep many tokens as fields of one Secure Note. One item for each token
gives each token its own history, tags, and access scope. A service account
can then read some tokens and not others.

Use the **API Credential** category:

| Field | Content |
| --- | --- |
| `username` | A non-secret identifier, such as an access key ID or a client ID |
| `credential` | The secret value |
| `hostname` | The endpoint URL, if the token belongs to one |
| `notesPlain` | Where the token is used, and how to rotate it |

Put a key ID and its secret, or a client ID and its secret, in the same item.

Title format: `<Service> – <purpose>`, for example `Cloudflare R2 – backups`.

## Tags

Give each item these tags:

- `token`
- `service/<service>`, for example `service/cloudflare`
- `project/<project>`, for example `project/example-app`

Filter with `op item list --tags token`.

## References

Use the item ID in `op://` references, not the title:

```text
op://Developer/<item-id>/credential
```

A title that contains `/` breaks the path. An ID also stays the same when you
rename the item.

## Split an old item

1. Find every reference to the old item before you change it: search code,
   config, launch agents, and docs for its title and its ID.
2. Create the new items. Copy each value from field to field with `op` and a
   JSON template. Do not print values.
3. Compare each new value with the old value by hash only, for example
   `printf '%s' "$value" | shasum -a 256`.
4. Update the references, then check that each new path resolves:
   `op read 'op://<vault>/<item-id>/credential' >/dev/null`.
5. Archive the old item (`op item delete <id> --archive`). Do not delete it.

When you list fields of an item, show labels and types only. Field type `OTP`
is not concealed in JSON output: filter it out with the concealed fields.

## Do not

- Do not `grep` or `cat` env files or shell profiles that contain secret
  values. List variable names only.
- Do not put tokens in shell profiles in plain text. Load them from 1Password
  when the command starts, or write them to a user-only file (mode 600).
