# Rollback: COSMIC/NVIDIA suspend VT-switch bypass

Applied 2026-08-30. This is what to do if suspend/resume behaves *worse*
after this change (e.g. suspend no longer completes, laptop won't wake,
fans stay on after closing the lid, battery drains during sleep).

## What was changed

Three systemd drop-in files were added (nothing in `/usr/lib` or the
NVIDIA driver package was touched):

```
/etc/systemd/system/nvidia-suspend.service.d/50-cosmic-no-vt-switch.conf
/etc/systemd/system/nvidia-hibernate.service.d/50-cosmic-no-vt-switch.conf
/etc/systemd/system/nvidia-suspend-then-hibernate.service.d/50-cosmic-no-vt-switch.conf
```

Each one overrides the service's `ExecStart` to skip `nvidia-sleep.sh`
(which does `chvt 63`, the thing that was breaking `cosmic-comp`) and
write directly to `/proc/driver/nvidia/suspend` instead.

## How to check if this is still in place

```bash
systemctl cat nvidia-suspend.service
```
If you see a `50-cosmic-no-vt-switch.conf` section at the bottom, the
override is active.

## How to roll back

### Option A — remove just this fix (recommended first step)

```bash
sudo rm /etc/systemd/system/nvidia-suspend.service.d/50-cosmic-no-vt-switch.conf
sudo rm /etc/systemd/system/nvidia-hibernate.service.d/50-cosmic-no-vt-switch.conf
sudo rm /etc/systemd/system/nvidia-suspend-then-hibernate.service.d/50-cosmic-no-vt-switch.conf

# optional: clean up the now-empty directories
sudo rmdir /etc/systemd/system/nvidia-suspend.service.d
sudo rmdir /etc/systemd/system/nvidia-hibernate.service.d
sudo rmdir /etc/systemd/system/nvidia-suspend-then-hibernate.service.d

sudo systemctl daemon-reload
```

Verify it's gone:
```bash
systemctl cat nvidia-suspend.service
```
You should only see the original `[Unit]`/`[Service]`/`[Install]` block,
no `50-cosmic-no-vt-switch.conf` section.

No reboot is required — the next suspend after `daemon-reload` will use
the original (`nvidia-sleep.sh`-based, `chvt 63`-including) behavior
again, i.e. back to the pre-fix state (which had the blank-screen bug,
but was otherwise a known, stock configuration).

### Option B — if something is stuck *right now* (mid-suspend, won't wake)

1. Hold the power button ~10s to force a hard shutdown.
2. Power back on. The system will boot normally — this override only
   affects the suspend path, not boot.
3. Once booted, follow Option A above to remove the drop-ins before
   suspending again.

## Other things also changed in this session (only touch these if asked)

These are unrelated to this specific fix and don't need to be rolled
back for this — listed here only so you have the full picture of what
changed recently:

- `/etc/modprobe.d/blacklist-spd5118.conf` — blacklists the unrelated
  `spd5118` RAM-sensor driver (cosmetic fix, unrelated to this bug).
  Rollback: `sudo rm /etc/modprobe.d/blacklist-spd5118.conf && sudo update-initramfs -u`
- System packages were updated via `apt upgrade` (cosmic-comp,
  cosmic-session, mesa, etc.) — this is a normal package update, not
  something to roll back per-package; use your distro's usual downgrade
  process only if truly necessary.

## If rollback doesn't fix a NEW problem

If you roll back and still see new/different suspend problems, the
override probably wasn't the cause — check `sudo journalctl -b -p err`
and `sudo dmesg | tail -100` for what's actually happening, or run
`/home/jlc/suspend-diag.sh` from a text VT to capture the state before
touching anything else.
