# Cloudflare WARP for Teams on GL.iNet routers

A practical guide to routing your network through your organization's Cloudflare WARP-for-Teams tunnel, using your GL.iNet router's GUI for almost everything.

Tested on the **Flint 2 (GL-MT6000)** running firmware 4.8.x. Should work on any GL.iNet router with firmware 4.x.

---

## Before you start

You'll need:

- A **GL.iNet router** with admin access.
- A **Cloudflare WARP for Teams** organization (you can log in at `https://<your-org>.cloudflareaccess.com/warp`).
- A **computer** (Mac, Linux, or Windows) with a terminal — you'll open it briefly twice.
- About **15 minutes**.

You may notice this isn't 100% GUI. There are two unavoidable terminal steps because:

1. WARP only registers devices over its API (no GUI). You'll run a small Python tool on your computer once to get your WireGuard credentials.
2. The router needs one in-kernel firewall rule that the GL.iNet GUI doesn't expose. You'll SSH in once to install it.

After that, **everything is GUI-managed forever**: connect, disconnect, change which clients/domains use WARP, kill switch, etc.

---

## Why your router can't just speak WireGuard to WARP

Cloudflare's WARP uses a custom WireGuard variant. Every packet carries a 3-byte "routing ID" in WireGuard's reserved header bytes that the WARP load balancer reads to dispatch packets to the right backend. Stock WireGuard (what every router speaks) leaves those bytes as zero — so WARP silently drops the connection.

The fix is small: tell the router's firewall to **fill in those 3 bytes on outbound, zero them on inbound**. Nine lines of configuration, fully in-kernel, no extra software running, no performance hit.

---

## Step 1 — Get your WARP credentials (5 min on your computer)

Open a terminal on your computer.

### 1a. Get your authentication cookie

1. Open a browser and go to `https://<YOUR_ORG>.cloudflareaccess.com/warp`. Replace `<YOUR_ORG>` with your organization's name.
2. Log in normally.
3. Open the browser's developer tools:
   - **Chrome / Edge / Brave**: `F12` (Windows/Linux) or `Cmd+Option+I` (Mac).
   - **Firefox**: same shortcuts.
   - **Safari**: enable "Show Develop menu" in Preferences first, then `Cmd+Option+I`.
4. Go to **Application** (Chrome) or **Storage** (Firefox/Safari) → **Cookies** → click your org's domain.
5. Find the cookie named **`CF_Authorization`** and copy its value. It's a very long string starting with `eyJ...`.

### 1b. Run wgcf-teams

In your computer's terminal:

```sh
# Get the tool
git clone https://github.com/izzoa/wgcf-teams
cd wgcf-teams

# Install dependencies (needs Python 3.10+)
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# Patch the script to print the full registration response
# (the default output hides the field we actually need)
python3 -c "
src = open('wgcf_teams.py').read()
src = src.replace(
  '    return data[\"result\"]',
  '    import json, sys; print(json.dumps(data, indent=2), file=sys.stderr)\n    return data[\"result\"]'
)
open('wgcf_teams.py','w').write(src)
"

# Save your cookie to a file (replace eyJ...your-cookie... with what you copied in 1a)
echo 'eyJ...your-cookie...' > token.txt
chmod 600 token.txt

# Register a device
.venv/bin/python wgcf_teams.py -t token.txt 2>&1 | tee register.log
```

The output is a long JSON blob followed by a generated WireGuard config. **Save these three values somewhere** (you'll use them in steps 2 and 3):

1. **Private key** — find the line `PrivateKey = ...` near the bottom of the output.
2. **Endpoint v4** — search the JSON for `"v4":` inside a `"peers"` block, e.g. `"v4": "162.159.193.2:0"`. The IP part (`162.159.193.2`) is what you want.
3. **Routing ID** — search the JSON for `"client_id":`, e.g. `"client_id": "T/zz"`. Get the 6-character hex routing ID with:
   ```sh
   echo -n 'T/zz' | base64 -d | xxd -p
   # output: 4ffcf3
   ```

> WARP will create a new device entry in your Zero Trust admin dashboard, named something like `iPad13,8` (the script registers as an iPad to look unobtrusive). You can rename it in the dashboard.

---

## Step 2 — Add the WireGuard tunnel in GL.iNet GUI (3 min)

1. Open your router admin (typically `http://192.168.8.1`).
2. In the sidebar: **VPN → WireGuard Client**.
3. Click **+ Add Profile** (or **Add a New WireGuard Profile**).
4. Choose **Manual Input** (not "Import from Server" or any GL.iNet portal option).
5. Give it a name like `WARP` and paste this configuration, replacing the two placeholders with **your** values from Step 1:

   ```ini
   [Interface]
   PrivateKey = YOUR_PRIVATE_KEY_FROM_STEP_1
   Address = 172.16.0.2/32
   Address = 2606:4700:cf1:1000::4/128
   DNS = 1.1.1.1
   MTU = 1420

   [Peer]
   PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
   AllowedIPs = 0.0.0.0/0
   AllowedIPs = ::/0
   Endpoint = YOUR_ENDPOINT_V4_FROM_STEP_1:2408
   PersistentKeepalive = 25
   ```

6. **Save** the profile. **Do not click Connect yet** — it will fail until Step 3 is done.

> The `PublicKey` above is Cloudflare's shared WARP gateway public key — it's the same for everyone. If the JSON output from Step 1 happens to show a different `public_key`, use that one instead.

---

## Step 3 — Install the firewall rule via SSH (5 min, one-time)

This is the only step that requires SSH. After this, the rule lives on the router permanently (survives reboots, firmware doesn't touch it), and you'll never need SSH again for WARP.

### 3a. Find your router's admin password

It's whatever you set when you first configured the router. If you forgot, you can reset it in the GUI under **Admin Panel** or **System → Admin Password**.

### 3b. Connect via SSH from your computer

In your computer's terminal:

```sh
ssh root@192.168.8.1
```

When prompted for the password, type your admin password. (If your router uses a different IP, replace it.)

### 3c. Paste this whole block into the SSH session

**Before pasting**, replace `YOUR_ENDPOINT_V4` and `YOUR_ROUTING_ID_HEX` with your values:

- `YOUR_ENDPOINT_V4` = the IP from Step 1 (e.g. `162.159.193.2`).
- `YOUR_ROUTING_ID_HEX` = the 6 hex characters from Step 1 (e.g. `4ffcf3`).

```sh
# Write the firewall rule
cat > /etc/nftables.warp.conf <<'EOF'
table inet warp {
    chain egress {
        type filter hook output priority 0;
        ip daddr YOUR_ENDPOINT_V4 udp dport 2408 @th,72,24 set 0xYOUR_ROUTING_ID_HEX udp checksum set 0 counter
    }
    chain ingress {
        type filter hook input priority -100;
        ip saddr YOUR_ENDPOINT_V4 udp sport 2408 @th,72,24 set 0 udp checksum set 0 counter
    }
}
EOF

# Write the boot init that loads the rule
cat > /etc/init.d/warp-nft <<'EOF'
#!/bin/sh /etc/rc.common
START=60
STOP=10
start()  { nft -f /etc/nftables.warp.conf 2>&1 | logger -t warp-nft; }
stop()   { nft delete table inet warp 2>/dev/null; }
reload() { stop; start; }
EOF
chmod +x /etc/init.d/warp-nft

# Enable for boot and start now
/etc/init.d/warp-nft enable
/etc/init.d/warp-nft start

# Verify it loaded
nft list table inet warp
```

The `nft list` at the end should print your table with both chains. If you see an error or nothing, the placeholders didn't get replaced — re-paste with your values.

Type `exit` to leave SSH.

---

## Step 4 — Connect WARP and verify

1. Back in the GL.iNet GUI: **VPN → WireGuard Client**, find your WARP profile, and toggle it **on / Connect**.
2. Within a few seconds, the GUI should show **Connected** with a recent handshake timestamp.
3. From any device on your network (or right in the router admin's terminal page), visit:
   ```
   https://www.cloudflare.com/cdn-cgi/trace
   ```
   Look for `warp=plus` in the response — that confirms your traffic is going through WARP for Teams.

If it doesn't connect, see **Troubleshooting** below.

---

## Step 5 — Route traffic through WARP (GUI only)

Now you're back to pure GUI-land. Use GL.iNet's normal VPN tools:

- **VPN Dashboard** or **VPN Policy / Domain Policy** — pick which clients or domains use WARP and leave the rest direct. This is the most useful feature if you only want certain sites or certain devices on WARP.
- **Global Mode** — send all traffic through WARP.
- **Kill Switch** — drop traffic if WARP goes down instead of falling back to direct. Recommended if you depend on WARP for privacy.

Add, remove, or change these freely. They all just work with the WARP profile you set up.

---

## Troubleshooting

**GUI shows "Connected" but `warp=plus` doesn't appear / sites don't load.**
The firewall rule probably isn't applying. SSH back in and run:
```sh
nft list table inet warp
```
You should see both chains with non-zero counters when traffic is flowing. If counters stay at 0, the rule isn't matching — most likely the `endpoint v4` in your nft rule doesn't match what's actually in the GUI's WireGuard config. They must be byte-for-byte the same IP.

**GUI shows "Disconnected" / handshake never completes.**
Check the rule loaded:
```sh
nft list table inet warp
```
If empty or missing: `/etc/init.d/warp-nft start` and check `logread | grep warp-nft` for errors.

If the rule is loaded but handshake still fails: `tcpdump -i any -n 'host YOUR_ENDPOINT_V4'` to confirm packets are leaving. If they're leaving and you're getting no reply, the most likely cause is a wrong routing ID — double-check the base64 decode of `client_id`.

**It worked, then stopped working after a few hours/days.**
WARP tokens for Teams can require periodic re-authentication (some orgs enforce session timeouts). Check your Zero Trust admin dashboard — your device should still be listed. The WireGuard tunnel itself doesn't time out, so if it's failing repeatedly, your org may have policies that need the official WARP client's posture checks. This setup doesn't perform those checks.

**I re-registered the device and now nothing works.**
A new registration gives a new private key and routing ID. You need to update:
- The `PrivateKey` line in the GL.iNet GUI's WireGuard profile.
- The `0xYOUR_ROUTING_ID_HEX` value in `/etc/nftables.warp.conf` on the router, then `/etc/init.d/warp-nft reload`.

The endpoint usually stays the same per organization.

**My internet speed is capped through WARP.**
Normal. Cloudflare typically rate-limits individual WARP-for-Teams sessions to a few hundred Mbps. The same cap shows up with the official WARP client — not the router or this setup.

---

## How this works (skip if you don't care)

Cloudflare's WARP backend differs from stock WireGuard in two specific ways:

1. The 3 bytes immediately after the message-type byte in every WireGuard packet (officially "reserved" and zero in stock WG) are repurposed as a per-device routing identifier. Cloudflare's load balancer reads them to dispatch each packet to the backend that holds your session.
2. When the backend verifies a packet's `mac1` cryptographic authentication code, it **zeros those 3 bytes first** — so the MAC is computed over the same form stock WireGuard would have signed.

This means stock kernel WireGuard, which always sends bytes 1–3 as zero and signs `mac1` over the zero form, is *almost* compatible. Just flip the bytes on the wire — after the kernel signs the outbound packet, and before the kernel verifies an inbound one. That's exactly what our nftables rules do, using two netfilter hooks (`output` and `input`). No userspace daemon, no patched kernel module, no decryption involved.

The `udp checksum set 0` line is necessary because modifying payload bytes invalidates the UDP checksum. IPv4 RFC 768 explicitly permits checksum=0 ("no checksum present") which both Cloudflare's edge and the local kernel accept.

---

## What doesn't work (and why)

Recorded so others don't waste time:

- **Plain stock WireGuard with no byte rewrite.** Handshake reaches Cloudflare but gets silently dropped — load balancer can't dispatch.
- **AmneziaWG H1–H4 parameters** (the obvious-looking match if your firmware ships AmneziaWG). Produces correct wire bytes, but AmneziaWG modifies the first 4 bytes *and* includes them in `mac1`; WARP's verifier expects `mac1` over the zero form. Mismatch → drop. No AmneziaWG knob fixes this. AmneziaWG 2.0's new I1–I5 parameters don't help either.
- **wireproxy / boringtun / other userspace WireGuard clients with `Reserved =` support.** They work, but cost CPU (every packet round-trips through userspace) and don't expose a real kernel interface, so GUI policy routing can't see them.
- **udpfwd-style sidecar relays.** Work, but are an earlier version of the same idea; the in-kernel nftables approach is strictly simpler.

The nftables approach is the only one we found that's both functional and zero-overhead.

---

## For non-GL.iNet OpenWRT users (advanced)

If you're on plain OpenWRT or a non-GL.iNet device, you can do the same thing without any GUI.

### 1. WireGuard interface via UCI

Add to `/etc/config/network`:

```
config interface 'wgwarp'
    option proto 'wireguard'
    option private_key 'YOUR_PRIVATE_KEY'
    list addresses '172.16.0.2/32'
    list addresses '2606:4700:cf1:1000::4/128'
    option mtu '1420'

config wireguard_wgwarp
    option public_key 'bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo='
    option endpoint_host 'YOUR_ENDPOINT_V4'
    option endpoint_port '2408'
    list allowed_ips '0.0.0.0/0'
    list allowed_ips '::/0'
    option persistent_keepalive '25'
    option route_allowed_ips '1'
```

Then:
```sh
uci commit network
ifup wgwarp
```

Add the interface to a firewall zone (typically wan-like) in `/etc/config/firewall` with masquerading enabled.

### 2. nftables rule (same as Step 3 above)

```sh
cat > /etc/nftables.warp.conf <<'EOF'
table inet warp {
    chain egress {
        type filter hook output priority 0;
        ip daddr YOUR_ENDPOINT_V4 udp dport 2408 @th,72,24 set 0xYOUR_ROUTING_ID_HEX udp checksum set 0 counter
    }
    chain ingress {
        type filter hook input priority -100;
        ip saddr YOUR_ENDPOINT_V4 udp sport 2408 @th,72,24 set 0 udp checksum set 0 counter
    }
}
EOF

cat > /etc/init.d/warp-nft <<'EOF'
#!/bin/sh /etc/rc.common
START=60
STOP=10
start()  { nft -f /etc/nftables.warp.conf 2>&1 | logger -t warp-nft; }
stop()   { nft delete table inet warp 2>/dev/null; }
reload() { stop; start; }
EOF
chmod +x /etc/init.d/warp-nft
/etc/init.d/warp-nft enable
/etc/init.d/warp-nft start
```

Verified on kernel 5.4 with `nft` 0.9.6. Should work on any newer combination.

### Notes for advanced users

- The rules above match all traffic to `YOUR_ENDPOINT_V4:2408`. If you run multiple WARP tunnels on the same Cloudflare IP (consumer + Teams, multiple orgs), they'd collide. Add a per-tunnel `meta mark` discriminator.
- This is IPv4 in the rules. For an IPv6 WARP endpoint, mirror the chains with `ip6 daddr` / `ip6 saddr`.
- Newer `nft` versions may handle L4 checksum recomputation natively, making `udp checksum set 0` optional. Test before relying on it.

---

## License

MIT. Use freely, no warranty.
