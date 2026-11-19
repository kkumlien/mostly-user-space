+++
title = "Fingerprint (fprintd) authentication on Debian with Dell's Broadcom ControlVault 3"
date = 2026-11-19
description = "Getting the Broadcom ControlVault 3 fingerprint reader (0a5c:5843) working on Debian forky with Ubuntu's libfprint TOD build and Broadcom's 5.12.018 driver."
+++

*[Diátaxis](https://diataxis.fr/): tutorial · **how-to guide** · reference · explanation*

Fingerprint authentication with the Broadcom ControlVault 3 reader on the power button of many Dell Latitude and Precision laptops works on Debian 14 ("forky" - testing), using libfprint TOD pair from Ubuntu 26.04 ("Resolute") plus Broadcom's 5.12.018 driver, all three on `apt-mark hold`. Tested on a Dell Latitude 7640 (i7-1365U) on 18 November 2026. The same sensor sits in much of the 2022-2023 Latitude and Precision lines (7x40, 5x30/5x40 and siblings), which are likely to behave the same: check `lsusb` below.

Please note: tested on forky only. On Debian 13 ("trixie" - stable) the Ubuntu 26.04 packages used here won't install; see the trixie note under [Why Ubuntu's libfprint](#why-ubuntu-s-libfprint).

## TL;DR

On `0a5c:5843` (Citadel A0 CID7) the two 5.15 drivers tested fail out of the box with status (11), so the target is driver **5.12.018** with chip firmware AAI 5.12.18.0 / SBI 181 (the two firmware version numbers the driver logs). Debian's libfprint lacks TOD (Touch OEM Drivers) support, which is what loads closed vendor drivers like Broadcom's; Ubuntu resolute's `libfprint-2-2` + `libfprint-2-tod1` **1.95.1+tod1** install cleanly on forky and run the 5.12.018 driver. A chip already on 5.12.18.0 (e.g. from a previous install) isn't flashed at all; otherwise do a **cold boot after the flash**, i.e. full poweroff and boot again before trying enrollment.

## Disclaimer

At your own risk. The driver flashes the ControlVault firmware when it doesn't match, and **interrupting a flash can brick the sensor**: AC power, no suspend, no service restarts until it completes. 5.12.18.0 predates the ReVault fixes (see [Why not 5.15](#why-not-5-15)).

## Does this affect me?

- **Symptoms**: enrollment runs through all stages, then `Enrollment failed : Device status = (11)`; verify stopped matching after a driver update.
- **Hardware**: `lsusb | grep 0a5c` shows `0a5c:5843` (BCM58200 ControlVault 3). `0a5c:5865` or newer is the cv3plus family: different package, versions here don't apply.

## Pre-checks

Just in case something has been fixed since. None of these commands touch the chip.

1. **Online**: anything in the [Broadcom pool](http://dell.archive.canonical.com/updates/pool/public/libf/libfprint-2-tod1-broadcom/) newer than `5.15.285-5.15.010.0`, or in Yao Wei's PPA newer than 5.15.377, or recent movement on [LP bug 2119302](https://bugs.launchpad.net/libfprint-2-tod1-broadcom/+bug/2119302)? A fixed driver would make this post obsolete.

2. **apt** (read-only commands):

   ```bash
   apt policy libfprint-2-2 fprintd libpam-fprintd
   apt-cache search --names-only fprint
   ```

   No `libfprint-2-tod1` in the results means Debian still has no TOD build.

3. **Stock run**: Debian's packages alone, no driver, no flash risk:

   ```bash
   sudo apt install -y fprintd libpam-fprintd
   fprintd-list $USER
   ```

   Expected: `No devices available`. Anything else, stop and investigate.

## Why Ubuntu's libfprint

The Broadcom driver needs the TOD fork of libfprint, which only Ubuntu packages. Resolute's build is the one that fits forky:

| Source | Version | On forky |
|---|---|---|
| Debian | 1:1.94.10-1 | Installs, but built without TOD: no driver can load |
| Ubuntu 24.04 (noble) | 1:1.94.7+tod1-0ubuntu5~24.04.9 | Refused: needs `libgusb2`, which forky [renamed to `libgusb2a`](https://tracker.debian.org/news/1707833/accepted-libgusb-049-6-source-amd64-into-experimental/) over an ABI change; Debian's fprintd also wants libfprint ≥ 1:1.94.9 |
| Ubuntu 26.04 (resolute) | 1:1.95.1+tod1-0ubuntu2 | Works: depends on `libgusb2a` and `libssl3t64`; newer than Debian's, so no downgrade |

Arch-based guides pin libfprint-tod at 1.94.8 ([example](https://github.com/Lberonio/Dell-Latitude-5531-fprint-arch-linux)), and newer TOD builds have been seen crashing this driver there (`G_IS_TASK` assertion). Ubuntu's 1.95.1 build shows no such crash.

**On trixie**: the libgusb rename hasn't reached stable, which still ships [`libgusb2` 0.4.9-1](https://packages.debian.org/trixie/libgusb2), so resolute's pair (needs `libgusb2a`) won't install. The likely fit is Ubuntu 25.10's (questing) `1.94.9+tod1` build, which predates the rename; take the current one from the [Ubuntu pool](http://archive.ubuntu.com/ubuntu/pool/main/libf/libfprint/). Not tested: let apt tell you whether trixie's fprintd accepts it, and expect the same steps otherwise.

## Install

When fprintd starts, the Broadcom driver compares the chip's firmware with the version it ships and flashes the chip if they differ, which can take a few minutes. systemd's default start timeout could kill fprintd mid-flash, so first add an override (a drop-in) to the fprintd unit that lifts the timeouts. It only matters if a flash happens, but it's harmless to keep (see [Flash and cold boot](#flash-and-cold-boot)). Run `sudo systemctl edit fprintd` and in the top section:

```ini
[Service]
ExecStart=
ExecStart=/usr/libexec/fprintd --no-timeout
TimeoutStartSec=5min
```

Drop-ins merge with the unit, so `Type` and `BusName` are inherited. `ExecStart` is a list, hence the empty line to clear it first.

Fetch the three debs and install them (commands work in bash and fish):

```bash
mkdir -p /tmp/fprint-debs && cd /tmp/fprint-debs
curl -fL --remote-name-all \
  http://archive.ubuntu.com/ubuntu/pool/main/libf/libfprint/libfprint-2-2_1.95.1+tod1-0ubuntu2_amd64.deb \
  http://archive.ubuntu.com/ubuntu/pool/main/libf/libfprint/libfprint-2-tod1_1.95.1+tod1-0ubuntu2_amd64.deb \
  http://dell.archive.canonical.com/updates/pool/public/libf/libfprint-2-tod1-broadcom/libfprint-2-tod1-broadcom_5.12.018-0ubuntu1~22.04.01_amd64.deb
printf '%s\n' \
  "aa09cc1ac94608ac1dcaca0bfda0b172a466b6d6ad99a2fc8f5477c23222842a  libfprint-2-2_1.95.1+tod1-0ubuntu2_amd64.deb" \
  "aad2e14758121dede1d30aafed754374c7716f735e58074105eded2fb1c8ab9e  libfprint-2-tod1_1.95.1+tod1-0ubuntu2_amd64.deb" \
  "c2b0822ce0a0b7b916c77259445b7fa06ea200f1204b5c62d01d3203db8b7e6a  libfprint-2-tod1-broadcom_5.12.018-0ubuntu1~22.04.01_amd64.deb" \
  | sha256sum -c
sudo apt install -y fprintd libpam-fprintd ./*.deb
```

Both pools are plain HTTP, so the checksums above are what protects the download. Checked on 19 November 2026: the two Ubuntu hashes against Ubuntu's GPG-signed resolute-updates index, the Broadcom hash against the same deb downloaded in July 2026. Stop if any line doesn't say `OK`.

- The Ubuntu build number moves with Ubuntu's stable release updates (SRUs): if the download 404s, take the current resolute one from the [Ubuntu pool](http://archive.ubuntu.com/ubuntu/pool/main/libf/libfprint/).
- Copy the debs somewhere persistent: `/tmp` doesn't survive a reboot, and reinstalling is a plain `apt install ./…`.

Then start fprintd and watch:

```bash
sudo systemctl restart fprintd
sudo journalctl -u fprintd -b -f
```

Expected with the chip already on 5.12.18.0, no flash (typo verbatim):

```txt
AAI version matches - it is up do date
SBI version matches - it is up do date
Citadel A0 CID7 Chip Found....
Sensor firmware versions in file and on device match
Started fprintd.service - Fingerprint Authentication Daemon.
```

`Going to update BCM (bcmCitadel_7.otp)` instead means it's flashing: hands off, then [cold boot](#flash-and-cold-boot).

## Flash and cold boot

Only when the chip isn't on 5.12.18.0 yet, with the timeout override from the [Install](#install) section in place.

1. Start fprintd and watch its journal, as in the Install section, and wait for `Firmware Upgrade Complete` … `Control Vault firmware upgrade successful`.
2. **Cold boot, not reboot**: `systemctl poweroff`, then power on. A warm chip after a flash often fails init (`unspecified error` / `G_IS_TASK`), which looks like the bug but isn't. The journal should then show `Current AAI Version = 5.12.18.0` and `AAI version matches`.

Keep the override afterwards. fprintd is D-Bus activated, not started at boot (`systemctl is-enabled fprintd` should say `static`), so it can't hold up booting. The only effect is that fprintd stays resident after first use instead of exiting when idle; in exchange, any future driver that flashes the chip is covered.

## Enroll and PAM

Enrolling needs polkit authorisation, and if your session runs no polkit agent (common on minimal Wayland setups) nothing can ask for it (`PermissionDenied: Not Authorized`), so enroll as root for your user.

Choose a different finger based on your preference:

```bash
sudo fprintd-enroll -f right-index-finger $USER
fprintd-verify
```

After a reinstall `fprintd-list` shows no fingers even though old templates are still on the chip. Just re-enroll; if it complains about a duplicate, `fprintd-delete $USER` first.

PAM: `libpam-fprintd` ships a pam-auth-update profile. Run `sudo pam-auth-update` and check "Fingerprint authentication" is ticked; no hand-editing of `/etc/pam.d`.

## Checks

| Check | How | Result |
|---|---|---|
| Verify | `fprintd-verify` | `verify-match (done)` |
| sudo | `sudo -kv` | Prompts for the finger straight away |
| Lock screen | Lock, touch the sensor | Unlocks screen |
| Login screen | Log out, touch the sensor | Logs in user |

If the lock or login screen doesn't react to the finger, press Enter on the empty password field first to start the fingerprint prompt. Some greeters and screen lockers won't submit an empty field: type any character, then press Enter.

## Hold

Once the checks pass:

```bash
sudo apt-mark hold libfprint-2-2 libfprint-2-tod1 libfprint-2-tod1-broadcom
```

- **What it guards against**: Debian's own libfprint overtaking the version number. Today Debian's 1:1.94.10-1 sorts below 1:1.95.1+tod1, but a future 1:1.95.2-1 would win and silently swap in a build without TOD. Symptom: no device, back to password only; no flash, no brick.
- **Broadcom driver**: nothing in the apt sources ships it, so the hold is redundant there; kept for completeness. A firmware flash can only come from a driver installed by hand.
- **No updates either way**: these debs have no repo behind them, so there are no security fixes for libfprint. Check the resolute pool now and then; `apt-mark unhold`, install the newer pair, hold again.
- **Reminder**: held packages show up in `apt upgrade` output.

## Notes

- **Enrollments survive flashes**: templates live in the chip's storage, not in the firmware. If verify fails after a flash, `fprintd-delete` + re-enroll.
- **Security**: the trade-off of staying on 5.12 is in [Why not 5.15](#why-not-5-15).
- **Quirks**: sensor missing from `lsusb` after suspend → cold boot.
- **When to revisit**: a Broadcom driver newer than 5.15.377 that fixes status (11), in the Dell pool or Yao Wei's PPA, or Debian packaging the TOD fork. Either one ends the holds.

## Why not 5.15

Both 5.15 drivers tried on this sensor (on the 7640, July 2026) initialise cleanly, then enroll and verify fail with `Device status = (11)`:

| Driver | Chip firmware | Result |
|---|---|---|
| 5.12.018 | AAI 5.12.18.0 / SBI 181 | Works |
| 5.15.285 (Dell pool) | AAI 5.15.010.0 / SBI 229 | Status (11) |
| 5.15.021 build via AUR, very likely 5.15.377 | AAI 5.15.21.0 / SBI 240 | Status (11) |

"5.15.021" was the AUR's label, taken from the firmware number. It was built from Canonical's packaging git around the 5.15.377 release (7 July 2026), so very likely the same driver as in [Yao Wei's PPA](https://launchpad.net/~medicalwei/+archive/ubuntu/dell-cv3-cv3plus), though not installed from there.

**The trade-off of staying on 5.12**: chip firmware 5.12.18.0 predates the ReVault fixes ([Cisco Talos](https://blog.talosintelligence.com/revault-when-your-soc-turns-against-you/), [Dell DSA-2025-053](https://www.dell.com/support/kbdoc/en-us/000276106/dsa-2025-053), remediated in 5.15.7.0+). On a single-user Linux laptop the exposure is mostly physical: the chip is root-gated by udev, and a root-level attacker can down-flash even a patched chip, as this guide shows. The realistic residual is a running, locked machine with the chassis opened and the matcher tampered with to accept any finger; powering off before leaving it unattended in untrusted places closes that window. Suspend users, shared machines, Windows dual-boot and fleets: think twice.

A [report on the Arch forum](https://bbs.archlinux.org/viewtopic.php?id=312437) (Latitude 5420, same sensor, also seen on a 7400) traces status (11) to a host-timestamp interrupt that the vendor driver never answers. The poster fixed it with an `LD_PRELOAD` shim injected into fprintd plus a patched libfprint-tod loader. Haven't tested it. This alternative, 5.15 plus the shim, gets the patched firmware but runs an unofficial library inside the root daemon that decides authentication, on top of a patched libfprint. Neither option is clean; this guide picks the one with only vendor-built binaries.

## References

- [LP bug 2119302, enrollment fails with Device status = (11)](https://bugs.launchpad.net/libfprint-2-tod1-broadcom/+bug/2119302)
- [Dell/Canonical OEM archive pool (Broadcom debs)](http://dell.archive.canonical.com/updates/pool/public/libf/libfprint-2-tod1-broadcom/)
- [Ubuntu pool, libfprint](http://archive.ubuntu.com/ubuntu/pool/main/libf/libfprint/) · [resolute libfprint-2-2 1.95.1 package record](https://ubuntuupdates.org/listfile_data/3403055)
- [Debian tracker, libfprint](https://tracker.debian.org/pkg/libfprint-2-2) (no `libfprint-2-tod1` binary) · [libgusb2 → libgusb2a rename](https://tracker.debian.org/news/1707833/accepted-libgusb-049-6-source-amd64-into-experimental/)
- [Cisco Talos: ReVault](https://blog.talosintelligence.com/revault-when-your-soc-turns-against-you/) · [Dell DSA-2025-053](https://www.dell.com/support/kbdoc/en-us/000276106/dsa-2025-053)
- [Yao Wei's PPA, ppa:medicalwei/dell-cv3-cv3plus](https://launchpad.net/~medicalwei/+archive/ubuntu/dell-cv3-cv3plus) (5.15.377)
- [Arch forum: status (11) root cause and shim](https://bbs.archlinux.org/viewtopic.php?id=312437)
