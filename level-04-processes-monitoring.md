# Level 4 — Processes & System Monitoring

> **Goal:** Understand how Linux manages processes, monitor system resources (CPU, RAM, disk I/O), manage foreground/background jobs, send signals, and troubleshoot runaway processes like a sysadmin.

> **Lab Environment:** WSL2 Ubuntu or VirtualBox VM

> **Prerequisite:** Complete Levels 1–3

---

## 4.1 — What Is a Process?

### Minimum Theory

A **process** is a running instance of a program. When you type `ls`, the shell creates a new process that runs the `ls` binary, produces output, and then exits.

Every process has:

| Attribute | Description |
|-----------|-------------|
| **PID** | Process ID — unique number assigned by the kernel |
| **PPID** | Parent Process ID — the PID of the process that created it |
| **UID** | User ID — which user owns the process |
| **State** | Running, Sleeping, Stopped, Zombie, etc. |
| **Priority** | How much CPU time it gets relative to others |
| **Memory** | How much RAM it uses |
| **Command** | The command/program that started it |

**📌 Important — Process hierarchy:**

Every process is created by another process (except PID 1). This creates a tree:

```
systemd (PID 1)
├── sshd (PID 500)
│   └── bash (PID 1200)    ← your shell
│       └── vim (PID 1250)  ← program you started
├── nginx (PID 600)
│   ├── nginx (PID 601)     ← worker
│   └── nginx (PID 602)     ← worker
├── cron (PID 700)
└── rsyslogd (PID 800)
```

**PID 1 — `systemd` (or `init` on older systems):**
- The first process started by the kernel at boot
- Parent of all other processes
- If PID 1 dies, the system crashes

**🎯 Interview Point:** "What is PID 1?"  
> PID 1 is the init system (usually `systemd` on modern Linux). It's the first userspace process started by the kernel. It starts and manages all other services and processes.

### Process States

| State | Code | Meaning |
|-------|------|---------|
| **Running** | `R` | Actively using CPU or ready to run |
| **Sleeping (Interruptible)** | `S` | Waiting for something (I/O, input, signal). Most processes are here. |
| **Sleeping (Uninterruptible)** | `D` | Waiting for I/O (disk, network). Cannot be interrupted. Often seen with NFS or disk issues. |
| **Stopped** | `T` | Paused by a signal (e.g., `Ctrl+Z`) |
| **Zombie** | `Z` | Process finished but parent hasn't collected its exit status. Takes no resources but occupies a PID. |

**💡 Admin Tip:** A few zombie processes are normal. Many zombies indicate a buggy parent process that isn't handling its children properly. You can't `kill` a zombie — you need to fix or kill the parent.

---

## 4.2 — Viewing Processes with `ps`

### `ps` — Process Status

```bash
ps
```
**What it does:** Shows processes in your current terminal session only.  
**Expected output:**
```
  PID TTY          TIME CMD
 1200 pts/0    00:00:00 bash
 1350 pts/0    00:00:00 ps
```

This is not very useful — you usually want to see ALL processes.

### Common `ps` Combinations

#### `ps aux` — BSD Style (Most Common)

```bash
ps aux
```

**What each column means:**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.3 169536 13280 ?        Ss   09:00   0:05 /sbin/init
root       500  0.0  0.1  15432  5680 ?        Ss   09:00   0:00 /usr/sbin/sshd
www-data   601  0.0  0.4  43780 17920 ?        S    09:01   0:10 nginx: worker process
abhishek  1200  0.0  0.1   8960  4720 pts/0    Ss   10:00   0:00 -bash
```

| Column | Meaning |
|--------|---------|
| `USER` | Process owner |
| `PID` | Process ID |
| `%CPU` | CPU usage percentage |
| `%MEM` | Memory usage percentage |
| `VSZ` | Virtual memory size (KB) — total memory the process can access |
| `RSS` | Resident Set Size (KB) — actual physical memory used |
| `TTY` | Terminal associated (`?` = no terminal, daemon process) |
| `STAT` | Process state (see below) |
| `START` | When the process started |
| `TIME` | Total CPU time consumed |
| `COMMAND` | The command that started the process |

**STAT column codes:**

| Code | Meaning |
|------|---------|
| `R` | Running |
| `S` | Sleeping (interruptible) |
| `D` | Sleeping (uninterruptible — usually disk I/O) |
| `T` | Stopped |
| `Z` | Zombie |
| `s` | Session leader |
| `l` | Multi-threaded |
| `+` | Foreground process group |
| `<` | High priority (not nice) |
| `N` | Low priority (nice) |

**Example:** `Ss` = Sleeping + session leader, `R+` = Running in foreground.

#### `ps -ef` — System V Style

```bash
ps -ef
```

```
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 09:00 ?        00:00:05 /sbin/init
root       500     1  0 09:00 ?        00:00:00 /usr/sbin/sshd
abhishek  1200   500  0 10:00 pts/0    00:00:00 -bash
```

| Column | Meaning |
|--------|---------|
| `UID` | User |
| `PID` | Process ID |
| `PPID` | Parent PID |
| `C` | CPU utilization |
| `STIME` | Start time |
| `TTY` | Terminal |
| `TIME` | CPU time |
| `CMD` | Command |

**📌 Important:** `ps -ef` shows PPID (parent PID), `ps aux` does not. Use `-ef` when you need to trace process parentage.

### Useful `ps` Filters

```bash
# Find a specific process
ps aux | grep nginx

# Avoid grep showing itself in results
ps aux | grep [n]ginx
# The [n] trick: shell expands [n]ginx literally, but grep sees regex [n]ginx
# which matches "nginx" but the grep process itself has "[n]ginx" not "nginx"

# Show processes for a specific user
ps -u www-data

# Show process tree (parent-child relationships)
ps -ef --forest

# Show only PID and command
ps -eo pid,comm

# Custom output — CPU and memory hogs
ps -eo pid,user,%cpu,%mem,comm --sort=-%cpu | head -15

# Show the top 10 memory consumers
ps -eo pid,user,%mem,rss,comm --sort=-%mem | head -11
```

**💡 Admin Tip:** This is one of the most useful one-liners for troubleshooting:
```bash
ps -eo pid,user,%cpu,%mem,comm --sort=-%cpu | head -11
```
Shows the top 10 CPU-consuming processes. Replace `-%cpu` with `-%mem` for memory.

### `pstree` — Process Tree

```bash
pstree
```
**What it does:** Shows processes as a tree, visualizing parent-child relationships.

```bash
pstree -p
```
**`-p`:** Show PIDs alongside process names.

```bash
pstree -p abhishek
```
Show the process tree rooted at user `abhishek`.

**Expected output:**
```
systemd(1)─┬─sshd(500)───sshd(1199)───bash(1200)───pstree(1400)
            ├─nginx(600)─┬─nginx(601)
            │            └─nginx(602)
            ├─cron(700)
            └─rsyslogd(800)
```

---

## 4.3 — Real-Time Monitoring with `top` and `htop`

### `top` — Real-Time Process Viewer

```bash
top
```

**What it does:** Shows a continuously updating view of system processes, sorted by CPU usage by default.

**Top section (system summary):**
```
top - 14:30:00 up 5 days,  3:00,  2 users,  load average: 0.15, 0.10, 0.05
Tasks: 128 total,   1 running, 127 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.3 us,  1.0 sy,  0.0 ni, 96.5 id,  0.2 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :  7953.2 total,  3245.8 free,  2100.4 used,  2607.0 buff/cache
MiB Swap:  2048.0 total,  2048.0 free,     0.0 used.  5520.8 avail Mem
```

**Understanding the summary:**

| Line | Field | Meaning |
|------|-------|---------|
| Line 1 | `up 5 days` | System uptime |
| Line 1 | `load average: 0.15, 0.10, 0.05` | CPU load: 1 min, 5 min, 15 min averages |
| Line 2 | `Tasks` | Total processes and their states |
| Line 3 | `us` | User space CPU time (your applications) |
| Line 3 | `sy` | System/kernel CPU time |
| Line 3 | `ni` | Nice (low priority) CPU time |
| Line 3 | `id` | Idle CPU time (higher = less busy) |
| Line 3 | `wa` | I/O wait (waiting for disk — high value = disk bottleneck) |
| Line 3 | `st` | Steal time (VM only — hypervisor took CPU away) |
| Line 4 | `Mem` | Physical RAM usage |
| Line 5 | `Swap` | Swap space usage |

**📌 Important — Load Average:**

Load average represents the number of processes waiting for CPU over time periods.

| Load Average | On 1-CPU system | On 4-CPU system |
|-------------|-----------------|-----------------|
| 0.5 | 50% utilized | 12.5% utilized |
| 1.0 | 100% utilized (threshold) | 25% utilized |
| 4.0 | 4x overloaded! | 100% utilized (threshold) |
| 8.0 | 8x overloaded! | 2x overloaded! |

**Rule of thumb:** Load average should be **at or below your CPU core count**.

```bash
# Check your CPU count
nproc
# OR
grep -c processor /proc/cpuinfo
```

**Interactive keys in `top`:**

| Key | Action |
|-----|--------|
| `q` | Quit |
| `h` | Help |
| `M` | Sort by memory usage |
| `P` | Sort by CPU usage (default) |
| `T` | Sort by cumulative time |
| `k` | Kill a process (enter PID) |
| `r` | Renice a process (change priority) |
| `u` | Filter by user |
| `1` | Toggle per-CPU display |
| `c` | Show full command path |
| `V` | Tree view |
| `f` | Select which fields to display |
| `W` | Save current settings |

**💡 Admin Tip:** Press `1` in `top` to see per-CPU utilization. This helps identify whether one core is pegged at 100% (single-threaded bottleneck) or all cores are busy.

---

### `htop` — Better Process Viewer

```bash
# Install if not present
sudo apt install htop    # Ubuntu
sudo dnf install htop    # Rocky/RHEL (requires EPEL)

htop
```

**What it does:** An improved version of `top` with:
- Colorful, visual CPU/memory bars
- Mouse support
- Easier process management
- Horizontal and vertical scrolling
- Tree view built-in
- Search/filter functionality

**Key differences from `top`:**

| Feature | `top` | `htop` |
|---------|-------|--------|
| Visual CPU/mem bars | No | Yes |
| Mouse support | No | Yes |
| Scroll horizontally | No | Yes |
| Kill without PID | No | Yes (select + F9) |
| Tree view | `V` key | `F5` key |
| Search | No | `F3` or `/` |
| Filter | `u` for user only | `F4` for any filter |
| Colors | Minimal | Full color |

**htop keyboard shortcuts:**

| Key | Action |
|-----|--------|
| `F1` | Help |
| `F2` | Setup (configure display) |
| `F3` or `/` | Search for a process |
| `F4` or `\` | Filter processes |
| `F5` | Tree view |
| `F6` | Sort by column |
| `F9` | Kill selected process |
| `F10` or `q` | Quit |
| `Space` | Tag (select) a process |
| `u` | Filter by user |

**✅ Best Practice:** Use `htop` for interactive monitoring. Use `ps` and `top` for scripting and when `htop` isn't installed (it's not part of default installs).

---

## 4.4 — Finding Processes: `pgrep` and `pidof`

### `pgrep` — Find PIDs by Name

```bash
pgrep nginx
```
**What it does:** Returns PIDs of all processes matching "nginx".  
**Expected output:**
```
600
601
602
```

| Option | Meaning | Example |
|--------|---------|---------|
| `-l` | Show PID and process name | `pgrep -l nginx` |
| `-a` | Show PID and full command | `pgrep -a nginx` |
| `-u user` | Match by user | `pgrep -u www-data` |
| `-c` | Count matching processes | `pgrep -c nginx` |
| `-f` | Match against full command line | `pgrep -f "python app.py"` |
| `-x` | Exact match only | `pgrep -x nginx` |

**Examples:**
```bash
# Count nginx worker processes
pgrep -c nginx

# Find all processes by a user
pgrep -la -u www-data

# Find a Python script by its full command
pgrep -af "python.*myapp"
```

### `pidof` — Get PID of a Running Program

```bash
pidof nginx
```
**Expected output:** `600 601 602`

**Difference from `pgrep`:** `pidof` matches the exact program name; `pgrep` matches patterns.

---

## 4.5 — Signals and Killing Processes

### What Are Signals?

Signals are software interrupts sent to processes. They tell a process to do something — stop, restart, terminate, etc.

**📌 Important signals every admin must know:**

| Signal | Number | Name | Meaning | Can Be Caught? |
|--------|--------|------|---------|----------------|
| `SIGHUP` | 1 | Hangup | Terminal closed or config reload | Yes |
| `SIGINT` | 2 | Interrupt | `Ctrl+C` pressed | Yes |
| `SIGQUIT` | 3 | Quit | Like SIGINT but creates core dump | Yes |
| `SIGKILL` | 9 | Kill | Force kill — process cannot catch or ignore | **No** |
| `SIGTERM` | 15 | Terminate | Graceful shutdown (default `kill` signal) | Yes |
| `SIGSTOP` | 19 | Stop | Pause the process (like `Ctrl+Z`) | **No** |
| `SIGCONT` | 18 | Continue | Resume a stopped process | Yes |
| `SIGUSR1` | 10 | User-defined 1 | Application-specific (e.g., log rotation) | Yes |
| `SIGUSR2` | 12 | User-defined 2 | Application-specific | Yes |

**🎯 Interview Point:** "What is the difference between SIGTERM and SIGKILL?"  
> **SIGTERM (15):** Asks the process to terminate gracefully. The process can catch it, clean up (close files, release locks), and exit.  
> **SIGKILL (9):** Forces immediate termination. The process cannot catch, block, or ignore it. No cleanup happens. Use only as a last resort.

### `kill` — Send a Signal to a Process

```bash
kill PID
```
**What it does:** Sends SIGTERM (15) to the process. This is a **graceful** shutdown request.

```bash
kill -9 PID
```
**What it does:** Sends SIGKILL. Force kills the process immediately.

**Syntax:** `kill [-signal] PID`

```bash
# Graceful termination (default — SIGTERM)
kill 1250

# Force kill
kill -9 1250
kill -KILL 1250
kill -SIGKILL 1250

# Reload configuration (many daemons support this)
kill -1 600
kill -HUP 600

# Pause a process
kill -STOP 1250

# Resume a paused process
kill -CONT 1250
```

**✅ Best Practice — Escalation order:**

```bash
# Step 1: Ask nicely (SIGTERM)
kill PID
sleep 5

# Step 2: Check if it's still running
ps -p PID

# Step 3: Force kill (SIGKILL) only if still running
kill -9 PID
```

**⚠️ Common Mistake:** Immediately using `kill -9`. This prevents the process from cleaning up — it may leave:
- Lock files that block restart
- Corrupted data files
- Orphaned child processes
- Incomplete transactions

**Always try `kill` (SIGTERM) first. Wait a few seconds. Only use `kill -9` if it doesn't work.**

### List All Available Signals

```bash
kill -l
```

---

### `killall` — Kill Processes by Name

```bash
killall nginx
```
**What it does:** Sends SIGTERM to ALL processes named `nginx`.

```bash
# Force kill all nginx processes
killall -9 nginx

# Kill all processes by a user
killall -u john

# Interactive — confirm each kill
killall -i nginx
```

**⚠️ Common Mistake:** On Solaris, `killall` kills ALL processes on the system (different behavior than Linux). Be careful on non-Linux systems.

---

### `pkill` — Kill Processes by Pattern

```bash
pkill nginx
```
**What it does:** Like `killall` but matches patterns (similar to `pgrep`).

```bash
# Kill by pattern
pkill -f "python app.py"

# Kill all processes by a user
pkill -u john

# Send HUP signal to nginx (reload config)
pkill -HUP nginx
```

| Option | Meaning |
|--------|---------|
| `-signal` | Specify signal (default: SIGTERM) |
| `-u user` | Match by user |
| `-f` | Match against full command line |
| `-9` | Force kill (SIGKILL) |

---

## 4.6 — Foreground, Background & Job Control

### Foreground vs Background

| Type | Description | Terminal Access |
|------|------------|-----------------|
| **Foreground** | Process takes over your terminal | Blocked — you can't type other commands |
| **Background** | Process runs without blocking terminal | Free — you can keep working |

### Running a Process in the Background

```bash
# Run in background by adding &
sleep 300 &
```
**Output:** `[1] 1500`
- `[1]` = Job number
- `1500` = PID

### `jobs` — List Background Jobs

```bash
jobs
```
**Expected output:**
```
[1]+  Running                 sleep 300 &
[2]-  Stopped                 vim file.txt
```

| Symbol | Meaning |
|--------|---------|
| `+` | Current job (default target of `fg`, `bg`) |
| `-` | Previous job |

### Moving Between Foreground and Background

```bash
# Move a background job to foreground
fg %1          # Bring job 1 to foreground
fg             # Bring the most recent job to foreground

# Suspend (pause) a foreground process
# Press Ctrl+Z

# Resume a suspended process in the background
bg %1          # Resume job 1 in background
bg             # Resume most recent job in background
```

### Practical Workflow

```bash
# 1. Start a long-running process in foreground
tar -czf backup.tar.gz /home/

# 2. Oh wait, I need the terminal! Press Ctrl+Z
# [1]+  Stopped                 tar -czf backup.tar.gz /home/

# 3. Resume it in the background
bg %1
# [1]+ tar -czf backup.tar.gz /home/ &

# 4. Check on it
jobs
# [1]+  Running                 tar -czf backup.tar.gz /home/ &

# 5. Continue working while it runs...
```

### `nohup` — Keep Running After Logout

```bash
nohup long_script.sh &
```
**What it does:** Runs the command so that it continues even after you close the terminal or SSH session. Output goes to `nohup.out` by default.

```bash
# Run and redirect output to a custom file
nohup long_script.sh > /var/log/myscript.log 2>&1 &
```

**⚠️ Common Mistake:** Starting a background process with `&` but without `nohup`. If you close the SSH session, the process dies (receives SIGHUP).

**💡 Admin Tip:** For long-running tasks over SSH, use `tmux` or `screen` instead of `nohup`. They give you a persistent terminal session you can reattach to:
```bash
tmux new -s mywork      # Create a new session
# ... run your commands ...
# Press Ctrl+B, then D to detach
tmux attach -t mywork   # Reattach later (even from a different SSH session)
```

### `disown` — Detach a Running Job

```bash
# Start a process in background
long_command &

# Detach it from the shell (won't die when shell exits)
disown %1
```

---

## 4.7 — Process Priority: `nice` and `renice`

### How Priority Works

Linux uses a scheduling priority system from **-20 (highest priority)** to **19 (lowest priority)**.

| Value | Priority | Who Can Set |
|-------|----------|-------------|
| -20 | Highest (gets most CPU) | root only |
| 0 | Default | Any user |
| 19 | Lowest (gets least CPU) | Any user |

Regular users can only LOWER their priority (set nice values 0–19).  
Only root can RAISE priority (set nice values -20 to -1).

### `nice` — Start a Process with Modified Priority

```bash
# Run with lower priority (nice to other processes)
nice -n 10 tar -czf backup.tar.gz /home/

# Run with higher priority (root only)
sudo nice -n -5 critical_script.sh
```

### `renice` — Change Priority of a Running Process

```bash
# Lower priority of a running process
renice 10 -p 1500

# Raise priority (root only)
sudo renice -5 -p 1500

# Change priority for all processes by a user
sudo renice 15 -u john
```

**Admin use case:** A user's process is hogging CPU. Lower its priority so other services aren't affected:
```bash
# Find the offending process
ps -eo pid,user,ni,%cpu,comm --sort=-%cpu | head -5

# Lower its priority
sudo renice 19 -p 1500
```

---

## 4.8 — System Resource Monitoring

### CPU Information

```bash
# Number of CPU cores
nproc

# Detailed CPU info
lscpu

# CPU model and details
cat /proc/cpuinfo | head -30

# Per-CPU usage (from top summary)
top -bn1 | head -5
```

### Memory Information

```bash
# Memory usage summary
free -h
```
**Expected output:**
```
               total        used        free      shared  buff/cache   available
Mem:           7.8Gi       2.1Gi       3.2Gi       120Mi       2.5Gi       5.4Gi
Swap:          2.0Gi          0B       2.0Gi
```

| Column | Meaning |
|--------|---------|
| `total` | Total physical RAM |
| `used` | Memory actively used by processes |
| `free` | Completely unused memory |
| `shared` | Memory shared between processes (tmpfs) |
| `buff/cache` | Memory used for disk caching (can be reclaimed) |
| `available` | Memory available for new processes (free + reclaimable cache) |

**📌 Important:** Linux aggressively uses free RAM for disk caching. **This is normal and good.** The important number is `available`, not `free`. If `available` is low, you're running out of memory.

**⚠️ Common Mistake:** Seeing `free` is small and panicking. Check `available` instead. Linux uses idle RAM for file caching to improve performance, which shows as `buff/cache`.

```bash
# Watch memory in real-time (update every 2 seconds)
watch -n 2 free -h
```

### `vmstat` — Virtual Memory Statistics

```bash
vmstat 1 5
```
**What it does:** Reports system stats every 1 second, 5 times.

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 3321892  93428 2566412    0    0     5    20   80  150  2  1 97  0  0
```

| Field | Meaning | Concern When |
|-------|---------|-------------|
| `r` | Processes waiting for CPU | Higher than CPU count |
| `b` | Processes in uninterruptible sleep | Consistently > 0 (I/O bottleneck) |
| `si`/`so` | Swap in/out (KB/s) | Any swap activity = memory pressure |
| `wa` | I/O wait (%) | High value = disk bottleneck |
| `us` | User CPU (%) | Application CPU usage |
| `sy` | System CPU (%) | Kernel CPU usage |
| `id` | Idle CPU (%) | Low = system is busy |

### Uptime and Load Average

```bash
uptime
```
```
14:30:00 up 5 days,  3:00,  2 users,  load average: 0.15, 0.10, 0.05
```

```bash
# Just the load average
cat /proc/loadavg
```
```
0.15 0.10 0.05 1/128 1400
```
- `0.15 0.10 0.05` = 1-min, 5-min, 15-min load averages
- `1/128` = 1 running process out of 128 total
- `1400` = last PID assigned

**Interpreting load trends:**

| Pattern | Meaning |
|---------|---------|
| 1-min > 5-min > 15-min | Load is INCREASING (investigate now!) |
| 1-min < 5-min < 15-min | Load is DECREASING (system recovering) |
| All three similar | Stable load |

---

## 4.9 — The `/proc` Filesystem

### What Is `/proc`?

`/proc` is a **virtual filesystem** — it doesn't exist on disk. The kernel generates its contents on the fly to expose system and process information.

### System Information in `/proc`

```bash
# CPU info
cat /proc/cpuinfo

# Memory info (detailed)
cat /proc/meminfo

# Kernel version
cat /proc/version

# Uptime in seconds
cat /proc/uptime

# Load average
cat /proc/loadavg

# Mounted filesystems
cat /proc/mounts

# Kernel command line (boot parameters)
cat /proc/cmdline

# Partitions
cat /proc/partitions

# Network connections
cat /proc/net/tcp
```

### Per-Process Information

Every process has a directory `/proc/[PID]/`:

```bash
# Look at your shell's /proc entry
ls /proc/$$/

# Process status
cat /proc/$$/status

# Process command line
cat /proc/$$/cmdline | tr '\0' ' '; echo

# Process environment variables
cat /proc/$$/environ | tr '\0' '\n'

# Open file descriptors
ls -la /proc/$$/fd/

# Memory maps
cat /proc/$$/maps | head -20

# Current working directory
ls -la /proc/$$/cwd

# Executable path
ls -la /proc/$$/exe
```

**💡 Admin Tip:** `/proc/[PID]/fd/` shows all open files. This is invaluable for troubleshooting "too many open files" errors:
```bash
# Count open files for a process
ls /proc/1500/fd | wc -l

# See what files it has open
ls -la /proc/1500/fd/
```

**🎯 Interview Point:** "What is `/proc` filesystem?"  
> `/proc` is a virtual/pseudo filesystem that provides an interface to kernel data structures. It contains system info (`/proc/cpuinfo`, `/proc/meminfo`) and per-process info (`/proc/[PID]/`). It exists only in memory, not on disk.

---

## 4.10 — Practical Monitoring Commands

### `iostat` — CPU and Disk I/O Statistics

```bash
# Install if needed
sudo apt install sysstat    # Ubuntu
sudo dnf install sysstat    # Rocky

iostat 1 5
```
**What it does:** Reports CPU and disk I/O stats every 1 second, 5 times.

### `mpstat` — Per-CPU Statistics

```bash
mpstat -P ALL 1 5
```
**What it does:** Shows per-CPU stats. Useful for identifying if one CPU core is bottlenecked.

### `sar` — Historical System Activity

```bash
# CPU usage history
sar -u

# Memory usage history
sar -r

# Disk I/O history
sar -d

# Network statistics
sar -n DEV
```

**📌 Important:** `sar` keeps historical data in `/var/log/sysstat/` (if `sysstat` service is enabled). This lets you look back at what happened hours or days ago.

### `lsof` — List Open Files

```bash
# List ALL open files (very long output!)
sudo lsof | head -20

# Files opened by a specific process
sudo lsof -p 1500

# Who has a specific file open?
sudo lsof /var/log/syslog

# What's using a specific port?
sudo lsof -i :80

# All network connections for a process
sudo lsof -i -p 1500

# Files opened by a user
sudo lsof -u www-data | head -20
```

**💡 Admin Tip:** `lsof -i :PORT` is one of the most useful troubleshooting commands:
```bash
# Who is listening on port 80?
sudo lsof -i :80
# COMMAND  PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
# nginx    600     root    6u  IPv4 12345    0t0     TCP *:http (LISTEN)
```

### `strace` — Trace System Calls

```bash
# Trace a running process
sudo strace -p 1500

# Trace a command
strace ls /tmp

# Count system calls
strace -c ls /tmp
```

**Admin use case:** A process is hanging or behaving oddly. `strace` shows exactly what system calls it's making (reading files, network connections, etc.).

---

## 4.11 — Hands-On Labs

### Lab 1: Process Exploration

```bash
# 1. View all running processes
ps aux | head -20

# 2. Count total processes
ps aux | wc -l

# 3. Find the top 5 CPU consumers
ps -eo pid,user,%cpu,%mem,comm --sort=-%cpu | head -6

# 4. Find the top 5 memory consumers
ps -eo pid,user,%cpu,%mem,rss,comm --sort=-%mem | head -6

# 5. View the process tree
pstree -p | head -30

# 6. Find your shell's PID and PPID
echo "My PID: $$"
ps -p $$ -o pid,ppid,comm

# 7. Check your shell's parent
ps -p $(ps -p $$ -o ppid=) -o pid,comm

# 8. Check system info from /proc
cat /proc/cpuinfo | grep "model name" | head -1
cat /proc/meminfo | head -5
cat /proc/loadavg
```

### Lab 2: Signals and Process Control

```bash
# 1. Start a long-running process
sleep 600 &
echo "PID: $!"

# 2. Verify it's running
ps -p $!
jobs

# 3. Send SIGSTOP (pause it)
kill -STOP $!
jobs    # Should show "Stopped"

# 4. Resume it
kill -CONT $!
jobs    # Should show "Running"

# 5. Terminate it gracefully
kill $!
jobs    # Should show "Terminated"

# 6. Start another and force kill it
sleep 600 &
PID=$!
kill -9 $PID
jobs    # Should show "Killed"

# 7. Practice job control
sleep 300 &
sleep 400 &
sleep 500 &
jobs              # See all three
fg %2             # Bring job 2 to foreground
# Press Ctrl+Z    # Suspend it
bg %2             # Resume in background
kill %1 %2 %3     # Kill all three
```

### Lab 3: Monitoring System Resources

```bash
# 1. Check system load
uptime
cat /proc/loadavg

# 2. Check memory usage
free -h

# 3. Check CPU count
nproc

# 4. Run top for a quick snapshot
top -bn1 | head -20

# 5. Install and use htop
sudo apt install -y htop
htop    # Explore the interface, press F5 for tree, q to quit

# 6. Simulate CPU load (in background)
dd if=/dev/urandom of=/dev/null bs=1M &
CPU_PID=$!

# 7. Watch it in top
top -p $CPU_PID -bn3

# 8. Check its priority
ps -o pid,ni,%cpu,comm -p $CPU_PID

# 9. Lower its priority
renice 19 -p $CPU_PID
ps -o pid,ni,%cpu,comm -p $CPU_PID

# 10. Clean up
kill $CPU_PID
```

### Lab 4: Process Investigation with `/proc`

```bash
# 1. Start a background process
sleep 1000 &
PID=$!

# 2. Explore its /proc entry
ls /proc/$PID/

# 3. Check its status
cat /proc/$PID/status | head -15

# 4. See its command line
cat /proc/$PID/cmdline | tr '\0' ' '; echo

# 5. Check open file descriptors
ls -la /proc/$PID/fd/

# 6. Check its working directory
ls -la /proc/$PID/cwd

# 7. Check the executable
ls -la /proc/$PID/exe

# 8. Clean up
kill $PID
```

---

## 4.12 — Troubleshooting Scenarios

### Scenario 1: Process Consuming 100% CPU

**Problem:** The server is slow. A process is consuming 100% CPU.

**Task:**
1. Identify which process is using the most CPU
2. Find out who owns it and what it's doing
3. Determine if it's a legitimate workload or a problem
4. Take appropriate action

**Try this yourself first.**

<details>
<summary><strong>Solution</strong></summary>

```bash
# Step 1: Find the culprit
top -bn1 | head -15
# OR
ps -eo pid,user,%cpu,%mem,comm --sort=-%cpu | head -5

# Step 2: Get details about the process (assuming PID 1500)
ps -fp 1500                          # Full process details
cat /proc/1500/cmdline | tr '\0' ' '  # Exact command
ls -la /proc/1500/exe                 # What binary
ls -la /proc/1500/cwd                 # Working directory
pstree -p 1500                        # Child processes

# Step 3: Check how long it's been running
ps -o pid,etime,%cpu,comm -p 1500

# Step 4: Decide and act
# If it's a runaway process:
kill 1500                             # Try graceful first
sleep 5
ps -p 1500 && kill -9 1500           # Force if still running

# If it's a legitimate but low-priority task:
sudo renice 19 -p 1500               # Lower priority
```

</details>

---

### Scenario 2: Cannot Kill a Process

**Problem:** You tried `kill -9` but the process won't die.

**Task:** Diagnose why and fix it.

<details>
<summary><strong>Solution</strong></summary>

```bash
# Check the process state
ps -o pid,stat,comm -p 1500

# If state is 'D' (uninterruptible sleep):
# The process is waiting for I/O (usually disk or NFS).
# kill -9 cannot interrupt it. You must fix the underlying I/O issue.
# Common causes:
# - NFS mount to an unavailable server
# - Failing disk
# - Kernel bug

# If state is 'Z' (zombie):
# The process is already dead! It's waiting for its parent to collect the exit status.
# Find and fix (or kill) the parent:
ps -o pid,ppid,stat,comm -p 1500
# Note the PPID
kill PPID              # Kill the parent
# OR
# Wait — systemd will eventually clean up orphaned zombies

# If it's truly unkillable (state D):
# Check for I/O issues
dmesg | tail -20       # Kernel messages
mount | grep nfs       # NFS mounts?
iostat                 # Disk I/O issues?
# Last resort: reboot (only after exhausting other options)
```

</details>

---

### Scenario 3: Server Running Out of Memory

**Problem:** Applications are crashing with "out of memory" errors or being killed by the OOM killer.

**Task:** Investigate and resolve the memory issue.

<details>
<summary><strong>Solution</strong></summary>

```bash
# Step 1: Check memory status
free -h

# Step 2: Check if swap is being used heavily
free -h | grep Swap
# If swap used is high, system is under memory pressure

# Step 3: Find memory-hungry processes
ps -eo pid,user,%mem,rss,comm --sort=-%mem | head -10

# Step 4: Check OOM killer activity
dmesg | grep -i "oom\|out of memory" | tail -10
grep -i "oom" /var/log/syslog | tail -10

# Step 5: Check for memory leaks (process using increasing memory)
# Watch a suspect process over time
watch -n 5 'ps -o pid,rss,%mem,comm -p SUSPECT_PID'

# Step 6: Take action
# Option A: Kill the memory hog
kill PID

# Option B: Add swap (temporary fix)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Option C: Reduce memory usage of the application (config change)

# Step 7: Check system buffers/cache
# Linux uses free RAM for caching. You can drop caches (safe, temporary):
sudo sync
sudo sh -c 'echo 3 > /proc/sys/vm/drop_caches'
free -h
```

</details>

---

### Scenario 4: Zombie Processes

**Problem:** You notice zombie processes on the system.

**Task:** Find them, understand why they exist, and clean them up.

<details>
<summary><strong>Solution</strong></summary>

```bash
# Step 1: Find zombies
ps aux | grep -w Z
# OR
ps -eo pid,ppid,stat,comm | grep Z

# Step 2: Identify the parent
# Note the PPID of the zombie process
ps -o pid,ppid,stat,comm -p ZOMBIE_PID

# Step 3: Check the parent process
ps -fp PARENT_PID

# Step 4: Options
# Option A: Send SIGCHLD to parent (tells it to reap children)
kill -SIGCHLD PARENT_PID

# Option B: Kill the parent (zombies are cleaned up by systemd)
kill PARENT_PID

# Option C: If zombies persist and parent is critical, restart the service
sudo systemctl restart service_name

# Note: A few zombies are harmless. They use no CPU, memory, or disk.
# They only consume a PID entry. Only worry if you have hundreds.
```

</details>

---

## 4.13 — Practice Questions

1. **Find the 5 processes** using the most CPU right now. Show their PID, user, CPU%, memory%, and command.

2. **Start three background sleep processes** (sleep 100, 200, 300). List them with `jobs`. Kill the second one by job number. Verify it's gone.

3. **Using `/proc`**, find out:
   - How many CPU cores your system has
   - How much total RAM (in KB)
   - The current system uptime in seconds

4. **A process with PID 5000 is using too much CPU.** Show the exact sequence of commands to gracefully stop it, with a fallback to force-kill if necessary.

5. **Find all zombie processes** on your system. If there are any, identify their parent process.

6. **Use `lsof`** to find which process is listening on port 22 (SSH).

7. **Start a CPU-intensive process** (`dd if=/dev/urandom of=/dev/null bs=1M`), lower its priority to 15, verify the priority change, then kill it.

8. **Memory analysis:** What is the difference between `free`, `available`, and `buff/cache` in the output of `free -h`? Why is `available` more important than `free`?

---

## Level 4 Summary

### What I Learned

- Every process has a PID, PPID, UID, state, and priority
- PID 1 (`systemd`) is the root of all processes
- `ps aux` and `ps -ef` show all running processes with different detail levels
- `top` and `htop` provide real-time monitoring
- Signals control process behavior — SIGTERM (graceful) vs SIGKILL (force)
- Always try SIGTERM before SIGKILL
- Foreground/background job control with `fg`, `bg`, `Ctrl+Z`, `&`
- `nice`/`renice` control process CPU priority (-20 to 19)
- `/proc` is a virtual filesystem exposing kernel and process info
- Load average should be at or below CPU core count
- Linux uses free RAM for caching — check `available` not `free`
- Zombie processes indicate parent bugs — kill the parent to fix

### Commands to Remember

| Command | Purpose |
|---------|---------|
| `ps aux` | Show all processes (BSD style) |
| `ps -ef` | Show all processes with PPID (SysV style) |
| `ps -eo pid,user,%cpu,%mem,comm --sort=-%cpu` | Custom sorted process list |
| `top` | Real-time process monitor |
| `htop` | Better real-time process monitor |
| `pgrep -la name` | Find PIDs by process name |
| `kill PID` | Send SIGTERM (graceful stop) |
| `kill -9 PID` | Send SIGKILL (force stop) |
| `kill -HUP PID` | Reload configuration |
| `killall name` | Kill all processes by name |
| `pkill -f pattern` | Kill by command pattern |
| `jobs` | List background jobs |
| `fg %N` / `bg %N` | Foreground / background a job |
| `Ctrl+Z` | Suspend foreground process |
| `nohup cmd &` | Run command immune to hangup |
| `nice -n 10 cmd` | Start with lower priority |
| `renice 19 -p PID` | Lower running process priority |
| `free -h` | Memory usage |
| `uptime` | Load average and uptime |
| `lsof -i :PORT` | Find process using a port |
| `nproc` | CPU core count |
| `pstree -p` | Process tree with PIDs |
| `vmstat 1` | Virtual memory stats |

### Practical Skills Gained

- Find and identify any process on the system
- Safely stop runaway processes with proper signal escalation
- Monitor CPU, memory, and I/O usage in real-time
- Use job control to manage long-running tasks
- Investigate process details through `/proc`
- Diagnose and resolve zombie processes
- Identify memory pressure and OOM situations
- Find which process is using a specific port

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `kill -9` first | Always try `kill` (SIGTERM) first, wait, then `-9` |
| Panicking at low `free` memory | Check `available` instead — caching is normal |
| Starting SSH tasks without `nohup` or `tmux` | Use `nohup cmd &` or `tmux` for persistence |
| Forgetting `-a` with `pgrep` | Without `-a` you only get PIDs, not command names |
| Trying to kill zombies | Kill the parent process, not the zombie |
| Ignoring load average trends | Watch the 1/5/15 min trend, not just one number |

### Administrator-Level Knowledge

- **Load average rule:** Should be ≤ number of CPU cores
- **Memory rule:** Watch `available`, not `free`. Swap usage = memory pressure.
- **Process escalation:** SIGTERM → wait 5s → SIGKILL → check parent if zombie
- **Real-time log + process:** `tail -f log` in one terminal, `top` in another
- **Finding port usage:** `sudo lsof -i :PORT` or `sudo ss -tlnp`
- **Historical data:** Configure `sysstat`/`sar` for retrospective performance analysis
- **OOM killer:** Check `dmesg | grep oom` when processes mysteriously die

### Interview Questions

1. **What is the difference between a process and a program?**
2. **Explain PID, PPID, and the process tree.**
3. **What are zombie processes and how do you handle them?**
4. **What is the difference between SIGTERM (15) and SIGKILL (9)?**
5. **What does load average represent? When should you be concerned?**
6. **How do you find which process is using port 80?**
7. **Explain the difference between `free` and `available` memory.**
8. **What is the `/proc` filesystem?**
9. **How do you run a process that survives SSH disconnection?**
10. **A server is slow. Describe your troubleshooting process step by step.**

---

*When you've completed all the labs and practice questions, proceed to [Level 5 — Services & systemd](level-05-services-systemd.md)*
