# Ubuntu Swap Expansion Runbook — 8 GiB to 16 GiB

Status: **Implemented and manually validated — 2026-10-15** (automatic activation on the next reboot not yet tested)
Purpose: Safely expand an existing file-backed Ubuntu swap configuration by adding a second 8 GiB swap file, without disabling the original swap or restarting running services.
Scope: **Independent Ubuntu host operations procedure**, not part of Server Health Phase 1 or SMART Disk Health Phase 2.
Related docs: [Operations index](./README.md), [Common commands](./common-commands.md), [Base server setup](../platform/base-server-setup.md).

> **As-built runbook, not an auto-executed script.** Commands below document a successful interactive change to the homelab. Always verify the current machine first. In particular, the direct file-creation method here is for an `ext4` root filesystem; Btrfs, ZFS, CoW, compressed or otherwise special filesystems need different swap-file considerations. Do not blindly rerun creation or formatting commands on an existing active swap file.

## 1. Why expand swap?

At the time of the change, the host had approximately 30 GiB RAM (around 23 GiB available), an existing `/swap.img` of 8 GiB, and 6.7–6.9 GiB swap in use. Glances warned about the high **percentage** of used swap.

`vmstat 1 5` showed no swap-in/swap-out activity in the four live samples (the first line summarizes activity since boot). A search of the available current-boot kernel journal found no OOM-killer records. Those short checks did **not** establish any ongoing memory-pressure problem or guarantee that a past problem never occurred.

The additional 8 GiB was an **optional capacity/headroom preference**, not a demonstrated performance fix. Adding it reduces the percentage reported by Glances without moving pages out of the original swap.

### Verified pre-change snapshot

| Item | Observed value |
| --- | --- |
| Ubuntu host | `piroman-server` |
| Root filesystem | `/`, `ext4` on then-current `/dev/sde2` (device name is not a stable ID) |
| Root free space | About 1.6 TB |
| Existing swap | `/swap.img`, 8 GiB, active |
| Existing swap usage | About 6.9 GiB |
| New swap file | `/swap-extra.img`, 8 GiB |
| Swap target | Two active 8 GiB files; nominal **16 GiB** total |
| Unrelated services | Docker/Compose, Samba, mount configuration, NUT/UPS shutdown left unchanged |

## 2. Read-only preflight

Check available space, root filesystem type, active swap and persistent entries:

```bash
df -h /
findmnt -no TARGET,SOURCE,FSTYPE -T /
swapon --show
free -h
sudo grep -n 'swap' /etc/fstab
sudo test ! -e /swap-extra.img && echo 'New swap path is unused'
```

**Stop** if `/swap-extra.img` already exists; inspect it rather than recreating or formatting it. Confirm that `/` is `ext4`, there is comfortably more than 8 GiB of free space, and the original swap is still active. Do not assume this procedure is suitable for another filesystem.

Optional memory-pressure investigation before deciding whether expansion is needed:

```bash
vmstat 1 5
sudo journalctl -k -b --no-pager |
  grep -Ei 'out of memory|oom-killer|killed process|oom_reaper|memory cgroup out of memory' |
  tail -n 50
```

No matching log lines only means there were no matches in the currently available journal; it is not proof that no incident has ever occurred.

## 3. Create the additional 8 GiB file

Use `dd` rather than creating a sparse file, and `umask 077` for root-only permissions. `conv=excl` refuses to overwrite an existing target; `conv=fsync` flushes the completed output.

```bash
sudo sh -c 'umask 077; dd if=/dev/zero of=/swap-extra.img bs=1M count=8192 status=progress conv=excl,fsync'
```

This deliberately writes 8 GiB of zero bytes to storage and can temporarily cause disk I/O. The observed initial write took around one minute. It does **not** change the existing swap file.

Verify the file **before** formatting it:

```bash
sudo ls -lh /swap-extra.img
sudo stat -c '%n | %s bytes | permissions: %a' /swap-extra.img
sudo du -h /swap-extra.img
swapon --show
```

Expected: a root-owned 8 GiB file, mode `600`, `8589934592` logical bytes, roughly 8 GiB of allocated blocks, and only the original `/swap.img` active. `du` may show slightly more due to filesystem allocation/accounting.

**Actual troubleshooting note:** `oflag=excl` was attempted first and failed with `dd: invalid input flag: ‘excl’`. The validated GNU `dd` form is **`conv=excl,fsync`**. That failed command did not create the destination file.

## 4. Initialize and activate — no swapoff of the original

Only run `mkswap` after verifying that the path is the new, inactive file:

```bash
sudo mkswap /swap-extra.img
sudo blkid /swap-extra.img
```

Check `TYPE="swap"`. The 8 GiB `mkswap` display may report a slightly smaller usable size because of its header. UUID values are machine-specific; do not copy them from an example.

Add the new swap without disabling the original:

```bash
sudo swapon /swap-extra.img
swapon --show
free -h
```

Validated on the host:

```text
NAME            TYPE SIZE USED PRIO
/swap.img       file   8G 6.9G   -1
/swap-extra.img file   8G   0B   -1
```

Both priorities were `-1`. `free -h` displayed `15Gi` total, `6.9Gi` used and `9.1Gi` free due to human-readable rounding/truncation; Glances displayed `16.0G` total. This is consistent with two nominally 8 GiB swap files, not a missing swap file. Existing pages stayed in `/swap.img`; no `swapoff` or service restart was needed.

## 5. Persist across reboot

Back up `/etc/fstab` before editing:

```bash
sudo cp -a /etc/fstab "/etc/fstab.bak-$(date +%Y%m%d-%H%M%S)"
```

Add **exactly one** line for the second file, retaining the existing `/swap.img` entry:

```fstab
/swap-extra.img none swap sw 0 0
```

The following append command was used on the host and avoids adding a duplicate *active* line if it already exists:

```bash
grep -qE '^[[:space:]]*/swap-extra\.img[[:space:]]' /etc/fstab ||
  printf '%s\n' '/swap-extra.img none swap sw 0 0' | sudo tee -a /etc/fstab
```

Review the resulting file rather than assuming that absence of duplicate active lines proves the rest of `fstab` is correct:

```bash
sudo findmnt --verify --verbose
grep -n swap /etc/fstab
sudo systemctl daemon-reload
swapon --show
systemctl list-units --type=swap --all
```

**Actual validation:** `findmnt` returned **0 parse errors, 0 errors, 14 warnings**. Warnings concerned duplicate `none` swap targets, swap files being regular files, the existing `ntfs-3g`/FUSE entries, and the need for `systemctl daemon-reload` before it was executed. They did not warrant modifying the five NTFS mounts; assess warnings against your own system rather than ignoring all warnings categorically.

After `daemon-reload`, systemd reported both units as `loaded active active`:

```text
swap.img.swap
swap\x2dextra.img.swap
```

**Persistence caveat:** the `fstab` entry and systemd units were verified without reboot. Re-activation after a future ordinary boot is **configured but not yet empirically verified**. Check `swapon --show` and `systemctl list-units --type=swap --all` after the next planned reboot; do not reboot solely for this change.

## 6. Post-change acceptance and monitoring

- Both swap files are listed and active, with an aggregate nominal 16 GiB.
- The original swap file and its existing data are undisturbed.
- `/etc/fstab` includes both entries and has a dated backup.
- systemd recognizes both swap units as active after `daemon-reload`.
- Glances SWAP changed from roughly 85–87% to about 43.4% with 6.9 GiB still used; historical `MEMSWAP` alerts can remain in its recent-alert list.
- No change was made to Docker/Compose, the Homepage/Glances/PeaNUT configuration, NTFS/Samba, drive mount points, or host-native NUT UPS shutdown.

More swap provides reserve capacity, **not** faster RAM or proof of reduced memory pressure. Continue investigating if swapping becomes sustained (`vmstat` `si`/`so`), OOM logs appear, or applications slow down.

## 7. Safe rollback (only if deliberately requested)

Rollback is **not** needed for normal operation. Removing swap can require reading swapped pages back into RAM; if the extra file has pages in use, `swapoff` can cause memory pressure, stall or fail. Plan a maintenance window, check memory headroom, and do not force this while the host is under pressure.

1. Check `free -h`, `swapon --show`, and ensure the original `/swap.img` is healthy and active.
2. If safe, run `sudo swapoff /swap-extra.img` **only** (never `swapoff -a`); confirm successful exit and that only `/swap.img` remains active.
3. Back up the current `/etc/fstab` again; remove **only** the exact `/swap-extra.img none swap sw 0 0` line using `sudoedit /etc/fstab`. Do not restore an old backup if unrelated `fstab` entries have changed since.
4. Run `sudo findmnt --verify --verbose`, `sudo systemctl daemon-reload`, and `swapon --show`.
5. Only after confirming the extra file is not active **and** is no longer referenced in `fstab`, optionally remove it with `sudo rm -- /swap-extra.img`. Verify the filename before executing.

If `swapoff` fails or would cause memory pressure, **stop**: retain the extra file and its persistence entry; do not delete an active swap file.

## 8. Boundaries

This runbook changes only Linux host swap-file configuration. It does not involve repartitioning, filesystem conversion, tuning `vm.swappiness`, changing Glances warning thresholds, upgrading a monitoring stack, restarting Docker, or modifying UPS shutdown behavior. Swap files can also have special implications for hibernation; hibernation/resume configuration was not in scope and was not tested.
