# Setup Xray VLESS Reality Server

Set up a self-hosted encrypted proxy server (Xray VLESS Reality) on a Debian/Ubuntu VPS. Provides traffic encryption with TLS 1.3 fingerprint mimicry for privacy. Generates keypair, writes config, starts service, and outputs a ready-to-use sing-box client snippet.

## When to use

When asked to:
- "поднять self-hosted proxy сервер"
- "настроить Reality на новом дроплете"
- "setup a new Xray server"
- "deploy VLESS Reality"
- "self-hosted encrypted proxy for privacy"

## Workflow

### 1. Get VPS access

Need: root SSH access to a fresh Debian/Ubuntu VPS.

If no VPS exists, create a DigitalOcean droplet:
```bash
doctl compute droplet create vpn-server \
  --region ams3 \
  --image debian-12-x64 \
  --size s-1vcpu-512mb-10gb \
  --ssh-keys $(doctl compute ssh-key list --format ID --no-header | head -1) \
  --wait
```

Get IP: `doctl compute droplet get vpn-server --format PublicIPv4`

### 2. Run setup script

```bash
# Default donor: habr.com
ssh root@<SERVER_IP> 'bash <(curl -fsSL https://raw.githubusercontent.com/cyb3ralbert/sing-box/master/setup-server.sh)'

# Custom donor domain:
ssh root@<SERVER_IP> 'SERVER_DOMAIN=yahoo.com bash <(curl -fsSL https://raw.githubusercontent.com/cyb3ralbert/sing-box/master/setup-server.sh)'
```

Script does automatically:
- Installs Xray
- Generates Reality keypair (x25519) + short_id + client UUID
- Writes `/etc/xray/config.json`
- Starts `xray.service`
- Prints sing-box client snippet with all values

### 3. Add more clients (optional)

Each device gets its own UUID. On the server:
```bash
NEW_UUID=$(cat /proc/sys/kernel/random/uuid)
jq --arg id "$NEW_UUID" --arg email "phone" \
  '.inbounds[0].settings.clients += [{"id": $id, "flow": "xtls-rprx-vision", "email": $email}]' \
  /etc/xray/config.json | sponge /etc/xray/config.json
systemctl restart xray
echo "New UUID: $NEW_UUID"
```

### 4. Configure client (sing-box)

Use the snippet from step 2. Key fingerprint values by sing-box version:
- 1.12.8 (Android/Termux) → `firefox`
- 1.12.12+ (desktop/VM) → `randomized`

### 5. Verify

```bash
# On client machine:
no_proxy="" curl -x socks5h://127.0.0.1:<PORT> --max-time 8 ifconfig.me
# Should return server IP
```

## Notes

- Donor domain must support TLS 1.3. **habr.com** is the recommended default (stable, tested).
- **CRITICAL — UseIPv4**: if server has IPv6, xray exits via IPv6 by default → clients see IPv6 IP, some apps break. Always add to outbounds: `{"protocol": "freedom", "settings": {"domainStrategy": "UseIPv4"}}`.
- **CRITICAL — fingerprint**: use `firefox` (mobile) or `randomized` (desktop). `chrome` is broken with Xray 25.3.6+ on many clients — connects but no traffic.
- Recommended xray version: **25.3.6** (pinned in setup-server.sh). Newer versions (25.8.x+) may break Reality client compat.
- Port 443 mimics HTTPS — works through most firewalls.
- If using 3x-ui panel instead: config is managed via web UI at port 2053, not config.json directly.
- Repo with scripts: https://github.com/cyb3ralbert/sing-box