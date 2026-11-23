# Cloudflare Tunnel for public web traffic

Status: plan. This page is not yet tested on a server.

Written on 2026-11-23. All names and addresses below are examples.

## Why

With a tunnel, the server accepts no inbound connections. `cloudflared` opens
outbound connections to Cloudflare, and Cloudflare sends visitors through them.
The provider firewall and UFW can block all inbound traffic, and the server's
IP address does not appear in DNS. Use Tailscale for SSH and a tunnel for web
traffic.

## Plan

1. Run the application on `127.0.0.1` only. For Docker, publish the port on
   loopback (`"127.0.0.1:8080:80"`). Docker published ports can bypass UFW.
2. Install `cloudflared` from the Cloudflare package source.
3. Create the tunnel and its DNS record with `cloudflared` or the Cloudflare
   API (`cf`, `wrangler`).
4. Point the tunnel ingress at `http://127.0.0.1:8080`.
5. Run `cloudflared` as a systemd service.
6. Test the public hostname. Check from outside that ports 80 and 443 on the
   server are closed.

## netcup and other stateless UDP firewalls

`cloudflared` uses QUIC on UDP port 7844 by default. The netcup firewall drops
the UDP replies. Choose one:

- Set `protocol: http2` in the `cloudflared` configuration. It then uses TCP.
- Add an SCP policy: INGRESS, UDP, source port `7844`, ACCEPT.

See [VPS setup with Tailscale SSH](vps-tailscale-setup.md#netcup-firewall).
