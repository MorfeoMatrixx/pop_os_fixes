# Fix: spd5118 Suspend/Resume Crash — Pop!_OS on Lenovo LOQ 15IAX9

## System
- **Laptop:** Lenovo LOQ 15IAX9 (model 83GS)
- **GPU:** NVIDIA RTX 4050, hybrid mode with Intel iGPU
- **OS:** Pop!_OS, dual-boot with Windows 11
- **RAM:** DDR5 (with SPD hub sensor chip)

## Symptom
On suspend/resume (sleep and wake), the system either hung or logged repeated
kernel errors on the console:

```
spd5118 14-0050: Failed to write b = 0: -6
spd5118 14-0050: PM: dpm_run_callback(): spd5118_resume [spd5118] returns -6
spd5118 14-0050: PM: failed to resume async: error -6
Freezing user space processes failed after 20.000 seconds (4 tasks refusing to freeze, wq_busy=0)
```

## Root Cause
The `spd5118` kernel module is a driver for the SPD hub chip on DDR5 RAM
(reads temperature/voltage/manufacturer info from the memory sticks). On this
hardware, the chip fails to resume properly after suspend, causing the kernel
resume callback to error out (`-6` = ENXIO).

This is **unrelated** to:
- NVIDIA/Intel hybrid GPU switching
- `system76-power` / GPU mode toggling
- PRIME offloading

It's purely a RAM sensor/telemetry driver — safe to disable entirely, no
functional impact on memory operation.

## Fix Applied

### 1. Blacklist the module
```bash
echo "blacklist spd5118" | sudo tee /etc/modprobe.d/blacklist-spd5118.conf
sudo update-initramfs -u
sudo reboot
```

### 2. Verify it's not loading
```bash
lsmod | grep spd5118
cat /etc/modprobe.d/blacklist-spd5118.conf
```
- No output from `lsmod` = module not loaded = fix working.
- File should contain: `blacklist spd5118`

### 3. (Not needed in this case, but a stronger fallback if blacklist alone fails)
If the module still loads despite the blacklist (e.g. pulled in as a
dependency by another module), force it off:
```bash
echo "install spd5118 /bin/false" | sudo tee -a /etc/modprobe.d/blacklist-spd5118.conf
sudo update-initramfs -c -k all
sudo reboot
```

## Result
- **Before fix:** system froze on wake ("Freezing user space processes failed"),
  requiring hard reset. Turned out to be unrelated crashed file manager
  instances stacking up over multiple sleep cycles, compounded by the
  spd5118 resume errors.
- **After blacklist:** module confirmed unloaded (`lsmod` empty). Suspend/resume
  works normally, no more hangs.
- **Residual `dmesg` messages:** Even with the module blacklisted, the SPD hub
  chip itself still attempts to communicate on resume at the hardware level,
  and the kernel logs a similar-looking error to console. This is now purely
  cosmetic/informational — the driver is not loaded, nothing hangs, no
  functional impact.

## Alternative Fixes (Not Needed Here, Kept for Reference)

These were considered as follow-up options in case the module blacklist alone
hadn't resolved the hang. Not required in this case, but useful if a similar
issue resurfaces after a reinstall.

### Alternative fix: disable deep suspend
Some laptops handle `s2idle` (a lighter suspend state) more reliably than
`deep` sleep, which is more prone to device resume failures like this one.

```bash
# Check current sleep state
cat /sys/power/mem_sleep

# Force s2idle instead of deep sleep
sudo kernelstub -a "mem_sleep_default=s2idle"
sudo reboot
```

### Alternative fix: NVIDIA resume/freeze issues
If "tasks refusing to freeze" persists and is related to the NVIDIA driver
(rather than crashed background processes, as it was in this case), these
kernel parameters and services can help NVIDIA handle suspend/resume more
gracefully:

```bash
sudo kernelstub -a "nvidia.NVreg_PreserveVideoMemoryAllocations=1"
sudo kernelstub -a "nvidia.NVreg_TemporaryFilePath=/var/tmp"

sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service

sudo reboot
```

## Optional: Reduce console log noise
If the residual `dmesg`-style messages on screen during resume are
undesired, you can lower the console log verbosity:
```bash
sudo kernelstub -a "loglevel=3"
sudo reboot
```
Note: this only affects what's printed to the console/screen. It does
**not** affect what's captured in logs — full detail remains available for
troubleshooting via:
```bash
sudo dmesg
sudo dmesg --level=err,warn
sudo journalctl -b -p err
```
This is recommended if you ever need to debug a *different* hardware issue
later — the errors are still there in the logs, just not on-screen.

## Confirmed Non-Impacts
- Does **not** affect Windows 11 dual-boot (initramfs update only touches
  the Pop!_OS `/boot` partition, not the bootloader or Windows partitions).
- Does **not** affect hybrid GPU switching / `gpu-mode` TUI / `system76-power`.

## Quick Reference — All Commands
```bash
# Apply the fix
echo "blacklist spd5118" | sudo tee /etc/modprobe.d/blacklist-spd5118.conf
sudo update-initramfs -u
sudo reboot

# Verify
lsmod | grep spd5118
cat /etc/modprobe.d/blacklist-spd5118.conf

# Fallback if module still loads
echo "install spd5118 /bin/false" | sudo tee -a /etc/modprobe.d/blacklist-spd5118.conf
sudo update-initramfs -c -k all
sudo reboot

# Alternative: force s2idle instead of deep sleep (if needed)
sudo kernelstub -a "mem_sleep_default=s2idle"

# Alternative: NVIDIA suspend/resume tuning (if freeze issue is NVIDIA-related)
sudo kernelstub -a "nvidia.NVreg_PreserveVideoMemoryAllocations=1"
sudo kernelstub -a "nvidia.NVreg_TemporaryFilePath=/var/tmp"
sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service

# Optional: quiet console log level
sudo kernelstub -a "loglevel=3"

# Troubleshooting other hw issues later
sudo dmesg --level=err,warn
sudo journalctl -b -p err
```
