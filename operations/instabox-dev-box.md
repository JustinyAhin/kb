# instabox development box

Notes for an instabox.sh machine used as a remote development box through
Tailscale.

Written on 2026-11-23 against the instabox documentation
(`https://instabox.sh/llms.txt`). All names below are examples.

## What the machine is

- A Firecracker microVM with Debian 13, its own kernel, systemd, and Docker.
- It logs in as a non-root user with passwordless `sudo`. You do not need to
  create another user.
- Home and `/data` persist. `/tmp` is RAM.
- SSH goes through the instabox gateway with short-lived certificates:
  `instabox ssh <machine>`.

## Free plan limits

- The machine pauses after about 30 minutes of idle time.
- A run starts at 120 minutes. While the machine is in use, the run grows up
  to 480 minutes. Then it pauses, with a cooldown of 5 to 20 minutes.
- A pause keeps memory and processes. A stop or restart ends them.
- An idle SSH or Rex connection does not keep the machine awake.

A paused machine leaves the tailnet. Tailscale and Rex cannot wake it. Wake it
from the instabox dashboard (open the machine's Terminal, Computer, or Files
tab) or with:

```bash
instabox wake <machine>
```

## Tailscale

The kernel has the TUN device, so Tailscale runs in normal mode:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --hostname=<machine> --advertise-tags=tag:<name> --ssh
```

Give the box its own tag. Limit its grant to the client devices that need it,
with device addresses in `hosts`:

```json
{
  "hosts": { "my-laptop": "<laptop-tailscale-ip>", "my-phone": "<phone-tailscale-ip>" },
  "grants": [{ "src": ["my-laptop", "my-phone"], "dst": ["tag:<name>"], "ip": ["*"] }],
  "ssh": [{ "action": "accept", "src": ["admin@example.com"], "dst": ["tag:<name>"], "users": ["user"] }]
}
```

If a client device is added to the tailnet again, its address changes. Update
`hosts` then.

Do not make the box an exit node. The instabox abuse check (`abuse_proxy`) can
pause and lock the machine for 24 hours.

## Firewall

UFW is not necessary: SSH from the internet goes only through the gateway.
`sshd` still listens on port 22 inside the machine, and the machine can have a
public IPv6 address. Keep `PasswordAuthentication no` as a second barrier.

## Latency

The free plan does not let you choose the region. If the machine is far from
you, each key press takes one round trip. On a client that uses OpenSSH, SSH
connection reuse makes new connections faster:

```text
Host <machine>
  ControlMaster auto
  ControlPath ~/.ssh/cm/%C
  ControlPersist 10m
  ServerAliveInterval 30
```

Create `~/.ssh/cm` with mode 700.
