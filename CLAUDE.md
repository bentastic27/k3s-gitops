# CLAUDE.md

Notes for working on this repo and the machine it deploys to. See README.md for what's
deployed and how Flux is bootstrapped.

## The node

A single k3s server on a Lenovo ThinkCentre M715q (`192.168.1.10`). It's also
the kid's Ubuntu desktop (Ubuntu 26.04, GNOME), so the minecraft pods share
the box with someone actually playing on it. SSH in as `root@192.168.1.10`.

- CPU: AMD Ryzen 5 PRO 2400GE (Raven Ridge, Vega iGPU via amdgpu)
- RAM: 30 GiB, 8 GiB swap file
- BIOS: M1XKT39A (2019-04-08), the original release
- Secure Boot is on, so kernel lockdown is active. MSR writes (e.g. ZenStates C6
  disable) and hibernation are blocked.

## Random hard freezes (2026-11-02)

The machine hard-locks every so often and needs a power cycle. It froze on 2026-11-02
at ~01:04 (idle, overnight) and again at 18:41:53.

What the logs showed:

- The journal just stops mid-stream. No panic, no oops, no amdgpu errors, no
  shutdown.
- Not suspend: nothing tried to sleep. GNOME's `sleep-inactive-ac-timeout` is 0
  (never); only the screen blanks after 5 min.
- Not memory: ~25% RAM used, swap untouched, low load.
- kdump was armed but never fired, so the kernel never panicked. The lockup
  detector only warns by default, and that warning never made it to disk.

Likely cause: the well-known Raven Ridge idle freeze (and/or an amdgpu hang), made
worse by the 2019 BIOS.

### Changes made on the host

These are host config, not managed by Flux:

| File | What | Effect |
|---|---|---|
| `/etc/sysctl.d/90-lockup-panic.conf` | `kernel.hardlockup_panic=1`, `kernel.softlockup_panic=1` | A detected lockup panics, so kdump writes to `/var/crash`, then `kernel.panic=10` reboots the box on its own |
| `/etc/default/grub.d/90-ryzen-idle.cfg` | appends `idle=nomwait` to the kernel cmdline | Idle with HLT instead of MWAIT. Needs a reboot and `update-grub` |

`/etc/default/grub.bak-2026-11-02` is a backup of the original GRUB defaults (that file
itself wasn't edited).

Why `idle=nomwait` and not `processor.max_cstate`: no cpuidle driver loads on this box
(`/sys/devices/system/cpu/cpuidle/current_driver` is `none`; dmesg says "using mwait
in idle threads"), so `max_cstate` has no effect.

To undo: delete the file, then run `update-grub` for the GRUB one, and reboot.

### If it freezes again

1. Confirm the cmdline has `idle=nomwait`: `cat /proc/cmdline`.
2. Check `/var/crash` for a kdump capture.
3. Read the end of the previous boot without the AppArmor noise:
   `journalctl -b -1 | grep -v 'apparmor="DENIED"' | grep -v kauditd | tail -100`
4. Next step is a BIOS update (`fwupdmgr get-updates`, or Lenovo's USB updater). After
   that, try amdgpu options such as `amdgpu.sg_display=0`.

## Known log noise

`spark-async-sam` (the spark profiler in a minecraft server pod) sends SIGVTALRM, and
the `cri-containerd.apparmor.d` profile denies it, hundreds of times a second (~470k
audit lines in 40 min). It probably didn't cause the freezes, but it buries everything
else in the journal. Turning off spark in the server config would fix it.
