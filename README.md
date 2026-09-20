# fingerprint-sensor

[![build](https://github.com/benjaminjamesxyz/fingerprint-sensor/actions/workflows/ci.yml/badge.svg)](https://github.com/benjaminjamesxyz/fingerprint-sensor/actions/workflows/ci.yml)

**Fingerprint sensor (FPC `10a5:a900`) support on Linux — HONOR MagicBook X14 Pro / Art 14.**

Working enrollment, login, sudo and lock-screen unlock, verified end to end on
Arch Linux: `fprintd-enroll` → `Verify result: verify-match`.

Built on [cityji's upstream driver](https://github.com/cityji/honor-magickbookpro-fingerprint-driver);
this repo adds an **AUR package**, a **polkit rule** that lets you enroll
without a desktop polkit agent, **hardened CI**, and a complete setup guide.
Upstream's original README is preserved in
[docs/UPSTREAM-README.md](docs/UPSTREAM-README.md), together with its
protocol teardown ([docs/INVESTIGATION.md](docs/INVESTIGATION.md)).

---

## Find it fast

| I want to… | Go to |
|---|---|
| check it works on my laptop | [§1 Is this for your laptop?](#1-is-this-for-your-laptop) |
| understand what gets installed | [§2 What this actually does](#2-what-this-actually-does-30-second-version) |
| install on Arch | [§3 Install — Arch Linux](#3-install--arch-linux-package) |
| install on Ubuntu / Fedora / openSUSE | [§4 Install — other distros](#4-install--other-distros-upstream-bootstrap) |
| enroll a finger (or several) | [§5 Enroll your finger](#5-enroll-your-finger) · [More than one](#more-than-one-finger) |
| unlock login / sudo / lock screen | [§6 Fingerprint login, sudo, lock screen](#6-fingerprint-login-sudo-lock-screen) |
| set up hyprlock | [Example: hyprlock](#example-hyprlock) |
| fix something that broke | [§7 Troubleshooting](#7-troubleshooting) |
| remove everything | [§8 Uninstall](#8-uninstall) |
| know if updates break it | [Will a system update break it?](#will-a-system-update-break-it) |
| read how the sensor was reverse-engineered | [docs/INVESTIGATION.md](docs/INVESTIGATION.md) |

## 1. Is this for your laptop?

```bash
lsusb | grep 10a5
```

| What you see | Status |
|---|---|
| `10a5:a900` — product string `FPC Sensor Controller L:0002 FW:22.26.2.29` | **This repo.** Tested on HONOR MagicBook X14 Pro. |
| `10a5:9800` | The sibling sensor (various Lenovo ThinkBook/IdeaPad). Likely works — see upstream [docs/DEVICE.md](docs/DEVICE.md) first. |
| any other `10a5:…` | Possibly reachable; upstream docs have an identification checklist. |
| no `10a5:` device | Not this project — check [libfprint's supported devices](https://fprint.freedesktop.org/supported-devices.html). |

## 2. What this actually does (30-second version)

No distribution supports this sensor. It is a **match-on-host** device: the
sensor only captures and encrypts an image; the matching runs on your CPU
inside FPC's proprietary `libfpcbep.so`. That library cannot be redistributed,
so it is **fetched at install time from Lenovo's public driver download and
checksum-verified** (md5 `f7136fd774d5208e629bbeaa4974543a`).

The fix is a patched `libfprint` (v1.94.6 + the `fpcmoh` driver from upstream
merge request 396 + the patches in `patches/`) wired into **`fprintd` only**,
via a systemd drop-in. Your distribution's `libfprint` stays exactly where it
is and keeps serving everything else. Removal is one `pacman -R`.

If you want the full story — USB traces, the three root-caused bugs, and why
enrollment used to report success while rejecting every image — read
[docs/INVESTIGATION.md](docs/INVESTIGATION.md). It is excellent.

## 3. Install — Arch Linux (package)

```bash
git clone https://github.com/benjaminjamesxyz/fingerprint-sensor
cd fingerprint-sensor/packaging/aur
makepkg -si
```

`makepkg` needs network access (it fetches libfprint sources and the
proprietary matcher) and takes about two minutes. The package installs:

```
/usr/lib/fpc-a900/                              patched libfprint + matcher
/usr/lib/systemd/system/fprintd.service.d/10-fpc-a900.conf   fprintd sees it
/usr/lib/udev/rules.d/60-libfprint-2-device-fpc-a900.rules
/usr/share/polkit-1/rules.d/50-fpc-a900.rules   agent-less enrollment
/usr/lib/tmpfiles.d/fpc-a900.conf               creates /var/log/fpc
```

Nothing under `/usr/lib` is overwritten; the drop-in only affects `fprintd`.

## 4. Install — other distros (upstream bootstrap)

Ubuntu, Fedora, openSUSE and Arch are covered by upstream's one-command
installer, which builds the same thing from source:

```bash
curl -fsSL https://raw.githubusercontent.com/cityji/honor-magickbookpro-fingerprint-driver/main/bootstrap.sh | bash
```

Reasonable people do not pipe scripts to bash unread. The script was reviewed
line by line when this repo was set up: it installs distro build
dependencies, clones the upstream repo to `~/.cache/fpc-a900-build`, fetches
and checksum-verifies the matcher, builds, and installs to `/opt/fpc-a900`
(never touching `/usr/lib`). Removal: `sudo ./uninstall.sh` in that directory.

## 5. Enroll your finger

```bash
fprintd-enroll
```

What to expect — this trips everyone up once:

- Press, lift, press again. Each press is one stage. It needs **10–16
  accepted samples** and takes about a minute.
- `enroll-remove-and-retry` and `enroll-retry-scan` appear many times. **That
  is normal, not an error.** Lift completely and press again, shifting the
  finger slightly between presses (one edge → center → other edge).
- You only get `Enroll result: enroll-completed` at the very end. **Ctrl+C
  discards all progress** — there are no partial saves.
- Enroll as your own user, **not with sudo** — `sudo fprintd-enroll` stores
  the print under root, where it is useless for your login.

Then verify:

```bash
fprintd-verify
```

`Verify result: verify-match` is the **only** proof that matters. For most of
this sensor's history, enrollment reported success while the matcher rejected
every image — so trust the verify, not the enroll.

### More than one finger

```bash
fprintd-enroll -f right-thumb
fprintd-enroll -f left-index-finger
```

Names: `left`/`right` × `thumb`, `index-finger`, `middle-finger`,
`ring-finger`, `little-finger`. Two or three is plenty — every PAM consumer
(sudo, login, lock screen) matches **any** enrolled print automatically;
nothing to configure per finger. `fprintd-list $USER` shows what is enrolled.

## 6. Fingerprint login, sudo, lock screen

Add `pam_fprintd` as a **sufficient** auth method — fingerprint first,
password still always works, so there is no lock-out risk:

```bash
# Arch:
sudo sed -i '1i auth        sufficient    pam_fprintd.so' /etc/pam.d/system-auth

# Debian/Ubuntu:
sudo pam-auth-update            # tick "Fingerprint authentication"

# Fedora (and Arch with authselect):
sudo authselect enable-feature with-fingerprint
```

Test sudo without a password prompt race: `sudo -k && sudo true` — it will
wait for a finger first, Enter falls through to your password.

Lock-screen caveat: `pam_fprintd`'s default window is roughly **10 seconds
and one attempt**, and some lock screens print no "place your finger" prompt
at all. Have the finger moving as the lock screen appears, or raise
`timeout=` in the PAM line.

**Enrollment without a desktop polkit agent.** Plain `fprintd-enroll` needs
polkit authorization, and without an agent you get
`Not Authorized: net.reactivated.fprint.device.enroll`. The Arch package
ships a scoped rule (`50-fpc-a900.rules`) that allows wheel-group members in
local, active sessions to enroll/verify/list/delete prints without a prompt.
It grants nothing beyond fingerprint management and does not touch PAM. On
other distros, install the same file from `polkit/` into
`/etc/polkit-1/rules.d/`.

### Example: hyprlock

```bash
sudo sed -i '1i auth        sufficient    pam_fprintd.so timeout=20' /etc/pam.d/hyprlock
```

Behaviour, verified on v0.9.6: the sensor arms the moment the lock screen
appears — touch unlocks instantly. Typing your password and pressing
**Enter** abandons the fingerprint wait and authenticates against the
password instead. `timeout=20` widens the default 10-second window; add
`max-tries=3` if you want more sensor attempts per unlock. The same one-line
pattern works for any other locker — prepend it to that locker's file in
`/etc/pam.d/`.

Test before trusting it: run `hyprlock` from a terminal first (fingerprint
unlock, then password path), with `Ctrl+Alt+F2` → `pkill hyprlock` as the
escape hatch.

### Will a system update break it?

No. The patched library lives in `/usr/lib/fpc-a900/` and is reached only
through the fprintd drop-in; `pacman -Syu` updating the stock `libfprint`
does not touch it. Only the package itself (or upstream tag bumps, which we
track in the AUR package) changes what fprintd loads.

## 7. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `Not Authorized: …device.enroll` | Polkit rule missing or misnamed (rules **must** end in `.rules`). See §6. |
| `NoEnrolledFingers` on verify | Enrollment was interrupted (no partial saves), or you enrolled with sudo (prints stored under root). Re-enroll as your user, see it through. |
| Enroll works, verify never matches | You are not running the patched library. Check `grep fpc-a900 /proc/$(pidof fprintd)/maps` — must show `/usr/lib/fpc-a900` (or `/opt/fpc-a900` for the bootstrap install). |
| Sensor "disappears" or resets after a capture | The unpatched-driver behaviour (firmware watchdog reset). The `patches/0006` fix reads the image completely; make sure the patched build is loaded. |
| Fingerprint dead after every wake from sleep | Fixed by `patches/0011` (package ≥ 1.0.0-3): the driver double-reported resume, tripping libfprint's `suspend_resume_task` assertion and killing the first post-wake attempt. A `system-sleep` hook also restarts fprintd on wake as a safety net. If a touch does nothing after a long lock, press **Enter** first — it re-arms the fingerprint wait (`timeout=` on the PAM line). |
| `fprintd` fails to start after installing | Prebuilt-library ABI mismatch — build from source instead (§4). Upstream documents this in its README. |
| Password login broke after PAM edit | Should not happen with `sufficient` (§6). If it did: boot with a live USB, remove the `pam_fprintd` line you added from `/etc/pam.d/system-auth`. |

More: upstream's [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md).

## 8. Uninstall

```bash
# Arch package:
sudo pacman -R fpc-a900-fingerprint-driver

# Bootstrap install:
sudo ~/.cache/fpc-a900-build/uninstall.sh
```

⚠️ **Do not run upstream's `uninstall.sh` while the Arch package is
installed.** It deletes `/var/log/fpc`, which the package's systemd drop-in
requires (`ReadWritePaths=`) — `fprintd` then fails to start with
`status=226/NAMESPACE` until the directory is recreated:
`sudo systemd-tmpfiles --create fpc-a900.conf` (or a reboot).

Both remove everything they added. Enrolled prints live in
`/var/lib/fprint` (or `/var/lib/fprintd`); delete with `fprintd-delete` if you
want them gone too.

## 9. Credits

All hard problems were solved upstream. This repo stands on:

- **[cityji](https://github.com/cityji/honor-magickbookpro-fingerprint-driver)**
  — the driver, the patches, the investigation. This repo's reason to exist.
- **Jason Huang (Fingerprint Cards AB)** — the `fpcmoh` driver,
  [libfprint MR 396](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/396).
- **Lenovo** — publishes `libfpcbep.so` publicly, which is the only reason
  this works without an NDA.

Local deviations from upstream files are documented in
[VENDOR-NOTES.md](VENDOR-NOTES.md). Licence: LGPL-2.1-or-later (matching
libfprint); `libfpcbep.so` remains proprietary Fingerprint Cards AB property.
