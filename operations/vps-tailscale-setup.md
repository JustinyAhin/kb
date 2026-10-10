# Ubuntu VPS setup with Tailscale SSH

Use this runbook for a fresh Ubuntu 24.04 LTS or newer VPS. It sets up private
administration through Tailscale SSH, then blocks public SSH with UFW. Public
application ports stay closed unless you open them later.

Reviewed against the linked upstream documentation on 2026-10-10. All names,
addresses, and identities below are examples. Replace them before use.

A VPS is a virtual private server. A tailnet is your private Tailscale network.
UFW is Ubuntu's firewall management tool. MagicDNS resolves Tailscale device names.

## Prerequisites

- A fresh VPS with working OpenSSH access and your public SSH key installed.
- A working provider console or rescue system. Confirm access before changing
  the firewall.
- A client computer signed in to your tailnet with your user identity.
- Permission to edit the tailnet policy and approve devices, if required.
- MagicDNS enabled, or the server's Tailscale IP address available.

Keep the initial SSH session open until the final checks pass. Keep its SSH key
for recovery. This runbook does not disable OpenSSH or change its listener.

## Choose the SSH mode

| Mode | SSH server | Authentication |
| --- | --- | --- |
| OpenSSH over Tailscale | The existing `sshd` | SSH keys and OpenSSH settings |
| Tailscale SSH, used here | An SSH server inside `tailscaled` | Tailnet identity and SSH access rules |

Tailscale SSH takes over tailnet traffic to port 22. Public-address SSH still
uses OpenSSH. It does not change `sshd_config` or `authorized_keys`.
Do not bind `sshd` or `ssh.socket` to a fixed Tailscale address for this setup.
See [Tailscale SSH](https://tailscale.com/docs/features/tailscale-ssh).

## 1. Prepare the server account

From your client, connect with the account supplied by the provider. For a
root-based image, use:

```bash
ssh root@203.0.113.10
```

On the server, install the base tools:

If the provider account is `root` and `sudo` is not installed, omit `sudo` from
these initial package commands.

```bash
sudo apt update
sudo apt install -y ca-certificates curl ufw sudo
```

Use a separate Linux account for routine administration. The examples use
`admin`. If it does not exist, create it. Set a password for `sudo` when asked:

```bash
sudo adduser admin
sudo usermod -aG sudo admin
```

If you use another existing account, replace `admin` in both the commands and
the SSH policy below. Tailscale SSH does not create Linux accounts.

## 2. Add narrow tailnet access rules

Open **Access controls** in the Tailscale admin console. Merge these entries
into the existing policy. Do not replace the policy with this example.
Replace `admin@example.com` with the intended operator's tailnet identity.

```json
{
  "tagOwners": {
    "tag:managed-vps": ["admin@example.com"]
  },
  "grants": [
    {
      "src": ["admin@example.com"],
      "dst": ["tag:managed-vps"],
      "ip": ["tcp:22"]
    }
  ],
  "ssh": [
    {
      "action": "check",
      "src": ["admin@example.com"],
      "dst": ["tag:managed-vps"],
      "users": ["admin"],
      "checkPeriod": "12h"
    }
  ]
}
```

The network grant allows TCP port 22. The separate SSH rule allows login as
the existing Linux user `admin` and requires periodic sign-in checks. Root
login is not included. Add `root` only if you need direct root access and have
reviewed who can use that rule.

The tag identifies servers; `tagOwners` controls who can assign it. Do not
tag your client computer: tagged devices use a service identity instead of
their user's identity. Use this tag only on servers that need these rules.
See [server setup](https://tailscale.com/docs/how-to/set-up-servers) and
[tags](https://tailscale.com/docs/features/tags).

Review existing ACLs, grants, and SSH rules too. Rules are additive: a narrow
new rule does not cancel an existing broad rule. Check the merged policy in
the policy editor, including intended access and access that must be denied.
Do not remove unrelated rules without reviewing their users and services.
See [grants syntax](https://tailscale.com/docs/reference/syntax/grants).

## 3. Install and enroll Tailscale

Run these commands in the original public-address SSH session:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
sudo tailscale up --hostname=runner-1 --advertise-tags=tag:managed-vps
```

Open the sign-in link on your client. Sign in with the authorized tag owner's
account. If your tailnet requires device approval, approve the device. Confirm
that the server appears with the intended tag in the admin console.

For automated enrollment, use a scoped, tagged auth key and keep it out of
shell history, repositories, and logs. This manual path does not need an auth
key. See [Linux installation](https://tailscale.com/docs/install/linux).

Enable Tailscale SSH while the public-address session is still available:

```bash
sudo tailscale set --ssh
tailscale status
tailscale ip -4
```

Enabling this feature can interrupt existing SSH connections to the server's
Tailscale address. The public-address bootstrap session is separate.

## 4. Test a new private connection

In a second terminal on your client, connect through MagicDNS:

```bash
ssh admin@runner-1
```

If name resolution is unavailable, replace `runner-1` with the address returned
by `tailscale ip -4` on the server. Complete the sign-in check if requested.
Then run:

```bash
whoami
sudo -v
```

Confirm that `whoami` returns `admin` and that `sudo` works. Do not restrict
public SSH until this new connection succeeds. A successful `tailscale ping`
alone does not prove that the SSH policy or Linux account is correct.

## 5. Check device key expiry

Inspect the server's key expiry in the admin console. When a device is first
tagged and authenticated, key expiry is disabled by default. Changing tags on
an already enrolled device does not always change its existing expiry state.
Verify the displayed setting. See [tag expiry behavior](https://tailscale.com/docs/features/tags#key-expiry).

For a trusted unattended server, disabled device key expiry can avoid loss of
remote access. If you enable expiry, plan renewal and console recovery.
Keep expiry on user devices unless you have a specific reason to change it.
Device key expiry and the SSH rule's 12-hour sign-in check are separate.
See [key expiry](https://tailscale.com/docs/features/access-control/key-expiry).

## 6. Restrict public access with UFW

Continue from the working Tailscale SSH session. These commands assume a fresh
server. Inspect existing rules first; do not reset an active service's firewall.

```bash
sudo ufw status verbose
sudo ufw show added
```

Keep a temporary public SSH rule during the change. Schedule a rollback that
disables UFW after ten minutes if you lose access:

```bash
sudo ufw allow 22/tcp comment 'temporary bootstrap SSH'
sudo systemd-run --unit=vps-firewall-rollback --on-active=10m /usr/sbin/ufw disable
sudo systemctl status vps-firewall-rollback.timer --no-pager
```

Do not continue unless the rollback timer is active and the provider console
is available. Apply the rules, then answer the UFW confirmation prompt:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw enable
sudo ufw status verbose
```

UFW now allows traffic on the Tailscale interface. Tailnet policy still
controls which peers and ports can connect. The example grant permits only
TCP port 22. Other private services need their own network grants.
See [Tailscale's UFW guide](https://tailscale.com/docs/how-to/secure-ubuntu-server-with-ufw).

Open another new Tailscale SSH connection. Check `sudo -v`. If it works,
remove the temporary public SSH rule:

```bash
sudo ufw delete allow 22/tcp
sudo ufw status numbered
```

If other public SSH rules exist, such as an `OpenSSH` profile rule, remove only
those rules after reviewing them. Use `sudo ufw delete N`, where `N` is the
current rule number. List the rules again after each deletion because numbers
change. Check both IPv4 and IPv6 entries. Leave the `tailscale0` rule in place.

## 7. Verify before canceling rollback

Keep the working session open. From your client, make a new private connection:

```bash
ssh admin@runner-1
```

Also try the public address with a short timeout:

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 root@203.0.113.10
```

The public connection must fail before authentication, usually with a timeout
or connection refusal. `Permission denied (publickey)` means port 22 is still
reachable. Test the public IPv6 address too, if one is assigned.

On the server, confirm that UFW is active and that no unwanted public allow
rules remain. Only after all checks pass, cancel the rollback:

```bash
sudo ufw status verbose
sudo systemctl stop vps-firewall-rollback.timer
sudo systemctl is-active vps-firewall-rollback.timer
```

The last command should report `inactive`. If the timer already fired, UFW
will be disabled; repeat the firewall steps and verification. Do not treat a
working connection as proof that the firewall is active.

Reboot once at a suitable time. Test a new Tailscale SSH connection.
Check `systemctl is-active tailscaled` and `sudo ufw status verbose` again.

## Docker and application ports

Docker published ports can bypass UFW. A mapping such as `8080:80` can expose a
container on public host addresses despite UFW's incoming default. Do not rely
on UFW alone for container access. See [Docker firewall behavior](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw).

For a private local service, bind the published port to loopback. For example,
the service's Compose entry can use:

```yaml
ports:
  - "127.0.0.1:8080:80"
```

Use Docker 28 or newer for this boundary. Older releases can allow hosts on
the same layer-2 network to reach ports published to loopback.

Access that service from your client with an SSH tunnel:

```bash
ssh -N -L 8080:127.0.0.1:8080 admin@runner-1
```

Then open `http://127.0.0.1:8080` on your client. Alternatively, bind a service
to the server's Tailscale address and add only the required tailnet grant.
Review Docker's routing mode before relying on an address binding. Use a
supported Docker-aware firewall or the provider's firewall where required.
See [Docker port publishing](https://docs.docker.com/engine/network/port-publishing/).

Public services are opt-in. For a host-based HTTPS service, allow `443/tcp` in
UFW and in any provider firewall only when you intend public access. Review
Docker exposure separately if the service runs in a container.

`fail2ban` is optional if public OpenSSH remains accessible. It does not replace
Tailscale SSH identity checks and is not required for this default setup.

## Recovery

Use the provider console if Tailscale access fails. To undo the UFW change:

```bash
sudo ufw disable
sudo systemctl status tailscaled --no-pager
```

This can restore public OpenSSH access if OpenSSH is still running and the
provider firewall permits it. Fix the cause, then repeat the private access
and firewall checks before enabling UFW again.

If a device key expired, use the admin console's temporary key extension or
reauthenticate through the provider console. Do not run
`tailscale up --force-reauth` through your only remote connection: it can
disconnect that connection. Review the existing enrollment flags before
reauthentication. See [expired-device recovery](https://tailscale.com/docs/features/access-control/key-expiry#renewing-keys-for-an-expired-device).

To return to standard OpenSSH over Tailscale, use the provider console or another
recovery path. Configure an SSH public key for the chosen Linux user.
Run `sudo tailscale set --ssh=false`. Test a new private SSH connection.
If it fails, use the console to restore Tailscale SSH with `sudo tailscale set --ssh`.
Keep the TCP port 22 network grant. The Tailscale SSH policy does not authorize
OpenSSH logins.
