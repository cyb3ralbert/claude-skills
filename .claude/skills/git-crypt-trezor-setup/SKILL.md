---
name: git-crypt-trezor-setup
description: Wire git-crypt to a Trezor hardware wallet via GPG, so the git-crypt symmetric key is encrypted to a Trezor-backed GPG identity and the private key material never touches disk. Sets up trezor-agent, initializes the GPG identity, and connects it to git-crypt with add-gpg-user. Use when asked to connect Trezor to git-crypt, set up hardware-backed git-crypt, keep a repo key off the host disk, подключить Trezor к git-crypt, or настроить git-crypt через аппаратный ключ.
---

# git-crypt + Trezor (via GPG)

git-crypt's default mode stores the symmetric key as a plaintext file on disk
(`.git-crypt/keys/default`). This wires it through GPG instead, backed by
`trezor-agent` (romanz/trezor-agent — hardware-backed GPG/SSH/age agent): the
key lives on the Trezor, only sign/decrypt *results* leave the device.

Chain: **git-crypt (GPG mode) → GPG → trezor-agent → Trezor**. Nothing custom
to build, just wiring three existing pieces together.

## Prerequisites

- GPG ≥ 2.1.11 (`gpg --version`)
- Trezor One or Model T, firmware with `Capability.Crypto`
  (`trezorctl get-features` — check `capabilities`)
- If the Trezor is on a different host than where this runs (e.g. a VM),
  USB passthrough first — see Notes.

## Workflow

### 1. Install trezor-agent in a venv

```bash
sudo apt-get install -y python3-venv python3-full libusb-1.0-0 python3-dev \
  build-essential libudev-dev pkg-config pinentry-curses
python3 -m venv venv && source venv/bin/activate
pip install trezor_agent
```

Installs `trezorctl`, `trezor-gpg`, `trezor-gpg-agent` into `venv/bin`.

### 2. udev access for the USB device

Without this, `trezorctl` fails with `LIBUSB_ERROR_ACCESS [-3]`:

```bash
sudo tee /etc/udev/rules.d/51-trezor.rules > /dev/null <<'EOF'
SUBSYSTEM=="usb", ATTR{idVendor}=="534c", ATTR{idProduct}=="0001", MODE="0660", GROUP="plugdev", TAG+="uaccess"
SUBSYSTEM=="usb", ATTR{idVendor}=="1209", ATTR{idProduct}=="53c0", MODE="0660", GROUP="plugdev", TAG+="uaccess"
SUBSYSTEM=="usb", ATTR{idVendor}=="1209", ATTR{idProduct}=="53c1", MODE="0660", GROUP="plugdev", TAG+="uaccess"
KERNEL=="hidraw*", ATTRS{idVendor}=="534c", ATTRS{idProduct}=="0001", MODE="0660", GROUP="plugdev", TAG+="uaccess"
KERNEL=="hidraw*", ATTRS{idVendor}=="1209", ATTRS{idProduct}=="53c0", MODE="0660", GROUP="plugdev", TAG+="uaccess"
KERNEL=="hidraw*", ATTRS{idVendor}=="1209", ATTRS{idProduct}=="53c1", MODE="0660", GROUP="plugdev", TAG+="uaccess"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
sudo usermod -aG plugdev "$USER"   # re-login (or `sg plugdev`) to pick it up
```

Verify: `sg plugdev -c "venv/bin/trezorctl get-features"` should print device
info, not a USB access error.

### 2b. USB passthrough into a VM (only if needed)

If the Trezor is physically on a hypervisor host and this runs inside a VM
(libvirt/KVM), passthrough is a host-side action:

```bash
# on the host, find the device
lsusb | grep -i trezor   # e.g. ID 1209:53c1

# attach into the running VM (persists across reboots via --config)
virsh -c qemu:///system attach-device <vm-name> --live --config /dev/stdin <<'EOF'
<hostdev mode='subsystem' type='usb'>
  <source>
    <vendor id='0x1209'/>
    <product id='0x53c1'/>
  </source>
</hostdev>
EOF
```

Then `lsusb` inside the VM should show the device.

### 3. Init the GPG identity on the Trezor

**This step is interactive and must run in a real terminal — not through an
automated tool-call environment.** `trezor-gpg init` calls a `pinentry`
program to collect the PIN, and that program needs a real TTY (curses) or a
real X `DISPLAY` (GUI). Neither exists inside a non-interactive command
runner, so the call throws `UnexpectedError: ... isatty ...` there. Hand this
step to the human running the session.

```bash
sg plugdev -c "bash -c 'source venv/bin/activate && trezor-gpg init \
  \"Name <email>\" \
  --pin-entry-binary=/usr/bin/pinentry-curses \
  --passphrase-entry-binary=/usr/bin/pinentry-curses'"
```

**PIN entry is a position cipher, not the PIN itself.** The prompt shows a
fixed position map:

```
7 8 9
4 5 6
1 2 3
```

The Trezor's own screen shows the *same* 3×3 grid with digits shuffled fresh
each attempt. Find where each real PIN digit sits on the device screen, and
type the corresponding *position number* from the map above — not the digit,
and not the map's visual order. Same scheme Electrum's clickable PIN grid
uses, just as a typed sequence instead of mouse clicks.

If GUI pinentry is preferred over curses (e.g. connecting over VNC with a
real X server): install `pinentry-gtk2`, pass
`--pin-entry-binary=/usr/bin/pinentry-gtk-2` instead, and make sure `DISPLAY`
is set. Functionally identical, just a dialog instead of a terminal prompt.

Creates `~/.gnupg/trezor/` (homedir) with a shadow secret key — `gpg
--list-secret-keys` shows `sec` but no private key material is on disk, only
a reference to the device. Note the fingerprint printed at the end; it's
needed in the next step.

To switch an already-initialized identity between curses and GUI pinentry
later, edit the `--pin-entry-binary`/`--passphrase-entry-binary` flags baked
into `~/.gnupg/trezor/run-agent.sh` directly — no need to re-run `init`.

### 4. Wire it into git-crypt

```bash
sudo apt-get install -y git-crypt   # if not already present
cd <repo>
git-crypt init                      # generates git-crypt's own symmetric key
export GNUPGHOME=~/.gnupg/trezor
sg plugdev -c "bash -c 'export GNUPGHOME=~/.gnupg/trezor && \
  git-crypt add-gpg-user <fingerprint-from-step-3>'"
```

`add-gpg-user` only *encrypts* git-crypt's key to the Trezor identity's
public key — this step needs no device interaction, no PIN.

### 5. Verify the full cycle

```bash
git-crypt lock      # no device needed — re-encrypts working tree files
sg plugdev -c "bash -c 'export GNUPGHOME=~/.gnupg/trezor && git-crypt unlock'"
```

`unlock` needs the PIN once (again interactive — hand to the human), then
decrypts every git-crypt file for the whole repo in one device confirmation,
not once per file.

## Notes

- Tested against: Trezor One, firmware 1.12.1, GPG 2.2.40, `trezor_agent`
  0.13.0 (`libagent`), git-crypt 0.7.0, Debian 12.
- Key curve is `nistp256` by default (`trezor-gpg init` doesn't take a curve
  flag for the primary GPG identity). If you ever move to GPG ≥ 2.5.x with
  ed25519 keys, check upstream issue
  [romanz/trezor-agent#515](https://github.com/romanz/trezor-agent/issues/515)
  (decrypt failures) before upgrading — doesn't affect nistp256.
- Security model: git-crypt's symmetric key is encrypted to the Trezor
  identity's public key and stored in `.git-crypt/keys/`. Decrypting it
  requires the device to sign/decrypt on-device; a compromised host can at
  most prompt you to confirm an operation on the physical Trezor screen, it
  cannot exfiltrate the key material itself.
- `pinentry-gnome3` fails outright in a headless/VNC-less environment
  (`DISPLAY not defined` → dbus/gcr errors) — use `pinentry-curses` (needs a
  real TTY) or `pinentry-gtk2` (needs a real `DISPLAY`, e.g. over VNC) instead.
- No PIN-cache expiry was configured here (`--cache-expiry-seconds` left at
  the library default); if a long-lived unlocked session is undesirable,
  check upstream issue
  [romanz/trezor-agent#474](https://github.com/romanz/trezor-agent/issues/474).
