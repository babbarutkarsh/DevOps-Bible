# Linux — DevOps Interview Preparation

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
- [Scenario-Based Questions](#scenario-based-questions)
- [Key Commands Cheat Sheet](#key-commands-cheat-sheet)

---

## Linux Fundamentals

### Q: What is the Linux boot process?

**Answer:**

1. **BIOS/UEFI** — Firmware performs POST (Power-On Self-Test), detects hardware, looks for a bootable device.
2. **Bootloader (GRUB2)** — Loads the kernel into memory. GRUB reads `/boot/grub2/grub.cfg` to know which kernel and initramfs to load.
3. **Kernel Initialization** — Kernel decompresses itself, initializes hardware, mounts the initial RAM filesystem (initramfs).
4. **initramfs** — Temporary root filesystem that contains drivers needed to mount the real root filesystem.
5. **Init System (systemd/SysVinit)** — PID 1 process starts. Systemd reads `/etc/systemd/system/default.target` and starts services in parallel.
6. **Runlevel / Target** — System reaches the configured target (e.g., `multi-user.target` for CLI, `graphical.target` for GUI).
7. **Login Prompt** — getty or display manager presents the login interface.

### Q: What is the difference between a process and a thread?

| Aspect | Process | Thread |
|--------|---------|--------|
| Memory | Own address space | Shares parent process's address space |
| Creation | `fork()` — heavier | `pthread_create()` — lighter |
| Communication | IPC (pipes, sockets, shared mem) | Direct shared memory access |
| Crash Impact | Isolated — one crash doesn't affect others | One thread crash can kill the entire process |
| Context Switch | Expensive (TLB flush, page table swap) | Cheap (same address space) |

### Q: Explain Linux file system hierarchy.

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

### Q: What are inodes?

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

### Q: Hard link vs Soft link?

| Feature | Hard Link | Soft Link (Symlink) |
|---------|-----------|---------------------|
| Inode | Same as target | Different inode |
| Cross filesystem | No | Yes |
| Link to directory | No (usually) | Yes |
| Target deleted | Still works (data persists) | Becomes dangling/broken |
| Command | `ln target link` | `ln -s target link` |

---

## File System & Storage

### Q: Explain LVM and its components.

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

### Q: What is RAID? Explain levels.

| RAID Level | Min Disks | Redundancy | Performance | Usable Capacity |
|-----------|-----------|------------|-------------|-----------------|
| RAID 0 | 2 | None | Best read/write | 100% |
| RAID 1 | 2 | Mirroring | Good read, normal write | 50% |
| RAID 5 | 3 | 1 disk parity | Good read, slower write | (N-1)/N |
| RAID 6 | 4 | 2 disk parity | Good read, slower write | (N-2)/N |
| RAID 10 | 4 | Mirror + Stripe | Excellent | 50% |

**FAANG Tip:** RAID is not a backup strategy — it protects against disk failure, not data corruption, accidental deletion, or ransomware.

### Q: A disk is full. How do you troubleshoot?

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

### Q: Explain process states in Linux.

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

### Q: What are zombie processes and how to fix them?

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

### Q: Explain signals in Linux. Key signals to know:

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

### Q: What is the OOM Killer?

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

### Q: Explain Linux memory management (Virtual Memory, Pages, Swap).

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

### Q: What are huge pages and why use them?

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

### Q: How does DNS resolution work in Linux?

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

### Q: Explain iptables / nftables.

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

### Q: What is the difference between TCP and UDP?

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

### Q: Explain Linux file permissions in depth.

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

### Q: What is sudoers and how does it work?

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

### Q: Explain systemd and its key components.

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

### Q: journalctl usage for log analysis?

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

### Q: Explain package management across distributions.

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

### Q: A server is slow. Walk me through your troubleshooting methodology.

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

### Q: How do you investigate high CPU usage?

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

### Q: How do you investigate high I/O wait?

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

### Q: Important log files to know.

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

### Q: What are kernel parameters and how to tune them?

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

### Q: What are cgroups and namespaces?

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

---

## Security Hardening

### Q: How do you harden a Linux server?

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

### Q: What is SELinux?

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

---

## Scenario-Based Questions

### Q: A user cannot SSH into a server. Troubleshoot.

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

### Q: Server is unreachable. How do you troubleshoot?

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

### Q: A process is consuming 100% CPU. What do you do?

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
- **The Linux Command Line (book)** — William Shotts
- **How Linux Works (book)** — Brian Ward
- **Linux man pages** — `man <command>` is your best friend
- **RHCSA/RHCE Certification Guides** — Great structured learning path
