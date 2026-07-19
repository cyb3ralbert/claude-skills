---
name: trezor-totp-vault
description: Store TOTP/HOTP secrets encrypted at rest, decryptable only via a Trezor device touch (+ PIN). Uses the CipherKeyValue primitive so the authenticator secret is derived and decrypted on-device, never sitting in plaintext on disk. Use when asked to replace an authenticator app with a hardware key, keep 2FA secrets off the host disk, set up a Trezor-backed TOTP vault, хранить TOTP секреты на аппаратном ключе, or заменить authenticator app на Trezor.
---

# Trezor-backed TOTP vault

Encrypts TOTP secrets with `CipherKeyValue`
(`trezorlib.misc.encrypt_keyvalue`/`decrypt_keyvalue`) — the same primitive
the legacy Trezor Password Manager used. The AES key is derived on-device
from a fixed BIP32 node plus a per-entry label; only ciphertext + IV are ever
written to disk. Decrypting to compute a code requires touching the device
(and PIN, if set) every time.

Repo: https://github.com/cyb3ralbert/trezor-totp

## Prerequisites

- Trezor with `Capability.Crypto` (`trezorctl get-features` — check `capabilities`)
- udev access to the device (`LIBUSB_ERROR_ACCESS` otherwise) — same rules as
  [git-crypt-trezor-setup](../git-crypt-trezor-setup/SKILL.md) step 2, reuse
  them if already set up.

## Workflow

### 1. Clone and set up the venv

```bash
git clone https://github.com/cyb3ralbert/trezor-totp
cd trezor-totp
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt   # trezor, pyotp
```

### 2. Add a secret

**Interactive — must run in a real terminal.** PIN entry (if the device has
one set) needs a real TTY for `input()`; running this through a
non-interactive tool-call environment will hang or throw waiting on stdin.
Hand this step to the human.

```bash
./run.sh add github --secret JBSWY3DPEHPK3PXP
```

Omit `--secret` to be prompted instead (keeps it out of shell history).
Confirms once on the device (button/touch), then encrypts and stores
ciphertext + IV in `~/.trezor-totp/vault.json`.

If a PIN is set, entry uses the same position-cipher scheme as GPG/SSH
Trezor flows: the prompt shows a fixed map

```
7 8 9
4 5 6
1 2 3
```

and you type the *position number* of each real PIN digit as shown
(shuffled) on the device screen — not the digit itself.

### 3. Get the current code

```bash
./run.sh code github
```

Also interactive (device touch + PIN each time — no session caching).
Decrypts via device, base32-encodes back into a TOTP secret, and prints the
current 6-digit code with seconds remaining in the 30s window.

### 4. List / remove

```bash
./run.sh list
./run.sh remove github
```

## Notes

- Tested against: Trezor One, firmware 1.12.1, `trezor` (trezorlib) 0.20.1, `pyotp` 2.10.0.
- Fixed BIP32 node `m/13'/0'` for all entries; separation between entries
  comes from the per-entry key label (`trezor-totp:<name>`), not the path —
  each label derives a cryptographically independent key on-device.
- No session/code caching by design — every `code` call re-touches the
  device. If this is too much friction for frequent use, that's a known open
  gap upstream (see
  [romanz/trezor-agent#472](https://github.com/romanz/trezor-agent/issues/472)
  for the equivalent request against `age-plugin-trezor`); not implemented
  here to avoid caching key material on disk.
- Underlying primitive (`CipherKeyValue`) is the same one the discontinued
  Trezor Password Manager used — well-supported in firmware, not deprecated,
  just under-documented for uses other than the original password manager.
- Vault format is a flat JSON file, not git-crypt-integrated — this is a
  standalone secrets vault, unrelated to
  [git-crypt-trezor-setup](../git-crypt-trezor-setup/SKILL.md) beyond sharing
  the udev/USB setup step.
