# Ubuntu VPS setup with Tailscale SSH

Use this runbook for a fresh Ubuntu 24.04 LTS or newer VPS. It also applies to
Debian 13 (trixie). It sets up private administration through Tailscale SSH,
then blocks public SSH with UFW. Public application ports stay closed unless
you open them later.

Reviewed against the linked upstream documentation on 2026-10-10. Debian 13,
netcup, base hardening, and SSH action notes added on 2026-11-23. All names,
addresses, and identities below are examples. Replace them before use.

For the tools and logins of a new development machine, see
[New machine bootstrap](new-machine-bootstrap.md).

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

A minimal Debian image can have only `root`, with no `sudo` and no `ufw`. The
package command above installs both. Store the `sudo` password of the new
account in your password manager. To set it without showing it, pipe it into
`chpasswd` from the password manager.

### Bootstrap SSH key from 1Password

If the 1Password SSH agent supplies your keys, it offers only the items in
`~/.config/1Password/ssh/agent.toml`. After you create a new SSH Key item, add
an entry for it, then check that `ssh-add -L` lists the key:

```toml
[[ssh-keys]]
item = "SSH – example-server"
vault = "Personal"
```

The entry uses the item title. If you rename the item, change the entry at the
same time. The 1Password CLI cannot edit or rename SSH Key items: use the app.

Without the entry, the server rejects each offered key. After several rejected
attempts, OpenSSH 9.8 and newer blocks the client address for a period
(`PerSourcePenalties`). The log shows `penalty: failed authentication`. Wait
for the block to expire, or connect through Tailscale.

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

### Choose `check` or `accept`

`check` asks for a browser sign-in again after each `checkPeriod`. A person at
a terminal sees the link. A background client does not see it. For example,
a terminal multiplexer that reconnects through the system `ssh` fails until
you reconnect by hand. The tailscaled log on the server then shows repeated
`error sending auth welcome message: EOF` lines.

Use `accept` when background clients must reconnect without a prompt:

```json
{
  "action": "accept",
  "src": ["admin@example.com"],
  "dst": ["tag:managed-vps"],
  "users": ["admin"]
}
```

With `accept`, any device that is signed in as the operator can open SSH
without a second prompt. Keep `src` narrow. To limit the rule to some devices,
limit the network grant: put device addresses in `hosts` and use those host
names in the grant `src`. SSH rules accept users, groups, and tags, not hosts.

### Use one tag for each login model

Rules attach to tags. If one tag permits `root` for older servers, a new server
with that tag also permits `root`. Give a server that uses a non-root account
its own tag, grant, and SSH rule. Do not add its user to a shared rule.

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

Turn on automatic Tailscale updates. Unattended upgrades do not cover the
Tailscale package source by default:

```bash
sudo tailscale set --auto-update
```

### Change the tag of an enrolled server

You can change the tags of a device in the admin console (**Machines** →
device → **Edit ACL tags**). The device keeps its old advertised tags in its
local preferences. A later `tailscale up` or a new sign-in can restore the old
tag and its rules. After a tag change in the console, set the same tag on the
server. `tailscale set` has no `--advertise-tags` flag, so use `tailscale up`
with all flags that you used before:

```bash
sudo tailscale up --hostname=runner-1 --ssh --advertise-tags=tag:new-tag
sudo tailscale debug prefs | grep -A2 AdvertiseTags
```

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

Renewing an expired key does not disable expiry. The renewed key gets a new
expiry date. If you want expiry disabled, select **Disable key expiry** after
the renewal. Then check the displayed setting again.

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

## 8. Add a provider firewall

A provider firewall filters traffic before it reaches the server. Docker cannot
bypass it. Use one on every server that runs Docker. The example uses the
Hetzner Cloud CLI. Other providers have equivalent controls.

Prerequisites:

- A provider API token with write access, read from a password manager.
- A working Tailscale SSH connection and the provider console.

Allow only the public services that you need. This example allows HTTPS and
direct Tailscale connections:

```bash
export HCLOUD_TOKEN="$(op read "op://<vault>/<item>/<field>")"

cat > fw-rules.json <<'EOF'
[
  {"direction": "in", "protocol": "tcp", "port": "80", "source_ips": ["0.0.0.0/0", "::/0"]},
  {"direction": "in", "protocol": "tcp", "port": "443", "source_ips": ["0.0.0.0/0", "::/0"]},
  {"direction": "in", "protocol": "udp", "port": "443", "source_ips": ["0.0.0.0/0", "::/0"]},
  {"direction": "in", "protocol": "udp", "port": "41641", "source_ips": ["0.0.0.0/0", "::/0"]},
  {"direction": "in", "protocol": "icmp", "source_ips": ["0.0.0.0/0", "::/0"]}
]
EOF

hcloud firewall create --name web-only --rules-file fw-rules.json
hcloud firewall apply-to-resource web-only --type server --server runner-1
```

Remove the `80` and `443` rules if the server has no public web service.
Outbound traffic stays open. Tailscale and Cloudflare Tunnel use outbound
connections, so they continue to work. UDP `41641` lets peers make direct
Tailscale connections. Without it, Tailscale uses relays, which are slower.

From your client, test each public port that must be closed:

```bash
nc -z -w 3 203.0.113.10 5432 && echo OPEN || echo closed
```

Then make a new Tailscale SSH connection. Test each public site.

Traffic from a container to the server's own public address does not leave
the server. The provider firewall does not block it. Do not use this path as
an access method. See [Dokploy private services](dokploy-private-services.md).

### netcup firewall

netcup manages its firewall in the Server Control Panel (SCP), under
**Firewall Policies** and the server's **Firewall** tab. It differs from the
Hetzner firewall in two ways:

- It is stateful for TCP only. It does not track UDP. It drops the reply to an
  outbound UDP request unless an incoming rule accepts that reply.
- The implicit rules at the end are **Accept all**. When a policy with your own
  incoming rule is attached and saved, the implicit incoming rule changes to
  **DROP**. The default policies `netcup Mail block` and `netcup Ping allow` do
  not cause this change.

Create one policy for each rule, then attach the policies to the interface and
save. The policy form copies a new rule row from the previous row, so set one
rule for each policy. The server view shows ports as server side, then remote
side. Check each rule in the policy editor, which shows source and destination
ports in separate columns.

| Policy rule | Why |
| --- | --- |
| INGRESS, UDP, destination port `41641`, ACCEPT | Direct Tailscale connections |
| INGRESS, UDP, source port `123`, ACCEPT | NTP replies, so the clock stays correct |
| INGRESS, UDP, source port `53`, ACCEPT | DNS replies for resolvers other than netcup's |

DNS to the netcup resolvers is always allowed. Tailscale MagicDNS forwards to
them, so the host works without the DNS rule. Add the DNS rule before you
install Docker, `cloudflared`, or another tool with its own resolver. QUIC and
HTTP/3 also use UDP: see [Cloudflare Tunnel](cloudflare-tunnel.md).

Keep `netcup Ping allow`. IPv6 needs ICMPv6 for neighbor discovery and path
MTU. Keep `netcup Mail block` unless you need outbound mail.

UFW on the host is stateful. It still drops unsolicited packets that these
provider rules accept.

`tailscale netcheck` can show `UDP: false` after this change. The command tests
from random local ports, which the firewall drops. Check the real path with
`tailscale ping <server>` from a client: it must show a direct address, not
`DERP`.

See the [netcup firewall documentation](https://www.netcup.com/en/helpcenter/documentation/server/firewall).

## 9. Harden the base system

Do these steps after the private connection works.

Turn off SSH password login and X11 forwarding with a drop-in file:

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
X11Forwarding no
EOF
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin|x11forwarding) '
```

Turn on automatic security updates:

```bash
sudo apt install -y unattended-upgrades apt-listchanges
echo 'unattended-upgrades unattended-upgrades/enable_auto_updates boolean true' | sudo debconf-set-selections
sudo dpkg-reconfigure -f noninteractive unattended-upgrades
```

The default origins are the distribution only. Packages from other sources,
such as Tailscale, need their own update path. Decide if the server may reboot
by itself (`Unattended-Upgrade::Automatic-Reboot`).

Add a swap file if the image has none. Example for 4 GB:

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swap.conf
sudo sysctl -p /etc/sysctl.d/99-swap.conf
```

## Docker and application ports

Docker published ports can bypass UFW. A mapping such as `8080:80` can expose a
container on public host addresses despite UFW's incoming default. Do not rely
on UFW alone for container access. See [Docker firewall behavior](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw).

A `DROP` rule in the iptables `INPUT` chain does not block a published port.
Docker translates the destination address before filtering. The packet then
goes through the `FORWARD` chain, not `INPUT`. Use the provider firewall, or
add rules to the `DOCKER-USER` chain.

List the ports that listen on all addresses:

```bash
sudo ss -tlnpH | awk '{print $4}' | grep -E '^(0\.0\.0\.0|\[::\]|\*):' | sort -u
```

Every port in this list is public unless a provider firewall blocks it.
Docker Swarm also opens TCP `2377` and TCP and UDP `7946`. Do not allow these
ports in a provider firewall for a single-node Swarm.

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

Use the provider console if Tailscale access fails. On netcup, use the SCP
**Screen** tab. Sign in as `root` with the password from the provider's
welcome email; keep that password in your password manager. The provider
firewall can also be changed in SCP without access to the server.

To undo the UFW change:

```bash
sudo ufw disable
sudo systemctl status tailscaled --no-pager
```

This can restore public OpenSSH access if OpenSSH is still running and the
provider firewall permits it. Fix the cause, then repeat the private access
and firewall checks before enabling UFW again.

To remove the provider firewall from the server:

```bash
hcloud firewall remove-from-resource web-only --type server --server runner-1
```

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
