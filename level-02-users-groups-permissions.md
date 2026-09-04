# Level 2 — Users, Groups & Permissions

> **Goal:** Understand how Linux manages users, groups, file ownership, and permissions. Be able to create users, assign permissions, troubleshoot access issues, and configure sudo.

> **Lab Environment:** WSL2 Ubuntu or VirtualBox VM (VM preferred for full user management)

> **Prerequisite:** Complete Level 1 — Linux Fundamentals

---

## 2.1 — Understanding Users in Linux

### Minimum Theory

Linux is a **multi-user operating system**. Multiple users can be logged in and working simultaneously. Every file, process, and action on the system is associated with a user.

**📌 Important — Two types of users:**

| Type | UID Range | Purpose | Examples |
|------|-----------|---------|----------|
| **Root user** | 0 | Superuser — unrestricted access to everything | `root` |
| **System users** | 1–999 (Debian/Ubuntu) or 1–499 (RHEL) | Run services/daemons, cannot log in interactively | `www-data`, `mysql`, `sshd`, `nobody` |
| **Regular users** | 1000+ (Debian/Ubuntu) or 500+ (RHEL) | Real people who log in and work | `abhishek`, `john`, `deploy` |

**Why this matters in real systems:**
- Every process runs as a specific user
- File access is controlled by user ownership
- Security depends on proper user configuration
- Services should NEVER run as root (principle of least privilege)

**🎯 Interview Point:** "What is the root user?"  
> Root (UID 0) is the superuser with unlimited access to the entire system. It bypasses all permission checks. In production, you should never log in as root directly — use `sudo` instead.

---

## 2.2 — User Database Files

### `/etc/passwd` — User Account Information

```bash
cat /etc/passwd
```

Every user on the system has a line in this file. Despite the name, it does **NOT** contain passwords (historically it did).

**Format — 7 fields separated by `:`:**

```
abhishek:x:1000:1000:Abhishek Maurya:/home/abhishek:/bin/bash
   │      │  │    │        │              │             │
   │      │  │    │        │              │             └── Login shell
   │      │  │    │        │              └── Home directory
   │      │  │    │        └── Comment/Full name (GECOS field)
   │      │  │    └── Primary GID (Group ID)
   │      │  └── UID (User ID)
   │      └── Password placeholder ('x' means password is in /etc/shadow)
   └── Username
```

**Practical example — extracting useful info:**

```bash
# List all usernames
cut -d ":" -f 1 /etc/passwd

# List all users with their shells
cut -d ":" -f 1,7 /etc/passwd

# Find users who can log in (have a real shell)
grep -v "nologin\|false" /etc/passwd | cut -d ":" -f 1

# Count total users
wc -l /etc/passwd

# Find a specific user
grep "^abhishek:" /etc/passwd
```

**💡 Admin Tip:** The `^abhishek:` pattern uses `^` to match the beginning of the line. This prevents matching a user like `abhishek2` or a comment containing "abhishek".

---

### `/etc/shadow` — Password Hashes

```bash
sudo cat /etc/shadow
```

**📌 Important:** This file is readable ONLY by root. It contains the actual password hashes.

**Format — 9 fields separated by `:`:**

```
abhishek:$6$xyz...hash...:19500:0:99999:7:::
   │          │              │   │   │    │
   │          │              │   │   │    └── Days before expiry to warn
   │          │              │   │   └── Maximum password age (days)
   │          │              │   └── Minimum password age (days)
   │          │              └── Last password change (days since Jan 1, 1970)
   │          └── Password hash ($6$ = SHA-512, $y$ = yescrypt)
   └── Username
```

**Special password field values:**

| Value | Meaning |
|-------|---------|
| `$6$...` | SHA-512 hashed password |
| `$y$...` | yescrypt hashed password (newer systems) |
| `!` or `!!` | Account is locked (cannot log in with password) |
| `*` | Account never had a password set |
| _(empty)_ | No password required (DANGEROUS!) |

**⚠️ Common Mistake:** Never edit `/etc/shadow` manually. Use `passwd`, `usermod`, or `chage` commands instead.

**Check password status:**
```bash
sudo passwd -S abhishek
```
**Expected output:**
```
abhishek P 09/01/2026 0 99999 7 -1
```
- `P` = password is set and usable
- `L` = account is locked
- `NP` = no password

---

### `/etc/group` — Group Information

```bash
cat /etc/group
```

**Format — 4 fields separated by `:`:**

```
developers:x:1001:abhishek,john,sara
    │       │  │        │
    │       │  │        └── Group members (comma-separated)
    │       │  └── GID (Group ID)
    │       └── Password placeholder (rarely used)
    └── Group name
```

**Practical commands:**

```bash
# List all groups
cut -d ":" -f 1 /etc/group

# Find which groups a user belongs to
groups abhishek

# Alternative — more detailed
id abhishek

# Find members of a specific group
grep "^developers:" /etc/group
```

**`id` command output:**
```bash
id abhishek
```
```
uid=1000(abhishek) gid=1000(abhishek) groups=1000(abhishek),4(adm),27(sudo),1001(developers)
```

This shows: UID, primary GID, and all groups the user belongs to.

---

### `/etc/gshadow` — Group Password Hashes

```bash
sudo cat /etc/gshadow
```

Rarely used. Contains group passwords and group administrators. You'll almost never need to edit this directly.

---

## 2.3 — Managing Users

### `useradd` — Create a New User

```bash
sudo useradd john
```
**What it does:** Creates a user called `john` with default settings.

**🔴 IMPORTANT:** On most systems, `useradd` alone does NOT:
- Create a home directory (on some distros)
- Set a password
- Set a login shell

**✅ Best Practice — Always use these options:**

```bash
sudo useradd -m -s /bin/bash -c "John Smith" john
```

| Option | Meaning |
|--------|---------|
| `-m` | Create home directory (`/home/john`) |
| `-s /bin/bash` | Set the login shell |
| `-c "John Smith"` | Set the comment/full name |
| `-d /custom/path` | Custom home directory path |
| `-u 1500` | Specify a custom UID |
| `-g groupname` | Set primary group |
| `-G group1,group2` | Add to supplementary groups |
| `-e 2027-12-31` | Account expiration date |
| `-r` | Create a system user (UID < 1000, no home dir) |

**Complete example — create a new developer:**
```bash
sudo useradd -m -s /bin/bash -c "John Smith" -G sudo,developers john
sudo passwd john
```

**What happens internally:**
1. A line is added to `/etc/passwd`
2. A line is added to `/etc/shadow`
3. A line is added to `/etc/group` (creates a group with the same name)
4. Home directory is created (if `-m` is used)
5. Default files from `/etc/skel` are copied to the home directory

**Verify the user was created:**
```bash
id john
ls -la /home/john
grep "^john:" /etc/passwd
```

---

### `/etc/skel` — Skeleton Directory

```bash
ls -la /etc/skel
```
**What it is:** A template directory. Every file in `/etc/skel` is copied to a new user's home directory when the account is created.

**Typical contents:**
```
.bash_logout
.bashrc
.profile
```

**💡 Admin Tip:** If you want all new users to have a `README.txt` or specific `.bashrc` configuration, add it to `/etc/skel` BEFORE creating users.

```bash
sudo cp custom_bashrc /etc/skel/.bashrc
```

---

### `/etc/login.defs` — Default Values for User Creation

```bash
cat /etc/login.defs
```
Contains system-wide defaults for:
- UID/GID ranges
- Password aging defaults
- Home directory creation
- umask

**💡 Admin Tip:** Check this file when you need to understand why new users are created with specific settings.

---

### `adduser` (Debian/Ubuntu Only) — Interactive User Creation

```bash
sudo adduser john
```

**📌 Important:** `adduser` is a **Debian/Ubuntu-specific** friendly wrapper around `useradd`. It's interactive and automatically:
- Creates the home directory
- Copies `/etc/skel` files
- Prompts for a password
- Asks for full name and other info

| Command | Distribution | Behavior |
|---------|-------------|----------|
| `useradd` | All Linux | Low-level, non-interactive, requires options |
| `adduser` | Debian/Ubuntu | High-level, interactive, user-friendly |

**✅ Best Practice:** On Ubuntu, use `adduser` for interactive creation. Use `useradd` in scripts (non-interactive).

---

### `usermod` — Modify an Existing User

```bash
sudo usermod [options] username
```

| Option | Meaning | Example |
|--------|---------|---------|
| `-aG group` | Add to supplementary group | `sudo usermod -aG docker john` |
| `-s /bin/zsh` | Change login shell | `sudo usermod -s /bin/zsh john` |
| `-d /new/home` | Change home directory | `sudo usermod -d /new/home john` |
| `-d /new/home -m` | Change home AND move files | `sudo usermod -d /new/home -m john` |
| `-l newname` | Rename the user | `sudo usermod -l john_smith john` |
| `-L` | Lock the account | `sudo usermod -L john` |
| `-U` | Unlock the account | `sudo usermod -U john` |
| `-e 2027-12-31` | Set expiration date | `sudo usermod -e 2027-12-31 john` |
| `-c "New Name"` | Change comment/full name | `sudo usermod -c "John D. Smith" john` |

**🔴 CRITICAL — The `-aG` vs `-G` trap:**

```bash
# CORRECT — ADD to existing groups
sudo usermod -aG docker john

# WRONG — REPLACES all supplementary groups with only "docker"!
sudo usermod -G docker john
```

**⚠️ Common Mistake:** Forgetting the `-a` (append) flag removes the user from ALL other supplementary groups. This is one of the most common admin mistakes.

**Verify the change:**
```bash
groups john
id john
```

---

### `userdel` — Delete a User

```bash
sudo userdel john
```
**What it does:** Removes the user account. Does NOT delete the home directory.

```bash
sudo userdel -r john
```
**`-r`:** Remove the home directory and mail spool too.

**🔴 DANGER — Understand before executing:**
- `userdel -r` permanently deletes the user's home directory and all their files
- Back up important data first if needed
- Check for running processes owned by the user first:

```bash
# Check if user has running processes
ps -u john

# Kill all processes owned by user (if needed)
sudo pkill -u john

# Then delete
sudo userdel -r john
```

**💡 Admin Tip:** In production, instead of deleting users immediately, lock them first:
```bash
sudo usermod -L john                    # Lock account
sudo usermod -s /sbin/nologin john      # Prevent login
# Wait for a retention period, then delete
sudo userdel -r john
```

---

### `passwd` — Manage Passwords

```bash
sudo passwd john
```
**What it does:** Set or change the password for user `john`.

```bash
passwd
```
**What it does:** Change YOUR OWN password (no sudo needed).

| Option | Meaning | Example |
|--------|---------|---------|
| `-l` | Lock the password | `sudo passwd -l john` |
| `-u` | Unlock the password | `sudo passwd -u john` |
| `-d` | Delete password (allow passwordless login — DANGEROUS) | `sudo passwd -d john` |
| `-e` | Expire password (force change at next login) | `sudo passwd -e john` |
| `-S` | Show password status | `sudo passwd -S john` |
| `-n 7` | Minimum days between password changes | `sudo passwd -n 7 john` |
| `-x 90` | Maximum password age (days) | `sudo passwd -x 90 john` |
| `-w 14` | Warning days before expiry | `sudo passwd -w 14 john` |

**💡 Admin Tip:** Force a new user to change their password on first login:
```bash
sudo useradd -m -s /bin/bash newuser
sudo passwd newuser
sudo passwd -e newuser    # Expire immediately
```

---

### `chage` — Change Password Aging

```bash
sudo chage -l john
```
**What it does:** List password aging information for user `john`.

**Expected output:**
```
Last password change                    : Sep 04, 2026
Password expires                        : Dec 03, 2026
Password inactive                       : never
Account expires                         : never
Minimum number of days between password change  : 0
Maximum number of days between password change  : 90
Number of days of warning before password expires: 7
```

| Option | Meaning |
|--------|---------|
| `-l` | List password aging info |
| `-M 90` | Max days before password must be changed |
| `-m 7` | Min days between password changes |
| `-W 14` | Warning days before expiry |
| `-I 30` | Days of inactivity after expiry before account is locked |
| `-E 2027-12-31` | Account expiration date |
| `-d 0` | Force password change at next login (set last change to epoch) |

**✅ Best Practice — Set a password policy for new users:**
```bash
sudo chage -M 90 -m 7 -W 14 john
```
This means: Password must be changed every 90 days, cannot be changed within 7 days of the last change, and user gets a warning 14 days before expiry.

**🎯 Interview Point:** "How do you enforce password expiration on Linux?"  
> Use `chage -M 90 username` to set max password age to 90 days. Use `chage -l username` to verify.

---

## 2.4 — Managing Groups

### Why Groups Matter

Groups let you manage permissions for multiple users at once. Instead of giving each user individual access to a file, you:
1. Create a group
2. Add the relevant users to it
3. Set the file's group ownership to that group

**Example:** A team of 5 developers all need access to `/var/www/project`:
- Create group `webdevs`
- Add all 5 developers to `webdevs`
- Set `/var/www/project` to be owned by group `webdevs`

### Primary vs Secondary Groups

| Type | Description | How Set |
|------|------------|---------|
| **Primary group** | The default group for files the user creates. Usually same as username. | Set in `/etc/passwd` (GID field) |
| **Secondary (supplementary) groups** | Additional groups the user belongs to | Listed in `/etc/group` |

```bash
# Check primary and secondary groups
id abhishek
```
```
uid=1000(abhishek) gid=1000(abhishek) groups=1000(abhishek),4(adm),27(sudo)
```
- Primary group: `abhishek` (GID 1000)
- Secondary groups: `adm`, `sudo`

**📌 Important:** When a user creates a file, it's owned by the user and their **primary group**.

---

### `groupadd` — Create a Group

```bash
sudo groupadd developers
```
**What it does:** Creates a new group called `developers`.

| Option | Meaning |
|--------|---------|
| `-g 2000` | Specify a custom GID |
| `-r` | Create a system group (GID < 1000) |

**Verify:**
```bash
grep "^developers:" /etc/group
getent group developers
```

---

### `groupmod` — Modify a Group

```bash
sudo groupmod -n dev_team developers
```
**What it does:** Renames group `developers` to `dev_team`.

| Option | Meaning |
|--------|---------|
| `-n newname` | Rename the group |
| `-g newGID` | Change the GID |

---

### `groupdel` — Delete a Group

```bash
sudo groupdel developers
```
**What it does:** Removes the group. Cannot delete a group if it's any user's primary group.

**⚠️ Common Mistake:** Trying to delete a group that is someone's primary group:
```bash
sudo groupdel abhishek
# Error: cannot remove the primary group of user 'abhishek'
```
**Fix:** Change the user's primary group first, then delete.

---

### `gpasswd` — Manage Group Membership

```bash
# Add a user to a group
sudo gpasswd -a john developers

# Remove a user from a group
sudo gpasswd -d john developers

# Set group administrators
sudo gpasswd -A abhishek developers
```

| Option | Meaning |
|--------|---------|
| `-a user` | Add user to group |
| `-d user` | Remove user from group |
| `-A user` | Make user a group administrator |
| `-M user1,user2` | Set the member list (replaces all members!) |

**💡 Admin Tip:** `gpasswd -a` is safer than `usermod -aG` because it only adds — there's no risk of accidentally removing groups.

---

### `newgrp` — Switch Active Group

```bash
newgrp developers
```
**What it does:** Temporarily changes your primary group for the current session. New files you create will be owned by the `developers` group.

```bash
touch testfile.txt    # This file will be owned by group "developers"
exit                  # Return to your original primary group
```

---

## 2.5 — File Ownership

Every file and directory in Linux has:
1. **User owner** — one user
2. **Group owner** — one group

```bash
ls -la
```
```
-rw-r--r-- 1 abhishek developers 1024 Sep 04 10:00 project.conf
               │          │
               │          └── Group owner
               └── User owner
```

### `chown` — Change File Owner

```bash
sudo chown john file.txt
```
**What it does:** Changes the user owner of `file.txt` to `john`.

**Syntax:** `chown [options] user[:group] file`

```bash
# Change owner only
sudo chown john file.txt

# Change owner and group
sudo chown john:developers file.txt

# Change group only (note the colon before the group)
sudo chown :developers file.txt

# Change ownership recursively (all files in a directory)
sudo chown -R john:developers /var/www/project/
```

| Option | Meaning |
|--------|---------|
| `-R` | Recursive — apply to all contents of a directory |
| `-v` | Verbose — show each change |
| `--reference=reffile` | Copy ownership from another file |

**Admin use case — web server files:**
```bash
# Set proper ownership for a web application
sudo chown -R www-data:www-data /var/www/html/
```

**⚠️ Common Mistake:** Forgetting `-R` when changing ownership of a directory with contents:
```bash
sudo chown john:developers /var/www/project     # Changes only the directory itself!
sudo chown -R john:developers /var/www/project   # Changes directory AND all contents
```

---

### `chgrp` — Change Group Owner

```bash
sudo chgrp developers file.txt
```
**What it does:** Changes only the group owner. Equivalent to `chown :developers file.txt`.

```bash
# Recursive group change
sudo chgrp -R developers /shared/project/
```

---

## 2.6 — File Permissions (chmod)

### Understanding Permission Bits

Every file has three sets of permissions for three categories:

```
-rwxr-xr--
 │││ │││ │││
 │││ │││ └┴┴── Others (everyone else)
 │││ └┴┴────── Group (members of the group owner)
 └┴┴────────── User/Owner (the file's owner)
```

| Symbol | Permission | On Files | On Directories |
|--------|-----------|----------|----------------|
| `r` | Read | View file contents | List directory contents (`ls`) |
| `w` | Write | Modify file contents | Create/delete files in the directory |
| `x` | Execute | Run the file as a program | Enter the directory (`cd`) |
| `-` | No permission | — | — |

**📌 Important — Permissions on directories:**

| Permission | File | Directory |
|-----------|------|-----------|
| `r` (read) | `cat`, `less` the file | `ls` the directory |
| `w` (write) | Edit the file | Create/delete files inside |
| `x` (execute) | Run the file | `cd` into the directory |

**⚠️ Common Mistake:** A directory without `x` permission is unusable even if it has `r`. You need `x` to enter (`cd`) a directory and access its files.

---

### Numeric (Octal) Permissions

Each permission has a numeric value:

| Permission | Value |
|-----------|-------|
| Read (`r`) | 4 |
| Write (`w`) | 2 |
| Execute (`x`) | 1 |
| No permission (`-`) | 0 |

Add the values for each category:

| Numeric | Symbolic | Meaning |
|---------|----------|---------|
| `7` | `rwx` | Read + Write + Execute (4+2+1) |
| `6` | `rw-` | Read + Write (4+2) |
| `5` | `r-x` | Read + Execute (4+1) |
| `4` | `r--` | Read only (4) |
| `3` | `-wx` | Write + Execute (2+1) |
| `2` | `-w-` | Write only (2) |
| `1` | `--x` | Execute only (1) |
| `0` | `---` | No permissions (0) |

**Common permission combinations:**

| Numeric | Symbolic | Meaning | Used For |
|---------|----------|---------|----------|
| `755` | `rwxr-xr-x` | Owner: full, Others: read+exec | Directories, scripts, executables |
| `644` | `rw-r--r--` | Owner: read+write, Others: read | Regular files, config files |
| `700` | `rwx------` | Owner only | Private directories, SSH keys dir |
| `600` | `rw-------` | Owner read+write only | Private files, SSH keys, secrets |
| `777` | `rwxrwxrwx` | Everyone: full access | Almost NEVER appropriate |
| `750` | `rwxr-x---` | Owner: full, Group: read+exec | Shared project directories |
| `640` | `rw-r-----` | Owner: read+write, Group: read | Shared config files |

---

### `chmod` — Change Permissions

#### Numeric Method

```bash
chmod 755 script.sh
```
**What it does:** Sets permissions to `rwxr-xr-x` (owner: full, group: read+exec, others: read+exec).

```bash
chmod 644 config.conf
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
```

#### Symbolic Method

**Syntax:** `chmod [who][operator][permission] file`

**Who:**

| Symbol | Meaning |
|--------|---------|
| `u` | User (owner) |
| `g` | Group |
| `o` | Others |
| `a` | All (u+g+o) |

**Operators:**

| Symbol | Meaning |
|--------|---------|
| `+` | Add permission |
| `-` | Remove permission |
| `=` | Set exact permission |

**Examples:**

```bash
# Add execute permission for the owner
chmod u+x script.sh

# Remove write permission for others
chmod o-w file.txt

# Add read permission for group
chmod g+r report.txt

# Set exact permissions: owner=rwx, group=rx, others=none
chmod u=rwx,g=rx,o= project_dir/

# Add execute for everyone
chmod a+x deploy.sh

# Remove all permissions for others
chmod o= secret.txt

# Make a file readable by everyone
chmod a+r public_doc.txt
```

#### Recursive Permission Change

```bash
chmod -R 755 /var/www/html/
```
**`-R`:** Apply recursively to all files and subdirectories.

**⚠️ Common Mistake:** Using `chmod -R 755` on a web directory sets execute on ALL files including images, CSS, HTML. Usually you want:
- Directories: `755` (need `x` to enter)
- Files: `644` (no need for `x` on non-scripts)

**✅ Best Practice — Set different permissions for files and directories:**
```bash
# Set directories to 755
find /var/www/html -type d -exec chmod 755 {} \;

# Set files to 644
find /var/www/html -type f -exec chmod 644 {} \;
```

**🎯 Interview Point:** "A user cannot `cd` into a directory but can `ls` it. What's wrong?"  
> The directory has `r` permission but not `x`. You need `x` (execute) to enter a directory. Fix: `chmod u+x directory/`

---

## 2.7 — Special Permissions: SUID, SGID, Sticky Bit

These are advanced permission bits that exist beyond the standard `rwx`.

### SUID (Set User ID) — Bit Value: 4

```
-rwsr-xr-x  1 root root  59976 Sep  1 00:00 /usr/bin/passwd
   ^
   s = SUID is set
```

**What it does:** When a file with SUID is executed, it runs with the **owner's permissions**, not the user's permissions.

**Why it exists:** The `passwd` command needs to write to `/etc/shadow` (owned by root). Normal users can't write to that file. SUID lets `passwd` run as root temporarily so it can update the shadow file.

```bash
# Find all SUID files on the system
find / -perm -4000 -type f 2>/dev/null
```

**Common SUID files:**

| File | Why SUID |
|------|----------|
| `/usr/bin/passwd` | Users need to change their own password (writes to `/etc/shadow`) |
| `/usr/bin/sudo` | Needs to execute commands as root |
| `/usr/bin/su` | Needs to switch to another user |
| `/usr/bin/ping` | Needs raw network socket access |

**Set SUID:**
```bash
chmod u+s program
chmod 4755 program
```

**🔴 DANGER:** SUID on the wrong file is a security vulnerability. A SUID shell (`chmod u+s /bin/bash`) gives anyone root access. Never set SUID unless you fully understand the implications.

**💡 Admin Tip:** Periodically audit SUID files on your system:
```bash
find / -perm -4000 -type f -ls 2>/dev/null
```
Compare against a known-good baseline. Unexpected SUID files could indicate a compromised system.

---

### SGID (Set Group ID) — Bit Value: 2

#### On Files:
```
-rwxr-sr-x  1 root tty  22872 Sep  1 00:00 /usr/bin/wall
         ^
         s = SGID is set
```

**On files:** The program runs with the **group's permissions** instead of the user's group.

#### On Directories (More Commonly Used):

```bash
chmod g+s /shared/projects/
```

**What it does on a directory:** Any new file or subdirectory created inside inherits the **directory's group**, not the creating user's primary group.

**Why it's useful:** Team collaboration.

**Example — Shared project directory:**
```bash
# Create shared directory
sudo mkdir /shared/projects
sudo groupadd webteam
sudo chown root:webteam /shared/projects
sudo chmod 2775 /shared/projects

# Now any file created inside will automatically belong to group "webteam"
# regardless of who creates it
```

Without SGID: Files get the creator's primary group → other team members can't access them.  
With SGID: Files get the directory's group (`webteam`) → everyone in the team can access them.

**Set SGID:**
```bash
chmod g+s directory/
chmod 2775 directory/
```

**🎯 Interview Point:** "How do you set up a shared directory where all files are group-accessible?"  
> Use SGID: `chmod 2775 /shared/dir`. This makes all new files inherit the directory's group ownership.

---

### Sticky Bit — Bit Value: 1

```bash
ls -ld /tmp
```
```
drwxrwxrwt  10 root root 4096 Sep  4 10:00 /tmp
          ^
          t = sticky bit is set
```

**What it does:** In a directory with the sticky bit, only the **file owner** (or root) can delete or rename files — even if others have write permission on the directory.

**Why it exists:** `/tmp` is world-writable. Without the sticky bit, any user could delete any other user's temporary files.

**Set sticky bit:**
```bash
chmod +t directory/
chmod 1777 directory/
```

**📌 Important — Reading the fourth digit:**

| Numeric | Meaning |
|---------|---------|
| `4xxx` | SUID |
| `2xxx` | SGID |
| `1xxx` | Sticky bit |
| `6xxx` | SUID + SGID |
| `3xxx` | SGID + Sticky |
| `7xxx` | All three |

**Visual indicator in `ls -l`:**

| Character | Position | Meaning |
|-----------|----------|---------|
| `s` in owner's `x` position | `-rw**s**r-xr-x` | SUID is set (owner has `x` too) |
| `S` in owner's `x` position | `-rw**S**r-xr-x` | SUID is set but owner lacks `x` |
| `s` in group's `x` position | `-rwxr-**s**r-x` | SGID is set (group has `x` too) |
| `S` in group's `x` position | `-rwxr-**S**r-x` | SGID is set but group lacks `x` |
| `t` in others' `x` position | `drwxrwxrw**t**` | Sticky bit (others have `x` too) |
| `T` in others' `x` position | `drwxrwxrw**T**` | Sticky bit but others lack `x` |

**Uppercase (`S`, `T`) means the underlying execute permission is NOT set — usually a misconfiguration.**

---

## 2.8 — umask — Default Permission Mask

### What Is umask?

When you create a file or directory, it doesn't get `777` permissions by default. The **umask** subtracts permissions from the maximum.

**Default maximums:**
- Files: `666` (no execute by default — security measure)
- Directories: `777`

**Default umask (usually):** `0022`

**Calculation:**

| | Files | Directories |
|--|-------|------------|
| Maximum | 666 | 777 |
| Minus umask | -022 | -022 |
| **Result** | **644** (`rw-r--r--`) | **755** (`rwxr-xr-x`) |

```bash
# Check current umask
umask
```
**Output:** `0022`

```bash
# Check umask in symbolic form
umask -S
```
**Output:** `u=rwx,g=rx,o=rx`

### Changing umask

```bash
# More restrictive — no permissions for others
umask 027
# Files: 640 (rw-r-----), Directories: 750 (rwxr-x---)

# Very restrictive — only owner has access
umask 077
# Files: 600 (rw-------), Directories: 700 (rwx------)
```

**Temporary vs Permanent:**
- Running `umask 027` in terminal only affects the current session
- To make it permanent, add to `~/.bashrc` or `/etc/profile`

```bash
# Make permanent for a user
echo "umask 027" >> ~/.bashrc

# Make permanent system-wide
sudo sh -c 'echo "umask 027" >> /etc/profile'
```

**✅ Best Practice:** On servers, use `umask 027` or `umask 077` for better security. The default `022` is too permissive for sensitive environments.

**🎯 Interview Point:** "What is umask and why is it important?"  
> umask defines the default permissions for newly created files and directories. It subtracts from the maximum (666 for files, 777 for dirs). A value of `022` gives files `644` and dirs `755`.

---

## 2.9 — Access Control Lists (ACLs)

### Why ACLs?

Standard Linux permissions allow only ONE user owner and ONE group owner. What if you need:
- User `john` gets read-only access
- User `sara` gets read-write access
- Group `devs` gets read-execute access
- Everyone else gets nothing

Standard `chmod` can't do this. **ACLs** provide fine-grained access control.

### Checking ACL Support

```bash
# Check if the filesystem supports ACLs
mount | grep -i acl
```

Most modern filesystems (ext4, XFS) support ACLs by default.

### `getfacl` — View ACLs

```bash
getfacl file.txt
```
**Expected output:**
```
# file: file.txt
# owner: abhishek
# group: abhishek
user::rw-
group::r--
other::r--
```

This shows standard permissions (no extra ACL entries yet).

### `setfacl` — Set ACLs

```bash
# Give user john read-write access
setfacl -m u:john:rw file.txt

# Give group devs read-execute access
setfacl -m g:devs:rx file.txt

# Remove ACL for user john
setfacl -x u:john file.txt

# Remove all ACLs
setfacl -b file.txt
```

**Syntax:** `setfacl -m [type]:[name]:[permissions] file`

| Option | Meaning |
|--------|---------|
| `-m` | Modify (add/change ACL entry) |
| `-x` | Remove a specific ACL entry |
| `-b` | Remove ALL ACLs |
| `-R` | Recursive |
| `-d` | Set default ACL (for new files in a directory) |

**Types:**

| Type | Meaning |
|------|---------|
| `u:username:perms` | ACL for a specific user |
| `g:groupname:perms` | ACL for a specific group |
| `o::perms` | ACL for others |
| `m::perms` | Effective rights mask |

### Practical Example — Project Directory with ACLs

```bash
# Create project directory
sudo mkdir /projects/webapp
sudo chown root:root /projects/webapp
sudo chmod 770 /projects/webapp

# Give developer john full access
sudo setfacl -m u:john:rwx /projects/webapp

# Give designer sara read-only access
sudo setfacl -m u:sara:r-x /projects/webapp

# Give QA group read-execute access
sudo setfacl -m g:qa:r-x /projects/webapp

# Set default ACL — new files will inherit these ACLs
sudo setfacl -d -m u:john:rwx /projects/webapp
sudo setfacl -d -m u:sara:r-x /projects/webapp
sudo setfacl -d -m g:qa:r-x /projects/webapp

# Verify
getfacl /projects/webapp
```

### How to Spot ACLs

```bash
ls -la
```
```
-rw-rw-r--+ 1 abhishek abhishek 1024 Sep 4 10:00 file.txt
          ^
          + sign means ACLs are set on this file
```

The `+` at the end of the permission string indicates ACLs are present. Use `getfacl` to see them.

### ACL Mask

The **mask** defines the maximum effective permissions for group and ACL entries:

```bash
setfacl -m m::r file.txt
```
This limits ALL group and ACL entries to read-only, regardless of what's set.

**💡 Admin Tip:** If ACL permissions aren't working as expected, check the mask:
```bash
getfacl file.txt
# Look for: mask::r--  ← this limits effective permissions
```

---

## 2.10 — sudo and `/etc/sudoers`

### What Is sudo?

`sudo` (Superuser Do) lets authorized users execute commands as root (or another user) without knowing root's password.

```bash
sudo apt update
```
**What happens:**
1. Checks if the user is authorized in `/etc/sudoers`
2. Asks for the USER's own password (not root's)
3. Runs the command as root
4. Logs the action to `/var/log/auth.log`

### Why sudo Instead of Logging In as Root

| Root Login | sudo |
|-----------|------|
| Anyone with the password has full access | Individual user accounts are tracked |
| No audit trail of who did what | Every sudo command is logged |
| One shared password for multiple admins | Each admin uses their own password |
| Unlimited, permanent access | Temporary, per-command access |
| If compromised, everything is exposed | Can limit which commands a user can run |

**✅ Best Practice:** Disable direct root login. Use sudo for all administrative tasks.

### The `sudo` Group

On **Ubuntu/Debian**, users in the `sudo` group can run any command as root.  
On **RHEL/Rocky**, the group is called `wheel`.

```bash
# Add a user to sudo group (Ubuntu/Debian)
sudo usermod -aG sudo john

# Add a user to wheel group (RHEL/Rocky)
sudo usermod -aG wheel john
```

**Verify:**
```bash
groups john
# Should show: john : john sudo
```

### `/etc/sudoers` — sudo Configuration

**🔴 DANGER — Never edit `/etc/sudoers` directly with a text editor. Always use `visudo`:**

```bash
sudo visudo
```

**Why `visudo`?**
- It validates the syntax before saving
- A syntax error in `/etc/sudoers` can lock you out of sudo entirely
- Uses a safe temporary file

### sudoers File Format

```
# User privilege specification
root    ALL=(ALL:ALL) ALL
```

**Format:** `who where=(as_whom) what`

| Field | Meaning | Example |
|-------|---------|---------|
| `who` | Username or group (%group) | `john` or `%developers` |
| `where` | Host (usually `ALL`) | `ALL` |
| `as_whom` | Run as user:group | `(ALL:ALL)` = any user, any group |
| `what` | Commands allowed | `ALL` = any command |

### Practical sudoers Examples

```bash
sudo visudo
```

```
# Allow john to run ALL commands as root
john    ALL=(ALL:ALL) ALL

# Allow john to run commands without typing a password
john    ALL=(ALL:ALL) NOPASSWD: ALL

# Allow the devops group to restart nginx and apache only
%devops ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl restart apache2

# Allow backup user to run rsync as root without password
backup  ALL=(root) NOPASSWD: /usr/bin/rsync

# Allow sara to manage users only
sara    ALL=(root) /usr/sbin/useradd, /usr/sbin/usermod, /usr/sbin/userdel, /usr/bin/passwd
```

### Drop-in sudoers Files

Instead of editing the main sudoers file, create separate files in `/etc/sudoers.d/`:

```bash
sudo visudo -f /etc/sudoers.d/developers
```

```
# /etc/sudoers.d/developers
%developers ALL=(root) /usr/bin/systemctl restart nginx
%developers ALL=(root) /usr/bin/systemctl restart apache2
%developers ALL=(root) /usr/bin/systemctl status *
```

**✅ Best Practice:**
- Use `/etc/sudoers.d/` for custom rules (easier to manage)
- Name files descriptively: `01-admins`, `02-developers`, `03-backup`
- Always use `visudo -f` to create/edit them

```bash
# Verify sudoers syntax
sudo visudo -c
```
**Output if valid:** `parsed OK`

### Useful sudo Commands

```bash
# Run a command as root
sudo command

# Run a command as another user
sudo -u john command

# Open a root shell
sudo -i

# Open a root shell preserving environment
sudo -s

# List your sudo permissions
sudo -l

# Edit a file with root privileges
sudo -e /etc/config_file    # or: sudoedit /etc/config_file

# Run the previous command with sudo
sudo !!
```

**💡 Admin Tip:** `sudo -l` is very useful when you connect to a server and want to know what you can do:
```bash
sudo -l
```
```
User abhishek may run the following commands on server01:
    (ALL : ALL) ALL
```

**⚠️ Common Mistake:** Using `sudo su -` to become root permanently. This defeats the purpose of sudo (individual command logging). Use `sudo command` for individual commands instead.

---

## 2.11 — Hands-On Labs

### Lab 1: User Management

```bash
# 1. Create three users with home directories
sudo useradd -m -s /bin/bash -c "Alice Developer" alice
sudo useradd -m -s /bin/bash -c "Bob Tester" bob
sudo useradd -m -s /bin/bash -c "Carol Admin" carol

# 2. Set passwords
sudo passwd alice    # Set a password
sudo passwd bob
sudo passwd carol

# 3. Verify users were created
id alice
id bob
id carol

# 4. Check the entries in /etc/passwd
grep -E "^(alice|bob|carol):" /etc/passwd

# 5. Check home directories
ls -la /home/

# 6. Check password status
sudo passwd -S alice

# 7. Force alice to change password on next login
sudo passwd -e alice

# 8. Lock bob's account
sudo passwd -l bob
sudo passwd -S bob    # Should show "L"

# 9. Unlock bob
sudo passwd -u bob

# 10. Set password expiry for carol (90 days max, warning at 14 days)
sudo chage -M 90 -W 14 carol
sudo chage -l carol
```

### Lab 2: Group Management

```bash
# 1. Create groups
sudo groupadd developers
sudo groupadd testers
sudo groupadd admins

# 2. Add users to groups
sudo usermod -aG developers alice
sudo usermod -aG testers bob
sudo usermod -aG admins carol
sudo usermod -aG developers carol     # Carol is in admins AND developers

# 3. Verify
groups alice
groups bob
groups carol
id carol

# 4. Check /etc/group
grep -E "^(developers|testers|admins):" /etc/group

# 5. Remove bob from testers
sudo gpasswd -d bob testers
groups bob
```

### Lab 3: Permissions

```bash
# 1. Create a test directory structure
mkdir -p ~/perm_lab
cd ~/perm_lab

# 2. Create test files
touch public.txt private.txt shared.txt script.sh

# 3. Check default permissions
ls -la

# 4. Make script executable
chmod u+x script.sh
ls -la script.sh

# 5. Set numeric permissions
chmod 644 public.txt      # rw-r--r--
chmod 600 private.txt     # rw-------
chmod 660 shared.txt      # rw-rw----
chmod 755 script.sh       # rwxr-xr-x
ls -la

# 6. Test with symbolic method
chmod o+r private.txt     # Add read for others
ls -la private.txt
chmod o-r private.txt     # Remove it again
ls -la private.txt

# 7. Change ownership
sudo chown alice:developers shared.txt
ls -la shared.txt

# 8. Create a shared directory with SGID
sudo mkdir /shared
sudo chown root:developers /shared
sudo chmod 2775 /shared
ls -ld /shared
# Notice the 's' in group execute position

# 9. Test umask
umask
touch default_perms.txt
ls -la default_perms.txt      # Should be 644 (with umask 022)
umask 077
touch restricted_perms.txt
ls -la restricted_perms.txt   # Should be 600 (with umask 077)
umask 022                     # Reset umask
```

### Lab 4: ACLs

```bash
# 1. Create a test file
touch ~/perm_lab/acl_test.txt
echo "Sensitive project data" > ~/perm_lab/acl_test.txt

# 2. Check current ACLs
getfacl ~/perm_lab/acl_test.txt

# 3. Give alice read-write access via ACL
sudo setfacl -m u:alice:rw ~/perm_lab/acl_test.txt

# 4. Give bob read-only access
sudo setfacl -m u:bob:r ~/perm_lab/acl_test.txt

# 5. View the ACLs
getfacl ~/perm_lab/acl_test.txt

# 6. Check ls -la (notice the + sign)
ls -la ~/perm_lab/acl_test.txt

# 7. Remove ACL for bob
sudo setfacl -x u:bob ~/perm_lab/acl_test.txt
getfacl ~/perm_lab/acl_test.txt

# 8. Remove all ACLs
sudo setfacl -b ~/perm_lab/acl_test.txt
getfacl ~/perm_lab/acl_test.txt
```

### Lab 5: sudo Configuration

```bash
# 1. Check your sudo permissions
sudo -l

# 2. Create a sudoers drop-in file for the developers group
sudo visudo -f /etc/sudoers.d/developers

# Add this line:
# %developers ALL=(root) /usr/bin/systemctl status *, /usr/bin/systemctl restart nginx

# 3. Validate the configuration
sudo visudo -c

# 4. Check auth log for sudo entries
sudo grep "sudo" /var/log/auth.log | tail -5
```

---

## 2.12 — Troubleshooting Exercises

### Exercise 1: "Permission Denied" When Accessing a File

**Problem:** User `alice` cannot read the file `/projects/config.yaml`.

**Diagnosis steps:**

```bash
# 1. Check the file's permissions and ownership
ls -la /projects/config.yaml

# 2. Check who alice is and what groups she belongs to
id alice

# 3. Check if ACLs are involved
getfacl /projects/config.yaml

# 4. Check parent directory permissions (need 'x' to traverse)
ls -ld /projects

# 5. Check for SELinux context (RHEL/Rocky)
ls -Z /projects/config.yaml    # Only on SELinux-enabled systems
```

**Common fixes:**
```bash
# Fix: Add alice to the file's group
sudo usermod -aG projectgroup alice
# Alice must log out and back in for group changes to take effect!

# OR fix: Add a specific ACL
sudo setfacl -m u:alice:r /projects/config.yaml

# OR fix: Change group ownership
sudo chown :projectgroup /projects/config.yaml
sudo chmod g+r /projects/config.yaml
```

**⚠️ Common Mistake:** Group membership changes don't take effect until the user logs out and back in. Use `newgrp groupname` as a workaround.

---

### Exercise 2: Script Won't Execute

**Problem:** You created a script but get "Permission denied" when trying to run it.

```bash
./deploy.sh
# bash: ./deploy.sh: Permission denied
```

**Diagnosis:**
```bash
ls -la deploy.sh
# -rw-r--r-- 1 abhishek abhishek 256 Sep 4 10:00 deploy.sh
#  ^ No execute permission!
```

**Fix:**
```bash
chmod u+x deploy.sh
# OR
chmod 755 deploy.sh
```

**Alternative (runs without execute permission):**
```bash
bash deploy.sh
```

---

### Exercise 3: New Files Not Group-Accessible

**Problem:** Team members create files in `/shared/project` but other team members can't access them.

**Diagnosis:**
```bash
ls -la /shared/project/
# -rw-r--r-- 1 alice alice 1024 Sep 4 10:00 report.txt
#                     ^^^^^
#                     File has alice's primary group, not the team group!

ls -ld /shared/project/
# drwxrwxr-x 2 root developers 4096 Sep 4 10:00 /shared/project
# No SGID set!
```

**Fix:**
```bash
# Set SGID on the directory
sudo chmod g+s /shared/project/

# Fix existing files
sudo chgrp -R developers /shared/project/
sudo chmod -R g+rw /shared/project/

# Verify SGID is set
ls -ld /shared/project/
# drwxrwsr-x 2 root developers 4096 Sep 4 10:00 /shared/project
#        ^
#        s = SGID is set
```

---

### Exercise 4: User Locked Out

**Problem:** User `bob` reports he cannot log in.

**Diagnosis:**
```bash
# Check if account is locked
sudo passwd -S bob
# bob L 09/04/2026 0 99999 7 -1
#     ^
#     L = locked

# Check if account is expired
sudo chage -l bob
# Account expires: Sep 01, 2026  ← expired!

# Check if shell is set to nologin
grep "^bob:" /etc/passwd
# bob:x:1001:1001::/home/bob:/usr/sbin/nologin  ← can't log in!
```

**Fixes:**
```bash
# Unlock the account
sudo passwd -u bob

# Extend or remove expiration
sudo chage -E -1 bob    # Remove expiration
sudo chage -E 2027-12-31 bob   # Extend

# Fix the shell
sudo usermod -s /bin/bash bob
```

---

## 2.13 — Practical Scenarios

### Scenario 1: Set Up a Shared Development Environment

**Task:** You have three developers (alice, bob, carol) who all need to work on files in `/var/www/webapp`. Set it up so:
- All three can read, write, and execute files
- New files automatically belong to the shared group
- Other users cannot access the directory
- No one can delete another person's files

**Try this yourself first.**

<details>
<summary><strong>Solution</strong></summary>

```bash
# Create the group
sudo groupadd webdevs

# Add users to the group
sudo usermod -aG webdevs alice
sudo usermod -aG webdevs bob
sudo usermod -aG webdevs carol

# Create and set up the directory
sudo mkdir -p /var/www/webapp
sudo chown root:webdevs /var/www/webapp

# Set permissions: SGID + Sticky bit
# SGID (2) = new files inherit group
# Sticky (1) = only owners can delete their own files
# 3770 = SGID+Sticky, owner rwx, group rwx, others nothing
sudo chmod 3770 /var/www/webapp

# Verify
ls -ld /var/www/webapp
# drwxrws--T  2 root webdevs 4096 Sep  4 10:00 /var/www/webapp
```

The `s` in group position = SGID, `T` in others position = sticky bit (uppercase because others have no `x`).

</details>

### Scenario 2: Audit User Accounts

**Task:** You're a new admin on a server. Audit the user accounts:
1. List all users who can log in (have a real shell)
2. Find all users with UID 0 (potential security issue — only root should have UID 0)
3. Find any users with empty passwords
4. List all SUID files
5. Find all world-writable files in /etc

**Try this yourself first.**

<details>
<summary><strong>Solution</strong></summary>

```bash
# 1. Users with real shells
grep -v "nologin\|false\|sync\|halt\|shutdown" /etc/passwd | cut -d: -f1,7

# 2. Users with UID 0 (should only be root!)
awk -F: '$3 == 0 {print $1}' /etc/passwd

# 3. Users with empty password field
sudo awk -F: '$2 == "" {print $1}' /etc/shadow

# 4. All SUID files
find / -perm -4000 -type f -ls 2>/dev/null

# 5. World-writable files in /etc (security risk!)
find /etc -perm -o+w -type f -ls 2>/dev/null
```

</details>

---

## 2.14 — Practice Questions

1. **Create a user** `testuser` with:
   - Home directory at `/home/testuser`
   - Shell set to `/bin/bash`
   - Full name "Test User"
   - Member of groups `developers` and `docker`
   - Password that expires in 60 days

2. **Permission puzzle:** A file has permissions `750`. Who can do what?

3. **Create a shared directory** at `/data/shared`:
   - Group `analysts` should have full access
   - New files should automatically belong to the `analysts` group
   - Users should only be able to delete their own files
   - Others should have no access

4. **Fix this scenario:** A web server (running as user `www-data`) cannot read files in `/var/www/html`. The files are owned by `deploy:deploy` with permissions `640`. Fix it without changing ownership.

5. **Decode these permissions:**
   - `-rwsr-xr-x` — What is the special bit? What does it mean?
   - `drwxrws---` — What is the special bit? What happens when files are created here?
   - `drwxrwxrwt` — What is the special bit? What does it prevent?

6. **Create a sudoers rule** that allows the `backup` group to run `/usr/bin/rsync` as root without a password prompt.

---

## Level 2 Summary

### What I Learned

- Linux has root, system, and regular users with distinct UID ranges
- User info is stored in `/etc/passwd`, passwords in `/etc/shadow`, groups in `/etc/group`
- `useradd` creates users; `usermod` modifies them; `userdel` removes them
- Groups allow bulk permission management; primary vs secondary groups matter
- `chmod` sets permissions using numeric (755) or symbolic (u+x) methods
- SUID, SGID, and sticky bit provide special permission behaviors
- `umask` controls default permissions for new files
- ACLs (`setfacl`/`getfacl`) provide fine-grained access beyond basic rwx
- `sudo` provides controlled root access with full audit logging
- `/etc/sudoers` must only be edited with `visudo`

### Commands to Remember

| Command | Purpose |
|---------|---------|
| `useradd -m -s /bin/bash user` | Create user with home dir and shell |
| `passwd user` | Set/change password |
| `usermod -aG group user` | Add user to group (DON'T forget -a!) |
| `userdel -r user` | Delete user and home directory |
| `groupadd name` | Create a group |
| `id user` | Show UID, GID, and groups |
| `groups user` | Show user's groups |
| `chown user:group file` | Change ownership |
| `chown -R user:group dir/` | Recursive ownership change |
| `chmod 755 file` | Set numeric permissions |
| `chmod u+x file` | Add execute for owner |
| `chmod g+s dir/` | Set SGID on directory |
| `chmod +t dir/` | Set sticky bit |
| `umask` | Show/set default permission mask |
| `getfacl file` | View ACLs |
| `setfacl -m u:user:rw file` | Set ACL |
| `sudo visudo` | Safely edit sudoers |
| `sudo -l` | List your sudo permissions |
| `chage -l user` | Show password aging info |
| `passwd -S user` | Show password status |

### Practical Skills Gained

- Create and manage user accounts for a team
- Set up shared directories with proper group access
- Configure SGID for collaborative work
- Use sticky bit to protect shared directories
- Set up fine-grained access with ACLs
- Configure sudo for different admin roles
- Troubleshoot permission-related access issues

### Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| `usermod -G` without `-a` | Removes user from all other groups | Always use `usermod -aG` |
| Editing `/etc/sudoers` without `visudo` | Syntax error can lock you out | Always use `sudo visudo` |
| Group changes not taking effect | User must re-login | `newgrp groupname` or re-login |
| `chmod -R 777 /var/www` | Massive security hole | Use `755` for dirs, `644` for files |
| Missing `x` on a directory | Can't `cd` into it even with `r` | `chmod +x directory` |
| Setting SUID on scripts | Security vulnerability | Only set SUID on compiled binaries |

### Administrator-Level Knowledge

- Audit SUID files regularly: `find / -perm -4000 -type f -ls 2>/dev/null`
- Use `/etc/sudoers.d/` for drop-in rules instead of modifying the main file
- Set password policies with `chage`: `-M 90 -m 7 -W 14`
- Use SGID + sticky bit for team shared directories: `chmod 3770`
- Disable direct root login; enforce sudo for auditability
- Check for world-writable files in sensitive directories
- Default to `umask 027` or `077` on servers

### Interview Questions

1. **What is the difference between `/etc/passwd` and `/etc/shadow`?**
2. **Explain SUID, SGID, and sticky bit with examples.**
3. **What happens if you run `usermod -G docker john` (without `-a`)?**
4. **How do you set up a shared directory for a team?**
5. **What is umask? What does a umask of `027` mean for new files?**
6. **How do you configure sudo for a user to run only specific commands?**
7. **What is the difference between `chmod 755` and `chmod u=rwx,g=rx,o=rx`?**
8. **How do ACLs work and when would you use them?**
9. **A user was added to a group but still can't access files. Why?**
10. **How do you audit SUID files on a system and why is it important?**

---

*When you've completed all the labs and practice questions, proceed to [Level 3 — Package Management](level-03-package-management.md)*
