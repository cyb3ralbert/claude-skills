---
name: xray-reality-setup
description: Set up a self-hosted Xray VLESS Reality proxy server on a Debian/Ubuntu VPS. Generates the Reality keypair, writes the config, starts the service, and outputs a ready-to-use sing-box client snippet. Use when asked to deploy VLESS Reality, set up a new Xray server, configure Reality on a fresh droplet, поднять self-hosted proxy сервер, or настроить Reality на новом дроплете.
---

# Setup Xray VLESS Reality Server

Self-hosted encrypted proxy (Xray VLESS Reality) on a Debian/Ubuntu VPS. Traffic
encryption with TLS 1.3 fingerprint mimicry.

## Workflow

### 1. Get VPS access

Need: root SSH access to a fresh Debian/Ubuntu VPS.

If no VPS exists, create a DigitalOcean droplet. Pick the SSH key explicitly —
`head -1` grabs whichever key happens to sort first, which is not reproducible:

```bash
doctl compute ssh-key list          # note the ID of the key you want
doctl compute droplet create vpn-server \
  --region ams3 \
  --image debian-12-x64 \
  --size s-1vcpu-512mb-10gb \
  --ssh-keys <SSH_KEY_ID> \
  --wait
```

Get IP: `doctl compute droplet get vpn-server --format PublicIPv4`

### 2. Review the setup script, then run it

The script runs as root and configures the box end to end. Pin it to a release
tag rather than `master` — `master` is a moving target, and this skill pins the
Xray version precisely to avoid the compat breakage described in Notes. Pinning
the installer but not its entry point defeats that.

Fetch it, check it against the checksum pinned here, and review it before
running. The hash lives in this skill, not in the repo — so a moved tag or a
tampered file fails the check (a bare `sha256sum` with nothing to compare
against does not):

```bash
SETUP_REF=v1.0.1   # pinned release tag
EXPECTED_SHA256=921244ea7bc6ff0b4172c42ed9bbd150d02046ff683558c6c01431fc27d5e19f
SETUP_URL="https://raw.githubusercontent.com/cyb3ralbert/sing-box-public/${SETUP_REF}/setup-server.sh"

curl -fsSL "$SETUP_URL" -o /tmp/setup-server.sh
echo "${EXPECTED_SHA256}  /tmp/setup-server.sh" | sha256sum -c - || {
  echo "Checksum mismatch — refusing to run." >&2; exit 1; }
less /tmp/setup-server.sh
```

Once the user has confirmed, run it:

```bash
# Default donor: habr.com
scp /tmp/setup-server.sh root@<SERVER_IP>:/tmp/
ssh root@<SERVER_IP> 'bash /tmp/setup-server.sh'

# Custom donor domain:
ssh root@<SERVER_IP> 'SERVER_DOMAIN=yahoo.com bash /tmp/setup-server.sh'
```

Script does automatically:
- Installs Xray (pinned to 25.3.6)
- Generates Reality keypair (x25519) + short_id + client UUID
- Writes `/etc/xray/config.json`
- Starts `xray.service`
- Prints sing-box client snippet with all values

Gate on it before moving on — don't discover a dead service at step 6 (adjust
`:443` if you set a custom `SERVER_PORT`):

```bash
ssh root@<SERVER_IP> 'systemctl is-active xray && ss -tlnp | grep -q ":443 " && echo OK'
```

### 3. Confirm IPv4 egress

`setup-server.sh` (≥ v1.0.1) already writes the `freedom` outbound with
`domainStrategy: UseIPv4`. Without it, an IPv6-capable server exits over IPv6,
clients see an IPv6 address, and some apps break. This step just confirms it:

```bash
ssh root@<SERVER_IP> 'jq -c ".outbounds" /etc/xray/config.json'
```

Expected:

```json
[{"protocol":"freedom","settings":{"domainStrategy":"UseIPv4"}}]
```

Only if it is missing — a server provisioned before v1.0.1, or a hand-edited
config — set it with the same backup → `xray -test` → `mv` pattern as step 4:

```bash
ssh root@<SERVER_IP> bash -s <<'EOF'
set -euo pipefail
CONFIG=/etc/xray/config.json
cp "$CONFIG" "$CONFIG.bak"
jq '(.outbounds[] | select(.protocol == "freedom") | .settings.domainStrategy) = "UseIPv4"' \
  "$CONFIG" > "$CONFIG.new"
if xray -test -config "$CONFIG.new"; then
  mv "$CONFIG.new" "$CONFIG"
  systemctl restart xray
else
  echo "Config test failed — keeping the current config" >&2
  rm -f "$CONFIG.new"
fi
EOF
```

### 4. Add more clients (optional)

Each device gets its own UUID.

`jq` is not on a stock Debian image:

```bash
apt-get update && apt-get install -y jq
```

Edit with a backup and a config test. A bare `jq | sponge` will silently write
an empty file if `jq` fails, and the following restart then takes the service
down — including the tunnel you are connected through:

```bash
CONFIG=/etc/xray/config.json
NEW_UUID=$(cat /proc/sys/kernel/random/uuid)

cp "$CONFIG" "$CONFIG.bak"
jq --arg id "$NEW_UUID" --arg email "phone" \
  '.inbounds[0].settings.clients += [{"id": $id, "flow": "xtls-rprx-vision", "email": $email}]' \
  "$CONFIG" > "$CONFIG.new"

if xray -test -config "$CONFIG.new"; then
  mv "$CONFIG.new" "$CONFIG"
  systemctl restart xray
  echo "New UUID: $NEW_UUID"
else
  echo "Config test failed — keeping the current config" >&2
  rm -f "$CONFIG.new"
fi
```

Remove a client by email:

```bash
CONFIG=/etc/xray/config.json
cp "$CONFIG" "$CONFIG.bak"
jq --arg email "phone" \
  '.inbounds[0].settings.clients |= map(select(.email != $email))' \
  "$CONFIG" > "$CONFIG.new"
if xray -test -config "$CONFIG.new"; then
  mv "$CONFIG.new" "$CONFIG"
  systemctl restart xray
else
  echo "Config test failed — keeping the current config" >&2
  rm -f "$CONFIG.new"
fi
```

### 5. Configure client (sing-box)

Use the snippet from step 2. Set the uTLS fingerprint by sing-box version —
`chrome` is broken with Xray 25.3.6+ (connects but passes no traffic):
- 1.12.8 (Android/Termux) → `firefox`
- 1.12.12+ (desktop/VM) → `randomized`

### 6. Verify

```bash
# On client machine:
no_proxy="" curl -x socks5h://127.0.0.1:<LOCAL_SOCKS_PORT> --max-time 8 ifconfig.me
# Should return server IP, and it should be IPv4 (see step 3)
```

`<LOCAL_SOCKS_PORT>` is the local inbound port from the client snippet, not the
server port.

If it hangs or returns nothing, triage in order:
- Server side: `ssh root@<SERVER_IP> 'journalctl -u xray -n 50 --no-pager'`.
- Client side: a wrong uTLS fingerprint is the usual cause — recheck step 5
  (`chrome` passes no traffic).
- Network: the exit IP came back IPv6 (step 3), or a firewall blocks the port —
  `nc -zv <SERVER_IP> 443`.

## Teardown

```bash
ssh root@<SERVER_IP> 'systemctl disable --now xray; rm -rf /etc/xray'
doctl compute droplet delete vpn-server
```

Rotating credentials means re-running step 2: the Reality keypair and all client
UUIDs are regenerated, so every client snippet must be reissued.

## Notes

- Donor domain must support TLS 1.3. **habr.com** is the recommended default (stable, tested).
- Recommended xray version: **25.3.6** (pinned in setup-server.sh). Newer versions (25.8.x+) may break Reality client compat.
- Port 443 mimics HTTPS — works through most firewalls.
- Config path here is `/etc/xray/config.json`. Upstream Xray-install defaults to `/usr/local/etc/xray/config.json` — check which one `xray.service` actually loads before editing (`systemctl cat xray`).
- The Reality private key is generated on the VPS, so the hosting provider is inside the trust boundary. Fine when the goal is privacy from third parties on the network; not adequate if the threat model includes the provider itself.
- If using 3x-ui panel instead: config is managed via web UI at port 2053, not config.json directly.
- Repo with scripts: https://github.com/cyb3ralbert/sing-box-public