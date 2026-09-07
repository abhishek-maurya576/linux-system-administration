# Level 6 — Storage & Filesystems

> **Goal:** Understand how Linux manages disks, partitions, and filesystems. Learn to partition drives, create filesystems, mount storage, configure persistent mounts, manage LVM, and set up basic RAID — the foundation of every production server.

> **Lab Environment:** VirtualBox VM (required — you need real block devices to practice safely)

> **Prerequisite:** Levels 1–5 completed. You should be comfortable with `sudo`, file permissions, and systemd basics.

---

## 6.1 — How Linux Sees Storage

### Minimum Theory

In Linux, **everything is a file** — including hard drives, SSDs, USB sticks, and partitions. They appear as special files under `/dev/` (device directory).

**📌 Important — Storage naming conventions:**

| Device Type | Naming Pattern | Example |
|---|---|---|
| SATA / SSD / Virtual disk | `/dev/sd[a-z]` | `/dev/sda` (first disk), `/dev/sdb` (second disk) |
| NVMe SSD | `/dev/nvme[0-9]n[1-9]` | `/dev/nvme0n1` (first NVMe) |
| Virtual (VirtIO) | `/dev/vd[a-z]` | `/dev/vda` (first VirtIO disk) |
| Partitions | device + number | `/dev/sda1` (first partition on sda) |

### Why It Matters in Production

- Every server needs properly partitioned and mounted storage
- Database servers need dedicated fast storage for data directories
- Log partitions prevent runaway logs from filling the root filesystem
- Admins add, resize, and replace disks regularly without downtime

### Checking Your Current Storage

```bash
# List all block devices (disks and partitions)
lsblk
```

**Expected output:**
```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0    25G  0 disk
├─sda1   8:1    0     1M  0 part
├─sda2   8:2    0   1.8G  0 part /boot
└─sda3   8:3    0  23.2G  0 part /
sdb      8:16   0    10G  0 disk
```

**Reading the output:**
- `sda` = first disk (25 GB), has 3 partitions
- `sda1` = BIOS boot partition (1 MB)
- `sda2` = boot partition mounted at `/boot`
- `sda3` = root partition mounted at `/`
- `sdb` = second disk (10 GB), no partitions yet — this is what we'll practice on

```bash
# More detailed view with filesystem info
lsblk -f
```

**Expected output:**
```
NAME   FSTYPE FSVER LABEL UUID                                 MOUNTPOINTS
sda
├─sda1
├─sda2 ext4   1.0         a1b2c3d4-e5f6-7890-abcd-ef1234567890 /boot
└─sda3 ext4   1.0         f0e1d2c3-b4a5-6789-0fed-cba987654321 /
sdb
```

This shows filesystem types, labels, and UUIDs.

```bash
# Show disk sizes and usage
df -h
```

**Expected output:**
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        23G   4.2G   18G  19% /
/dev/sda2       1.8G   120M  1.5G   8% /boot
tmpfs           994M     0  994M   0% /dev/shm
```

```bash
# Show all disks with sizes (including unmounted)
sudo fdisk -l
```

**📌 Important — Key difference:**
- `lsblk` = shows block devices and their relationships (tree view)
- `df -h` = shows mounted filesystems and their usage
- `fdisk -l` = shows detailed disk geometry and partition tables

**💡 Admin Tip:** `df -h` only shows **mounted** filesystems. If you added a new disk but don't see it in `df`, it's because you haven't mounted it yet. Use `lsblk` to find unmounted disks.

**🎯 Interview Point:** "How would you check disk space on a Linux server?"
> `df -h` for mounted filesystem usage. `lsblk` for all block devices. `du -sh /path` for directory-level usage. `fdisk -l` for raw disk information.

---

## 6.2 — Partition Tables: MBR vs GPT

### Minimum Theory

Before you can use a disk, it needs a **partition table** — a data structure that defines how the disk is divided into partitions.

| Feature | MBR (Master Boot Record) | GPT (GUID Partition Table) |
|---|---|---|
| Max disk size | 2 TB | 9.4 ZB (practically unlimited) |
| Max partitions | 4 primary (or 3 primary + 1 extended with logical) | 128 partitions |
| Boot mode | BIOS (legacy) | UEFI (modern) |
| Data integrity | No checksums | CRC32 checksums + backup table |
| Age | 1983 | 2000s |

**✅ Best Practice:** Always use **GPT** for new installations unless you have a specific legacy BIOS requirement. Most modern servers use UEFI + GPT.

### MBR Partition Types

If you encounter MBR disks (legacy systems):

| Type | Description |
|---|---|
| **Primary** | Up to 4 per disk. Can be bootable. |
| **Extended** | A container partition that holds logical partitions. Only 1 per disk. |
| **Logical** | Partitions inside the extended partition. Unlimited (practically). |

> **Preview:** On modern GPT disks, every partition is just a partition — no primary/extended/logical distinction. Much simpler.

**🎯 Interview Point:** "What is the difference between MBR and GPT?"
> MBR supports up to 2 TB disks and 4 primary partitions, uses BIOS. GPT supports disks larger than 2 TB, up to 128 partitions, uses UEFI, and includes integrity checksums and a backup partition table.

---

## 6.3 — Partitioning with `fdisk` (MBR) and `gdisk` / `parted` (GPT)

### Preparing a Practice Disk in VirtualBox

Before practicing, add a second virtual disk to your VM:

1. **Shut down** your VM
2. In VirtualBox → Select your VM → **Settings** → **Storage**
3. Click the controller (SATA), then the "Add Hard Disk" icon
4. **Create** a new VDI disk, 10 GB, dynamically allocated
5. Start your VM
6. Verify the new disk:

```bash
lsblk
```

You should see a new device like `/dev/sdb` with 10 GB and no partitions.

**🔴 DANGER:** All partitioning commands in this section MUST be practiced on the **second disk** (`/dev/sdb`), NOT on your system disk (`/dev/sda`). Partitioning the wrong disk will destroy your OS.

---

### `fdisk` — Interactive Partition Editor (MBR-focused)

```bash
sudo fdisk /dev/sdb
```

This opens an interactive prompt. Key commands inside `fdisk`:

| Command | Action |
|---|---|
| `p` | Print the current partition table |
| `n` | Create a new partition |
| `d` | Delete a partition |
| `t` | Change partition type |
| `l` | List known partition types |
| `w` | Write changes and exit (**destructive — only do this when ready**) |
| `q` | Quit without saving changes |

**Step-by-step: Create two partitions on `/dev/sdb`:**

```bash
sudo fdisk /dev/sdb
```

Inside `fdisk`:
```
Command (m for help): n       # New partition
Partition type: p              # Primary
Partition number: 1            # First partition
First sector: (press Enter for default)
Last sector: +5G               # 5 GB partition

Command (m for help): n       # New partition
Partition type: p              # Primary
Partition number: 2            # Second partition
First sector: (press Enter for default)
Last sector: (press Enter — use remaining space)

Command (m for help): p       # Print — verify your work

Command (m for help): w       # Write changes and exit
```

**Verify:**
```bash
lsblk /dev/sdb
```

**Expected output:**
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb      8:16   0   10G  0 disk
├─sdb1   8:17   0    5G  0 part
└─sdb2   8:18   0    5G  0 part
```

**⚠️ Common Mistake:** Pressing `w` before verifying with `p`. Always print first, then write. If you made a mistake, press `q` to quit without saving.

---

### `parted` — Partition Editor (MBR and GPT)

`parted` is more powerful than `fdisk` and supports both MBR and GPT. It also supports scripting.

**⚠️ Common Mistake:** Unlike `fdisk`, `parted` applies changes immediately — there is no "write" step. Be extra careful.

```bash
sudo parted /dev/sdb
```

Inside `parted`:

| Command | Action |
|---|---|
| `print` | Show current partition table |
| `mklabel gpt` | Create a GPT partition table (erases all data!) |
| `mklabel msdos` | Create an MBR partition table |
| `mkpart primary ext4 1MiB 5GiB` | Create a partition |
| `rm 1` | Remove partition 1 |
| `quit` | Exit |

**Step-by-step: Create a GPT partition table and two partitions:**

```bash
# First, remove old partitions (from fdisk practice above)
sudo parted /dev/sdb

(parted) mklabel gpt                           # Create GPT table (erases everything)
(parted) mkpart primary ext4 1MiB 5GiB         # Partition 1: 5 GB
(parted) mkpart primary ext4 5GiB 100%         # Partition 2: remaining space
(parted) print                                  # Verify
(parted) quit
```

**Non-interactive mode (scriptable):**

```bash
# Create GPT table and partitions in one go
sudo parted -s /dev/sdb mklabel gpt
sudo parted -s /dev/sdb mkpart primary ext4 1MiB 5GiB
sudo parted -s /dev/sdb mkpart primary ext4 5GiB 100%
```

**💡 Admin Tip:** Use `parted -s` (script mode) in automation. `fdisk` is better for interactive, manual work. `parted` is better for scripts and GPT disks.

**Verify with both tools:**
```bash
lsblk /dev/sdb
sudo parted /dev/sdb print
```

---

### `gdisk` — GPT-specific Partition Editor

`gdisk` is the GPT equivalent of `fdisk`. Same interactive interface, but designed specifically for GPT.

```bash
sudo gdisk /dev/sdb
```

Commands are similar to `fdisk`: `n` (new), `d` (delete), `p` (print), `w` (write), `q` (quit).

**✅ Best Practice — Which tool to use:**

| Situation | Recommended Tool |
|---|---|
| Quick interactive partitioning (MBR) | `fdisk` |
| GPT partitioning (interactive) | `gdisk` or `parted` |
| Scripted / automated partitioning | `parted -s` |
| Viewing partitions | `lsblk`, `parted print` |

---

## 6.4 — Filesystems

### What is a Filesystem?

A partition is just raw space. A **filesystem** organizes that space so the OS can store and retrieve files. Creating a filesystem is called **formatting**.

**📌 Important — Common Linux filesystems:**

| Filesystem | Use Case | Max File Size | Max Volume Size | Features |
|---|---|---|---|---|
| **ext4** | Default on most distros | 16 TB | 1 EB | Journaling, mature, reliable |
| **XFS** | Default on RHEL/Rocky | 8 EB | 8 EB | Excellent for large files, can only grow (not shrink) |
| **Btrfs** | SUSE, some Ubuntu | 16 EB | 16 EB | Snapshots, compression, CoW |
| **FAT32/vFAT** | USB drives, UEFI boot | 4 GB | 2 TB | Cross-platform compatibility |
| **swap** | Swap space | N/A | N/A | Virtual memory |

**🎯 Interview Point:** "What filesystem would you use for a database server?"
> **XFS** for large files and high throughput (default on RHEL). **ext4** for general purpose. Both support journaling which prevents data corruption on power loss.

### What is Journaling?

A **journal** is a log of changes about to be made. If the system crashes during a write:
- **Without journaling:** Filesystem check (`fsck`) must scan the entire disk — can take hours on large disks
- **With journaling:** Only replay the journal — recovery in seconds

**✅ Best Practice:** Always use a journaling filesystem (ext4 or XFS) on production servers.

---

### Creating Filesystems with `mkfs`

**🔴 DANGER:** `mkfs` erases ALL data on the partition. Triple-check the device path before running.

```bash
# Create ext4 filesystem on partition 1
sudo mkfs.ext4 /dev/sdb1
```

**Expected output:**
```
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 1310720 4k blocks and 327680 inodes
Filesystem UUID: a1b2c3d4-5678-9abc-def0-123456789abc
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

```bash
# Create XFS filesystem on partition 2
sudo mkfs.xfs /dev/sdb2
```

**Other filesystem commands:**

```bash
# Create ext4 with a label
sudo mkfs.ext4 -L "data_volume" /dev/sdb1

# Create XFS with a label
sudo mkfs.xfs -L "backup_volume" /dev/sdb2

# Create FAT32 (for USB drives)
sudo mkfs.vfat /dev/sdb1
```

**Verify the filesystem:**
```bash
lsblk -f /dev/sdb
```

**Expected output:**
```
NAME   FSTYPE FSVER LABEL         UUID                                 MOUNTPOINTS
sdb
├─sdb1 ext4   1.0   data_volume   a1b2c3d4-5678-9abc-def0-123456789abc
└─sdb2 xfs          backup_volume e1f2a3b4-5678-9abc-def0-abcdef123456
```

**⚠️ Common Mistake:** Running `mkfs` on the wrong partition (e.g., your root filesystem). Always verify with `lsblk` first.

---

## 6.5 — Mounting and Unmounting Filesystems

### What is Mounting?

**Mounting** attaches a filesystem to a directory in the file tree, making its contents accessible. The directory is called a **mount point**.

```
Before mounting:                After mounting:
/                               /
├── home/                       ├── home/
├── mnt/                        ├── mnt/
│   └── data/  (empty)          │   └── data/  → /dev/sdb1 contents visible here
└── ...                         └── ...
```

### Mounting Manually

```bash
# Step 1: Create a mount point (just a regular directory)
sudo mkdir -p /mnt/data

# Step 2: Mount the filesystem
sudo mount /dev/sdb1 /mnt/data

# Step 3: Verify
df -h /mnt/data
lsblk
```

**Expected output of `df -h /mnt/data`:**
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       4.9G   24K  4.6G   1% /mnt/data
```

```bash
# Now you can use it like any directory
cd /mnt/data
sudo touch testfile.txt
ls -la
```

### Mounting Options

```bash
# Mount as read-only
sudo mount -o ro /dev/sdb1 /mnt/data

# Mount with specific options
sudo mount -o rw,noexec,nosuid /dev/sdb1 /mnt/data
```

**Common mount options:**

| Option | Meaning |
|---|---|
| `rw` | Read-write (default) |
| `ro` | Read-only |
| `noexec` | Prevent execution of binaries on this filesystem |
| `nosuid` | Ignore SUID/SGID bits (security) |
| `nodev` | Ignore device files |
| `noatime` | Don't update access time on reads (improves performance) |
| `defaults` | Shorthand for `rw,suid,dev,exec,auto,nouser,async` |

**✅ Best Practice — Security-hardened mount options:**

```bash
# For /tmp — prevent execution and SUID exploits
sudo mount -o noexec,nosuid,nodev /dev/sdb1 /mnt/tmp_data

# For data partitions — disable access time updates for performance
sudo mount -o noatime /dev/sdb1 /mnt/data
```

### Unmounting

```bash
sudo umount /mnt/data
```

> Note: the command is `umount` (no 'n'), not 'unmount'.

**⚠️ Common Mistake:** Trying to unmount while inside the directory or while a process is using it:

```bash
cd /mnt/data
sudo umount /mnt/data    # ERROR: target is busy
```

**Fix:**
```bash
# Option 1: Leave the directory first
cd /
sudo umount /mnt/data

# Option 2: Find what's using the mount
sudo lsof /mnt/data

# Option 3: Find which processes are using it
sudo fuser -mv /mnt/data

# Option 4: Force unmount (last resort)
sudo umount -l /mnt/data    # Lazy unmount — detaches from tree, cleans up when not busy
```

**💡 Admin Tip:** `lsof /mnt/data` shows open files on the mount point. `fuser -mv /mnt/data` shows the PIDs of processes using it. Kill or stop them before unmounting.

---

## 6.6 — Persistent Mounts with `/etc/fstab`

### The Problem

Manual mounts (`sudo mount ...`) are lost after reboot. For permanent mounts, you need `/etc/fstab`.

### Understanding `/etc/fstab`

```bash
cat /etc/fstab
```

**Example `/etc/fstab` content:**
```
# <filesystem>                              <mountpoint>  <type>  <options>       <dump> <pass>
UUID=f0e1d2c3-b4a5-6789-0fed-cba987654321  /             ext4    defaults        0      1
UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890  /boot         ext4    defaults        0      2
UUID=1234abcd-5678-efgh-ijkl-mnopqrstuvwx  none          swap    sw              0      0
```

**📌 Important — Six fields of `/etc/fstab`:**

| Field | Purpose | Example |
|---|---|---|
| 1. **Filesystem** | What to mount (UUID or device path) | `UUID=a1b2c3d4...` or `/dev/sdb1` |
| 2. **Mount point** | Where to mount | `/mnt/data` |
| 3. **Type** | Filesystem type | `ext4`, `xfs`, `swap` |
| 4. **Options** | Mount options | `defaults`, `noatime,noexec` |
| 5. **Dump** | Backup flag (0 = skip, 1 = backup). Almost always 0. | `0` |
| 6. **Pass** | fsck order (0 = skip, 1 = root first, 2 = other). | `2` |

### Using UUIDs Instead of Device Names

**🔴 DANGER:** Never use `/dev/sdb1` in fstab on production servers. Device names can change if you add/remove disks (today's `/dev/sdb` could become `/dev/sdc` after adding a new disk). **Always use UUIDs.**

```bash
# Find the UUID of a partition
sudo blkid /dev/sdb1
```

**Expected output:**
```
/dev/sdb1: LABEL="data_volume" UUID="a1b2c3d4-5678-9abc-def0-123456789abc" TYPE="ext4"
```

```bash
# Or list all UUIDs
sudo blkid
```

### Adding a Permanent Mount Entry

**Step-by-step process:**

```bash
# Step 1: Get the UUID
sudo blkid /dev/sdb1
# Output: UUID="a1b2c3d4-5678-9abc-def0-123456789abc"

# Step 2: Create the mount point
sudo mkdir -p /mnt/data

# Step 3: BACKUP fstab first (critical!)
sudo cp /etc/fstab /etc/fstab.backup

# Step 4: Add the entry to fstab
echo 'UUID=a1b2c3d4-5678-9abc-def0-123456789abc  /mnt/data  ext4  defaults  0  2' | sudo tee -a /etc/fstab

# Step 5: Test BEFORE rebooting
sudo mount -a
```

**📌 Important:** `sudo mount -a` mounts everything in fstab that isn't already mounted. If there's an error in your fstab entry, this command will tell you — much safer than rebooting and finding out your system can't boot.

```bash
# Step 6: Verify
df -h /mnt/data
mount | grep sdb1
```

### Testing fstab Changes Safely

```bash
# Dry run — check for errors without mounting
sudo findmnt --verify
```

**🔴 DANGER — A broken fstab can prevent your system from booting!**

If you make a typo in `/etc/fstab` and reboot, your system may drop to an emergency shell. Always:

1. **Backup** fstab before editing: `sudo cp /etc/fstab /etc/fstab.backup`
2. **Test** with `sudo mount -a` before rebooting
3. **Verify** with `sudo findmnt --verify`

**🔧 Troubleshooting — System won't boot after fstab change:**

If your system drops to emergency mode after a bad fstab edit:
```bash
# In emergency mode, root filesystem may be read-only
# Remount root as read-write
mount -o remount,rw /

# Fix the fstab error
vi /etc/fstab
# Or restore the backup
cp /etc/fstab.backup /etc/fstab

# Reboot
reboot
```

**🎯 Interview Point:** "Why should you use UUID instead of device names in /etc/fstab?"
> Device names like `/dev/sdb1` can change when disks are added or removed (kernel assigns them based on detection order). UUIDs are unique identifiers permanently assigned to a filesystem and never change, ensuring mounts are reliable.

---

## 6.7 — Swap Space

### What is Swap?

**Swap** is disk space used as emergency extension of RAM. When physical RAM is full, the kernel moves less-used memory pages to swap.

**📌 Important:**
- Swap is **much slower** than RAM (even on SSD)
- It prevents out-of-memory (OOM) kills when RAM is temporarily full
- Production servers should have swap, but shouldn't rely on it

### Checking Current Swap

```bash
# Show swap usage
free -h

# Show swap details
swapon --show

# Show swap in /proc
cat /proc/swaps
```

### Creating a Swap Partition

```bash
# Step 1: Create a partition (using fdisk or parted)
# Assume /dev/sdb2 is available

# Step 2: Format as swap
sudo mkswap /dev/sdb2

# Step 3: Enable the swap
sudo swapon /dev/sdb2

# Step 4: Verify
free -h
swapon --show
```

### Creating a Swap File (Alternative — No Extra Partition Needed)

```bash
# Step 1: Create a 2 GB file filled with zeros
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048

# Step 2: Set correct permissions (MUST be 600)
sudo chmod 600 /swapfile

# Step 3: Format as swap
sudo mkswap /swapfile

# Step 4: Enable
sudo swapon /swapfile

# Step 5: Verify
free -h
```

### Making Swap Permanent

Add to `/etc/fstab`:
```bash
# For swap partition
echo 'UUID=<swap-uuid>  none  swap  sw  0  0' | sudo tee -a /etc/fstab

# For swap file
echo '/swapfile  none  swap  sw  0  0' | sudo tee -a /etc/fstab
```

### How Much Swap?

| RAM | Recommended Swap (Server) |
|---|---|
| < 2 GB | 2× RAM |
| 2–8 GB | Equal to RAM |
| 8–64 GB | At least 4 GB |
| > 64 GB | At least 4 GB (or none, depending on workload) |

### Swappiness

```bash
# Check current swappiness (0-100, default is 60)
cat /proc/sys/vm/swappiness

# Temporarily change (takes effect immediately)
sudo sysctl vm.swappiness=10

# Permanently change
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

- **Swappiness 60** (default): Kernel uses swap moderately
- **Swappiness 10**: Kernel avoids swap unless RAM is nearly full (better for servers)
- **Swappiness 0**: Disable swap almost entirely (not recommended)

**✅ Best Practice:** Set swappiness to **10** on production servers. You want the kernel to prefer RAM and only use swap as a last resort.

---

## 6.8 — LVM (Logical Volume Manager)

### Why LVM?

Traditional partitions have a major limitation: **you can't easily resize them**. If `/home` runs out of space, you can't just "borrow" space from `/var` without destructive repartitioning.

**LVM** solves this by adding a flexible layer between physical disks and filesystems.

### LVM Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Logical Volumes (LV)                   │
│     lv_root (20G)       lv_home (30G)    lv_data (50G)   │  ← Filesystems live here
├──────────────────────────────────────────────────────────┤
│                    Volume Group (VG)                       │
│                      vg_server (100G)                      │  ← Pool of space
├──────────────────────────────────────────────────────────┤
│                  Physical Volumes (PV)                     │
│        /dev/sdb (50G)           /dev/sdc (50G)            │  ← Actual disks
└──────────────────────────────────────────────────────────┘
```

**📌 Important — Three layers of LVM:**

| Layer | Abbreviation | What It Is | Analogy |
|---|---|---|---|
| **Physical Volume (PV)** | PV | A disk or partition prepared for LVM | Raw land |
| **Volume Group (VG)** | VG | A pool combining one or more PVs | A neighborhood (combines land plots) |
| **Logical Volume (LV)** | LV | A virtual partition carved from the VG | A house lot (can be resized) |

### Why It Matters in Production

- **Resize** filesystems without downtime
- **Combine** multiple disks into one large volume group
- **Snapshots** for backups before risky operations
- **Migrate** data between physical disks transparently
- Standard on **RHEL, Rocky, CentOS** default installations

### LVM Setup — Step by Step

**Lab setup: Add two 5 GB disks to your VM** (same process as adding `sdb` earlier — add `sdc` and `sdd` in VirtualBox settings).

```bash
# Verify new disks are visible
lsblk
# You should see /dev/sdc (5G) and /dev/sdd (5G)
```

#### Step 1: Create Physical Volumes (PV)

```bash
# Initialize disks for LVM
sudo pvcreate /dev/sdc /dev/sdd
```

**Expected output:**
```
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
```

```bash
# Verify PVs
sudo pvs          # Short summary
sudo pvdisplay    # Detailed info
```

#### Step 2: Create a Volume Group (VG)

```bash
# Create a volume group named "vg_data" from both PVs
sudo vgcreate vg_data /dev/sdc /dev/sdd
```

**Expected output:**
```
  Volume group "vg_data" successfully created
```

```bash
# Verify VG
sudo vgs          # Short summary
sudo vgdisplay    # Detailed info
```

**Expected `sudo vgs` output:**
```
  VG      #PV #LV #SN Attr   VSize  VFree
  vg_data   2   0   0 wz--n- 9.99g  9.99g
```

Two PVs, no LVs yet, ~10 GB total free space.

#### Step 3: Create Logical Volumes (LV)

```bash
# Create a 6 GB logical volume named "lv_projects"
sudo lvcreate -L 6G -n lv_projects vg_data

# Create a logical volume using remaining space
sudo lvcreate -l 100%FREE -n lv_backups vg_data
```

**Key options:**
- `-L 6G` = exact size (6 GB)
- `-l 100%FREE` = use 100% of remaining free space in the VG
- `-n lv_projects` = name of the LV

```bash
# Verify LVs
sudo lvs          # Short summary
sudo lvdisplay    # Detailed info
```

**Expected `sudo lvs` output:**
```
  LV          VG      Attr       LSize  Pool Origin Data%  Meta%
  lv_backups  vg_data -wi-a-----  3.99g
  lv_projects vg_data -wi-a-----  6.00g
```

#### Step 4: Create Filesystems and Mount

```bash
# Create filesystems on the logical volumes
sudo mkfs.ext4 /dev/vg_data/lv_projects
sudo mkfs.xfs /dev/vg_data/lv_backups

# Create mount points
sudo mkdir -p /mnt/projects /mnt/backups

# Mount
sudo mount /dev/vg_data/lv_projects /mnt/projects
sudo mount /dev/vg_data/lv_backups /mnt/backups

# Verify
df -h /mnt/projects /mnt/backups
```

**Expected output:**
```
Filesystem                      Size  Used Avail Use% Mounted on
/dev/mapper/vg_data-lv_projects 5.9G   24K  5.6G   1% /mnt/projects
/dev/mapper/vg_data-lv_backups  4.0G   33M  4.0G   1% /mnt/backups
```

> **Note:** LVM volumes appear as `/dev/mapper/vg_name-lv_name` or `/dev/vg_name/lv_name`. Both paths work.

**Add to fstab for persistence:**
```bash
# Get UUIDs
sudo blkid /dev/vg_data/lv_projects
sudo blkid /dev/vg_data/lv_backups

# Backup fstab
sudo cp /etc/fstab /etc/fstab.backup

# Add entries
echo '/dev/vg_data/lv_projects  /mnt/projects  ext4  defaults  0  2' | sudo tee -a /etc/fstab
echo '/dev/vg_data/lv_backups   /mnt/backups   xfs   defaults  0  2' | sudo tee -a /etc/fstab

# Test
sudo mount -a
```

**💡 Admin Tip:** For LVM volumes in fstab, you can use either UUID or the `/dev/vg_name/lv_name` path. The device mapper path is stable for LVM (unlike raw device paths), so both are safe.

---

### Resizing LVM — The Killer Feature

#### Extending a Logical Volume (Add Space)

**Scenario:** `/mnt/projects` is running out of space. The VG has no free space, so you need to add a new disk first.

```bash
# Step 1: Add a new disk to VM (e.g., /dev/sde, 5 GB)
# Step 2: Create PV
sudo pvcreate /dev/sde

# Step 3: Extend the VG with the new PV
sudo vgextend vg_data /dev/sde

# Step 4: Check available space
sudo vgs
# VFree should now show ~5 GB

# Step 5: Extend the LV
sudo lvextend -L +3G /dev/vg_data/lv_projects
# or extend to a specific size:
# sudo lvextend -L 9G /dev/vg_data/lv_projects

# Step 6: Resize the FILESYSTEM (critical — LV is bigger, but filesystem doesn't know yet)

# For ext4:
sudo resize2fs /dev/vg_data/lv_projects

# For XFS:
sudo xfs_growfs /mnt/backups    # XFS uses mount point, not device path
```

**📌 Important:** After `lvextend`, you MUST resize the filesystem. The LV is just the container — the filesystem inside it doesn't automatically expand.

**Shortcut — Extend LV and resize filesystem in one command:**
```bash
sudo lvextend -L +3G --resizefs /dev/vg_data/lv_projects
```

The `--resizefs` flag handles both steps automatically. Use this.

**✅ Best Practice:** Always use `lvextend --resizefs` to avoid forgetting the filesystem resize step.

#### Reducing a Logical Volume (Remove Space)

**🔴 DANGER:** Shrinking is risky. Data loss is possible if done incorrectly. XFS **cannot** be shrunk — only ext4 can.

```bash
# For ext4 ONLY — XFS does NOT support shrinking

# Step 1: Unmount the filesystem
sudo umount /mnt/projects

# Step 2: Check filesystem integrity
sudo e2fsck -f /dev/vg_data/lv_projects

# Step 3: Shrink the filesystem first (to 4 GB)
sudo resize2fs /dev/vg_data/lv_projects 4G

# Step 4: Then shrink the LV
sudo lvreduce -L 4G /dev/vg_data/lv_projects

# Step 5: Remount
sudo mount /dev/vg_data/lv_projects /mnt/projects

# Verify
df -h /mnt/projects
```

**⚠️ Common Mistake:** Shrinking the LV before shrinking the filesystem. This truncates the filesystem and destroys data. Always shrink the filesystem FIRST, then the LV.

### LVM Command Reference

| Command | Purpose |
|---|---|
| `pvcreate` | Initialize a disk as a PV |
| `pvs` / `pvdisplay` | Show physical volumes |
| `vgcreate` | Create a volume group |
| `vgs` / `vgdisplay` | Show volume groups |
| `vgextend` | Add a PV to a VG |
| `lvcreate` | Create a logical volume |
| `lvs` / `lvdisplay` | Show logical volumes |
| `lvextend --resizefs` | Grow an LV + filesystem |
| `lvreduce` | Shrink an LV (ext4 only, dangerous) |
| `lvremove` | Delete a logical volume |
| `vgremove` | Delete a volume group |
| `pvremove` | Remove PV label from a disk |

**🎯 Interview Point:** "How would you add more disk space to a running Linux server without downtime?"
> Add a new physical disk, create a PV (`pvcreate`), extend the VG (`vgextend`), extend the LV (`lvextend --resizefs`). The filesystem grows online — no unmount or reboot needed (for ext4 and XFS).

---

## 6.9 — RAID Overview (Software RAID with `mdadm`)

### What is RAID?

**RAID** (Redundant Array of Independent Disks) combines multiple disks for either performance, redundancy, or both.

**📌 Important — Common RAID levels:**

| Level | Min Disks | Description | Capacity | Fault Tolerance | Use Case |
|---|---|---|---|---|---|
| **RAID 0** (Stripe) | 2 | Data split across disks | 100% | None — 1 disk fails, all data lost | Performance (temp data) |
| **RAID 1** (Mirror) | 2 | Data duplicated on all disks | 50% | 1 disk can fail | OS drives, critical data |
| **RAID 5** (Stripe + Parity) | 3 | Data + parity distributed | (N-1)/N | 1 disk can fail | General server storage |
| **RAID 6** (Stripe + Double Parity) | 4 | Data + dual parity | (N-2)/N | 2 disks can fail | Large storage arrays |
| **RAID 10** (Mirror + Stripe) | 4 | Mirrored pairs, then striped | 50% | 1 disk per mirror pair | Databases, high performance |

### Software RAID with `mdadm`

Linux can do RAID in software using `mdadm` (no special hardware needed).

**Lab setup: Add three 2 GB disks to your VM** (`/dev/sdf`, `/dev/sdg`, `/dev/sdh`).

#### Creating a RAID 1 (Mirror)

```bash
# Install mdadm if not present
sudo apt install -y mdadm       # Ubuntu/Debian
# sudo dnf install -y mdadm     # Rocky/RHEL

# Create RAID 1 with two disks
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdf /dev/sdg
```

**Expected prompt:**
```
mdadm: Note: this array has metadata at the start and may not be suitable as a boot device.
Continue creating array? y
```

```bash
# Check RAID status
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

**Expected `cat /proc/mdstat` output:**
```
Personalities : [raid1]
md0 : active raid1 sdg[1] sdf[0]
      2095104 blocks super 1.2 [2/2] [UU]
```

`[UU]` means both disks are **Up**. `[U_]` would mean one disk has failed.

```bash
# Create filesystem on the RAID device
sudo mkfs.ext4 /dev/md0

# Mount it
sudo mkdir -p /mnt/raid1
sudo mount /dev/md0 /mnt/raid1

# Verify
df -h /mnt/raid1
```

#### Simulating a Disk Failure

```bash
# Mark a disk as failed
sudo mdadm --manage /dev/md0 --fail /dev/sdg

# Check status — should show [U_] (one disk down)
cat /proc/mdstat

# Remove the failed disk
sudo mdadm --manage /dev/md0 --remove /dev/sdg

# Add a replacement disk (using the spare disk sdh)
sudo mdadm --manage /dev/md0 --add /dev/sdh

# Watch the rebuild
watch cat /proc/mdstat
```

#### Making RAID Persistent

```bash
# Save RAID configuration
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf

# Update initramfs so RAID is available at boot
sudo update-initramfs -u

# Add to fstab
echo '/dev/md0  /mnt/raid1  ext4  defaults  0  2' | sudo tee -a /etc/fstab
```

**✅ Best Practice:** In production, use hardware RAID controllers when available. Use software RAID (`mdadm`) for cost-effective setups or when hardware RAID isn't an option.

**🎯 Interview Point:** "What is RAID 1 and RAID 5? When would you use each?"
> RAID 1 mirrors data across 2 disks — simple, reliable, but uses 50% capacity. RAID 5 distributes data and parity across 3+ disks — better capacity efficiency (only 1 disk lost to parity), but slower writes. RAID 1 for OS/boot drives, RAID 5 for general data storage.

---

## 6.10 — Filesystem Maintenance

### Checking Filesystem Health

```bash
# Check ext4 filesystem (must be unmounted)
sudo umount /dev/sdb1
sudo fsck.ext4 /dev/sdb1

# Force check even if clean
sudo fsck.ext4 -f /dev/sdb1

# Check and auto-fix errors
sudo fsck.ext4 -y /dev/sdb1
```

**⚠️ Common Mistake:** Running `fsck` on a mounted filesystem. This can cause data corruption. Always unmount first, or run it from a live USB.

```bash
# Check XFS filesystem (can check while mounted, read-only)
sudo xfs_repair /dev/sdb2         # Must be unmounted
sudo xfs_repair -n /dev/sdb2      # Dry run — check without fixing
```

### Checking Disk Usage

```bash
# Top-level directory usage
du -sh /*

# Find the largest directories under /var
du -h --max-depth=1 /var | sort -rh | head -10

# Find files larger than 100 MB
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# Inode usage (you can run out of inodes before disk space)
df -i
```

**📌 Important — Inodes:** Every file and directory uses one inode. A filesystem has a fixed number of inodes. If you have millions of tiny files, you can exhaust inodes even with free disk space.

```bash
# Check inode usage
df -i
```

If any filesystem shows 100% IUse%, you need to delete files (even empty files count).

### Tuning Filesystem Parameters

```bash
# View ext4 filesystem parameters
sudo tune2fs -l /dev/sdb1

# Set a filesystem label
sudo tune2fs -L "data_volume" /dev/sdb1

# Set a reserved block percentage (default 5% reserved for root)
sudo tune2fs -m 1 /dev/sdb1    # Reduce to 1% — good for data-only partitions
```

**💡 Admin Tip:** By default, ext4 reserves 5% of the disk for the root user. On a 1 TB data partition, that's 50 GB wasted. For non-root data partitions, reduce it to 1%: `sudo tune2fs -m 1 /dev/sdX`.

---

## 6.11 — Hands-On Labs

### Lab 1: Basic Partitioning and Mounting

**Objective:** Partition a disk, create a filesystem, mount it, and make it persistent.

```bash
# 1. Verify your practice disk
lsblk

# 2. Partition /dev/sdb with fdisk (create two partitions: 5G + remaining)
sudo fdisk /dev/sdb
# Inside fdisk: n → p → 1 → default → +5G → n → p → 2 → default → default → p → w

# 3. Create filesystems
sudo mkfs.ext4 -L "projects" /dev/sdb1
sudo mkfs.xfs -L "archives" /dev/sdb2

# 4. Create mount points
sudo mkdir -p /mnt/projects /mnt/archives

# 5. Mount
sudo mount /dev/sdb1 /mnt/projects
sudo mount /dev/sdb2 /mnt/archives

# 6. Verify
df -h /mnt/projects /mnt/archives
lsblk -f /dev/sdb

# 7. Create test files
echo "Project data" | sudo tee /mnt/projects/readme.txt
echo "Archive data" | sudo tee /mnt/archives/readme.txt

# 8. Get UUIDs
sudo blkid /dev/sdb1 /dev/sdb2

# 9. Add to fstab (replace UUIDs with your actual values)
sudo cp /etc/fstab /etc/fstab.backup
echo 'UUID=<your-sdb1-uuid>  /mnt/projects  ext4  defaults  0  2' | sudo tee -a /etc/fstab
echo 'UUID=<your-sdb2-uuid>  /mnt/archives  xfs   defaults  0  2' | sudo tee -a /etc/fstab

# 10. Test fstab
sudo umount /mnt/projects /mnt/archives
sudo mount -a

# 11. Verify files survived
cat /mnt/projects/readme.txt
cat /mnt/archives/readme.txt
```

### Lab 2: Complete LVM Setup

**Objective:** Create an LVM setup from scratch, then extend it.

```bash
# 1. Create PVs (assuming /dev/sdc and /dev/sdd are available, 5G each)
sudo pvcreate /dev/sdc /dev/sdd
sudo pvs

# 2. Create VG
sudo vgcreate vg_lab /dev/sdc /dev/sdd
sudo vgs

# 3. Create LVs
sudo lvcreate -L 4G -n lv_web vg_lab
sudo lvcreate -L 3G -n lv_db vg_lab
sudo lvs

# 4. Create filesystems
sudo mkfs.ext4 /dev/vg_lab/lv_web
sudo mkfs.xfs /dev/vg_lab/lv_db

# 5. Mount
sudo mkdir -p /mnt/web /mnt/db
sudo mount /dev/vg_lab/lv_web /mnt/web
sudo mount /dev/vg_lab/lv_db /mnt/db

# 6. Verify
df -h /mnt/web /mnt/db

# 7. Write test data
sudo dd if=/dev/urandom of=/mnt/web/testdata bs=1M count=100
sudo dd if=/dev/urandom of=/mnt/db/testdata bs=1M count=100

# 8. Extend lv_web by 2G
sudo lvextend -L +2G --resizefs /dev/vg_lab/lv_web

# 9. Verify the extension
df -h /mnt/web    # Should now show ~6G

# 10. Verify data is intact
ls -lh /mnt/web/testdata    # Should still be 100M
```

### Lab 3: Swap File Creation

**Objective:** Create and enable a swap file.

```bash
# 1. Check current swap
free -h
swapon --show

# 2. Create a 1 GB swap file
sudo dd if=/dev/zero of=/swapfile_lab bs=1M count=1024

# 3. Set permissions
sudo chmod 600 /swapfile_lab

# 4. Format as swap
sudo mkswap /swapfile_lab

# 5. Enable
sudo swapon /swapfile_lab

# 6. Verify
free -h
swapon --show

# 7. Clean up (disable and remove)
sudo swapoff /swapfile_lab
sudo rm /swapfile_lab
free -h
```

---

## 6.12 — Troubleshooting Exercises

### Exercise 1: Disk Full but No Large Files

**Problem:** `df -h` shows 100% usage, but `du -sh /*` doesn't account for all the space.

**Diagnosis:**
```bash
# Check for deleted files still held open by processes
sudo lsof | grep "(deleted)"

# These files won't free space until the process releases them
# Restart the service holding the file
sudo systemctl restart <service_name>

# Or — truncate the file if the process supports it
```

**Root cause:** A process (like a log handler) opened a file, the file was deleted, but the process still holds a file descriptor. The space isn't freed until the process closes the file.

### Exercise 2: Mount Fails with "wrong fs type"

**Problem:** `sudo mount /dev/sdb1 /mnt/data` fails with "wrong fs type, bad option, bad superblock".

**Diagnosis:**
```bash
# Check what filesystem is actually on the partition
sudo blkid /dev/sdb1

# If it shows nothing, the partition has no filesystem
sudo mkfs.ext4 /dev/sdb1    # Create one

# If it shows a filesystem type, specify it explicitly
sudo mount -t xfs /dev/sdb1 /mnt/data

# If the filesystem is corrupted
sudo fsck /dev/sdb1
```

### Exercise 3: LVM — Cannot Extend LV

**Problem:** `lvextend` fails with "Insufficient free space".

**Diagnosis:**
```bash
# Check free space in the VG
sudo vgs

# If VFree is 0, you need more physical storage
# Option 1: Add a new disk
sudo pvcreate /dev/sde
sudo vgextend vg_data /dev/sde

# Option 2: Shrink another LV in the same VG (ext4 only)
# Option 3: Move data and remove unused LVs
sudo lvremove /dev/vg_data/lv_unused
```

### Exercise 4: System Won't Boot After fstab Edit

**Problem:** You edited `/etc/fstab`, rebooted, and the system drops to emergency mode.

**Fix:**
```bash
# In emergency mode:
mount -o remount,rw /                    # Remount root as writable
cp /etc/fstab.backup /etc/fstab          # Restore backup
# OR edit manually:
vi /etc/fstab                            # Comment out the bad line with #
reboot
```

**Prevention:** Always `sudo mount -a` before rebooting after fstab changes.

---

## 6.13 — Practical Scenarios

### Scenario 1: Production Server Running Out of Space

**Task:** A web server's `/var/log` is filling up. The server uses LVM. Add space without downtime.

**Try this yourself first, then check the solution below.**

<details>
<summary><strong>Solution</strong></summary>

```bash
# 1. Identify the problem
df -h /var/log
du -h --max-depth=1 /var/log | sort -rh | head

# 2. Quick fix — clean old logs
sudo journalctl --vacuum-time=7d
sudo find /var/log -name "*.gz" -mtime +30 -delete

# 3. Long-term fix — extend the LV
# Check VG free space
sudo vgs

# If free space available:
sudo lvextend -L +5G --resizefs /dev/vg_server/lv_var

# If no free space — add a disk:
# (after physically adding a disk)
sudo pvcreate /dev/sde
sudo vgextend vg_server /dev/sde
sudo lvextend -L +5G --resizefs /dev/vg_server/lv_var

# 4. Verify
df -h /var/log
```

</details>

### Scenario 2: Adding a New Data Disk to a Server

**Task:** A new 100 GB disk has been added to the server for application data. Set it up with LVM, mount it at `/data`, and make it persistent.

**Try this yourself first.**

<details>
<summary><strong>Solution</strong></summary>

```bash
# 1. Identify the new disk
lsblk

# 2. Create PV
sudo pvcreate /dev/sdb

# 3. Create VG
sudo vgcreate vg_appdata /dev/sdb

# 4. Create LV using all space
sudo lvcreate -l 100%FREE -n lv_data vg_appdata

# 5. Create filesystem
sudo mkfs.ext4 /dev/vg_appdata/lv_data

# 6. Create mount point and set ownership
sudo mkdir -p /data
sudo mount /dev/vg_appdata/lv_data /data
sudo chown appuser:appgroup /data

# 7. Make persistent
sudo blkid /dev/vg_appdata/lv_data
sudo cp /etc/fstab /etc/fstab.backup
echo '/dev/vg_appdata/lv_data  /data  ext4  defaults  0  2' | sudo tee -a /etc/fstab
sudo mount -a

# 8. Verify
df -h /data
```

</details>

---

## 6.14 — Practice Questions

1. **Partition a new 10 GB disk** into three partitions (3G, 3G, 4G) using `fdisk`. Create ext4 on the first, XFS on the second, and swap on the third. Mount the first two and enable the swap.

2. **Create a complete LVM setup** with two 5 GB disks. Create a VG, then create two LVs (6G and 4G). Mount them, add data, then extend the smaller one by 2G (you'll need a third disk).

3. **Set up a swap file** of 1 GB. Enable it, verify with `free -h`, add it to fstab, then disable and remove it.

4. **Mount challenge:** Mount a partition at `/mnt/secure` with `noexec,nosuid,nodev` options. Copy a script there and try to execute it. Verify that `noexec` prevents execution.

5. **Disaster recovery:** Intentionally add a bad entry to fstab (with a non-existent UUID). Use `sudo mount -a` to see the error. Fix it before rebooting. (Do NOT actually reboot with a bad fstab.)

6. **Storage investigation:** Find all partitions on the system, their filesystem types, mount points, and usage percentages. Present the data in a clean format using `lsblk -f` and `df -h`.

---

## Level 6 Summary

### What I Learned

- Linux represents disks as block devices under `/dev/`
- GPT is the modern partition table format, replacing MBR
- `fdisk` for interactive MBR partitioning, `parted` for GPT and scripting
- Filesystems (ext4, XFS) organize data on partitions
- `mkfs` creates filesystems; `mount` attaches them to directories
- `/etc/fstab` makes mounts persistent across reboots — always use UUIDs
- Swap extends RAM to disk — use swap files or partitions
- LVM adds flexible volume management: PV → VG → LV
- LVM allows live resize with `lvextend --resizefs`
- RAID provides redundancy (RAID 1 mirror, RAID 5 parity)
- `fsck` checks filesystem health (must unmount first for ext4)

### Commands to Remember

| Command | Purpose |
|---|---|
| `lsblk` | List all block devices |
| `lsblk -f` | Show filesystems, UUIDs, mount points |
| `df -h` | Show disk usage of mounted filesystems |
| `du -sh /path` | Show directory size |
| `sudo fdisk /dev/sdX` | Partition a disk (interactive, MBR) |
| `sudo parted /dev/sdX` | Partition a disk (GPT, scriptable) |
| `sudo mkfs.ext4 /dev/sdX1` | Create ext4 filesystem |
| `sudo mkfs.xfs /dev/sdX1` | Create XFS filesystem |
| `sudo mount /dev/sdX1 /mnt/dir` | Mount a filesystem |
| `sudo umount /mnt/dir` | Unmount a filesystem |
| `sudo blkid` | Show UUIDs and filesystem info |
| `/etc/fstab` | Persistent mount configuration |
| `sudo mount -a` | Mount all entries in fstab |
| `sudo mkswap /dev/sdX` | Create swap space |
| `sudo swapon / swapoff` | Enable / disable swap |
| `sudo pvcreate` | Create LVM physical volume |
| `sudo vgcreate` | Create LVM volume group |
| `sudo lvcreate` | Create LVM logical volume |
| `sudo lvextend --resizefs` | Extend LV + filesystem |
| `sudo pvs / vgs / lvs` | Show LVM status |
| `sudo mdadm --create` | Create software RAID |
| `sudo fsck` | Check/repair filesystem |

### Practical Skills Gained

- Partition and format new disks
- Mount filesystems manually and persistently
- Manage storage with LVM (create, extend, monitor)
- Create and manage swap space
- Understand RAID levels and set up software RAID
- Troubleshoot disk full, mount failures, and fstab errors
- Check filesystem health and tune parameters

### Common Mistakes

| Mistake | Fix |
|---|---|
| Using `/dev/sdb1` in fstab (device names change) | Always use UUID: `UUID=a1b2c3d4...` |
| Running `mkfs` on the wrong partition | Triple-check with `lsblk` before formatting |
| Running `fsck` on a mounted filesystem | Unmount first: `sudo umount /dev/sdX1` |
| Forgetting `sudo mount -a` after editing fstab | Always test before rebooting |
| `lvextend` without `--resizefs` | Filesystem stays small even though LV grew |
| Shrinking LV before shrinking filesystem | Destroys data. Shrink filesystem FIRST |
| Rebooting with bad fstab entry | System drops to emergency mode |

### Administrator-Level Knowledge

- Always backup `/etc/fstab` before editing
- Use LVM on all production servers for flexibility
- Set swap to a reasonable size and tune swappiness to 10
- Reserve less than 5% for data-only ext4 partitions (`tune2fs -m 1`)
- Use `noexec,nosuid,nodev` on `/tmp` for security
- Monitor inode usage with `df -i` (not just disk space)
- Know how to recover from fstab boot failures

### Interview Questions

1. **What is the difference between MBR and GPT?**
2. **Explain the three layers of LVM (PV, VG, LV).**
3. **How would you add disk space to a running production server without downtime?**
4. **What is the difference between `ext4` and `XFS`?**
5. **Why should you use UUID instead of device names in `/etc/fstab`?**
6. **How do you check disk usage on a Linux system?**
7. **What happens if `/etc/fstab` has an error and you reboot?**
8. **What is swap? How much should a server have?**
9. **What is RAID 1 vs RAID 5? When would you choose each?**
10. **How do you extend a logical volume that is running out of space?**

---

*When you've completed all the labs and practice questions, proceed to [Level 7 — Networking & Connectivity](level-07-networking.md)*
