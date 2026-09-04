# Level 1 — Linux Fundamentals

> **Goal:** Understand what Linux is, navigate the filesystem, work with files, and use essential command-line tools confidently.

> **Lab Environment:** WSL2 Ubuntu (ideal for this level)

---

## 1.1 — What is Linux?

### Minimum Theory

Linux is an **operating system** — the software that manages your computer's hardware and lets you run applications. When people say "Linux," they usually mean the entire OS, but technically Linux is just the **kernel**.

**📌 Important — Three key terms:**

| Term | What It Is | Example |
|------|-----------|---------|
| **Kernel** | The core program that talks to hardware (CPU, RAM, disk, network). It manages memory, processes, and device drivers. | Linux kernel (created by Linus Torvalds, 1991) |
| **Operating System (OS)** | Kernel + system utilities + libraries + shell. Everything needed to make the computer usable. | GNU/Linux |
| **Distribution (Distro)** | A complete package: kernel + OS tools + package manager + desktop (optional) + default apps, assembled by an organization. | Ubuntu, Rocky Linux, Debian, Fedora |

### Linux Architecture (Simplified)

```
┌─────────────────────────────────────┐
│          User Applications          │  ← Firefox, Nginx, your scripts
├─────────────────────────────────────┤
│             Shell (Bash)            │  ← You type commands here
├─────────────────────────────────────┤
│         System Libraries (glibc)    │  ← Standard C library, etc.
├─────────────────────────────────────┤
│       System Calls Interface        │  ← Bridge between user space & kernel
├─────────────────────────────────────┤
│         Linux Kernel                │  ← Process mgmt, memory, filesystems
├─────────────────────────────────────┤
│         Hardware                    │  ← CPU, RAM, Disk, Network card
└─────────────────────────────────────┘
```

### Why Linux is Used in Real Systems

- **90%+ of servers** on the internet run Linux (AWS, Google, Azure all default to Linux)
- **Free and open source** — no licensing costs
- **Stable** — servers run for years without rebooting
- **Secure** — permission system, rapid security patches
- **Customizable** — you control everything
- **Automation-friendly** — everything can be done via the command line

### Major Distributions

| Distro Family | Distros | Package Manager | Used For |
|---------------|---------|-----------------|----------|
| **Debian-based** | Ubuntu, Debian, Linux Mint | `apt` | Desktops, cloud servers, WSL |
| **RHEL-based** | Rocky Linux, AlmaLinux, CentOS Stream, Fedora | `dnf` / `yum` | Enterprise servers, production |
| **SUSE-based** | openSUSE, SLES | `zypper` | Enterprise |
| **Arch-based** | Arch Linux, Manjaro | `pacman` | Power users, rolling release |

**💡 Admin Tip:** In the job market, **Ubuntu** and **RHEL/Rocky** are the two you must know. This course covers both.

**🎯 Interview Point:** "What is the difference between Linux kernel and a Linux distribution?"
> The kernel is the core that manages hardware. A distribution packages the kernel with tools, libraries, a package manager, and default configurations into a usable OS.

---

## 1.2 — Terminal, Shell & CLI

### What is a Terminal?

The **terminal** (also called terminal emulator) is the window/application that lets you type text commands. Think of it as the screen.

### What is a Shell?

The **shell** is the program running inside the terminal that interprets your commands. The default shell on most Linux systems is **Bash** (Bourne Again Shell).

```
Terminal (window) ──→ Shell (Bash) ──→ Kernel ──→ Hardware
```

### CLI vs GUI

| Feature | CLI (Command Line Interface) | GUI (Graphical User Interface) |
|---------|-----|-----|
| Speed | Faster for experienced users | Slower for repetitive tasks |
| Automation | Easily scriptable | Difficult to automate |
| Remote access | Works over SSH (low bandwidth) | Requires VNC/RDP (high bandwidth) |
| Server usage | Standard — most servers have no GUI | Rarely installed on servers |
| Learning curve | Steep initially | Easier initially |

**📌 Important:** As a System Administrator, **90% of your work will be on the CLI**. Servers typically don't even have a GUI installed.

### Opening Your Terminal

**On WSL2:**
- Open "Ubuntu" from the Windows Start Menu
- Or open Windows Terminal and select the Ubuntu tab

**On a VM:**
- Log in and you're already at the terminal (Ubuntu Server has no GUI)
- On a desktop VM: press `Ctrl + Alt + T`

### Your First Commands

```bash
whoami
```
**What it does:** Prints the username of the currently logged-in user.  
**Expected output:** `abhishek` (or whatever username you created)

```bash
hostname
```
**What it does:** Prints the name of the machine.

```bash
date
```
**What it does:** Shows the current date and time.

```bash
uptime
```
**What it does:** Shows how long the system has been running, number of logged-in users, and load average.

```bash
clear
```
**What it does:** Clears the terminal screen.  
**Shortcut:** `Ctrl + L` does the same thing.

```bash
echo "Hello, Linux!"
```
**What it does:** Prints the text to the terminal. `echo` is the simplest output command.

---

## 1.3 — Bash Basics

### Command Structure

```
command [options] [arguments]
```

| Part | Purpose | Example |
|------|---------|---------|
| `command` | The program to run | `ls` |
| `options` | Modify behavior (start with `-` or `--`) | `-l`, `--all` |
| `arguments` | What the command acts on | `/home` |

**Example:**
```bash
ls -la /home
```
- `ls` → command (list directory contents)
- `-la` → options (`-l` long format + `-a` show hidden files)
- `/home` → argument (which directory to list)

### Short vs Long Options

```bash
ls -a          # Short option
ls --all       # Long option (same thing)
```

Short options can be combined:
```bash
ls -l -a -h    # Three separate options
ls -lah        # Same thing, combined
```

### Tab Completion

**📌 Important:** This is your best friend. Press `Tab` to auto-complete:

- File and directory names
- Command names
- Options (in some shells)

```bash
cd /ho         # Press Tab → completes to /home
cd /home/ab    # Press Tab → completes to /home/abhishek
```

Press `Tab` twice to see all possible completions if there are multiple matches.

### Command History

```bash
history
```
**What it does:** Shows all previously executed commands with line numbers.

```bash
history 20
```
**What it does:** Shows the last 20 commands.

**Navigation shortcuts:**

| Shortcut | Action |
|----------|--------|
| `↑` / `↓` | Scroll through previous commands |
| `Ctrl + R` | Reverse search through history (type to find) |
| `!!` | Repeat the last command |
| `!50` | Run command number 50 from history |
| `!ls` | Run the most recent command starting with `ls` |

**💡 Admin Tip:** `Ctrl + R` is extremely useful. Press it, then start typing part of a previous command. It searches backward through your history. Press `Ctrl + R` again to find older matches. Press `Enter` to execute, or `Esc` to edit before executing.

### Essential Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl + C` | Cancel/kill the running command |
| `Ctrl + D` | Logout / end input (EOF) |
| `Ctrl + L` | Clear screen |
| `Ctrl + A` | Move cursor to beginning of line |
| `Ctrl + E` | Move cursor to end of line |
| `Ctrl + U` | Delete everything before cursor |
| `Ctrl + K` | Delete everything after cursor |
| `Ctrl + W` | Delete the word before cursor |
| `Ctrl + Z` | Suspend (pause) the running process |

---

## 1.4 — Linux Filesystem Hierarchy

### Minimum Theory

Unlike Windows (which uses `C:\`, `D:\`), Linux has a **single root** directory called `/` and everything branches from it. There are no drive letters.

**🎯 Interview Point:** "Explain the Linux filesystem hierarchy."

### The Directory Tree

```
/                     ← Root — the top of everything
├── bin/              ← Essential user commands (ls, cp, mv, cat)
├── sbin/             ← System/admin commands (fdisk, reboot, iptables)
├── etc/              ← Configuration files (ALL system config lives here)
├── home/             ← User home directories (/home/abhishek, /home/john)
├── root/             ← Home directory of the root (admin) user
├── var/              ← Variable data — logs, caches, spool
│   ├── log/          ← System and service logs
│   ├── tmp/          ← Temporary files (preserved across reboots)
│   └── www/          ← Web server files (on some distros)
├── tmp/              ← Temporary files (cleared on reboot)
├── usr/              ← User programs and data (read-only)
│   ├── bin/          ← User commands (most commands live here now)
│   ├── sbin/         ← System admin commands
│   ├── lib/          ← Libraries
│   ├── local/        ← Locally installed software (/usr/local/bin)
│   └── share/        ← Shared data (docs, man pages)
├── opt/              ← Optional/third-party software (Oracle, custom apps)
├── dev/              ← Device files (disks, terminals, etc.)
├── proc/             ← Virtual filesystem — process and kernel info
├── sys/              ← Virtual filesystem — hardware/device info
├── boot/             ← Boot files (kernel, GRUB bootloader)
├── lib/              ← Essential shared libraries
├── mnt/              ← Temporary mount points
├── media/            ← Auto-mounted removable devices (USB, CD)
└── srv/              ← Service data (FTP, HTTP — rarely used)
```

### Key Directories for Administrators

| Directory | What's Inside | Admin Relevance |
|-----------|--------------|-----------------|
| `/etc` | All config files | You'll edit files here daily: `/etc/ssh/sshd_config`, `/etc/fstab`, `/etc/hosts` |
| `/var/log` | Log files | First place to look when debugging |
| `/home` | User data | Manage user storage, quotas |
| `/tmp` | Temporary files | Cleared on reboot; world-writable |
| `/root` | Root user's home | Not `/home/root` — it's just `/root` |
| `/opt` | Third-party software | Install commercial/custom apps here |
| `/proc` | Process info (virtual) | Check CPU, memory, process details |

**⚠️ Common Mistake:** Confusing `/root` (root user's home directory) with `/` (the root of the filesystem).

### Absolute vs Relative Paths

| Type | Starts With | Example | Meaning |
|------|-------------|---------|---------|
| **Absolute** | `/` | `/home/abhishek/docs` | Full path from root — works from anywhere |
| **Relative** | anything else | `docs/file.txt` | Relative to your current directory |

**Special path symbols:**

| Symbol | Meaning | Example |
|--------|---------|---------|
| `.` | Current directory | `./script.sh` (run script in current dir) |
| `..` | Parent directory | `cd ..` (go up one level) |
| `~` | Home directory of current user | `cd ~` = `cd /home/abhishek` |
| `-` | Previous directory | `cd -` (go back to where you were) |

---

## 1.5 — Navigation Commands

### `pwd` — Print Working Directory

```bash
pwd
```
**What it does:** Shows the full absolute path of your current directory.  
**Expected output:**
```
/home/abhishek
```
**When to use:** When you're lost and need to know where you are.

---

### `cd` — Change Directory

```bash
cd /var/log
```
**What it does:** Changes your current working directory to `/var/log`.

**Syntax:** `cd [directory]`

| Command | Action |
|---------|--------|
| `cd /etc` | Go to `/etc` (absolute path) |
| `cd documents` | Go into `documents` subdirectory (relative) |
| `cd ..` | Go up one directory |
| `cd ../..` | Go up two directories |
| `cd ~` | Go to your home directory |
| `cd` | Go to your home directory (same as `cd ~`) |
| `cd -` | Go to the previous directory |

**⚠️ Common Mistake:** Typing `cd/etc` without a space. Always: `cd /etc`.

---

### `ls` — List Directory Contents

```bash
ls
```
**What it does:** Lists files and directories in the current location.

**Syntax:** `ls [options] [directory]`

**Important Options:**

| Option | Meaning | Example |
|--------|---------|---------|
| `-l` | Long format (permissions, owner, size, date) | `ls -l` |
| `-a` | Show all files including hidden (dotfiles) | `ls -a` |
| `-h` | Human-readable sizes (KB, MB, GB) | `ls -lh` |
| `-R` | List subdirectories recursively | `ls -R` |
| `-t` | Sort by modification time (newest first) | `ls -lt` |
| `-r` | Reverse sort order | `ls -ltr` |
| `-S` | Sort by file size (largest first) | `ls -lS` |
| `-d` | Show directory itself, not its contents | `ls -ld /etc` |
| `-i` | Show inode number | `ls -li` |

**Most used combination:**

```bash
ls -lah
```
**Expected output:**
```
total 32K
drwxr-xr-x  5 abhishek abhishek 4.0K Sep  4 10:00 .
drwxr-xr-x  3 root     root     4.0K Sep  1 09:00 ..
-rw-------  1 abhishek abhishek  220 Sep  1 09:00 .bash_history
-rw-r--r--  1 abhishek abhishek  807 Sep  1 09:00 .bashrc
drwxr-xr-x  2 abhishek abhishek 4.0K Sep  3 14:00 documents
-rw-r--r--  1 abhishek abhishek   45 Sep  4 10:00 notes.txt
```

**Reading the long format:**

```
-rw-r--r--  1 abhishek abhishek 807 Sep 1 09:00 .bashrc
│ │  │  │   │  │        │       │   │           │
│ │  │  │   │  │        │       │   │           └── Filename
│ │  │  │   │  │        │       │   └── Last modified date
│ │  │  │   │  │        │       └── File size (bytes)
│ │  │  │   │  │        └── Group owner
│ │  │  │   │  └── User owner
│ │  │  │   └── Hard link count
│ │  │  └── Others permissions (r--)
│ │  └── Group permissions (r--)
│ └── User permissions (rw-)
└── File type (- = regular file, d = directory, l = symlink)
```

**💡 Admin Tip:** `ls -ltr` (long format, sorted by time, reversed) is extremely useful — it shows the most recently modified files at the bottom, making it easy to spot recent changes.

**🎯 Interview Point:** "What does `ls -la` show that `ls` doesn't?"  
> It shows hidden files (starting with `.`) and detailed info like permissions, owner, size, and timestamps.

---

## 1.6 — Working with Files and Directories

### `touch` — Create Empty Files / Update Timestamps

```bash
touch myfile.txt
```
**What it does:** Creates an empty file called `myfile.txt`. If the file already exists, it updates the modification timestamp without changing the content.

**Admin use case:** Creating placeholder files, updating timestamps for scripts.

```bash
touch file1.txt file2.txt file3.txt
```
Creates three files at once.

---

### `mkdir` — Create Directories

```bash
mkdir projects
```
**What it does:** Creates a directory called `projects`.

**Important option:**

```bash
mkdir -p projects/web/frontend
```
**`-p` (parents):** Creates the entire directory path including any missing parent directories. Without `-p`, this would fail if `projects/` or `projects/web/` don't exist.

**⚠️ Common Mistake:** Forgetting `-p` when creating nested directories.

---

### `cp` — Copy Files and Directories

```bash
cp source.txt destination.txt
```
**What it does:** Copies `source.txt` to `destination.txt`.

**Syntax:** `cp [options] source destination`

| Option | Meaning |
|--------|---------|
| `-r` or `-R` | Copy directories recursively (required for dirs) |
| `-i` | Interactive — ask before overwriting |
| `-v` | Verbose — show what's being copied |
| `-p` | Preserve permissions, timestamps, ownership |
| `-a` | Archive mode (`-r` + `-p` + preserve links). Best for backups |

**Examples:**

```bash
# Copy a file
cp report.txt report_backup.txt

# Copy a file to another directory
cp report.txt /tmp/

# Copy a directory (must use -r)
cp -r projects/ projects_backup/

# Copy preserving everything (admin backup)
cp -a /etc/nginx/ /backup/nginx_config/
```

**⚠️ Common Mistake:** Trying to copy a directory without `-r`:
```bash
cp projects/ backup/        # ERROR: omitting directory 'projects/'
cp -r projects/ backup/     # CORRECT
```

---

### `mv` — Move or Rename Files

```bash
mv oldname.txt newname.txt
```
**What it does:** Renames `oldname.txt` to `newname.txt`.

```bash
mv file.txt /tmp/
```
**What it does:** Moves `file.txt` to the `/tmp/` directory.

**Syntax:** `mv [options] source destination`

| Option | Meaning |
|--------|---------|
| `-i` | Ask before overwriting |
| `-v` | Verbose |
| `-n` | Never overwrite |

**📌 Important:** `mv` works on both files AND directories without needing `-r`.

**⚠️ Common Mistake:** Using `mv` without realizing it **overwrites** the destination file silently. Use `mv -i` to be safe.

---

### `rm` — Remove Files and Directories

```bash
rm file.txt
```
**What it does:** Permanently deletes `file.txt`.

**🔴 DANGER — There is no recycle bin in Linux. `rm` is permanent.**

**Syntax:** `rm [options] file`

| Option | Meaning |
|--------|---------|
| `-r` | Remove directories and contents recursively |
| `-f` | Force — don't ask for confirmation, ignore nonexistent files |
| `-i` | Interactive — ask before each deletion |
| `-v` | Verbose — show what's being deleted |

**Examples:**

```bash
# Delete a file
rm old_log.txt

# Delete a directory and everything inside
rm -r old_project/

# Delete with confirmation (safer)
rm -ri old_project/

# Force delete (no questions asked)
rm -rf temp_files/
```

**🔴 DANGER — The most dangerous Linux command:**

```bash
rm -rf /                # DESTROYS THE ENTIRE SYSTEM
rm -rf ~                # DESTROYS YOUR ENTIRE HOME DIRECTORY
rm -rf /*               # SAME AS ABOVE — DESTROYS EVERYTHING
```

> **Never run `rm -rf` with `/` or `~` or `/*`. Modern systems have safeguards (`--preserve-root`), but NEVER rely on them.**

**✅ Best Practice:** Always use `rm -ri` when deleting directories you're unsure about. Check with `ls` first.

**💡 Admin Tip:** Before running a dangerous `rm`, test the pattern with `ls` first:
```bash
ls /var/log/*.old      # Check what matches
rm /var/log/*.old      # Then delete
```

---

### `rmdir` — Remove Empty Directories

```bash
rmdir empty_folder
```
**What it does:** Removes a directory ONLY if it's empty. Safer than `rm -r`.

---

## 1.7 — Viewing File Contents

### `cat` — Concatenate and Display

```bash
cat /etc/hostname
```
**What it does:** Dumps the entire file content to the terminal.

```bash
cat file1.txt file2.txt
```
**What it does:** Shows contents of both files one after another (concatenation).

```bash
cat -n /etc/passwd
```
**`-n`:** Shows line numbers.

**⚠️ Common Mistake:** Using `cat` on huge files — it floods your terminal. Use `less` for large files.

---

### `less` — Page Through Files

```bash
less /var/log/syslog
```
**What it does:** Opens the file in a scrollable viewer. Doesn't load the entire file into memory — perfect for huge files.

**Navigation inside `less`:**

| Key | Action |
|-----|--------|
| `Space` / `Page Down` | Next page |
| `b` / `Page Up` | Previous page |
| `g` | Go to beginning |
| `G` | Go to end |
| `/pattern` | Search forward for "pattern" |
| `?pattern` | Search backward |
| `n` | Next search match |
| `N` | Previous search match |
| `q` | Quit |

**💡 Admin Tip:** `less` is the go-to tool for reading log files. Combine with search: open with `less`, then type `/error` to find errors.

---

### `more` — Simple Pager

```bash
more /etc/services
```
**What it does:** Similar to `less` but simpler. Can only scroll forward. `less` is preferred in almost all cases.

---

### `head` — View Beginning of a File

```bash
head /etc/passwd
```
**What it does:** Shows the first 10 lines of the file (default).

```bash
head -n 20 /etc/passwd
```
**`-n 20`:** Shows the first 20 lines.

```bash
head -c 100 /etc/passwd
```
**`-c 100`:** Shows the first 100 bytes.

---

### `tail` — View End of a File

```bash
tail /var/log/syslog
```
**What it does:** Shows the last 10 lines of the file (default).

```bash
tail -n 50 /var/log/syslog
```
**`-n 50`:** Shows the last 50 lines.

```bash
tail -f /var/log/syslog
```
**`-f` (follow):** Continuously watches the file for new lines. **This is one of the most important admin commands.** It lets you watch logs in real-time.

**💡 Admin Tip:** When troubleshooting a service:
```bash
# Terminal 1: Watch the log
tail -f /var/log/nginx/error.log

# Terminal 2: Restart the service and watch errors appear
sudo systemctl restart nginx
```

Press `Ctrl + C` to stop `tail -f`.

**🎯 Interview Point:** "How do you monitor logs in real-time?"  
> `tail -f /var/log/syslog` — follows the file and shows new entries as they're written.

---

## 1.8 — Wildcards and Pattern Matching (Globbing)

Wildcards let you match multiple files at once. The shell expands them before the command runs.

| Wildcard | Meaning | Example |
|----------|---------|---------|
| `*` | Matches zero or more characters | `*.txt` → all `.txt` files |
| `?` | Matches exactly one character | `file?.txt` → `file1.txt`, `fileA.txt` |
| `[abc]` | Matches any one character in the set | `file[123].txt` → `file1.txt`, `file2.txt`, `file3.txt` |
| `[a-z]` | Matches any one character in the range | `file[a-c].txt` → `filea.txt`, `fileb.txt`, `filec.txt` |
| `[!abc]` | Matches any character NOT in the set | `file[!0-9].txt` → `fileA.txt` but not `file1.txt` |
| `{a,b,c}` | Brace expansion — generates multiple strings | `file{1,2,3}.txt` → `file1.txt file2.txt file3.txt` |

**Examples:**

```bash
# List all log files
ls /var/log/*.log

# List all files starting with "report"
ls report*

# Delete all .tmp files
rm *.tmp

# Copy all .conf files to backup
cp /etc/*.conf /backup/

# Create multiple files at once
touch report_{jan,feb,mar,apr}.txt
```

**⚠️ Common Mistake:** Using wildcards with `rm` without checking first:
```bash
# WRONG — what if *.bak matches more than you expect?
rm *.bak

# SAFE — check first
ls *.bak        # See what matches
rm -i *.bak     # Delete with confirmation
```

---

## 1.9 — Command Chaining

### Sequential Execution (`;`)

```bash
mkdir testdir; cd testdir; touch file.txt
```
**What it does:** Runs all three commands one after another, regardless of whether previous ones succeeded or failed.

### AND Operator (`&&`)

```bash
mkdir testdir && cd testdir && echo "Success"
```
**What it does:** Runs the next command **only if the previous one succeeded** (exit code 0).

**Admin use case:** `apt update && apt upgrade` — only upgrade if the update succeeded.

### OR Operator (`||`)

```bash
mkdir testdir || echo "Directory already exists"
```
**What it does:** Runs the next command **only if the previous one failed** (non-zero exit code).

**Admin use case:** `ping -c 1 google.com || echo "Network is down"`

### Combined Pattern

```bash
command && echo "Worked" || echo "Failed"
```

**✅ Best Practice:** Use `&&` in scripts and admin commands to prevent cascading failures.

---

## 1.10 — Pipes and Redirection

### Concept

**Pipes** (`|`) send the output of one command as input to the next command.  
**Redirection** sends command output to a file (or reads input from a file).

### Three Standard Streams

Every Linux command has three data streams:

| Stream | Number | Default | Purpose |
|--------|--------|---------|---------|
| **stdin** | 0 | Keyboard | Input to the command |
| **stdout** | 1 | Terminal screen | Normal output |
| **stderr** | 2 | Terminal screen | Error messages |

### Output Redirection

```bash
echo "Hello" > file.txt
```
**`>`:** Redirects stdout to a file. **Overwrites** the file if it exists.

```bash
echo "World" >> file.txt
```
**`>>`:** Redirects stdout to a file. **Appends** to the file.

```bash
ls /nonexistent 2> errors.txt
```
**`2>`:** Redirects stderr (errors) to a file.

```bash
ls /etc /nonexistent > output.txt 2>&1
```
**`2>&1`:** Redirects stderr to the same place as stdout. Both go to `output.txt`.

```bash
ls /etc /nonexistent &> all_output.txt
```
**`&>`:** Shorthand — redirects both stdout and stderr to a file (Bash 4+).

```bash
command > /dev/null 2>&1
```
**`/dev/null`:** The "black hole." Discards all output silently. Useful for cron jobs or when you don't care about output.

### Input Redirection

```bash
sort < unsorted_list.txt
```
**`<`:** Feeds the contents of `unsorted_list.txt` as input to `sort`.

```bash
wc -l < /etc/passwd
```
Counts lines in `/etc/passwd`.

### Here Document (Heredoc)

```bash
cat << EOF
Line 1
Line 2
Line 3
EOF
```
**What it does:** Provides multi-line input to a command. `EOF` is a delimiter (you can use any word). Useful in scripts.

### Pipes (`|`)

```bash
cat /etc/passwd | head -5
```
**What it does:** `cat` outputs the file, `|` sends that output as input to `head`, which shows the first 5 lines.

```bash
ls -la | grep ".txt"
```
**What it does:** Lists files, then filters to show only lines containing `.txt`.

```bash
cat /var/log/syslog | grep "error" | wc -l
```
**What it does:** Chain — read log → filter errors → count how many lines.

**📌 Important:** Pipes can be chained indefinitely. Each command processes the output of the previous one.

**💡 Admin Tip:** Real admin pipeline:
```bash
# Find the top 10 largest files in /var/log
du -ah /var/log | sort -rh | head -10
```

---

## 1.11 — Text Processing Commands

These are the Swiss army knives of Linux administration.

### `grep` — Search for Patterns

```bash
grep "error" /var/log/syslog
```
**What it does:** Searches for lines containing "error" in the file.

**Syntax:** `grep [options] "pattern" file`

| Option | Meaning |
|--------|---------|
| `-i` | Case-insensitive search |
| `-r` | Recursive — search in all files in a directory |
| `-n` | Show line numbers |
| `-c` | Count matching lines (don't show them) |
| `-v` | Invert — show lines that DON'T match |
| `-l` | Show only filenames that contain the match |
| `-w` | Match whole word only |
| `-A 3` | Show 3 lines After each match |
| `-B 3` | Show 3 lines Before each match |
| `-C 3` | Show 3 lines of Context (before and after) |
| `-E` | Extended regex (or use `egrep`) |
| `--color` | Highlight the matched text |

**Examples:**

```bash
# Find "failed" in auth log (case insensitive)
grep -i "failed" /var/log/auth.log

# Count how many times "error" appears in syslog
grep -c "error" /var/log/syslog

# Find "ssh" in all config files recursively
grep -r "ssh" /etc/

# Show lines that do NOT contain comments (lines starting with #)
grep -v "^#" /etc/ssh/sshd_config

# Show non-empty, non-comment lines in a config file
grep -v "^#" /etc/ssh/sshd_config | grep -v "^$"
```

**💡 Admin Tip:** The last example is something you'll use daily — config files are full of comments, and this shows you only the active settings.

**🎯 Interview Point:** "How would you find all failed SSH login attempts?"
```bash
grep "Failed password" /var/log/auth.log
```

---

### `sort` — Sort Lines

```bash
sort names.txt
```
**What it does:** Sorts lines alphabetically.

| Option | Meaning |
|--------|---------|
| `-n` | Sort numerically |
| `-r` | Reverse order |
| `-k 2` | Sort by 2nd column |
| `-t ":"` | Use `:` as field separator |
| `-u` | Remove duplicates while sorting |
| `-h` | Sort human-readable numbers (1K, 2M, 3G) |

**Example — sort disk usage:**
```bash
du -sh /var/log/* | sort -rh
```
Shows directory sizes sorted from largest to smallest.

---

### `uniq` — Remove Duplicate Lines

```bash
sort access.log | uniq
```
**What it does:** Removes **consecutive** duplicate lines. Input MUST be sorted first.

| Option | Meaning |
|--------|---------|
| `-c` | Prefix lines with count of occurrences |
| `-d` | Show only duplicated lines |
| `-u` | Show only unique lines |

**Example — find most common IP in a log:**
```bash
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```

---

### `cut` — Extract Columns

```bash
cut -d ":" -f 1 /etc/passwd
```
**What it does:** Uses `:` as delimiter (`-d ":"`), extracts field 1 (`-f 1`). Shows all usernames.

| Option | Meaning |
|--------|---------|
| `-d` | Field delimiter |
| `-f` | Field number(s) to extract |
| `-c` | Character positions |

**Examples:**
```bash
# Get username and shell from /etc/passwd
cut -d ":" -f 1,7 /etc/passwd

# Get first 10 characters of each line
cut -c 1-10 /etc/passwd
```

---

### `tr` — Translate or Delete Characters

```bash
echo "Hello World" | tr 'a-z' 'A-Z'
```
**Output:** `HELLO WORLD`  
**What it does:** Translates lowercase to uppercase.

```bash
echo "Hello   World" | tr -s ' '
```
**Output:** `Hello World`  
**`-s`:** Squeeze repeated characters.

```bash
echo "Hello123" | tr -d '0-9'
```
**Output:** `Hello`  
**`-d`:** Delete specified characters.

---

### `wc` — Word Count

```bash
wc /etc/passwd
```
**Output:** `42  45  2456 /etc/passwd` → 42 lines, 45 words, 2456 bytes

| Option | Meaning |
|--------|---------|
| `-l` | Lines only |
| `-w` | Words only |
| `-c` | Bytes only |
| `-m` | Characters only |

**Examples:**
```bash
# How many users exist?
wc -l /etc/passwd

# How many files in a directory?
ls /etc | wc -l
```

---

### `find` — Search for Files

```bash
find /home -name "*.txt"
```
**What it does:** Searches `/home` and all subdirectories for files ending in `.txt`.

**Syntax:** `find [path] [conditions] [actions]`

| Option | Meaning | Example |
|--------|---------|---------|
| `-name` | Match filename (case-sensitive) | `-name "*.log"` |
| `-iname` | Match filename (case-insensitive) | `-iname "readme*"` |
| `-type f` | Files only | `-type f` |
| `-type d` | Directories only | `-type d` |
| `-size +100M` | Files larger than 100MB | `-size +100M` |
| `-mtime -7` | Modified in the last 7 days | `-mtime -7` |
| `-mtime +30` | Modified more than 30 days ago | `-mtime +30` |
| `-user` | Owned by specific user | `-user abhishek` |
| `-perm` | Match permissions | `-perm 777` |
| `-empty` | Empty files or directories | `-empty` |
| `-maxdepth 2` | Don't go deeper than 2 levels | `-maxdepth 2` |
| `-exec` | Run a command on each match | see below |

**Practical examples:**

```bash
# Find all .log files larger than 50MB
find /var/log -name "*.log" -size +50M

# Find files modified in the last 24 hours
find /etc -mtime -1

# Find empty files and delete them
find /tmp -type f -empty -delete

# Find files owned by a specific user
find /home -user john -type f

# Find files and execute a command on each
find /var/log -name "*.log" -exec ls -lh {} \;
```

**Understanding `-exec`:**
- `{}` is replaced by each found file
- `\;` ends the `-exec` command
- Each file is processed separately

**🔴 DANGER:** Be very careful with `find ... -delete` and `find ... -exec rm`:
```bash
# ALWAYS preview first!
find /tmp -name "*.bak" -type f          # Preview
find /tmp -name "*.bak" -type f -delete  # Then delete
```

**💡 Admin Tip:** Find the top space consumers:
```bash
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null
```
The `2>/dev/null` suppresses "Permission denied" errors.

---

### `xargs` — Build Commands from Input

```bash
find /tmp -name "*.log" | xargs rm
```
**What it does:** Takes the output of `find` (a list of filenames) and passes them as arguments to `rm`.

**Why not just `-exec`?** `xargs` is faster because it batches arguments. `-exec` runs the command once per file.

```bash
# Find and compress old logs
find /var/log -name "*.log" -mtime +30 | xargs gzip

# Find and count lines in all Python files
find . -name "*.py" | xargs wc -l
```

**Handle filenames with spaces:**
```bash
find /tmp -name "*.log" -print0 | xargs -0 rm
```
`-print0` and `-0` use null characters as delimiters instead of spaces.

---

## 1.12 — Getting Help

### `man` — Manual Pages

```bash
man ls
```
**What it does:** Opens the manual page for `ls`. Shows complete documentation: synopsis, description, all options, examples.

**Navigation:** Same as `less` — `Space` for next page, `/` to search, `q` to quit.

**Manual sections:**

| Section | Content |
|---------|---------|
| 1 | User commands |
| 5 | File formats and config files |
| 8 | System administration commands |

```bash
man 5 passwd     # Explains the /etc/passwd FILE FORMAT
man 1 passwd     # Explains the passwd COMMAND
```

---

### `--help` — Quick Reference

```bash
ls --help
```
**What it does:** Shows a brief summary of the command's options. Faster than `man` when you just need to check an option.

---

### `info` — Detailed Documentation

```bash
info coreutils
```
**What it does:** More detailed than `man` for GNU tools. Less commonly used.

---

### `type` — What Kind of Command Is It?

```bash
type ls
type cd
type ll
```
Shows whether a command is a built-in, alias, function, or external program.

---

### `which` — Where Is the Command Located?

```bash
which python3
which nginx
```
Shows the full path of the command's executable.

---

### `apropos` — Search for Commands

```bash
apropos "copy files"
```
**What it does:** Searches all man page descriptions for the keyword. Useful when you know what you want to do but not which command does it.

---

## 1.13 — Hands-On Lab

### Lab 1: Filesystem Exploration

Open your WSL2 terminal and complete these tasks:

```bash
# 1. Check who you are and where you are
whoami
pwd

# 2. Go to the root of the filesystem and explore
cd /
ls -la

# 3. Explore key directories
ls /etc | head -20
ls /var/log
ls /home
ls /tmp

# 4. Go to your home directory
cd ~
pwd

# 5. Check the hostname and OS info
hostname
cat /etc/os-release
```

### Lab 2: File and Directory Operations

```bash
# 1. Go to your home directory
cd ~

# 2. Create a project directory structure
mkdir -p linux_labs/level1/{files,logs,backup}

# 3. Verify the structure
ls -R linux_labs/

# 4. Navigate into it
cd linux_labs/level1/files

# 5. Create some files
touch report.txt notes.txt data.csv script.sh

# 6. Verify
ls -la

# 7. Copy a file
cp report.txt report_backup.txt

# 8. Move/rename a file
mv notes.txt meeting_notes.txt

# 9. Copy a file to another directory
cp data.csv ../backup/

# 10. Verify backup directory
ls ../backup/

# 11. Create a file with content
echo "This is line 1" > testfile.txt
echo "This is line 2" >> testfile.txt
echo "This is line 3" >> testfile.txt

# 12. View the file
cat testfile.txt

# 13. Clean up — remove a specific file safely
rm -i report_backup.txt
```

### Lab 3: Viewing and Searching Files

```bash
# 1. Explore /etc/passwd
cat /etc/passwd
head -5 /etc/passwd
tail -5 /etc/passwd
wc -l /etc/passwd

# 2. Extract usernames
cut -d ":" -f 1 /etc/passwd

# 3. Search for your user
grep "abhishek" /etc/passwd

# 4. Count system users (UID < 1000, usually system accounts)
cat /etc/passwd | wc -l

# 5. View a config file without comments
grep -v "^#" /etc/ssh/sshd_config 2>/dev/null | grep -v "^$"

# 6. Explore with less
less /etc/services
# Inside less: search for "http" by typing /http then Enter
# Press n for next match, q to quit
```

### Lab 4: Pipes and Redirection

```bash
# 1. Create a test file
echo -e "banana\napple\ncherry\napple\nbanana\ndate\napple" > fruits.txt

# 2. Sort the file
sort fruits.txt

# 3. Sort and remove duplicates
sort fruits.txt | uniq

# 4. Count occurrences of each item
sort fruits.txt | uniq -c | sort -rn

# 5. Save output to a file
sort fruits.txt | uniq -c | sort -rn > fruit_counts.txt
cat fruit_counts.txt

# 6. Find all .conf files in /etc
find /etc -name "*.conf" -type f 2>/dev/null | head -20

# 7. Count .conf files
find /etc -name "*.conf" -type f 2>/dev/null | wc -l

# 8. Redirect errors
find / -name "*.conf" 2>/dev/null | wc -l

# 9. Pipeline — top 5 largest files in /etc
du -a /etc 2>/dev/null | sort -rn | head -5
```

---

## 1.14 — Troubleshooting Exercises

### Exercise 1: "Command Not Found"

**Problem:** You type a command and get `command not found`.

**Diagnosis steps:**
```bash
# Check if the command exists
which nginx
type nginx

# Check if the package is installed (Ubuntu)
dpkg -l | grep nginx

# Check if it's in your PATH
echo $PATH
```

**Common causes:**
1. The program isn't installed → install it
2. You made a typo
3. The command is in a directory not in your `$PATH`
4. You need to use the full path: `/usr/sbin/nginx`

### Exercise 2: "Permission Denied"

**Problem:** You try to view or edit a file and get `Permission denied`.

```bash
cat /etc/shadow    # Permission denied!
```

**Solution:**
```bash
ls -la /etc/shadow     # Check permissions
sudo cat /etc/shadow   # Use sudo for admin access
```

### Exercise 3: "No Such File or Directory"

**Problem:** A command says a file doesn't exist, but you're sure it does.

**Diagnosis:**
```bash
# Check your current location
pwd

# Are you in the right directory?
ls

# Check for typos — Linux is CASE-SENSITIVE
ls -la | grep -i "readme"    # Case-insensitive search

# Check if it's a symlink that points to nothing
ls -la filename
```

**⚠️ Common Mistake:** Linux filenames are case-sensitive. `File.txt`, `file.txt`, and `FILE.TXT` are three different files.

---

## 1.15 — Practical Scenarios

### Scenario 1: Investigate a System

**Task:** You've just logged into a new Linux server for the first time. Gather basic information about it.

**Try this yourself first, then check the solution below.**

<details>
<summary><strong>Solution</strong></summary>

```bash
# What OS and version?
cat /etc/os-release

# What's the hostname?
hostname

# What's the kernel version?
uname -r

# Full system info
uname -a

# How long has it been running?
uptime

# How much disk space?
df -h

# How much memory?
free -h

# Who is logged in?
who

# What's the IP address?
ip addr show    # or: hostname -I
```

</details>

### Scenario 2: Find Large Log Files

**Task:** Your server's disk is getting full. Find the largest files in `/var/log`.

**Try this yourself first.**

<details>
<summary><strong>Solution</strong></summary>

```bash
# Check overall disk usage
df -h

# Find large files in /var/log
du -ah /var/log 2>/dev/null | sort -rh | head -20

# Find files larger than 50MB
find /var/log -type f -size +50M -exec ls -lh {} \; 2>/dev/null

# Check individual log sizes
ls -lhS /var/log/

# Check total log directory size
du -sh /var/log
```

</details>

---

## 1.16 — Practice Questions

Complete these tasks on your own to reinforce what you've learned:

1. **Create a directory structure:**
   ```
   ~/projects/webapp/{src,tests,docs,config}
   ```
   Then create files: `src/app.py`, `tests/test_app.py`, `docs/README.md`, `config/settings.conf`

2. **Extract all unique shells** used by users on the system from `/etc/passwd`. Sort them and count how many users use each shell.

3. **Find all files** in `/etc` that were modified in the last 7 days. Save the list to `~/recent_configs.txt`.

4. **Create a file** called `server_info.txt` that contains:
   - The current date
   - The hostname
   - The OS version
   - The kernel version
   - The disk usage
   - The memory usage

   Do this using only redirection (`>` and `>>`), not a text editor.

5. **Pipeline challenge:** Using a single pipeline, find all lines in `/etc/passwd` that contain `/bin/bash`, extract just the usernames (first field), sort them, and count how many there are.

6. **History:** Find the 5 most frequently used commands in your bash history.
   Hint: `history | awk '{print $2}' | sort | uniq -c | sort -rn | head -5`

---

## Level 1 Summary

### What I Learned

- Linux is an open-source OS used on 90%+ of servers
- The kernel manages hardware; distributions package everything together
- The terminal and Bash shell are the primary admin interface
- Linux has a single root (`/`) filesystem with a standard hierarchy
- Navigation: `pwd`, `cd`, `ls`
- File operations: `touch`, `mkdir`, `cp`, `mv`, `rm`
- Viewing files: `cat`, `less`, `head`, `tail`, `tail -f`
- Pipes (`|`) chain commands; redirection (`>`, `>>`, `2>`) sends output to files
- Text processing: `grep`, `sort`, `uniq`, `cut`, `tr`, `wc`
- File searching: `find` with conditions and actions
- `xargs` builds commands from piped input
- Getting help: `man`, `--help`, `apropos`

### Commands to Remember

| Command | Purpose |
|---------|---------|
| `pwd` | Print current directory |
| `cd` | Change directory |
| `ls -lah` | List files with details |
| `mkdir -p` | Create nested directories |
| `cp -r` | Copy directories |
| `mv` | Move or rename |
| `rm -r` | Remove directories (CAREFUL!) |
| `cat` | Display file contents |
| `less` | Page through large files |
| `head -n` / `tail -n` | View start/end of files |
| `tail -f` | Follow log files in real-time |
| `grep -i` | Search text (case-insensitive) |
| `find` | Search for files by name, size, date |
| `sort` / `uniq` | Sort and deduplicate |
| `cut -d -f` | Extract columns |
| `wc -l` | Count lines |
| `xargs` | Build commands from stdin |
| `>` / `>>` | Redirect / append output |
| `\|` | Pipe output to next command |
| `2>/dev/null` | Suppress errors |
| `man` / `--help` | Get help |

### Practical Skills Gained

- Navigate the Linux filesystem confidently
- Create, copy, move, and delete files and directories
- View and search through files and logs
- Build command pipelines for data processing
- Get information about a system you've never seen before

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| `rm -rf /` or `rm -rf ~` | NEVER do this. Always double-check paths |
| Forgetting `-r` when copying directories | Use `cp -r` for directories |
| Using `cat` on huge files | Use `less` instead |
| Case sensitivity errors | Linux cares: `File.txt` ≠ `file.txt` |
| Spaces in filenames causing issues | Quote them: `"my file.txt"` or escape: `my\ file.txt` |
| Forgetting `sudo` for admin operations | Check the error — "Permission denied" means try `sudo` |

### Administrator-Level Knowledge

- Know the filesystem hierarchy and what each directory contains
- Use `tail -f` for real-time log monitoring
- Use `grep -v "^#" | grep -v "^$"` to read config files efficiently
- Use `find` with `-size`, `-mtime` for storage investigation
- Use `du -ah | sort -rh | head` to find space consumers
- Redirect errors with `2>/dev/null` for clean output

### Interview Questions

1. **What is the difference between the Linux kernel and a distribution?**
2. **Explain the Linux filesystem hierarchy.**
3. **What is the difference between `>` and `>>`?**
4. **How do you find all files larger than 100MB?**
5. **How do you watch a log file in real-time?**
6. **What does `2>&1` mean?**
7. **What is the difference between absolute and relative paths?**
8. **How would you find the most recently modified files in a directory?**
9. **What is `/dev/null` and why would you redirect output there?**
10. **Explain what this pipeline does:** `cat /var/log/syslog | grep "error" | wc -l`

---

*When you've completed all the labs and practice questions, proceed to [Level 2 — Users, Groups & Permissions](level-02-users-groups-permissions.md)*
