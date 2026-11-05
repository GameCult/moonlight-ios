# Remote access: iPad to a Sunshine host on restrictive wifi

**Status: planning document. Every step below is untested.** Nothing here has been run
against the real host, iPad, or network. Doc claims are cited; anything not found in the
cited docs is marked *unverified*. Nothing in this repo changes: this is a design note.

## Problem

Moonlight (this client) streams from a Sunshine host (called **Raven** here) over several
TCP and UDP ports. The target situation is an iPad on managed wifi that blocks those
ports, with no mobile data fallback. The usual escape is an overlay VPN whose traffic
looks like ordinary HTTPS. Only outbound TCP 443 is assumed to work.

Sunshine's base port is 47989 and other ports are offset from it
([Sunshine configuration](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2configuration.html)).
That page does not list the offsets; read the exact list from Sunshine's config UI. Pairing
uses the Sunshine web UI (default `https://<host>:47990`, PIN page)
([Sunshine getting started](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2getting__started.html)).
The web UI is LAN-only by default (`origin_web_ui_allowed: lan`); a tailnet address is not
the LAN, so pairing may need that setting changed or must be done on the LAN first.
*Unverified: whether tailnet source addresses count as `lan`.*

## What the overlay must carry

Moonlight video/audio/input are UDP. Tailscale and Headscale carry arbitrary IP inside
WireGuard, so UDP flows unchanged. When no direct path exists, the WireGuard packets
travel inside a TCP/HTTPS connection to a DERP relay.

## Option A: Tailscale (hosted control plane, Tailscale DERP)

### How it behaves on a 443-only network

- Outbound requirements: TCP 443 for coordination and DERP; UDP 41641 and UDP 3478 only
  improve direct connections. If only 443 is allowed, "connectivity still works" via
  relays, but not as fast as peer-to-peer
  ([Tailscale firewall ports](https://tailscale.com/kb/1082/firewall-ports)).
- Every connection starts relayed and then tries to upgrade to direct; DERP is the
  fallback when direct fails. Relayed traffic is end-to-end WireGuard encrypted
  ([connection types](https://tailscale.com/kb/1257/connection-types)).
- Tailscale warns DERP has limited quality of service and lower throughput. Its sample
  `tailscale ping` output shows DERP at roughly 40 to 280 ms. Those are examples, not
  predictions for this site.

### Steps (all untested)

1. Create or identify the tailnet and its owner (see Open Questions).
2. Install Tailscale on Raven; sign in to that tailnet.
3. Install the Tailscale iOS app on the iPad; sign in to the same tailnet. *Unverified:
   whether the managed iPad's profile permits installing a VPN app or creating a VPN
   configuration. This is the first likely blocker.*
4. In the Tailscale admin console or `tailscale status` on Raven, note Raven's tailnet IP
   (`100.x.y.z`) or MagicDNS name.
5. On the LAN beforehand if possible, pair Moonlight with Raven normally. If pairing over
   the tailnet: in Moonlight, add the host manually by tailnet IP; enter the PIN in
   Sunshine's web UI.
6. Start a stream over the tailnet; check with `tailscale ping raven` (on any tailnet
   device) whether the path is `direct` or `via DERP(...)`.
7. If relayed and unplayable, lower Moonlight resolution, FPS and bitrate first.

### Latency limits when relayed

Relayed means TCP-in-the-middle for UDP-like traffic: head-of-line blocking on loss, an
extra round trip through the nearest DERP region, and a throughput ceiling. Expect
playable desktop and slow-paced use, not guaranteed competitive-game latency. *No
measurement exists for this site; measure with `tailscale ping` and a trial stream.*

## Option B: self-hosted Headscale on Yggdrasil behind 443

Headscale is an open-source reimplementation of the Tailscale control server; the
official Tailscale iOS app can point at it
([Apple clients](https://headscale.net/stable/usage/connect/apple/): account menu,
"Use custom coordination server", enter the `https://` URL).

### Needs Eureka's approval first

Nothing below is done until Eureka approves each item:

- A new public hostname and DNS record for the control server.
- A public TLS certificate for it, and where TLS terminates.
- A new daemon (Headscale) on Yggdrasil and who deploys and survives it (Idunn
  authority applies; this doc does not choose).
- Public exposure of 443 for this hostname, and possibly UDP 3478 (see below).
- A relay choice: reuse Tailscale's public DERP map (the iPad still reaches Tailscale's
  relays on 443), or run Headscale's embedded DERP.
- Who may register nodes, and key expiry policy.

### Minimal configuration shape (untested; read the example config for your release tag
before use, per [config docs](https://headscale.net/stable/ref/configuration/))

- `server_url`: the public `https://` hostname on 443.
- `listen_addr`: loopback or internal only; reverse proxy in front.
- TLS cert/key paths left empty when a proxy terminates TLS (Headscale then logs
  `listening without TLS`, which is expected)
  ([reverse proxy](https://headscale.net/stable/ref/integration/reverse-proxy/)).
- `trusted_proxies` set to the proxy address.
- A sqlite database path, a noise private key path, IP prefixes, and a DNS block as in the
  shipped example config.
- `derp.urls`: keep Tailscale's default map for the first trial. Removing it makes the
  embedded server a single point of failure
  ([DERP](https://headscale.net/stable/ref/derp/)).
- Optional: `derp.server.enabled: true` plus `ipv4`/`ipv6`, `verify_clients: true`.

### Proxy constraints from the docs

- The proxy must support WebSockets: the control protocol upgrades via POST with
  `Upgrade: tailscale-control-protocol`. Cloudflare proxies and Tunnels are unsupported.
- Port summary: tcp/443 public; tcp/80 only for built-in Let's Encrypt HTTP-01; udp/3478
  public if embedded DERP is enabled; tcp/50443 only for gRPC remote control; tcp/9090
  not public ([requirements](https://headscale.net/stable/setup/requirements/)).
- STUN for embedded DERP needs udp/3478, which an HTTPS reverse proxy cannot carry. On a
  443-only client network STUN is blocked anyway, so DERP-over-443 is the path that
  matters. *Unverified: the docs do not say whether the embedded DERP's HTTPS endpoint
  works behind a reverse proxy.* Simplest trial: leave Tailscale's DERP map in place.
- The reverse-proxy page is community maintained and may be outdated.

### Exposure added

One new public HTTPS hostname fronting a control plane; optionally a public relay and UDP
3478. Raven and the iPad gain no inbound ports; Sunshine ports stay on the tailnet. The
tailnet's ACLs should restrict the iPad to Raven's Moonlight ports only
(*unverified in this environment; ACL syntax differs between Headscale versions*).

## Failure ladder

Run top to bottom, stopping at the first success. Everything is untested.

1. Home/LAN: confirm Moonlight streams to Raven with no overlay (baseline).
2. Tailscale on both devices at the LAN: confirm tailnet IP works, `tailscale ping` is
   direct.
3. iPad on the restrictive wifi, Tailscale connected: does the app reach "Connected"?
   If not: is VPN installation or profile blocked by management (stop; this is policy,
   not network), or is 443 to the control plane filtered or TLS-intercepted?
4. `tailscale ping raven` from the iPad's side (use another tailnet device, or app
   status): `direct` is best, `via DERP` means relay.
5. Relayed but stream stutters: lower bitrate, resolution, FPS; try H.264 over HEVC if the
   iPad decoder is struggling (decode, not network, may be the limit).
6. Control plane reachable but no DERP: network may TLS-intercept or block Tailscale
   domains. Option B with a hostname that is not a known Tailscale domain is the next step.
7. Wifi blocks everything but ordinary web browsing, and VPN apps are policy-blocked:
   neither option works. Fall back to a browser-based path or a different network.

## A or B

| Question | A: Tailscale hosted | B: Headscale on Yggdrasil |
|---|---|---|
| Time to first test | Hours | Days (approvals, DNS, TLS, deploy) |
| New public exposure | None | Hostname, cert, daemon, maybe UDP 3478 |
| Works if network blocks Tailscale domains | No | Possibly (own hostname) |
| Works if network only does 443 | Yes, relayed | Yes, relayed, via Tailscale DERP or own |
| Control-plane ownership | Tailscale account holder | Eureka / GameCult |
| Latency when relayed | Tailscale DERP regions | Same, or own DERP region if run |
| Ongoing work | None | Upkeep, updates, backups |
| Recommended | Try first | Only if A fails at ladder step 3 or 6 |

## Open Questions

- Where does Raven sit on the network (same LAN as the iPad's home, behind carrier NAT,
  behind a router that can forward ports, wired or wifi)? It decides whether direct
  connections are plausible at all.
- Who holds the tailnet account for Option A (individual, org, shared), and who can add
  devices and edit ACLs?
- Does the iPad's management policy allow a third-party VPN app? Nothing else matters if
  not.
- Is any measurement available of latency from the restrictive network to the nearest
  DERP region?
- If Option B is chosen: does Eureka want Headscale in the Yggdrasil deployment model, and
  who owns it?

## Sources

- <https://tailscale.com/kb/1257/connection-types>
- <https://tailscale.com/kb/1082/firewall-ports>
- <https://headscale.net/stable/ref/derp/>
- <https://headscale.net/stable/ref/configuration/>
- <https://headscale.net/stable/ref/integration/reverse-proxy/>
- <https://headscale.net/stable/setup/requirements/>
- <https://headscale.net/stable/usage/connect/apple/>
- <https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2getting__started.html>
- <https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2configuration.html>
