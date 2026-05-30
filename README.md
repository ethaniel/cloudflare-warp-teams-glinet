# Cloudflare WARP-for-Teams on OpenWRT via stock kernel WireGuard + nftables

A short, in-kernel recipe to make a stock OpenWRT WireGuard client talk to Cloudflare WARP for Teams. No userspace relay, no patched WireGuard, no AmneziaWG, no out-of-tree kernel module. Nine lines of nftables.

Tested on a GL.iNet GL-MT6000 (Flint 2) running GL.iNet's stock OpenWRT 23.05-based firmware (kernel 5.4, `nft` 0.9.6). The approach should work on any OpenWRT 21.02+ with kernel 5.x+, `nft` ≥ 0.9.x, and `kmod-wireguard`.

## The problem

WARP for Teams uses WireGuard, but Cloudflare's WireGuard variant differs from stock in one critical way: it uses bytes 1–3 of every packet header (WireGuard's "reserved" field) as **out-of-band routing metadata**. Cloudflare's load balancer reads them to dispatch each packet to the right backend. The backend then **zeros those bytes for `mac1` verification**.

Stock kernel WireGuard always sends bytes 1–3 as zero and computes `mac1` over the zero form. So plain WireGuard handshakes reach Cloudflare and are silently dropped — the load balancer can't dispatch them.

## The solution

Modify bytes 1–3 of every WireGuard packet on the wire — set them to your routing ID on egress, zero them on ingress — using nftables hooks. Because Cloudflare zeros those bytes for `mac1`, and stock WG signed `mac1` over the zero form, the modification is **wire-compatible without touching any cryptographic state**.

```
LAN/router → kernel WG signs packet with reserved bytes = 0
              ↓
           nft egress hook: set reserved bytes to routing ID + UDP checksum = 0
              ↓
           WAN → Cloudflare edge (load balancer reads bytes, dispatches)
              ↓
           backend zeros bytes for mac1 verification → matches → handshake completes

Cloudflare reply → router WAN
              ↓
           nft ingress hook: zero reserved bytes + UDP checksum = 0
              ↓
           kernel WG verifies mac1 over zero form → matches → packet accepted
```

## Step 1 — Register a device and get your parameters

Use [`izzoa/wgcf-teams`](https://github.com/izzoa/wgcf-teams), a Python port of the original (now-stale) Rust tool.

```sh
git clone https://github.com/izzoa/wgcf-teams
cd wgcf-teams
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
```

Get your CF\_Authorization JWT:

1. In a browser, open `https://<YOUR_ORG>.cloudflareaccess.com/warp` and log in.
2. DevTools → Application → Cookies → copy the value of the `CF_Authorization` cookie.
3. Save it to a file:
   ```sh
   echo 'eyJ...your-jwt-here...' > token.txt
   chmod 600 token.txt
   ```

Now register. The default script reads `endpoint.host` from the response, which is a **generic anycast pool** that won't serve your device's session — patch it to also surface `endpoint.v4` (the org-specific endpoint) before running, or just dump the raw JSON response and pick the right field manually:

```sh
# In wgcf_teams.py, before `return data["result"]`, add:
import json
print(json.dumps(data, indent=2), file=sys.stderr)
```

Then run:

```sh
.venv/bin/python wgcf_teams.py -t token.txt 2> raw-response.json
```

From the raw response, you need three things:

| Field | Where in response | Example |
|---|---|---|
| Private key | generated locally, printed in output config as `PrivateKey =` | (yours) |
| Peer public key | `result.config.peers[0].public_key` | `bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=` |
| Endpoint v4 | `result.config.peers[0].endpoint.v4` | `162.159.193.2:0` — use port 2408 |
| Client ID (3 bytes) | `result.config.client_id` (base64, 3 bytes after decode) | `T/zz` → `4F FC F3` |

The peer public key has historically been stable across orgs (it's Cloudflare's shared WARP gateway), but verify in your response.

The 3-byte routing ID is the value you'll inject into the reserved bytes. Form a 24-bit hex literal from the bytes (`4F FC F3` → `0x4ffcf3`).

## Step 2 — Configure stock kernel WireGuard on OpenWRT

Either via your distribution's GUI, LuCI, or directly in `/etc/config/network`:

```
config interface 'wgwarp'
    option proto 'wireguard'
    option private_key '<YOUR_PRIVATE_KEY>'
    list addresses '172.16.0.2/32'
    list addresses '2606:4700:cf1:1000::4/128'  # or whatever your registration assigned
    option mtu '1420'

config wireguard_wgwarp
    option public_key 'bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo='
    option endpoint_host '<YOUR_ENDPOINT_V4>'   # e.g. 162.159.193.2
    option endpoint_port '2408'
    list allowed_ips '0.0.0.0/0'
    list allowed_ips '::/0'
    option persistent_keepalive '25'
```

If you use GL.iNet's GUI, add the tunnel as a WireGuard Client with the same fields. The interface will appear in their VPN Policy UI for per-client/per-domain routing.

## Step 3 — Install the nftables ruleset

`/etc/nftables.warp.conf`:

```nft
table inet warp {
    chain egress {
        type filter hook output priority 0;
        ip daddr <YOUR_ENDPOINT_V4> udp dport 2408 @th,72,24 set 0x<YOUR_ROUTING_ID> udp checksum set 0 counter
    }
    chain ingress {
        type filter hook input priority -100;
        ip saddr <YOUR_ENDPOINT_V4> udp sport 2408 @th,72,24 set 0 udp checksum set 0 counter
    }
}
```

- `@th,72,24` = 24 bits at offset 72 from the transport header = bytes 1–3 of the UDP payload = WireGuard's three reserved bytes.
- `udp checksum set 0` because raw payload SET on older `nft` does not recompute L4 checksums. IPv4 RFC 768 explicitly permits checksum=0 ("no checksum present") and both Cloudflare's edge and the local kernel accept it.
- `priority -100` on the ingress chain ensures we run before any default firewall rules; egress at `priority 0` is fine because we're not dropping anything.

## Step 4 — Persist across reboots

`/etc/init.d/warp-nft`:

```sh
#!/bin/sh /etc/rc.common
START=60
STOP=10

start() {
    nft -f /etc/nftables.warp.conf 2>&1 | logger -t warp-nft
}

stop() {
    nft delete table inet warp 2>/dev/null
}

reload() {
    stop
    start
}
```

Then:

```sh
chmod +x /etc/init.d/warp-nft
/etc/init.d/warp-nft enable
/etc/init.d/warp-nft start
```

## Step 5 — Bring up and verify

```sh
ifup wgwarp                                     # or click "Connect" in GUI
wg show wgwarp                                  # handshake should complete in seconds
nft list table inet warp                        # counters should grow with traffic
ping -I wgwarp -c 3 8.8.8.8                     # should respond
curl --interface wgwarp https://www.cloudflare.com/cdn-cgi/trace
# look for: warp=plus
```

## Why simpler approaches don't work

Recorded so others don't re-tread these dead ends.

- **Plain stock WireGuard alone** (no byte rewrite). Handshake init reaches Cloudflare, Cloudflare's load balancer can't dispatch it (no routing ID), silently dropped. Confirmed via tcpdump: packets leave WAN, never a reply.
- **AmneziaWG H1–H4** (the obvious-looking match if your firmware ships AmneziaWG). Produces correct wire bytes (verified with tcpdump), but Cloudflare still drops. Cause: AmneziaWG modifies the first 4 bytes *and* includes them in `mac1`; WARP's verifier expects `mac1` computed with bytes 1–3 zeroed. Mismatch → drop. No AmneziaWG config flag changes this. AmneziaWG 2.0 adds I1–I5 junk-packet parameters but not a reserved-bytes mode.
- **wireproxy or similar userspace WG client.** Works, but each packet round-trips through userspace, costing ~5–10% of one CPU core per 100 Mbps. Also can't expose a real kernel TUN, so distribution GUIs can't manage it like a normal tunnel.
- **A userspace UDP sidecar relay** (Go/C/Python) that injects/zeros reserved bytes between stock WG and Cloudflare. Works fine; we built one. Same userspace cost as wireproxy. Made obsolete by the nftables approach below.
- **nftables in-kernel byte rewrite** (this guide). Zero userspace overhead, nine lines of config. Works because WARP's protocol contract — set bytes on wire, zero for mac1 — maps cleanly to "modify after WG signs the packet, modify back before WG verifies."

## Common pitfalls

- **Wrong endpoint.** `wgcf-teams` defaults to `endpoint.host = engage.cloudflareclient.com:2408`. This is a generic anycast pool, **not** where your org's device is registered. Use `endpoint.v4` from the response instead.
- **`engage.cloudflareclient.com` looks tempting because it resolves.** It resolves and accepts UDP, but silently drops your handshake because your session is registered on a different backend.
- **Forgetting `udp checksum set 0`.** Without it, kernel WG on the receive side will discard packets with stale UDP checksums, even though Cloudflare may have replied correctly. You'll see ingress nft counter increment but `wg show` will show 0 bytes received.
- **`mac1` confusion.** WARP zeros reserved bytes *before* verifying `mac1`; AmneziaWG doesn't. This is why the seemingly equivalent AmneziaWG approach fails. Don't waste a day on it like we did.
- **Single-tunnel assumption.** The rules above match all traffic to `<YOUR_ENDPOINT_V4>:2408`. If you run multiple WARP tunnels (consumer + Teams, multiple orgs), each on the same Cloudflare IP, you'd inject the wrong routing ID into the wrong one. Add a `meta mark` or `meta nfproto`-style discriminator (out of scope here).

## Caveats

- The routing ID is per-device. If you re-register, you get a new one. Update the egress rule's hex literal.
- The endpoint is per-org. Likely stable, but if it ever changes in a re-registration response, update both the rule matches and the WG peer endpoint.
- `nft 0.9.6` was used for testing. Newer versions are also fine and may handle L4 checksum recompute natively, in which case the `udp checksum set 0` lines become optional.
- This is single-stack IPv4 in the rules above. For an IPv6 WARP endpoint, add parallel chains matching `ip6 daddr` / `ip6 saddr`.

## License

MIT. Use freely, no warranty.
