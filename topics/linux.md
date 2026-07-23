---
title: Linux
nav_order: 10
description: "Linux — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Linux — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents

- [Linux Fundamentals](#linux-fundamentals)
- [File System & Storage](#file-system--storage)
- [Process Management](#process-management)
- [Memory Management](#memory-management)
- [Networking](#networking-in-linux)
- [User & Permission Management](#user--permission-management)
- [Systemd & Init Systems](#systemd--init-systems)
- [Package Management](#package-management)
- [Shell Scripting Essentials](#shell-scripting-essentials)
- [Performance Troubleshooting](#performance-troubleshooting)
- [Log Management](#log-management)
- [Kernel & Boot Process](#kernel--boot-process)
- [Security Hardening](#security-hardening)
- [Containers: Namespaces & Cgroups Deep Dive](#containers-namespaces--cgroups-deep-dive)
- [Advanced Troubleshooting Scenarios](#advanced-troubleshooting-scenarios)
- [Command-Line One-Liners & Golf](#command-line-one-liners--golf)
- [Scenario-Based Questions](#scenario-based-questions)
- [Key Commands Cheat Sheet](#key-commands-cheat-sheet)

---

## Linux Fundamentals

### 🔥 Q: What is the Linux boot process?

**Answer:**

1. **BIOS/UEFI** — Firmware performs POST (Power-On Self-Test), detects hardware, looks for a bootable device.
2. **Bootloader (GRUB2)** — Loads the kernel into memory. GRUB reads `/boot/grub2/grub.cfg` to know which kernel and initramfs to load.
3. **Kernel Initialization** — Kernel decompresses itself, initializes hardware, mounts the initial RAM filesystem (initramfs).
4. **initramfs** — Temporary root filesystem that contains drivers needed to mount the real root filesystem.
5. **Init System (systemd/SysVinit)** — PID 1 process starts. Systemd reads `/etc/systemd/system/default.target` and starts services in parallel.
6. **Runlevel / Target** — System reaches the configured target (e.g., `multi-user.target` for CLI, `graphical.target` for GUI).
7. **Login Prompt** — getty or display manager presents the login interface.

### 🔥 Q: What is the difference between a process and a thread?

| Aspect | Process | Thread |
|--------|---------|--------|
| Memory | Own address space | Shares parent process's address space |
| Creation | `fork()` — heavier | `pthread_create()` — lighter |
| Communication | IPC (pipes, sockets, shared mem) | Direct shared memory access |
| Crash Impact | Isolated — one crash doesn't affect others | One thread crash can kill the entire process |
| Context Switch | Expensive (TLB flush, page table swap) | Cheap (same address space) |

### ⭐ Q: Explain Linux file system hierarchy.

```
/           Root of the entire filesystem
├── /bin    Essential user binaries (ls, cp, mv)
├── /sbin   System binaries (iptables, fdisk)
├── /etc    Configuration files
├── /home   User home directories
├── /var    Variable data (logs, spool, tmp)
├── /tmp    Temporary files (cleared on reboot)
├── /usr    Secondary hierarchy (user programs)
│   ├── /usr/bin     User commands
│   ├── /usr/lib     Libraries
│   └── /usr/local   Locally installed software
├── /opt    Optional/third-party software
├── /proc   Virtual filesystem — kernel & process info
├── /sys    Virtual filesystem — device/driver info
├── /dev    Device files (block, character)
├── /boot   Bootloader files, kernel, initramfs
├── /lib    Shared libraries for /bin and /sbin
├── /mnt    Temporary mount points
├── /media  Removable media mount points
└── /run    Runtime variable data since last boot
```

### 🔥 Q: What are inodes?

An **inode** is a data structure on a filesystem that stores metadata about a file:

- File size, owner (UID/GID), permissions
- Timestamps (atime, mtime, ctime)
- Number of hard links
- Pointers to data blocks on disk

**Key points:**
- Each file has exactly one inode; each inode has a unique inode number within its filesystem.
- Directory entries are mappings of filenames → inode numbers.
- You can run out of inodes before running out of disk space (common with millions of tiny files).
- Check with: `df -i`
- Soft links have their own inode; hard links share the same inode.

### 🔥 Q: Hard link vs Soft link?

| Feature | Hard Link | Soft Link (Symlink) |
|---------|-----------|---------------------|
| Inode | Same as target | Different inode |
| Cross filesystem | No | Yes |
| Link to directory | No (usually) | Yes |
| Target deleted | Still works (data persists) | Becomes dangling/broken |
| Command | `ln target link` | `ln -s target link` |

---

## File System & Storage

### ⭐ Q: Explain LVM and its components.

**LVM (Logical Volume Manager)** provides flexible disk management:

```
Physical Volumes (PV)  →  Volume Groups (VG)  →  Logical Volumes (LV)
  /dev/sda1                  vg_data                lv_app
  /dev/sdb1                                         lv_logs
```

- **PV** — Physical disk or partition initialized for LVM (`pvcreate /dev/sdb`)
- **VG** — Pool of storage from one or more PVs (`vgcreate vg_data /dev/sdb`)
- **LV** — Virtual partition carved from a VG (`lvcreate -L 50G -n lv_app vg_data`)

**Why LVM matters:**
- Resize volumes live (extend without downtime)
- Snapshots for backups
- Stripe across disks for performance
- Thin provisioning

**Common commands:**
```bash
pvs / vgs / lvs           # List PVs, VGs, LVs
lvextend -L +10G /dev/vg_data/lv_app
resize2fs /dev/vg_data/lv_app   # For ext4
xfs_growfs /mount/point          # For XFS
```

### ⭐ Q: What is RAID? Explain levels.

| RAID Level | Min Disks | Redundancy | Performance | Usable Capacity |
|-----------|-----------|------------|-------------|-----------------|
| RAID 0 | 2 | None | Best read/write | 100% |
| RAID 1 | 2 | Mirroring | Good read, normal write | 50% |
| RAID 5 | 3 | 1 disk parity | Good read, slower write | (N-1)/N |
| RAID 6 | 4 | 2 disk parity | Good read, slower write | (N-2)/N |
| RAID 10 | 4 | Mirror + Stripe | Excellent | 50% |

**FAANG Tip:** RAID is not a backup strategy — it protects against disk failure, not data corruption, accidental deletion, or ransomware.

### 🔥 Q: A disk is full. How do you troubleshoot?

```bash
# 1. Check disk usage
df -h

# 2. Find largest directories
du -sh /* | sort -rh | head -20

# 3. Find large files
find / -type f -size +100M -exec ls -lh {} \;

# 4. Check for deleted files still held open
lsof | grep '(deleted)'

# 5. Check inode usage
df -i

# 6. Common culprits
du -sh /var/log/*
journalctl --disk-usage

# 7. Clean up
journalctl --vacuum-size=500M
find /tmp -type f -atime +7 -delete
docker system prune -a    # If Docker is installed
```

---

## Process Management

### 🔥 Q: Explain process states in Linux.

```
         fork()
           |
           v
    ┌─────────────┐
    │   CREATED    │ (New)
    └──────┬──────┘
           │
           v
    ┌─────────────┐    schedule     ┌─────────────┐
    │   READY     │ ──────────────> │   RUNNING   │
    │  (Runnable) │ <────────────── │   (On CPU)  │
    └─────────────┘    preempt      └──────┬──────┘
                                           │
                           ┌───────────────┼───────────────┐
                           │               │               │
                           v               v               v
                    ┌────────────┐  ┌────────────┐  ┌────────────┐
                    │  SLEEPING  │  │  STOPPED   │  │   ZOMBIE   │
                    │ (D or S)   │  │  (T/t)     │  │   (Z)      │
                    └────────────┘  └────────────┘  └────────────┘
```

| State | Symbol | Description |
|-------|--------|-------------|
| Running | R | Actively using CPU or in run queue |
| Sleeping (Interruptible) | S | Waiting for event, can be interrupted by signal |
| Sleeping (Uninterruptible) | D | Waiting for I/O, cannot be killed (common with NFS issues) |
| Stopped | T | Suspended by signal (SIGSTOP/SIGTSTP) |
| Zombie | Z | Finished but parent hasn't read exit status via `wait()` |

### ⭐ Q: What are zombie processes and how to fix them?

A **zombie** is a process that has completed execution but still has an entry in the process table because its parent hasn't called `wait()` to collect the exit status.

```bash
# Find zombies
ps aux | awk '$8 ~ /Z/'

# Find the parent
ps -o ppid= -p <zombie_pid>

# Fix options:
# 1. Send SIGCHLD to parent (reminds it to reap)
kill -SIGCHLD <parent_pid>

# 2. Kill the parent (zombies get adopted by init/systemd which reaps them)
kill -9 <parent_pid>

# 3. If parent is essential, fix the application code to properly handle SIGCHLD
```

### 🔥 Q: Explain signals in Linux. Key signals to know:

| Signal | Number | Default Action | Description |
|--------|--------|---------------|-------------|
| SIGHUP | 1 | Terminate | Hangup — often used to reload config |
| SIGINT | 2 | Terminate | Interrupt (Ctrl+C) |
| SIGQUIT | 3 | Core dump | Quit (Ctrl+\\) |
| SIGKILL | 9 | Terminate | Force kill — **cannot be caught or ignored** |
| SIGTERM | 15 | Terminate | Graceful termination (default for `kill`) |
| SIGSTOP | 19 | Stop | Pause process — **cannot be caught** |
| SIGCONT | 18 | Continue | Resume a stopped process |
| SIGUSR1/2 | 10/12 | Terminate | User-defined signals |

**FAANG Tip:** Always try `SIGTERM` before `SIGKILL`. Many applications (e.g., databases) need to flush buffers and close connections gracefully.

### ⭐ Q: What is the OOM Killer?

The **Out-of-Memory Killer** is a Linux kernel mechanism that kills processes when the system runs critically low on memory.

- Each process gets an **oom_score** (0–1000) based on memory usage, runtime, and privileges.
- Adjust with **oom_score_adj** (-1000 to 1000):
  ```bash
  # Protect a critical process (e.g., database)
  echo -1000 > /proc/<pid>/oom_score_adj

  # Check current score
  cat /proc/<pid>/oom_score
  ```
- View OOM kills: `dmesg | grep -i "oom\|killed process"`
- Disable OOM killer (dangerous): `sysctl vm.overcommit_memory=2`

---

## Memory Management

### 🔥 Q: Explain Linux memory management (Virtual Memory, Pages, Swap).

**Virtual Memory:** Each process gets its own virtual address space. The MMU (Memory Management Unit) translates virtual addresses to physical addresses using page tables.

**Pages:** Memory is divided into fixed-size pages (typically 4KB). The kernel manages page allocation and tracks them via page tables.

**Key memory concepts:**
```bash
free -h
#               total    used    free    shared  buff/cache   available
# Mem:           16G      8G     1.2G     256M      6.8G        7.5G
# Swap:          4G       0B      4G
```

- **buff/cache** — Kernel uses free RAM as disk cache. This is **reclaimable**.
- **available** — Memory that can be allocated to processes (free + reclaimable cache).
- **Swap** — Disk-based extension of RAM. Slower but prevents OOM.

**Swap tunables:**
```bash
# Swappiness (0-100): How aggressively kernel swaps out memory
cat /proc/sys/vm/swappiness    # Default: 60
sysctl vm.swappiness=10        # Prefer keeping data in RAM
```

### 💡 Q: What are huge pages and why use them?

Regular pages are 4KB. **Huge Pages** are 2MB (or 1GB) pages that reduce TLB (Translation Lookaside Buffer) misses.

**Use case:** Databases (PostgreSQL, Oracle), JVMs, and any application with large memory footprints.

```bash
# Check current huge pages
cat /proc/meminfo | grep Huge

# Allocate 1024 huge pages (2MB each = 2GB)
echo 1024 > /proc/sys/vm/nr_hugepages

# Persistent via /etc/sysctl.conf
vm.nr_hugepages = 1024
```

---

## Networking in Linux

### ⭐ Q: How does DNS resolution work in Linux?

```
Application
    │
    ▼
/etc/nsswitch.conf  →  defines order: "files dns"
    │
    ├──► /etc/hosts (check local first)
    │
    └──► /etc/resolv.conf (DNS servers)
            │
            ▼
        DNS Resolver (stub resolver → systemd-resolved / NetworkManager)
            │
            ▼
        Recursive DNS Server (e.g., 8.8.8.8)
            │
            ├──► Root DNS (.)
            ├──► TLD DNS (.com)
            └──► Authoritative DNS (example.com)
```

**Troubleshooting DNS:**
```bash
# Check resolution
dig example.com
nslookup example.com
host example.com

# Check what resolver is configured
cat /etc/resolv.conf
resolvectl status

# Trace full DNS path
dig +trace example.com

# Clear DNS cache (systemd-resolved)
resolvectl flush-caches
```

### ⭐ Q: Explain iptables / nftables.

**iptables** is the traditional Linux firewall, using tables and chains:

```
Packet arrives
    │
    ▼
┌─────────────────────────────────────────┐
│           PREROUTING chain              │  (nat, mangle, raw)
└────────────────┬────────────────────────┘
                 │
         Is it for this host?
        ┌────────┴────────┐
        │ Yes             │ No
        ▼                 ▼
   INPUT chain      FORWARD chain         (filter, mangle)
        │                 │
        ▼                 ▼
   Local Process    POSTROUTING chain      (nat, mangle)
        │                 │
        ▼                 ▼
   OUTPUT chain      Network Out
        │
        ▼
   POSTROUTING
        │
        ▼
   Network Out
```

**Common iptables commands:**
```bash
# List rules
iptables -L -n -v

# Allow SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Block an IP
iptables -A INPUT -s 192.168.1.100 -j DROP

# NAT / Port forwarding
iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.5:8080

# Save rules
iptables-save > /etc/iptables/rules.v4
```

### 🔥 Q: What is the difference between TCP and UDP?

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery, ordering, retransmission | Best-effort, no guarantees |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Use cases | HTTP, SSH, FTP, databases | DNS, DHCP, video streaming, gaming |
| Flow control | Yes (sliding window) | No |
| Header size | 20 bytes minimum | 8 bytes |

**TCP 3-Way Handshake:**
```
Client              Server
  │── SYN ──────────►│
  │◄── SYN+ACK ──────│
  │── ACK ──────────►│
  │   Connection Established
```

---

## User & Permission Management

### 🔥 Q: Explain Linux file permissions in depth.

```
-rwxr-xr-- 1 root devops 4096 Jan 1 00:00 script.sh
│├─┤├─┤├─┤   │    │
│ │  │  │    │    └── Group
│ │  │  │    └── Owner
│ │  │  └── Others (r--)  = 4
│ │  └── Group (r-x)      = 5
│ └── Owner (rwx)          = 7
└── File type (- = regular, d = directory, l = symlink)

Numeric: 754
```

**Special permissions:**

| Permission | Numeric | On File | On Directory |
|-----------|---------|---------|-------------|
| SUID | 4000 | Runs as file owner | — |
| SGID | 2000 | Runs as group owner | New files inherit group |
| Sticky Bit | 1000 | — | Only owner can delete files (e.g., `/tmp`) |

```bash
chmod 4755 script.sh   # SUID + rwxr-xr-x
chmod 2755 /shared/    # SGID
chmod 1777 /tmp/       # Sticky bit

# Find SUID files (security audit)
find / -perm -4000 -type f 2>/dev/null
```

### ⭐ Q: What is sudoers and how does it work?

The `/etc/sudoers` file controls who can run what commands as what user.

```bash
# Edit safely (syntax check on save)
visudo

# Format: user  host=(runas_user:runas_group) commands
alice  ALL=(ALL:ALL) ALL                    # Full sudo
bob    ALL=(ALL) /usr/bin/systemctl restart nginx  # Specific command only
%devops ALL=(ALL) NOPASSWD: ALL             # Group-based, no password

# Sudoers drop-in directory (preferred for management)
/etc/sudoers.d/
```

---

## Systemd & Init Systems

### 🔥 Q: Explain systemd and its key components.

**systemd** is the init system (PID 1) on modern Linux, responsible for booting, service management, logging, and more.

**Key components:**
- **systemctl** — Service management
- **journalctl** — Log management
- **systemd-resolved** — DNS resolution
- **systemd-networkd** — Network management
- **systemd-tmpfiles** — Temp file management
- **systemd timers** — Cron alternative

**Service management:**
```bash
systemctl start/stop/restart/reload nginx
systemctl enable/disable nginx         # Start on boot
systemctl status nginx
systemctl list-units --type=service
systemctl list-unit-files --state=enabled

# View service file
systemctl cat nginx.service

# Edit override
systemctl edit nginx.service           # Creates override.conf drop-in
```

**Unit file anatomy (`/etc/systemd/system/myapp.service`):**
```ini
[Unit]
Description=My Application
After=network.target
Requires=postgresql.service

[Service]
Type=simple
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/start.sh
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment=ENV=production

[Install]
WantedBy=multi-user.target
```

**Restart policies:**

| Policy | When it restarts |
|--------|-----------------|
| `no` | Never |
| `on-success` | Clean exit (code 0) |
| `on-failure` | Non-zero exit, signal, timeout |
| `on-abnormal` | Signal, timeout |
| `on-abort` | Unhandled signal |
| `always` | Always (except `systemctl stop`) |

### ⭐ Q: journalctl usage for log analysis?

```bash
journalctl -u nginx                    # Logs for a service
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -f                 # Follow (like tail -f)
journalctl -p err                      # Priority: emerg,alert,crit,err,warning,notice,info,debug
journalctl -b                          # Current boot only
journalctl -b -1                       # Previous boot
journalctl --disk-usage                # Check journal size
journalctl --vacuum-size=500M          # Reclaim space
journalctl -o json-pretty -u nginx     # JSON output
```

---

## Package Management

### ⭐ Q: Explain package management across distributions.

| Task | Debian/Ubuntu (apt) | RHEL/CentOS (dnf/yum) |
|------|--------------------|-----------------------|
| Install | `apt install nginx` | `dnf install nginx` |
| Remove | `apt remove nginx` | `dnf remove nginx` |
| Search | `apt search nginx` | `dnf search nginx` |
| Update repos | `apt update` | `dnf check-update` |
| Upgrade all | `apt upgrade` | `dnf upgrade` |
| Info | `apt show nginx` | `dnf info nginx` |
| List installed | `dpkg -l` | `rpm -qa` |
| Which package owns a file | `dpkg -S /path/file` | `rpm -qf /path/file` |
| Clean cache | `apt clean` | `dnf clean all` |

---

## Performance Troubleshooting

### 🔥 Q: A server is slow. Walk me through your troubleshooting methodology.

**USE Method (Utilization, Saturation, Errors) by Brendan Gregg:**

```bash
# 1. LOAD AVERAGE — Quick health check
uptime
# load avg: 12.50, 11.30, 10.80 (1min, 5min, 15min)
# Compare with number of CPUs: nproc

# 2. CPU
top                          # Real-time view
mpstat -P ALL 1              # Per-CPU utilization
pidstat 1                    # Per-process CPU

# Key: %usr (user), %sys (kernel), %iowait (waiting for I/O), %steal (VM)

# 3. MEMORY
free -h                      # Memory overview
vmstat 1                     # Virtual memory stats (si/so = swap in/out)
slabtop                      # Kernel slab cache

# 4. DISK I/O
iostat -xz 1                 # Disk I/O stats
iotop                        # Per-process I/O
# Key: await (avg I/O time), %util (disk busy %)

# 5. NETWORK
sar -n DEV 1                 # Network interface stats
ss -tunapl                   # Socket stats (replaces netstat)
nstat                        # Network counters

# 6. PROCESSES
ps auxf                      # Process tree
strace -p <pid> -c           # System call summary
lsof -p <pid>                # Open files/sockets
```

### 🔥 Q: How do you investigate high CPU usage?

```bash
# 1. Identify the process
top -c                       # Press P to sort by CPU
# Or:
ps aux --sort=-%cpu | head

# 2. Profile the process
strace -p <pid> -c -S time   # System calls taking most time
perf top -p <pid>             # CPU profiling (if perf installed)

# 3. For Java/Python apps
jstack <pid>                  # Java thread dump
py-spy top --pid <pid>        # Python profiling

# 4. Check if it's user or kernel time
pidstat -p <pid> 1            # %usr vs %system

# 5. Common causes:
# - Infinite loops / busy waits
# - Regex backtracking
# - GC pressure (Java)
# - Compression/encryption workloads
# - Fork bombs: :(){ :|:& };:
```

### 🔥 Q: How do you investigate high I/O wait?

```bash
# 1. Confirm I/O wait
top                          # Check %wa (I/O wait)
vmstat 1                     # Check 'wa' column

# 2. Find which disk
iostat -xz 1                 # High %util, high await = saturated disk

# 3. Find which process
iotop -oP                    # Processes doing I/O
pidstat -d 1                 # Per-process I/O stats

# 4. Find what files the process is accessing
strace -p <pid> -e trace=read,write -c
lsof -p <pid>               # Open files

# 5. Common causes:
# - Excessive logging
# - Database without proper indexes
# - Swap thrashing (check vmstat si/so)
# - Undersized disks (HDD vs SSD)
```

---

## Log Management

### ⭐ Q: Important log files to know.

| Log File | Purpose |
|----------|---------|
| `/var/log/syslog` or `/var/log/messages` | General system logs |
| `/var/log/auth.log` or `/var/log/secure` | Authentication logs |
| `/var/log/kern.log` | Kernel logs |
| `/var/log/dmesg` | Boot-time hardware messages |
| `/var/log/apt/` or `/var/log/dnf.log` | Package manager logs |
| `/var/log/nginx/` | Web server logs |
| `/var/log/cron` | Cron job logs |
| `/var/log/audit/audit.log` | SELinux / audit logs |

```bash
# Real-time log monitoring
tail -f /var/log/syslog
journalctl -f

# Search logs
grep -r "error" /var/log/ --include="*.log"
zgrep "error" /var/log/syslog.*.gz    # Search compressed rotated logs
```

---

## Kernel & Boot Process

### ⭐ Q: What are kernel parameters and how to tune them?

```bash
# View all kernel parameters
sysctl -a

# View specific parameter
sysctl net.ipv4.ip_forward

# Set temporarily
sysctl -w net.ipv4.ip_forward=1

# Set permanently in /etc/sysctl.conf or /etc/sysctl.d/*.conf
net.ipv4.ip_forward = 1
net.core.somaxconn = 65535
vm.swappiness = 10
fs.file-max = 2097152
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535

# Apply
sysctl -p
```

**Commonly tuned parameters for production:**

| Parameter | Default | Tuned | Purpose |
|-----------|---------|-------|---------|
| `net.core.somaxconn` | 128 | 65535 | Max socket listen backlog |
| `net.ipv4.tcp_max_syn_backlog` | 128 | 65535 | SYN queue size |
| `fs.file-max` | 65536 | 2097152 | Max open files system-wide |
| `vm.swappiness` | 60 | 10 | Reduce swap usage |
| `net.ipv4.ip_forward` | 0 | 1 | Enable routing (needed for containers) |
| `vm.overcommit_memory` | 0 | 1 | Redis/forks need this |

### 🔥 Q: What are cgroups and namespaces?

**These are the two Linux kernel features that make containers possible.**

**Namespaces** — Provide **isolation** (what a process can see):

| Namespace | Isolates |
|-----------|----------|
| PID | Process IDs |
| NET | Network stack (interfaces, routing, ports) |
| MNT | Mount points (filesystem) |
| UTS | Hostname and domain name |
| IPC | Inter-process communication |
| USER | User and group IDs |
| CGROUP | Cgroup root directory |
| TIME | Boot and monotonic clocks (since kernel 5.6) |

**Cgroups** — Provide **resource limits** (what a process can use):

| Resource | Controls |
|----------|----------|
| CPU | CPU shares, quotas, pinning |
| Memory | Memory limits, swap limits |
| Block I/O | Disk I/O bandwidth limits |
| Network | Network bandwidth (via tc) |
| PIDs | Max number of processes |

```bash
# View cgroup of a process
cat /proc/<pid>/cgroup

# View cgroup limits (cgroup v2)
cat /sys/fs/cgroup/<group>/memory.max
cat /sys/fs/cgroup/<group>/cpu.max
```

### 💡 Q: What's the difference between cgroups v1 and cgroups v2?

**Cgroups v2** (unified hierarchy, default since RHEL 8 / Ubuntu 21.10):

| Feature | cgroups v1 | cgroups v2 |
|---------|-----------|-----------|
| Hierarchy | Multiple independent hierarchies per controller | Single unified hierarchy |
| Location | `/sys/fs/cgroup/<controller>/` | `/sys/fs/cgroup/` |
| Controllers | Spread across many dirs | All in one tree |
| Delegation | Unsafe, hard to secure | Safe, designed for unprivileged delegation |
| Memory accounting | Optional, can be off | Always-on, more accurate |
| Pressure Stall Information (PSI) | Not supported | Built-in (`cpu.pressure`, `memory.pressure`, `io.pressure`) |

```bash
# Check which version is in use
mount | grep cgroup

# cgroup v1 output:
# tmpfs on /sys/fs/cgroup type tmpfs
# cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup

# cgroup v2 output:
# cgroup2 on /sys/fs/cgroup type cgroup2

# Read PSI (cgroup v2 only) — shows resource contention
cat /sys/fs/cgroup/memory.pressure
# some avg10=2.50 avg60=1.20 avg300=0.80 total=54321000
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

**FAANG Tip:** Kubernetes 1.25+ defaults to cgroup v2. Older Docker versions may not support it — ensure runtime compatibility.

### ⭐ Q: Explain systemd timers vs cron. When would you use each?

**Cron** — Classic time-based job scheduler.

```bash
# Crontab format:
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of week (0 - 6) (Sunday=0)
# │ │ │ │ │
# * * * * * /path/to/script.sh

# Edit user crontab
crontab -e

# List current crontab
crontab -l

# Example: Run every day at 2:30 AM
30 2 * * * /opt/backup.sh
```

**Systemd Timers** — Modern, more powerful alternative:

```bash
# Create timer unit: /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=daily
OnCalendar=02:30
Persistent=true               # Run missed jobs on next boot
RandomizedDelaySec=300        # Jitter to avoid thundering herd

[Install]
WantedBy=timers.target

# Matching service unit: /etc/systemd/system/backup.service
[Unit]
Description=Backup job

[Service]
Type=oneshot
ExecStart=/opt/backup.sh
User=backup
Group=backup

# Enable and start
systemctl enable --now backup.timer
systemctl list-timers         # View all timers
journalctl -u backup.service  # View logs
```

**When to use systemd timers:**
- Need dependency management (run after network, database, etc.)
- Want integrated logging via journald
- Need resource limits (cgroup integration)
- Require persistent mode (run missed jobs)
- Random delay / spread load across time window

**When to use cron:**
- Simple time-based jobs
- Portability across systems (even non-systemd)
- User-level jobs (`crontab -e`)
- Legacy compatibility

---

## Security Hardening

### ⭐ Q: How do you harden a Linux server?

**Essential hardening checklist:**

1. **SSH Hardening** (`/etc/ssh/sshd_config`):
   ```
   PermitRootLogin no
   PasswordAuthentication no          # Use keys only
   MaxAuthTries 3
   AllowUsers deploy admin
   Port 2222                           # Non-standard port
   Protocol 2
   ```

2. **Firewall** — Enable and configure iptables/nftables/firewalld. Default deny policy.

3. **Automatic security updates:**
   ```bash
   # Ubuntu
   apt install unattended-upgrades
   dpkg-reconfigure unattended-upgrades
   ```

4. **Disable unused services:**
   ```bash
   systemctl list-unit-files --state=enabled
   systemctl disable --now <unused-service>
   ```

5. **File integrity monitoring:** Use AIDE or Tripwire.

6. **Audit logging:**
   ```bash
   auditctl -w /etc/passwd -p wa -k passwd_changes
   auditctl -w /etc/shadow -p wa -k shadow_changes
   ```

7. **Kernel hardening** (`/etc/sysctl.d/99-security.conf`):
   ```
   net.ipv4.conf.all.rp_filter = 1          # Reverse path filtering
   net.ipv4.conf.all.accept_redirects = 0
   net.ipv4.conf.all.send_redirects = 0
   net.ipv4.icmp_echo_ignore_broadcasts = 1
   kernel.randomize_va_space = 2             # ASLR
   ```

8. **SELinux / AppArmor** — Keep enabled, don't disable. Learn to write policies.

### ⭐ Q: What is SELinux?

**Security-Enhanced Linux** — Mandatory Access Control (MAC) system that confines processes to the minimum required permissions.

```bash
# Check status
getenforce                   # Enforcing, Permissive, or Disabled
sestatus

# Modes:
# Enforcing  — Blocks and logs violations
# Permissive — Logs but doesn't block (good for debugging)
# Disabled   — Off completely

# Temporary mode change
setenforce 0                 # Permissive
setenforce 1                 # Enforcing

# View context
ls -Z /var/www/html/
# unconfined_u:object_r:httpd_sys_content_t:s0

# Fix context after moving files
restorecon -Rv /var/www/html/

# Check for AVC denials
ausearch -m AVC -ts recent
sealert -a /var/log/audit/audit.log
```

### 💡 Q: What are Linux capabilities?

**Capabilities** split root's all-or-nothing privileges into fine-grained units. Instead of SUID binaries running as full root, they can hold only the specific capabilities they need.

```bash
# View capabilities of a file
getcap /usr/bin/ping
# /usr/bin/ping = cap_net_raw+ep

# Common capabilities:
# CAP_NET_BIND_SERVICE   — Bind to ports < 1024
# CAP_NET_RAW            — Use raw sockets (ping)
# CAP_SYS_ADMIN          — Mount filesystems, many admin tasks
# CAP_SYS_TIME           — Set system clock
# CAP_DAC_OVERRIDE       — Bypass file read/write/execute checks
# CAP_CHOWN              — Change file ownership

# Grant capability to a binary
setcap cap_net_bind_service=+ep /usr/local/bin/myapp

# View capabilities of a running process
getpcaps <pid>
cat /proc/<pid>/status | grep Cap

# Remove capabilities
setcap -r /usr/local/bin/myapp
```

**FAANG Tip:** Modern containers (Docker, Kubernetes) drop most capabilities by default. Use `--cap-add` sparingly — prefer the principle of least privilege.

### ⭐ Q: How do you handle secrets securely on Linux?

```bash
# 1. Never store secrets in:
# - Environment variables (visible in /proc/<pid>/environ)
# - Command line args (visible in ps)
# - Config files with 644 permissions
# - Git repos

# 2. Use secret management:
# - Vault (HashiCorp)
# - AWS Secrets Manager / Parameter Store
# - GCP Secret Manager
# - Kubernetes Secrets (with encryption at rest)

# 3. File permissions
chmod 600 /etc/myapp/secret.key       # Owner read/write only
chown myapp:myapp /etc/myapp/secret.key

# 4. Encrypted filesystems
# Use LUKS for disk encryption, or encrypted filesystems like eCryptfs

# 5. Tmpfs for secrets in memory (never touches disk)
mount -t tmpfs -o size=10M,mode=700 tmpfs /mnt/secrets
echo "secret_value" > /mnt/secrets/api_key
# Disappears on reboot

# 6. systemd LoadCredential (systemd 247+)
# Passes secrets to services without exposing in ps/environ
[Service]
LoadCredential=api_key:/run/secrets/api_key
# Access in service via $CREDENTIALS_DIRECTORY/api_key
```

---

## Containers: Namespaces & Cgroups Deep Dive

### 💡 Q: How would you manually create a container using namespaces?

```bash
# This demonstrates what Docker/containerd do under the hood

# 1. Create new namespaces with unshare
unshare --pid --net --mount --uts --ipc --fork /bin/bash
# Now you're in a new namespace

# 2. Inside the new namespace:
# - PID namespace: you are PID 1
echo $$        # Shows 1

# - UTS namespace: set hostname without affecting host
hostname container-demo

# - Mount namespace: create isolated filesystem
mkdir -p /tmp/container-root
# In real containers, this would be an extracted image layer
mount --bind /tmp/container-root /tmp/container-root
chroot /tmp/container-root /bin/bash

# - Network namespace: isolated network stack
ip link add veth0 type veth peer name veth1
ip link set veth1 netns <pid>
ip addr add 10.0.0.1/24 dev veth0
ip link set veth0 up
```

### 💡 Q: How do you limit container resource usage with cgroups v2?

```bash
# Create a cgroup for a container
mkdir /sys/fs/cgroup/mycontainer

# Set memory limit: 512MB
echo "536870912" > /sys/fs/cgroup/mycontainer/memory.max

# Set CPU limit: 50% of one core (500ms per 1000ms period)
echo "50000 100000" > /sys/fs/cgroup/mycontainer/cpu.max

# Set I/O weight (100-10000, default 100)
echo "500" > /sys/fs/cgroup/mycontainer/io.weight

# Add process to cgroup
echo <pid> > /sys/fs/cgroup/mycontainer/cgroup.procs

# Monitor memory usage
cat /sys/fs/cgroup/mycontainer/memory.current
cat /sys/fs/cgroup/mycontainer/memory.peak

# Check if memory limit was hit (OOM)
cat /sys/fs/cgroup/mycontainer/memory.events
# oom 0
# oom_kill 0
```

### ⭐ Q: What is the OOM score and how does the kernel decide which process to kill?

```bash
# Each process has an oom_score (0-1000)
# Higher score = more likely to be killed

# View OOM score
cat /proc/<pid>/oom_score

# Formula (simplified):
# oom_score = (process_memory_MB * 1000 / total_memory_MB) + oom_score_adj

# Adjust score (-1000 to +1000)
echo -500 > /proc/<pid>/oom_score_adj    # Less likely to be killed
echo 500 > /proc/<pid>/oom_score_adj     # More likely to be killed
echo -1000 > /proc/<pid>/oom_score_adj   # Never killed (except PID 1)

# Check OOM killer activity
dmesg | grep -i "killed process"
journalctl -k | grep -i oom

# Common OOM causes:
# 1. Memory leak in application
# 2. No swap configured
# 3. Cgroup memory limit too low
# 4. Sudden traffic spike
# 5. Large file caching exhausting memory
```

### 💡 Q: Explain `/proc` and `/sys` — what's the difference?

**`/proc`** — Virtual filesystem exposing **kernel and process information** (read-mostly):

```bash
/proc/<pid>/           # Per-process info
  ├── cmdline          # Command line with args
  ├── environ          # Environment variables
  ├── status           # Process state, memory, etc.
  ├── fd/              # Open file descriptors
  ├── maps             # Memory mappings
  └── limits           # Resource limits (ulimit)

/proc/sys/             # Kernel tunables (sysctl)
/proc/meminfo          # System memory stats
/proc/cpuinfo          # CPU info
/proc/loadavg          # Load averages
/proc/net/             # Network statistics
```

**`/sys`** — Virtual filesystem exposing **kernel objects, drivers, and hardware** (more structured, read-write):

```bash
/sys/class/            # Device classes (net, block, etc.)
/sys/block/            # Block devices
/sys/devices/          # Device tree
/sys/fs/cgroup/        # Cgroup v2 root
/sys/kernel/           # Kernel parameters
/sys/module/           # Loaded kernel modules
```

**Key difference:** `/proc` is process-centric and legacy; `/sys` is device/kernel-centric and more modern/structured.

---

## Advanced Troubleshooting Scenarios

### ⭐ Q: "Disk full" but `df` shows space available. What happened?

**Cause:** Deleted files still held open by processes.

```bash
# Files deleted while a process still has them open don't free space
# The inode and data blocks stay allocated until the process closes the FD

# Find deleted files held open
lsof +L1
# Or:
lsof | grep '(deleted)'

# Example output:
# nginx   1234 root   7w   REG   8,1   2G  (deleted) /var/log/nginx/access.log

# Fix options:
# 1. Restart the process holding the file
systemctl restart nginx

# 2. Or truncate the file descriptor
# Find the FD number (7 in example above)
> /proc/1234/fd/7
# This frees the space immediately without restarting

# 3. Kill the process (last resort)
kill 1234
```

### ⭐ Q: Load average is high but CPU usage is low. What's happening?

```bash
# Load average counts:
# 1. Processes using CPU (R state)
# 2. Processes waiting for I/O (D state)  ← This is the culprit

# 1. Check load average
uptime
# load average: 15.2, 12.8, 10.5    (but we have 4 CPUs)

# 2. Look at CPU usage
top
# %Cpu(s):  10.2 us,  2.5 sy,  0.0 ni, 70.0 id, 17.0 wa, 0.0 hi, 0.3 si
# ↑ High 'wa' (I/O wait) confirms disk bottleneck

# 3. Find processes in 'D' state (uninterruptible sleep)
ps aux | awk '$8 ~ /D/'
# Or:
ps -eo state,pid,cmd | grep "^D"

# 4. Check disk I/O
iostat -xz 1
# High %util, high await = saturated disk

# 5. Find the process doing I/O
iotop -oP

# Common causes:
# - Slow disk (HDD on a busy server)
# - Network filesystem timeout (NFS hang)
# - Failing disk (check dmesg for I/O errors)
# - Swap thrashing (vmstat si/so)
```

### ⭐ Q: A process won't die even with `kill -9`. What now?

```bash
# Check process state
ps aux | grep <pid>

# If state is 'D' (uninterruptible sleep):
# The process is waiting for kernel I/O and CANNOT be killed

# Common causes of unkillable D state:
# 1. NFS mount that's gone away
# 2. Disk hardware failure
# 3. Buggy device driver
# 4. Filesystem in bad state

# Diagnosis:
# 1. Check what the process is waiting for
cat /proc/<pid>/wchan        # Kernel wait channel
cat /proc/<pid>/stack        # Kernel stack trace

# 2. Check for I/O errors
dmesg | tail -50
journalctl -k -n 50

# 3. If it's an NFS issue:
mount | grep nfs
# Try unmounting with force (may not work if process is stuck)
umount -f -l /mnt/nfs        # -l = lazy unmount

# 4. Last resort: reboot
# There is no way to kill a process in D state waiting on broken I/O
# This is by design — the kernel is waiting for hardware to respond
```

### ⭐ Q: Server has high memory usage but no large processes. What's using it?

```bash
# Linux uses free RAM for caching — this is GOOD, not a problem
# But sometimes you need to distinguish real usage from cache

# 1. Check memory breakdown
free -h
#               total    used    free    shared  buff/cache   available
# Mem:           16G      2G     1G      256M      13G         13.5G
# ↑ 13G cached, but 13.5G "available" = no problem

# 2. If "available" is low, check slab cache (kernel memory)
slabtop
# Press 's' to sort by size
# Look for large caches: dentry, inode_cache, buffer_head

# 3. Check for memory leaks in kernel modules
cat /proc/meminfo | grep -i slab
# Slab:             3145728 kB     ← If this is huge, kernel leak

# 4. Check shared memory segments
ipcs -m
# If large segments exist, find which process
lsof | grep "/dev/shm"

# 5. Check for tmpfs usage
df -h | grep tmpfs
du -sh /dev/shm/*
du -sh /tmp/*

# 6. Check huge pages
cat /proc/meminfo | grep Huge
# HugePages_Total:    1024
# HugePages_Free:      512
# HugePages_Rsvd:        0
# Hugepagesize:       2048 kB
# = 1GB allocated to huge pages

# 7. Memory cgroup limits (if in container)
cat /sys/fs/cgroup/memory.current
cat /sys/fs/cgroup/memory.max
```

### ⭐ Q: How do you troubleshoot "too many open files" errors?

```bash
# Error: "error: too many open files"
# Cause: Process hit file descriptor limit

# 1. Check current limits
ulimit -n             # Current shell
ulimit -Hn            # Hard limit
ulimit -Sn            # Soft limit

# 2. Check process-specific limit
cat /proc/<pid>/limits | grep "open files"

# 3. Check current FD usage
ls /proc/<pid>/fd | wc -l
lsof -p <pid> | wc -l

# 4. Find what files/sockets are open
lsof -p <pid>
# Look for:
# - Too many sockets (network leak)
# - Too many log files (improper log rotation)
# - Leaked file descriptors (application bug)

# 5. System-wide limits
cat /proc/sys/fs/file-max        # Max open files system-wide
cat /proc/sys/fs/file-nr         # Currently open files
# Format: <allocated>  <unused>  <max>

# 6. Fix: Increase limits temporarily
ulimit -n 65535

# 7. Fix: Increase limits permanently
# /etc/security/limits.conf
myapp soft nofile 65535
myapp hard nofile 65535
* soft nofile 65535
* hard nofile 65535

# For systemd services:
# /etc/systemd/system/myapp.service
[Service]
LimitNOFILE=65535

# Then reload
systemctl daemon-reload
systemctl restart myapp

# 8. Increase system-wide limit
sysctl -w fs.file-max=2097152
# Persistent: add to /etc/sysctl.conf
fs.file-max = 2097152
```

### 💡 Q: How do you find which process is using a deleted library file after upgrade?

```bash
# After upgrading a shared library (glibc, OpenSSL, etc.),
# running processes still use the old version from memory
# This can be a security risk or cause crashes

# 1. Find processes using deleted files
lsof | grep '(deleted)'
# Or specific to libraries:
lsof | grep -E '\.so.*deleted'

# 2. Find which processes need restart after library upgrade
checkrestart          # Debian/Ubuntu (debian-goodies package)
needs-restarting      # RHEL/CentOS

# 3. Check specific process
lsof -p <pid> | grep DEL
cat /proc/<pid>/maps | grep deleted

# 4. Restart affected services
systemctl restart <service>

# 5. For critical libraries (libc, libssl), may need to restart many services
# Some distros provide a helper:
needrestart           # Interactive tool showing what needs restart
```

---

## Command-Line One-Liners & Golf

### 💡 Q: Common one-liner commands asked in screening rounds.

```bash
# Find top 10 largest files
find / -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head -10

# Find top 10 largest directories
du -ah / 2>/dev/null | sort -rh | head -10

# Count unique IPs in access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# Find files modified in last 24 hours
find /var/log -type f -mtime -1

# Kill all processes matching a name
pkill -f pattern
# Or:
ps aux | grep pattern | awk '{print $2}' | xargs kill

# Monitor live network connections
watch -n 1 'ss -tunapl | grep ESTABLISHED'

# Real-time disk I/O per process
iotop -oPa

# Count lines of code in a project
find . -name '*.py' | xargs wc -l | sort -rn

# Find all SUID binaries (security audit)
find / -perm -4000 -type f 2>/dev/null

# Check which process is listening on a port
lsof -i :8080
# Or:
ss -tulpn | grep :8080

# Continuously ping and log only failures
ping -i 1 8.8.8.8 | grep --line-buffered -v 'time=' >> ping_failures.log

# Show TCP connections per IP
ss -tan | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# Recursively find and replace in files
find . -name '*.txt' -exec sed -i 's/old/new/g' {} +

# Monitor log file for keyword and send alert
tail -f /var/log/app.log | grep --line-buffered ERROR | while read line; do
  echo "Error detected: $line" | mail -s "Alert" admin@example.com
done

# Backup with progress and compression
tar czf - /data | pv -s $(du -sb /data | awk '{print $1}') > backup.tar.gz

# Show processes sorted by memory
ps aux --sort=-%mem | head

# Generate random password
openssl rand -base64 32

# Show open network connections with process names
netstat -tulpn
# Or modern:
ss -tulpn

# Recursively change ownership
chown -R user:group /path

# Find zombie processes
ps aux | awk '$8 ~ /Z/ {print}'

# Compress old logs
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;

# Show directory sizes, sorted
du -h --max-depth=1 | sort -rh

# Check if a remote port is open
timeout 2 bash -c "</dev/tcp/example.com/443" && echo "Open" || echo "Closed"

# Extract IP addresses from text
grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' file.txt

# Monitor CPU usage per core
mpstat -P ALL 1

# Show systemd boot time breakdown
systemd-analyze blame
systemd-analyze critical-chain

# Find files with spaces in name
find . -name "* *" -type f

# Recursive case-insensitive grep
grep -ri "pattern" /path/

# Show largest packages installed (Debian/Ubuntu)
dpkg-query -Wf '${Installed-Size}\t${Package}\n' | sort -rn | head

# Show largest packages (RHEL/CentOS)
rpm -qa --queryformat '%{SIZE} %{NAME}\n' | sort -rn | head

# Monitor memory usage per process
watch -n 1 'ps aux --sort=-%mem | head -20'

# Clear systemd journal
journalctl --rotate
journalctl --vacuum-time=1d

# Test disk write speed
dd if=/dev/zero of=/tmp/test bs=1M count=1024 oflag=direct

# Test disk read speed
dd if=/tmp/test of=/dev/null bs=1M count=1024 iflag=direct

# Show established connections count by state
ss -tan | awk '{print $1}' | sort | uniq -c

# Find files owned by user
find / -user username 2>/dev/null

# Show kernel messages for a device
dmesg | grep -i sda

# Show routing table
ip route show
# Or:
route -n

# Flush DNS cache (systemd-resolved)
resolvectl flush-caches

# Show all listening services
systemctl list-units --type=service --state=running

# Check SSL certificate expiry
echo | openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -dates
```

---

## Scenario-Based Questions

### 🔥 Q: A user cannot SSH into a server. Troubleshoot.

```bash
# 1. Network connectivity
ping <server>
telnet <server> 22
nc -zv <server> 22

# 2. SSH service running?
systemctl status sshd

# 3. Firewall blocking?
iptables -L -n | grep 22
firewall-cmd --list-ports

# 4. Check SSH config
cat /etc/ssh/sshd_config | grep -E "AllowUsers|DenyUsers|PasswordAuth|PermitRoot"

# 5. Check user account
passwd -S <username>          # Account locked?
chage -l <username>           # Password expired?
grep <username> /etc/passwd   # Shell set to /sbin/nologin?

# 6. Check logs
journalctl -u sshd -n 50
tail -50 /var/log/auth.log

# 7. Key permissions (client-side)
# ~/.ssh/              → 700
# ~/.ssh/id_rsa        → 600
# ~/.ssh/authorized_keys → 644
# /home/<user>         → 755 (not 777!)

# 8. SELinux
getenforce
ausearch -m AVC -ts recent | grep ssh

# 9. TCP Wrappers
cat /etc/hosts.allow
cat /etc/hosts.deny

# 10. Disk full? (can prevent login)
df -h
```

### 🔥 Q: Server is unreachable. How do you troubleshoot?

```bash
# From another machine:
# 1. Is it the network or the server?
ping <server-ip>
traceroute <server-ip>
mtr <server-ip>

# 2. Is the port open?
nmap -p 22,80,443 <server-ip>
nc -zv <server-ip> 22

# 3. DNS issue?
dig <hostname>
nslookup <hostname>

# 4. Via out-of-band access (IPMI/iLO/console):
# Check if server is up
# Check network interfaces: ip a
# Check routes: ip r
# Check firewall: iptables -L -n
# Check if NIC is up: ethtool eth0
# Check logs: dmesg | tail, journalctl -xn
```

### 🔥 Q: A process is consuming 100% CPU. What do you do?

```bash
# 1. Identify the process
top -c                      # Find PID and command
ps aux --sort=-%cpu | head

# 2. Understand what it's doing
strace -p <pid> -c          # System calls
ls -la /proc/<pid>/fd       # Open file descriptors
cat /proc/<pid>/status      # Process details

# 3. Decision tree:
# - Known batch job? Let it finish or reschedule.
# - Application bug? Capture thread dump, restart.
# - Malicious? Kill it, investigate how it got there.

# 4. If you need to limit it immediately:
renice +19 <pid>            # Lower priority
cpulimit -p <pid> -l 50     # Limit to 50% CPU
# Or use cgroups for hard limits

# 5. Kill if necessary
kill <pid>                   # SIGTERM first
kill -9 <pid>                # SIGKILL if unresponsive
```

### 🔥 Q: Memory is full but applications are being OOM-killed. How do you investigate?

```bash
# 1. Check current memory state
free -h
cat /proc/meminfo

# 2. View OOM killer activity
dmesg | grep -i "killed process"
journalctl -xb | grep -i oom
grep -i oom /var/log/syslog

# 3. Identify memory hogs
ps aux --sort=-%mem | head -20
top -o %MEM

# 4. Check if swap is enabled and being used
swapon --show
vmstat 1 5
# Watch si/so (swap in/out) — high values = swap thrashing

# 5. Check for memory leaks
# Track process memory over time
watch -n 1 'ps aux --sort=-%mem | head'
# Or use pmem:
pmap -x <pid> | tail -1

# 6. Check slab memory (kernel)
slabtop
cat /proc/meminfo | grep -i slab

# 7. Check cgroup memory limits (if containerized)
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/memory.current
cat /sys/fs/cgroup/memory.events | grep oom

# 8. Adjust OOM preferences for critical processes
echo -1000 > /proc/<critical-pid>/oom_score_adj

# 9. Long-term fixes:
# - Add more RAM
# - Enable/increase swap
# - Fix memory leaks in application
# - Tune cgroup limits
# - Use memory profiling (valgrind, heaptrack)
```

### ⭐ Q: SSH login is slow. How do you troubleshoot?

```bash
# Common causes: DNS timeout, GSSAPI auth, slow home directory

# 1. Test with verbose output
ssh -vvv user@host
# Look for delays at specific steps

# 2. Disable DNS reverse lookup (server-side)
# /etc/ssh/sshd_config
UseDNS no
systemctl restart sshd

# 3. Disable GSSAPI authentication (client-side)
ssh -o GSSAPIAuthentication=no user@host
# Permanent: ~/.ssh/config
Host *
  GSSAPIAuthentication no

# 4. Check if home directory is on NFS or slow storage
df -h ~
mount | grep home

# 5. Check PAM configuration
# /etc/pam.d/sshd
# Comment out slow modules like pam_motd.so with network calls

# 6. Check DNS resolution
time host $(hostname)
# Should be instant

# 7. Check for slow .bashrc / .bash_profile
time bash -c "source ~/.bashrc"

# 8. Network latency
ping -c 5 <server>
mtr <server>

# 9. Check logs
tail -f /var/log/auth.log        # Ubuntu
tail -f /var/log/secure          # RHEL

# Quick fixes:
# Client: ssh -o GSSAPIAuthentication=no -o UseDNS=no user@host
# Server: Add to /etc/ssh/sshd_config:
#   UseDNS no
#   GSSAPIAuthentication no
```

### 💡 Q: How do you hunt down a memory leak in a long-running process?

```bash
# 1. Confirm it's a leak (memory usage grows over time)
watch -n 5 'ps aux | grep <process>'
# Or:
pidstat -r -p <pid> 1

# 2. Check memory maps
pmap -x <pid>
cat /proc/<pid>/smaps
# Look for growing anonymous memory regions

# 3. Use valgrind (requires restarting process)
valgrind --leak-check=full --log-file=valgrind.log ./myapp

# 4. Use heaptrack (C/C++)
heaptrack ./myapp
heaptrack_gui heaptrack.myapp.*.gz

# 5. For Python: memory_profiler
pip install memory-profiler
python -m memory_profiler script.py

# Or py-spy:
py-spy dump --pid <pid>

# 6. For Java: heap dump
jmap -dump:live,format=b,file=heap.bin <pid>
# Analyze with Eclipse MAT or jhat

# 7. For Go: pprof
curl http://localhost:6060/debug/pprof/heap > heap.prof
go tool pprof heap.prof

# 8. Monitor malloc stats (glibc)
gdb -p <pid>
(gdb) call malloc_stats()

# 9. Check for file descriptor leaks (can appear as memory leak)
ls /proc/<pid>/fd | wc -l
lsof -p <pid> | wc -l

# 10. Common leak patterns:
# - Growing heap without corresponding free()
# - Unclosed file descriptors
# - Retained references (Java, Python)
# - Memory mapped files not unmapped
# - Growing kernel slab cache (kernel memory leak)
```

---

## Key Commands Cheat Sheet

### System Info
```bash
uname -a                    # Kernel version
cat /etc/os-release         # OS info
hostname -I                 # IP addresses
uptime                      # Load averages
nproc                       # Number of CPUs
lscpu                       # CPU details
dmidecode -t memory         # Physical RAM info
```

### File Operations
```bash
find /path -name "*.log" -mtime +7 -delete    # Delete files older than 7 days
find / -type f -size +100M                      # Files larger than 100MB
rsync -avz --progress src/ dest/                # Sync directories
tar czf backup.tar.gz /path/                    # Create compressed archive
```

### Network
```bash
ss -tunapl                  # All sockets with process info
ip addr show                # Network interfaces
ip route show               # Routing table
curl -vvv https://example.com   # Verbose HTTP request
tcpdump -i eth0 port 80    # Packet capture
mtr <host>                  # Network path analysis
```

### Disk
```bash
lsblk                       # Block devices
blkid                       # Block device attributes
fdisk -l                    # Partition info
mount | column -t            # Mounted filesystems
fstrim -av                  # TRIM SSDs
```

### Text Processing
```bash
awk '{print $1}' file       # Print first column
sed -i 's/old/new/g' file   # Replace in-place
cut -d: -f1 /etc/passwd     # Cut by delimiter
sort | uniq -c | sort -rn   # Count unique occurrences
grep -rn "pattern" /path/   # Recursive grep with line numbers
```

---

## Key Resources

- **Brendan Gregg's Linux Performance Tools** — https://www.brendangregg.com/linuxperf.html
- **Linux Performance Observability Tools** — https://www.brendangregg.com/Perf/linux_observability_tools.png
- **USE Method** — https://www.brendangregg.com/usemethod.html
- **Linux Tracing Systems** — https://www.brendangregg.com/blog/2015-07-08/choosing-a-linux-tracer.html
- **The Linux Command Line (book)** — William Shotts
- **How Linux Works (book)** — Brian Ward
- **Systems Performance (book)** — Brendan Gregg
- **Linux Performance** — https://www.kernel.org/doc/html/latest/
- **Linux man pages** — `man <command>` is your best friend
- **RHCSA/RHCE Certification Guides** — Great structured learning path
- **CGroups v2 Documentation** — https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html
- **LWN.net** — In-depth Linux kernel articles — https://lwn.net/
- **Linux Insides** — Kernel internals book — https://0xax.gitbooks.io/linux-insides/
