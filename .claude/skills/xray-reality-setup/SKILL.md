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

Show the script to the user before executing it:

```bash
SETUP_REF=v1.0.0   # pinned release tag (or a commit SHA)
SETUP_URL="https://raw.githubusercontent.com/cyb3ralbert/sing-box-public/${SETUP_REF}/setup-server.sh"

curl -fsSL "$SETUP_URL" -o /tmp/setup-server.sh
sha256sum /tmp/setup-server.sh
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

### 3. Force IPv4 egress

Do this on every install. If the server has IPv6, Xray exits via IPv6 by
default, clients see an IPv6 address, and some apps break. Verify the outbound
is present and fix it if not:

```bash
ssh root@<SERVER_IP> 'jq ".outbounds" /etc/xray/config.json'
```

Expected:

```json
[{"protocol": "freedom", "settings": {"domainStrategy": "UseIPv4"}}]
```

If `domainStrategy` is missing, apply it using the safe-edit procedure in step 4.

### 4. Add more clients (optional)

Each device gets its own UUID.

Dependencies — neither is present on a stock Debian image:

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
xray -test -config "$CONFIG.new" && mv "$CONFIG.new" "$CONFIG" && systemctl restart xray
```

### 5. Configure client (sing-box)

Use the snippet from step 2. Key fingerprint values by sing-box version:
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

## Teardown

```bash
ssh root@<SERVER_IP> 'systemctl disable --now xray; rm -rf /etc/xray'
doctl compute droplet delete vpn-server
```

Rotating credentials means re-running step 2: the Reality keypair and all client
UUIDs are regenerated, so every client snippet must be reissued.

## Notes

- Donor domain must support TLS 1.3. **habr.com** is the recommended default (stable, tested).
- **CRITICAL — fingerprint**: use `firefox` (mobile) or `randomized` (desktop). `chrome` is broken with Xray 25.3.6+ on many clients — connects but no traffic.
- Recommended xray version: **25.3.6** (pinned in setup-server.sh). Newer versions (25.8.x+) may break Reality client compat.
- Port 443 mimics HTTPS — works through most firewalls.
- Config path here is `/etc/xray/config.json`. Upstream Xray-install defaults to `/usr/local/etc/xray/config.json` — check which one `xray.service` actually loads before editing (`systemctl cat xray`).
- The Reality private key is generated on the VPS, so the hosting provider is inside the trust boundary. Adequate for censorship circumvention; not adequate if the threat model includes the provider itself.
- If using 3x-ui panel instead: config is managed via web UI at port 2053, not config.json directly.
- Repo with scripts: https://github.com/cyb3ralbert/sing-box-public