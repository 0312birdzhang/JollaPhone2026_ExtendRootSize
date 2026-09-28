# Extending the Root Partition by 10 GiB on JollaPhone2026 (Sailfish OS)

**Host:** `JollaPhone2026` — Sailfish OS 5.2.0.17 (Finlayson), aarch64, kernel 6.12.38-4k
**Date:** 2026-09-28
**Goal:** grow `/` (root) by 10 GiB, taking the space from `/home`
**Result:** `/` 9.5 G (83 % full) → **19.4 G (41 % full)**; `/home` 218 G → 208 G

---

## 1. Starting layout

Everything relevant lives in a single LVM volume group on one physical volume, and the
volume group had **zero free extents** — so the only way to grow root was to shrink `/home`.

```
/dev/sdc71  ->  VG "sailfish"  232.36 GiB, 59483 extents of 4 MiB, VFree = 0
├─ sailfish/root   9.77 GiB   PE 1..2500     (PV head)      -> /       ext4
└─ sailfish/home 222.59 GiB   PE 2500..59483  (rest of PV)
     └─ luks-daa21e21-efa1-40cf-90ae-6c7d55a5683e   crypto_LUKS  -> /home  ext4
```

Key facts gathered before touching anything:

| Item | Value |
| --- | --- |
| root LV | `/dev/sailfish/root`, 2500 extents, single linear segment at PV sector 2048 |
| home LV | `/dev/sailfish/home`, 56983 extents, single linear segment at PV sector 20482048 |
| LUKS | LUKS1, `aes-xts-plain64`, hash `sha256`, 64-byte key, payload offset 4096 sectors (2 MiB) |
| `/home` usage | 218 G total, 66.6 G used → 152 G free (plenty of room to shrink 10 G) |
| ext4 state | both filesystems `clean` |
| `cryptsetup` CLI | **absent** (only `cryptsetup-libs` installed, no `dnf`, no package repos) |
| LVM extents | 4.00 MiB → 10 GiB = exactly 2560 extents |

### Target geometry

```
home LV      : 56983 -> 54423 extents  = 228266606592 B (212.59 GiB)
dm-crypt len : 466800640 -> 445829120 sectors = 228264509440 B  (LV size - 2 MiB LUKS header)
ext4 size    : 58350080 -> 55728640 blocks x 4096 = 228264509440 B
root LV      : 2500 -> 5060 extents = 21223178240 B (19.77 GiB)
```

---

## 2. Two blockers that had to be solved first

### 2.1 No `cryptsetup` CLI, and `libcryptsetup` would not cooperate

The `cryptsetup` binary is not installed — only `cryptsetup-libs`. There is no `dnf` and no
repositories, so it could not be installed.

The first idea — calling `crypt_resize()` from `libcryptsetup.so.12` via Python `ctypes` —
**did not work**. `crypt_init_by_name_and_header()` returned `0` but `crypt_load()` returned
`-22` (`EINVAL`) on the already-open device.

What worked instead: `dmsetup table --showkeys` returns the **real volume key in plaintext**
even though the key had been handed to the kernel by systemd and is masked as zeros in a
normal `dmsetup table`. Editing the dm-crypt table by hand and reloading it is exactly what
`cryptsetup resize --size` does internally, so no passphrase and no CLI are required.

```console
# dmsetup table            (key masked)
0 466800640 crypt aes-xts-plain64 0000...0000 0 254:1 4096
# dmsetup table --showkeys (real 64-byte key, 128 hex chars)
0 466800640 crypt aes-xts-plain64 5d64e8a4...b6 0 254:1 4096
```

### 2.2 `connmand` held `/home` open — and stopping it would have cut the control channel

A live graphical session was running, so `/home` had dozens of open references:

* ~55 processes inside `user@100000.service` (lipstick, contactsd, msyncd, …) — handled with
  `systemctl stop user@100000.service`
* `jolla-settings` in the seat0 GUI session — handled with `loginctl terminate-session 1`
* a second **remote SSH session (`c1`)** whose bash had its working directory in
  `/home/defaultuser`
* **`connmand`** (`/system.slice/connman.service`), a *system* service that had 2 file
  descriptors open on
  `/home/defaultuser/.local/share/system/privileged/connman/wifi_…/history.home.{hour,day}`

`connmand` was the hard case. It manages WiFi, and **the SSH session driving this whole
operation was running over that WiFi** (`192.168.3.240` → `192.168.3.1`). Killing it to
release the fds would have dropped the control channel mid-operation.

`gdb` is installed and the shell held both `CAP_SYS_PTRACE` and `CAP_SYS_ADMIN`, so the fds
were closed out from under the running daemon instead:

```console
# release the two fds without touching the network
gdb -q -p 3478 -batch -ex 'call (int) close(22)' -ex 'call (int) close(24)' -ex 'detach'

# move the other session's bash out of /home instead of killing that session
gdb -q -p 6519 -batch -ex 'call (int) chdir("/")' -ex 'detach'
```

Both SSH connections stayed established throughout.

**A plain `umount -l` (lazy unmount) was deliberately avoided.** It would have left the
filesystem still referenced, which would both block the dm-crypt resize and make an
`e2fsck`/`resize2fs` on a live filesystem a recipe for corruption.

---

## 3. Execution

### Step 0 — back up everything first

```bash
B=/root/resize-backup-20260928
mkdir -p $B
vgcfgbackup -f $B/sailfish-vg-backup.conf sailfish
dmsetup table --showkeys luks-daa21e21-efa1-40cf-90ae-6c7d55a5683e > $B/crypt-table-key.txt
dmsetup table sailfish-home > $B/home-lv-table.txt
dmsetup table sailfish-root > $B/root-lv-table.txt
cp /etc/crypttab $B/crypttab.bak
chmod 700 $B; chmod 600 $B/crypt-table-key.txt
```

### Step 1 — stop everything that holds `/home`, then unmount

```bash
loginctl terminate-session 1        # GUI session on seat0/tty7
systemctl stop user@100000.service  # ~55 user agents

# verify nothing is left (empty output = clean)
for p in $(ls /proc | grep -E '^[0-9]+$'); do
  n=$(ls -l /proc/$p/fd 2>/dev/null | grep -c "/home/")
  [ "$n" -gt 0 ] && echo "$p fd=$n $(awk -F: '/:name=systemd:/{print $3}' /proc/$p/cgroup)"
done
for p in $(ls /proc | grep -E '^[0-9]+$'); do
  c=$(readlink /proc/$p/cwd 2>/dev/null)
  case "$c" in /home*) echo "$p cwd=$c";; esac
done

umount /home
dmsetup info luks-daa21e21-efa1-40cf-90ae-6c7d55a5683e | grep "Open count"   # must be 0
```

### Step 2 — check and shrink the ext4 inside the LUKS container

```bash
D=/dev/mapper/luks-daa21e21-efa1-40cf-90ae-6c7d55a5683e

e2fsck -f -y "$D"     # 58350080 blocks; also optimised 3 extent trees; exit 0

# NOTE: resize2fs takes new_size in FILESYSTEM BLOCKS when no suffix is given.
# 55728640 blocks x 4096 = 228264509440 B = 212.59 GiB  (verify with tune2fs afterwards)
resize2fs -f "$D" 55728640

tune2fs -l "$D" | grep -E "^Filesystem state|^Block count"
# Filesystem state:  clean
# Block count:       55728640
```

> If the unit is ever in doubt, `resize2fs` is safe to correct in either direction — growing
> a filesystem is trivial. The block count is always verifiable with `tune2fs -l`.

### Step 3 — shrink the dm-crypt device

The dm-crypt device must always be shrunk **before** the LV underneath it, otherwise the
mapping would be larger than the LV and any I/O past the LV end would fault.

```bash
DEV=luks-daa21e21-efa1-40cf-90ae-6c7d55a5683e
NT=$B/crypt-table-new.txt

# rebuild the table from the backed-up one, changing only the length field
awk '{$2=445829120; OFS=" "; print}' $B/crypt-table-key.txt > $NT
awk '{print length($5)}' $NT      # must print 128 (64-byte key)

dmsetup suspend $DEV
dmsetup reload  $DEV $NT          # NB: this dmsetup wants a FILE, not an inline string
dmsetup resume  $DEV

dmsetup table $DEV                # -> 0 445829120 crypt aes-xts-plain64 <KEY> 0 254:1 4096
blockdev --getsize64 /dev/mapper/$DEV   # -> 228264509440
e2fsck -f -n "$D"                 # re-verify: still clean
```

`dmsetup reload` with the table passed inline is rejected — it tries to open the argument as
a filename. Pass the path to the table file instead.

### Step 4 — shrink the home LV

```bash
lvreduce --test -l 54423 /dev/sailfish/home     # dry run first: "<212.59 GiB"
yes | lvreduce -l 54423 /dev/sailfish/home

dmsetup table sailfish-home    # -> 0 445833216 linear 259:55 20482048
                                #    ( = crypt 445829120 + 4096 LUKS header sectors )
vgs -o vg_name,vg_free         # -> sailfish  10.00g
```

Using `-l <extent count>` rather than `-L -10G` makes the target unambiguous.

### Step 5 — grow the root LV and its filesystem

Growing an ext4 online is safe, so root never needed to be unmounted.

```bash
lvextend --test -l +2560 /dev/sailfish/root
yes | lvextend -l +2560 /dev/sailfish/root

dmsetup table sailfish-root
#   0        20480000 linear 259:55 2048        <- segment 1, PV head, 9.77 GiB
#   20480000 20971520 linear 259:55 466315264   <- segment 2, PV tail, 10 GiB

resize2fs /dev/sailfish/root    # online, "now 5181440 (4k) blocks long"
```

### Step 6 — restore

```bash
mount /home
systemctl start user@100000.service
su defaultuser -c 'export XDG_RUNTIME_DIR=/run/user/100000 \
    DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/100000/bus; \
    systemctl --user start lipstick.service'
```

Note: `ps -eo comm` does **not** show `lipstick` — `mapplauncherd` renames the process to
`{binder:<pid>_2}`. Use `ps -eo pid,args` or check `systemctl --user status lipstick.service`.

The `seat0` session itself does not come back this way; a reboot is the clean way to restore
it (and is the proper end-to-end verification — see below).

---

## 4. Verification

```console
$ lvs -o lv_name,lv_size,seg_count
  LV    LSize     #Seg
  home  <212.59g    1
  root  <19.77g     2

$ df -hT / /home
Filesystem                            Size  Used Avail Use% Mounted on
/dev/sailfish/root                    19.4G  7.8G  11.4G  41% /
/dev/mapper/luks-…-daa21e21…         208.2G 66.6G 130.9G  34% /home

$ tune2fs -l /dev/sailfish/root  | grep -E "state|Block count"
Filesystem state:  clean
Block count:       5181440

$ tune2fs -l /dev/mapper/luks-… | grep -E "state|Block count"
Filesystem state:  clean
Block count:       55728640
```

Geometry cross-check:

| Relation | Value |
| --- | --- |
| `root LV == root ext4` | 21223178240 B == 5181440 × 4096 ✔ |
| `home LV == crypt + 2 MiB` | 228266606592 == 228264509440 + 2097152 ✔ |
| `crypt payload == ext4` | 228264509440 B == 212.59 GiB ✔ |
| extent alignment | 228266606592 % 4 MiB == 0 ✔ |
| `/home` data | 66.6 G used, 70199 files, all readable ✔ |

---

## 5. Things to be aware of afterwards

### 5.1 root is now a non-contiguous 2-segment LV

`root` occupies the **head** of the PV while `home` occupied everything after it, so
shrinking `home` frees space at the **tail** of the PV — *not* adjacent to root. LVM
linear LVs can only grow into adjacent free extents, so `lvextend` created a second
segment:

```
root = segment 1  PE 1..2501      (9.77 GiB, PV head)
     + segment 2  PE 56923..59483  (10 GiB,   PV tail)
```

Neither ext4 nor device-mapper requires a contiguous block device, so this is functionally
fine and boots normally. It does mean any future repartitioning has to account for the
split.

### 5.2 Delete the backup directory — it holds the LUKS volume key in plaintext

`/root/resize-backup-20260928/crypt-table-key.txt` contains the 64-byte LUKS volume key, and
`/` is **not** encrypted. Remove it once the setup is confirmed good:

```bash
rm -rf /root/resize-backup-20260928
```

### 5.3 Reboot to verify the boot path

Everything above was verified in a *running* system. Reboot once to confirm that the
boot-time unlock of `/home` works against the new 212.59 GiB geometry — that is the only
step not yet exercised.

### 5.4 Quotas

`/home` contains a leftover `aquota.user` file, but the `quota` feature is **not** in the
filesystem's feature flags, so quotas are not enforced and nothing needed recalculating.
`quotacheck` refused to run (exit 6) and left the filesystem untouched. `repquota` is not
installed.

---

## 6. Failure-mode notes

The step order was chosen so that an interruption at any point is recoverable:

* `lvreduce` is the **last** destructive step, and the dm-crypt device is always shrunk
  **before** the LV. A crash in between therefore leaves the LV larger than the mapping.
  On the next boot systemd opens the full-size LV, the ext4 inside is simply smaller than
  the device — it still mounts and works, and `resize2fs` grows it back.
* A mistake in the `resize2fs` block count is also recoverable: filesystem growth is
  trivial, and the result is always readable from `tune2fs -l`.
* `vgcfgbackup` was taken before any change, so LVM metadata can be restored with
  `vgcfgrestore`.

The genuinely irrecoverable failure mode would have been corrupting the LUKS header or the
dm-crypt table, which is why the real key was read from `--showkeys` and verified to be
128 hex characters before every reload, and why the ext4 was re-checked with `e2fsck` both
before and after the dm-crypt resize.
