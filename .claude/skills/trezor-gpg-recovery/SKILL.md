---
name: trezor-gpg-recovery
description: Reconstruct a Trezor-backed GPG identity's fingerprint from its BIP39 seed phrase alone, with no Trezor hardware present — a recovery drill to verify your backup actually reproduces the key before you need it. Use when asked to verify GPG key recovery without a device, restore a Trezor GPG identity from a seed phrase, test disaster recovery for git-crypt-trezor-setup, восстановить GPG identity по seed-фразе без Trezor, or проверить бэкап Trezor GPG ключа.
---

# Trezor GPG recovery drill (seed-only, no hardware)

`trezor-agent`'s GPG identity derivation is fully deterministic: same
(BIP39 seed, passphrase, `user_id`, curve, creation-time) always produces
the same key — same fingerprint. `trezor-gpg init` defaults creation-time
to `0` (epoch), so a default-configured identity only needs the seed phrase
and the `user_id` string to reproduce.

This reconstructs the identity in software (`libagent`'s own packet-
building code, `export_public_key`, fed a software `Device` backend instead
of hardware) — output is byte-identical to what the real Trezor produces,
verified against the official SLIP-0010 nist256p1 test vector and
cross-checked with `gpg --list-packets`.

Repo: https://github.com/cyb3ralbert/trezor-gpg-recovery

Answers [romanz/trezor-agent#335](https://github.com/romanz/trezor-agent/issues/335).

**Scope note:** this is a recovery-drill / verification tool, not a
software decrypt agent for daily use. It proves your recorded parameters
reconstruct the right fingerprint; it doesn't run a persistent gpg-agent
backed by the software key, since materializing the raw private key
routinely is exactly the risk [git-crypt-trezor-setup](../git-crypt-trezor-setup/SKILL.md)
exists to avoid. For actual decryption after losing the device, the
simplest path is: restore the same seed onto any Trezor and re-run
`trezor-gpg init` with the original `--time` value — determinism gives you
the identical hardware-backed identity back, no software tooling needed.

## Prerequisites

- The `git-crypt-trezor-setup` identity already exists (or you're setting
  one up and want to record recovery parameters up front).
- Note down, at setup time: the exact `user_id` string passed to
  `trezor-gpg init`, the `--time` value used (defaults to `0` if not
  explicitly set), and the curve (`nist256p1` unless overridden).
- The BIP39 seed phrase (the one already backed up for the Trezor itself —
  no separate backup needed).

## Workflow

### 1. Set up

```bash
git clone https://github.com/cyb3ralbert/trezor-gpg-recovery
cd trezor-gpg-recovery
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
```

### 2. Verify the implementation (optional, no seed needed)

```bash
python3 test_slip0010.py
```

Cross-checks the SLIP-0010 nist256p1 derivation against the official test
vector — confirms the crypto is correct before trusting it with a real seed.

### 3. Run the recovery drill

**Interactive — must run in a real terminal, and air-gapped.** The mnemonic
is read via hidden `getpass` prompts (two: mnemonic, then BIP39 passphrase).
The derived private key exists in process memory for the run's duration.

```bash
python3 recover.py "Name <email>" --time 0 --expect-fingerprint <fingerprint-you-recorded>
```

`user_id` must match byte-for-byte what was passed to `trezor-gpg init`
originally. Prints the reconstructed fingerprints and armored public key;
with `--expect-fingerprint`, exits non-zero on mismatch instead of just
printing — use this to make the drill a pass/fail check rather than eyeballing.

Get the fingerprint to check against from the working setup:
`GNUPGHOME=~/.gnupg/trezor gpg --list-secret-keys --with-colons` (see
[git-crypt-trezor-setup](../git-crypt-trezor-setup/SKILL.md) step 3).

## Notes

- Only `nist256p1` supported — matches `trezor-gpg init`'s default curve.
  If the original identity used `--ecdsa-curve` to pick something else,
  pass the matching `--curve` here (see the ed25519 caveat in
  git-crypt-trezor-setup's Notes — untested with this tool either way).
- Run this drill right after initial setup, while the device is still in
  hand — that's when a mismatch is cheap to fix (wrong `user_id` string,
  forgotten `--time` value), not during an actual emergency.
- Tested against: `trezor_agent`/`libagent` 0.13.0, `ecdsa` 0.19.2,
  `mnemonic` 0.21.
