# Linux Storage & File Systems Lab

**Storage & File Systems (EBS → Linux Mount + Persistence)**

In this lab, I proved I can perform **cloud storage operations end-to-end** by creating and attaching an **AWS EBS volume**, formatting it with a Linux file system, mounting it on the instance, and making the mount **persist across reboots** using **`/etc/fstab` + UUIDs**.

---

## 📋 Lab Overview

**Goal:**

* Create a new **EBS gp3** volume and attach it to an EC2 instance
* Identify the new disk inside Linux using `lsblk`
* Create a file system (`xfs` or `ext4`)
* Mount the disk to `/data`
* Persist the mount across reboot using:

  * `/etc/fstab`
  * `UUID`
  * safe options like `nofail`
* Validate persistence by rebooting and confirming auto-mount

**Learning Outcomes:**

* Understand what `/etc/fstab` controls and why it’s **boot-critical**
* Use `blkid` to retrieve stable **UUIDs**
* Understand why AWS device names can change (e.g., `xvd*` vs `nvme*`)
* Correctly format and mount a new disk using `mkfs` + `mount`
* Safely edit `/etc/fstab` with backup + `mount -a` validation
* Verify mounts using `df -h | grep`

---

## 🛠 Step-by-Step Journey

### Step 1: Understand `/etc/fstab` and Why UUID Matters

**What is `/etc/fstab`?**
`/etc/fstab` is the **file systems table** — a config file that tells Linux:

* what to mount
* where to mount it
* which file system type
* which mount options
* and whether to check it at boot

📌 **Mental model:**

> “When Linux boots, mount these disks here in this way.”

---

### Step 2: Understand the Last Two `fstab` Columns (`dump` + `fsck`)

A typical `fstab` line has **6 columns**:

1. what to mount (UUID=...)
2. where to mount (`/data`)
3. file system type (`xfs`, `ext4`)
4. mount options (`defaults`, `nofail`, etc.)
5. `dump`
6. `fsck order`

**Column 5 — dump (legacy):**

* `0` = do not dump (almost always used)
* `1` = include in dump backups (rare)

✅ In real-world Linux (especially in 2026), this is effectively **legacy** and is usually `0`.

**Column 6 — fsck order (boot-time checks):**

* `0` = never check
* `1` = check first (root `/` only)
* `2` = check after root (non-root file systems like `/data`)

✅ Typical pattern:

* root is `1`
* extra disks like `/data` are `2`

---

### Step 3: Create a New EBS Volume in AWS

**AWS actions:**

* Go to **EBS → Volumes → Create volume**
* Type: **gp3**
* Size: **5 GiB**
* Availability Zone: **same AZ as EC2** (e.g., `eu-west-2a`)

📌 **Important rule:**
EBS volumes must be in the **same Availability Zone** as the instance they attach to.

---

### Step 4: Attach the Volume to the EC2 Instance

**AWS actions:**

* Select the correct volume
* **Actions → Attach volume**
* Choose your EC2 instance
* Choose a device name (example): `/dev/sdb`

✅ You may see a warning like:

> Linux may rename your devices internally.

**Why does this happen?**

AWS device names are **logical labels**, not guaranteed physical names.
Modern EC2 instances (Nitro-based) expose disks as **NVMe devices**, so Linux may name it like:

* `/dev/nvme1n1` (new EBS disk)
* `/dev/nvme0n1` (usually the root disk)

📌 **Production rule:**
Device paths like `/dev/sdb` or `/dev/xvdf` can change → use **UUID** for persistence.

---

### Step 5: Identify the New Disk in Linux

**Command:**

```bash
lsblk
```

Example result showed a new disk like:

* `nvme1n1` (5G, TYPE=disk, no mountpoint)

**What the columns mean (high-value ones):**

* `NAME` → kernel device name (e.g., `nvme1n1`)
* `SIZE` → disk size (5G)
* `RM` → removable (0 = not removable)
* `RO` → read-only (0 = writable)
* `TYPE` → disk/part/lvm/rom
* `MOUNTPOINTS` → empty means **not mounted**

✅ Key takeaway:

> Disk is attached and visible, but it has no file system and isn’t mounted yet — exactly expected.

---

### Step 6: Choose a File System (XFS vs EXT4)

You considered **XFS vs EXT4**:

**EXT4:**

* Most widely compatible
* Stable and well-supported
* Great general-purpose choice

**XFS:**

* Excellent performance for large files + concurrency
* Common default in RHEL/Amazon Linux ecosystems

✅ Since Amazon Linux / RHEL-style systems commonly default to XFS, you chose **XFS**.

---

### Step 7: Create the File System

**Command (XFS):**

```bash
sudo mkfs.xfs /dev/nvme1n1
```

📌 Safety reminder: `mkfs` destroys data — confirm you are **not formatting the root disk**.

**If using ext4 instead:**

```bash
sudo mkfs.ext4 /dev/nvme1n1
```

**Why `/dev/` prefix?**
`lsblk` shows short names for readability, but device files live under:

* `/dev/nvme1n1`

---

### Step 8: Create the Mount Point and Mount the Disk

**What is `/data`?**
It’s just a directory used **by convention** as a mount point. Linux won’t create it automatically.

**Create mount directory:**

```bash
sudo mkdir -p /data
```

**Mount:**

```bash
sudo mount /dev/nvme1n1 /data
```

✅ After mounting:

> Anything stored under `/data` now lives on the EBS volume.

**Verify:**

```bash
df -h | grep /data
```

---

### Step 9: Create a Test File on the Mounted Volume

**Command:**

```bash
sudo touch /data/f1
ls -l /data/f1
```

---

### Step 10: Make the Mount Persistent Using UUID + `/etc/fstab`

#### 10.1 Get the UUID

```bash
sudo blkid /dev/nvme1n1
```

Copy the UUID value.

---

#### 10.2 Backup `/etc/fstab` (Boot-Critical Safety Step)

```bash
sudo cp /etc/fstab /etc/fstab.bak
```

📌 Why this is mandatory:
A bad `fstab` entry can prevent boot and drop the system into emergency mode — especially painful on EC2.

---

#### 10.3 Append the New Entry Safely (UUID + nofail)

```bash
echo 'UUID=<PASTE-UUID-HERE>  /data  xfs  defaults,nofail  0  2' | sudo tee -a /etc/fstab > /dev/null
```

**Why this exact pattern matters:**

* `UUID=...` → stable identifier (won’t change like `/dev/*`)
* `/data` → mount point
* `xfs` → file system type
* `defaults` → standard safe mount options
* `nofail` → don’t block boot if the disk is missing
* `0` → legacy dump field (almost always 0)
* `2` → fsck after root (non-root file systems)

**Why `sudo tee -a`?**

* `-a` = append (prevents overwrite catastrophe)
* redirection `>> /etc/fstab` won’t work reliably with sudo because redirection happens before sudo

---

### Step 11: Validate `/etc/fstab` Without Rebooting (Best Practice)

```bash
sudo mount -a
```

✅ This mounts all entries from `/etc/fstab` that aren’t already mounted.

📌 Why this is best practice:
It catches mistakes immediately (bad UUID, wrong fs type, bad options) **before** you risk a broken reboot.

---

### Step 12: Reboot and Prove Persistence

**Reboot:**

```bash
sudo reboot
```

Reconnect via SSH.

**Verify auto-mount:**

```bash
df -h | grep /data
```

✅ Confirmed: `/data` mounts automatically after reboot.

---

## ✅ Key Commands Summary

| Task                        | Command                             |                   
| --------------------------- | ----------------------------------- | 
| List disks (raw devices)    | `lsblk`                             |                               
| Create XFS file system      | `sudo mkfs.xfs /dev/nvme1n1`        |                          
| Create mount point          | `sudo mkdir -p /data`               |                          
| Mount disk                  | `sudo mount /dev/nvme1n1 /data`     |                            
| Verify mount                | `df -h \| grep /data`  |   
| Create test file            | `sudo touch /data/f1`               |                          
| Get UUID                    | `sudo blkid /dev/nvme1n1`           |                              
| Backup fstab                | `sudo cp /etc/fstab /etc/fstab.bak` |                               
| Append fstab entry safely   | `echo '...' sudo tee -a /etc/fstab > /dev/null` | 
| Validate fstab mounts       | `sudo mount -a`                     |                             
| Reboot to prove persistence | `sudo reboot`                       |                          

---

## 💡 Notes / Tips

* **Always use UUID in `/etc/fstab`** for production-safe mounts.
* AWS device labels (`/dev/sdb`) are **not guaranteed** to match Linux device names (`/dev/nvme1n1`).
* `/etc/fstab` is **boot-critical** — always back it up first.
* Always run `sudo mount -a` after editing `fstab` to catch errors safely.
* `nofail` prevents an EC2 instance from failing boot if an optional EBS volume is missing.

---

## 📌 Lab Summary

| Step                       | Status | Key Takeaways                            |
| -------------------------- | ------ | ---------------------------------------- |
| Create + attach EBS volume | ✅      | Same AZ required; attach correctly       |
| Identify disk with `lsblk` | ✅      | Disk visible but unmounted initially     |
| Format disk                | ✅      | `mkfs.xfs` creates usable file structure |
| Mount disk to `/data`      | ✅      | Mount point is just a directory          |
| Create data on disk        | ✅      | Verified with test file                  |
| Persist using UUID + fstab | ✅      | UUID prevents device rename failures     |
| Validate with `mount -a`   | ✅      | Catch mistakes before reboot             |
| Reboot verification        | ✅      | `/data` auto-mount confirmed             |
