# Level 3 — Package Management

> **Goal:** Install, update, remove, and troubleshoot software packages on both Debian/Ubuntu and RHEL/Rocky systems. Understand repositories, dependencies, and package management best practices.

> **Lab Environment:** WSL2 Ubuntu for APT labs. VirtualBox Rocky Linux VM for DNF labs (recommended to have both).

> **Prerequisite:** Complete Level 1 and Level 2

---

## 3.1 — What Is a Package?

### Minimum Theory

A **package** is a compressed archive containing:
- The software's compiled binaries (executables)
- Configuration files
- Documentation
- Metadata (version, dependencies, description)
- Pre/post installation scripts

**Why packages instead of compiling from source?**

| Compiling from Source | Using Packages |
|----------------------|----------------|
| Download source code, configure, compile, install | One command to install |
| You manage dependencies manually | Dependencies resolved automatically |
| No automatic updates | Easy updates with one command |
| Hard to uninstall cleanly | Clean removal tracked by package manager |
| Time-consuming | Seconds to install |

**📌 Important — Two major package formats:**

| Format | Extension | Distributions | Package Manager | Low-Level Tool |
|--------|-----------|--------------|-----------------|----------------|
| **DEB** | `.deb` | Debian, Ubuntu, Linux Mint | **APT** (`apt`) | `dpkg` |
| **RPM** | `.rpm` | RHEL, Rocky, AlmaLinux, Fedora, CentOS | **DNF** (`dnf`) / YUM (`yum`) | `rpm` |

**🎯 Interview Point:** "What is the difference between `dpkg` and `apt`?"  
> `dpkg` is the low-level tool that installs/removes individual `.deb` files but cannot resolve dependencies. `apt` is the high-level tool that uses `dpkg` internally but also handles dependency resolution, repository management, and updates.

### Package Management Layers

```
┌─────────────────────────────────┐
│   High-Level: apt / dnf         │  ← You use this 99% of the time
│   (resolves dependencies,       │
│    downloads from repositories)  │
├─────────────────────────────────┤
│   Low-Level: dpkg / rpm         │  ← Installs individual package files
│   (no dependency resolution)     │
├─────────────────────────────────┤
│   Package Files: .deb / .rpm    │  ← The actual software archive
└─────────────────────────────────┘
```

---

## 3.2 — Repositories

### What Is a Repository?

A **repository (repo)** is a remote server that stores thousands of packages. When you run `apt install nginx`, APT:

1. Checks configured repositories
2. Finds the `nginx` package and its dependencies
3. Downloads all required `.deb` files
4. Installs them in the correct order

**Types of repositories:**

| Type | Content | Example |
|------|---------|---------|
| **Official/Main** | Core packages maintained by the distro | Ubuntu Main, Rocky BaseOS |
| **Universe/EPEL** | Community-maintained packages | Ubuntu Universe, EPEL for RHEL |
| **Third-party/PPA** | Software from vendors or individuals | Docker repo, Node.js repo, PPAs |
| **Security** | Security updates only | Ubuntu Security |

---

## 3.3 — APT — Debian/Ubuntu Package Management

### Repository Configuration

**Main config file:**
```bash
cat /etc/apt/sources.list
```

**Expected content (Ubuntu 22.04):**
```
deb http://archive.ubuntu.com/ubuntu jammy main restricted
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted
deb http://archive.ubuntu.com/ubuntu jammy universe
deb http://archive.ubuntu.com/ubuntu jammy-security main restricted
```

**Format:**
```
deb [options] URL distribution component1 component2 ...
```

| Field | Meaning |
|-------|---------|
| `deb` | Binary packages (`deb-src` = source packages) |
| URL | Repository server address |
| `jammy` | Distribution codename (Ubuntu 22.04) |
| `main` | Officially supported, free software |
| `restricted` | Officially supported, non-free (drivers) |
| `universe` | Community-maintained |
| `multiverse` | Non-free software |

**Modern format (Ubuntu 22.04+):**

Newer Ubuntu versions use `.sources` files in `/etc/apt/sources.list.d/`:
```bash
ls /etc/apt/sources.list.d/
```

**💡 Admin Tip:** Always use `/etc/apt/sources.list.d/` for third-party repos. Keep one file per repository. This makes it easy to add and remove repos cleanly.

---

### `apt update` — Refresh Package Index

```bash
sudo apt update
```

**What it does:** Downloads the latest package lists from all configured repositories. **It does NOT install or upgrade anything.** It just updates the local database of what packages are available and at which versions.

**Expected output:**
```
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Get:2 http://archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB]
Get:3 http://security.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
Fetched 229 kB in 2s (115 kB/s)
Reading package lists... Done
Building dependency tree... Done
15 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

**📌 Important:** Always run `apt update` BEFORE installing or upgrading packages. Without it, you might install outdated versions or get "Unable to locate package" errors.

**⚠️ Common Mistake:** Running `apt install nginx` without `apt update` first and getting:
```
E: Unable to locate package nginx
```
The package index is stale. Run `apt update` first.

---

### `apt install` — Install Packages

```bash
sudo apt install nginx
```

**What it does:**
1. Checks the package index for `nginx`
2. Identifies all dependencies
3. Downloads `nginx` and its dependencies
4. Installs them in the correct order
5. Runs post-installation scripts (may start the service)

**Important options:**

| Option | Meaning | Example |
|--------|---------|---------|
| `-y` | Auto-answer yes to all prompts | `sudo apt install -y nginx` |
| `--no-install-recommends` | Skip recommended (optional) packages | `sudo apt install --no-install-recommends nginx` |
| `--dry-run` | Simulate — show what would happen without doing it | `sudo apt install --dry-run nginx` |

**Install multiple packages:**
```bash
sudo apt install -y nginx curl wget vim htop
```

**Install a specific version:**
```bash
# List available versions
apt list -a nginx

# Install specific version
sudo apt install nginx=1.18.0-6ubuntu14.4
```

**💡 Admin Tip:** Use `--no-install-recommends` on servers to keep installations minimal. Recommended packages are often desktop-related and unnecessary on servers.

---

### `apt upgrade` — Upgrade Installed Packages

```bash
sudo apt upgrade
```

**What it does:** Upgrades ALL installed packages to their latest versions. Will NOT remove packages or install new ones (safe upgrade).

```bash
sudo apt upgrade -y
```
Auto-confirm all upgrades.

### `apt full-upgrade` — Full System Upgrade

```bash
sudo apt full-upgrade
```

**What it does:** Like `upgrade`, but will also **remove** packages if necessary to resolve dependencies. Used for distribution upgrades.

**✅ Best Practice — Standard update workflow:**
```bash
sudo apt update && sudo apt upgrade -y
```
Update the index, then upgrade everything. This is what you'll run regularly on servers.

**⚠️ Common Mistake:** Running `apt upgrade` without `apt update` first. You'll upgrade to old cached versions, not the latest.

---

### `apt remove` vs `apt purge` — Remove Packages

```bash
# Remove the package but KEEP config files
sudo apt remove nginx

# Remove the package AND its config files
sudo apt purge nginx

# Remove unused dependencies left behind
sudo apt autoremove
```

| Command | Removes Binary | Removes Config Files | Removes Dependencies |
|---------|---------------|---------------------|---------------------|
| `apt remove` | Yes | No | No |
| `apt purge` | Yes | Yes | No |
| `apt autoremove` | No | No | Yes (orphaned only) |

**✅ Best Practice — Clean removal:**
```bash
sudo apt purge nginx && sudo apt autoremove -y
```
This removes the package, its config files, AND any orphaned dependencies.

**💡 Admin Tip:** Run `sudo apt autoremove` periodically to clean up unused packages and free disk space.

---

### `apt search` and `apt show` — Find and Inspect Packages

```bash
# Search for packages by name or description
apt search nginx
```

```bash
# Show detailed info about a package
apt show nginx
```

**Expected output of `apt show`:**
```
Package: nginx
Version: 1.18.0-6ubuntu14.4
Priority: optional
Section: web
Origin: Ubuntu
Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
Installed-Size: 42.5 kB
Depends: nginx-core (>= 1.18.0) | nginx-full (>= 1.18.0) | ...
Homepage: https://nginx.org
Description: small, powerful, scalable web/proxy server
```

```bash
# List installed packages
apt list --installed

# List upgradable packages
apt list --upgradable

# Check if a specific package is installed
apt list --installed | grep nginx
```

---

### `dpkg` — Low-Level Package Tool (Debian/Ubuntu)

Use `dpkg` when you need to work with individual `.deb` files or query the package database directly.

```bash
# Install a downloaded .deb file
sudo dpkg -i package.deb
```

**⚠️ Common Mistake:** `dpkg -i` does NOT resolve dependencies. If dependencies are missing:
```bash
sudo dpkg -i package.deb
# Error: dependency problems...

# Fix: Install missing dependencies
sudo apt install -f
```

**Useful `dpkg` queries:**

```bash
# List all installed packages
dpkg -l

# Check if a specific package is installed
dpkg -l | grep nginx
dpkg -s nginx

# List all files installed by a package
dpkg -L nginx

# Find which package owns a file
dpkg -S /usr/sbin/nginx

# Show package info from a .deb file
dpkg -I package.deb

# List contents of a .deb file (without installing)
dpkg -c package.deb
```

**💡 Admin Tip:** `dpkg -S /path/to/file` is incredibly useful when you find a file and want to know which package installed it:
```bash
dpkg -S /usr/bin/curl
# curl: /usr/bin/curl
```

**🎯 Interview Point:** "A package installation failed midway. How do you fix the package system?"
```bash
sudo dpkg --configure -a       # Configure any unpacked but unconfigured packages
sudo apt install -f             # Fix broken dependencies
```

---

### Adding Third-Party Repositories (APT)

**Method 1 — Add a PPA (Personal Package Archive) — Ubuntu only:**
```bash
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install php8.2
```

**Method 2 — Add a vendor repository (recommended for production):**

Example: Adding Docker's official repository:
```bash
# 1. Add Docker's GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 2. Add the repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. Update and install
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
```

**Remove a PPA:**
```bash
sudo add-apt-repository --remove ppa:ondrej/php
sudo apt update
```

**Remove a vendor repository:**
```bash
sudo rm /etc/apt/sources.list.d/docker.list
sudo rm /usr/share/keyrings/docker-archive-keyring.gpg
sudo apt update
```

**⚠️ Common Mistake:** Adding untrusted PPAs or third-party repos. Only add repos from trusted sources. Malicious repos can replace system packages.

---

### APT Cache and Cleanup

```bash
# Show cache size
du -sh /var/cache/apt/archives/

# Clean downloaded package files
sudo apt clean           # Remove ALL cached .deb files
sudo apt autoclean       # Remove only obsolete cached .deb files

# Remove unused packages
sudo apt autoremove -y
```

**✅ Best Practice — Regular server maintenance:**
```bash
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y && sudo apt clean
```

---

## 3.4 — DNF/YUM — RHEL/Rocky/Fedora Package Management

### DNF vs YUM

| Feature | YUM | DNF |
|---------|-----|-----|
| Used in | RHEL 7, CentOS 7 | RHEL 8+, Rocky 8+, Fedora 22+ |
| Speed | Slower | Faster (improved dependency resolution) |
| Status | Legacy (still works) | Current standard |
| Command syntax | Almost identical to DNF | Almost identical to YUM |

**📌 Important:** On RHEL 8+ and Rocky 8+, `yum` is actually a symlink to `dnf`. Both commands work, but `dnf` is the official tool.

```bash
ls -la /usr/bin/yum
# /usr/bin/yum -> dnf-3
```

### Repository Configuration

```bash
# List configured repos
dnf repolist

# List all repos (including disabled)
dnf repolist all
```

**Repo config files location:**
```bash
ls /etc/yum.repos.d/
```

**Example repo file (`/etc/yum.repos.d/rocky.repo`):**
```ini
[baseos]
name=Rocky Linux $releasever - BaseOS
mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=BaseOS-$releasever
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
```

| Field | Meaning |
|-------|---------|
| `[baseos]` | Repository ID |
| `name` | Human-readable name |
| `baseurl` or `mirrorlist` | Where to download packages |
| `gpgcheck=1` | Verify package signatures (keep this ON!) |
| `enabled=1` | Repository is active |
| `gpgkey` | Path to the GPG key for verification |

---

### DNF Core Commands

#### Update Package Index and Upgrade

```bash
# Check for updates (like apt update + shows available updates)
sudo dnf check-update

# Upgrade all packages
sudo dnf upgrade -y

# Upgrade a specific package
sudo dnf upgrade nginx
```

**📌 Important:** Unlike APT, `dnf upgrade` both refreshes the repo metadata AND upgrades packages. There's no separate "update index" step (though `dnf makecache` does that if needed).

**Note on `dnf update` vs `dnf upgrade`:** They are identical in DNF. `update` is kept for backward compatibility with YUM.

---

#### Install Packages

```bash
sudo dnf install nginx
sudo dnf install -y nginx curl wget vim htop
```

| Option | Meaning |
|--------|---------|
| `-y` | Auto-answer yes |
| `--nobest` | Don't fail if latest version has dependency issues |
| `--skip-broken` | Skip packages with dependency problems |

**Install a specific version:**
```bash
# List available versions
dnf list nginx --showduplicates

# Install specific version
sudo dnf install nginx-1.20.1-1.el9
```

**Install from a local `.rpm` file:**
```bash
sudo dnf install ./package.rpm
```

DNF resolves dependencies even for local RPM files (unlike `rpm -i`).

---

#### Remove Packages

```bash
# Remove a package
sudo dnf remove nginx

# Remove unused dependencies
sudo dnf autoremove
```

**📌 Important:** `dnf remove` also removes packages that depend on the package being removed. Always review the removal list before confirming.

---

#### Search and Info

```bash
# Search for packages
dnf search nginx

# Show package details
dnf info nginx

# List installed packages
dnf list installed

# List available packages
dnf list available | head -20

# Find which package provides a file
dnf provides /usr/sbin/nginx
dnf provides "*/nginx"
```

**💡 Admin Tip:** `dnf provides` is extremely useful when you need a command but don't know which package has it:
```bash
dnf provides "*/ifconfig"
# net-tools-2.0-0.62.20160912git.el9.x86_64 : Basic networking tools
```

---

#### Group Installs

DNF supports installing groups of related packages:

```bash
# List available groups
dnf group list

# Install a group (e.g., development tools)
sudo dnf group install "Development Tools"

# Show what's in a group
dnf group info "Development Tools"

# Remove a group
sudo dnf group remove "Development Tools"
```

**Common useful groups:**

| Group | Contains |
|-------|---------|
| `"Development Tools"` | gcc, make, git, etc. |
| `"System Tools"` | Admin utilities |
| `"Security Tools"` | Security scanners, audit tools |

---

### `rpm` — Low-Level Package Tool (RHEL/Rocky)

Equivalent to `dpkg` for the RPM world.

```bash
# Install an RPM (no dependency resolution!)
sudo rpm -ivh package.rpm

# Upgrade an RPM
sudo rpm -Uvh package.rpm

# Remove a package
sudo rpm -e package_name

# Query if a package is installed
rpm -q nginx

# List all installed packages
rpm -qa

# List files installed by a package
rpm -ql nginx

# Find which package owns a file
rpm -qf /usr/sbin/nginx

# Show package information
rpm -qi nginx

# Verify package integrity
rpm -V nginx
```

**RPM install flags:**
- `-i` = install
- `-v` = verbose
- `-h` = show progress with hash marks
- `-U` = upgrade (install if not present, upgrade if present)

**💡 Admin Tip:** `rpm -Va` verifies ALL installed packages against their expected states. Useful for detecting tampered files on a compromised system:
```bash
rpm -Va 2>/dev/null | head -20
```
Any output means files have been modified from their original package state.

---

### EPEL — Extra Packages for Enterprise Linux

EPEL provides additional packages not included in the base RHEL/Rocky repos.

```bash
# Install EPEL repository (Rocky/RHEL 9)
sudo dnf install epel-release -y

# Verify
dnf repolist | grep epel

# Now you can install packages from EPEL
sudo dnf install htop
```

**📌 Important:** Many common tools (`htop`, `fail2ban`, `certbot`) are in EPEL, not the base repos. Installing EPEL is one of the first things admins do on RHEL-based systems.

---

### Adding Third-Party Repositories (DNF)

**Example — Adding Docker's repository:**
```bash
# Add Docker repo
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker
sudo dnf install docker-ce docker-ce-cli containerd.io -y
```

**Enable/disable a repository:**
```bash
# Disable a repo
sudo dnf config-manager --set-disabled repo-name

# Enable a repo
sudo dnf config-manager --set-enabled repo-name

# Install from a specific repo
sudo dnf install --enablerepo=epel-testing package_name
```

**Remove a repository:**
```bash
sudo rm /etc/yum.repos.d/docker-ce.repo
sudo dnf clean all
```

---

### DNF Cache and Cleanup

```bash
# Clean all cached data
sudo dnf clean all

# Clean only package cache
sudo dnf clean packages

# Clean metadata cache
sudo dnf clean metadata

# Rebuild cache
sudo dnf makecache
```

---

### DNF History — Undo/Redo Operations

```bash
# Show transaction history
dnf history

# Show details of a specific transaction
dnf history info 15

# Undo a specific transaction (rollback!)
sudo dnf history undo 15

# Redo a transaction
sudo dnf history redo 15
```

**💡 Admin Tip:** `dnf history undo` is a lifesaver. If you accidentally remove a critical package and its dependencies, you can roll back the entire transaction.

---

## 3.5 — APT vs DNF Quick Reference

| Task | APT (Ubuntu/Debian) | DNF (RHEL/Rocky) |
|------|---------------------|-------------------|
| Update package index | `sudo apt update` | `sudo dnf makecache` (or auto with upgrade) |
| Upgrade all packages | `sudo apt upgrade -y` | `sudo dnf upgrade -y` |
| Install a package | `sudo apt install nginx` | `sudo dnf install nginx` |
| Remove a package | `sudo apt remove nginx` | `sudo dnf remove nginx` |
| Remove + config files | `sudo apt purge nginx` | `sudo dnf remove nginx` (configs removed too) |
| Remove orphaned deps | `sudo apt autoremove` | `sudo dnf autoremove` |
| Search for a package | `apt search nginx` | `dnf search nginx` |
| Show package info | `apt show nginx` | `dnf info nginx` |
| List installed packages | `apt list --installed` | `dnf list installed` |
| Find file's package | `dpkg -S /path/to/file` | `rpm -qf /path/to/file` or `dnf provides /path` |
| List package files | `dpkg -L nginx` | `rpm -ql nginx` |
| Install local package | `sudo dpkg -i file.deb` | `sudo dnf install ./file.rpm` |
| Fix broken packages | `sudo apt install -f` | `sudo dnf distro-sync` |
| Clean cache | `sudo apt clean` | `sudo dnf clean all` |
| Transaction history | — | `dnf history` |
| Undo transaction | — | `sudo dnf history undo N` |
| Add repo | Edit `/etc/apt/sources.list.d/` | Edit `/etc/yum.repos.d/` or `dnf config-manager` |
| List repos | — | `dnf repolist` |

---

## 3.6 — Package Pinning and Holding

### Prevent a Package from Being Upgraded (APT)

Sometimes you want to keep a specific version of a package (e.g., a tested version of a database).

```bash
# Hold a package (prevent upgrades)
sudo apt-mark hold nginx

# Show held packages
apt-mark showhold

# Remove the hold
sudo apt-mark unhold nginx
```

### Prevent a Package from Being Upgraded (DNF)

```bash
# Install the versionlock plugin
sudo dnf install python3-dnf-plugin-versionlock

# Lock a package version
sudo dnf versionlock add nginx

# List locked packages
dnf versionlock list

# Remove the lock
sudo dnf versionlock delete nginx
```

**💡 Admin Tip:** Pin/hold critical packages (databases, web servers) in production to prevent unexpected upgrades from breaking your application.

---

## 3.7 — Hands-On Labs

### Lab 1: APT Basics (Ubuntu/WSL2)

```bash
# 1. Update the package index
sudo apt update

# 2. Check how many packages can be upgraded
apt list --upgradable

# 3. Install some useful tools
sudo apt install -y tree htop net-tools curl wget

# 4. Verify installation
which tree
which htop
tree --version

# 5. Get info about the 'curl' package
apt show curl

# 6. Find which package provides a file
dpkg -S /usr/bin/curl

# 7. List files installed by curl
dpkg -L curl

# 8. Search for packages related to "json"
apt search json | head -20

# 9. Check total installed packages
dpkg -l | wc -l

# 10. Remove a package
sudo apt remove tree
which tree     # Should return nothing

# 11. Install it back
sudo apt install -y tree

# 12. Hold a package
sudo apt-mark hold curl
apt-mark showhold

# 13. Try upgrading (curl should be skipped)
sudo apt upgrade -y
# Should see: "curl set on hold"

# 14. Remove the hold
sudo apt-mark unhold curl

# 15. Clean up
sudo apt autoremove -y
sudo apt clean
du -sh /var/cache/apt/archives/
```

### Lab 2: Package Investigation

```bash
# 1. Find what package provides the 'ss' command
dpkg -S $(which ss)

# 2. Find what package provides 'ip' command
dpkg -S $(which ip)

# 3. List all packages installed on the system, sorted by size
dpkg-query -W --showformat='${Installed-Size}\t${Package}\n' | sort -rn | head -20

# 4. Find all installed packages related to "python"
dpkg -l | grep python | head -20

# 5. Check if a package is installed
dpkg -s nginx 2>/dev/null && echo "INSTALLED" || echo "NOT INSTALLED"
```

### Lab 3: Adding a Third-Party Repository (Ubuntu)

```bash
# 1. Check current repos
ls /etc/apt/sources.list.d/

# 2. Install prerequisites
sudo apt install -y apt-transport-https ca-certificates gnupg

# 3. Add the Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 4. Add the Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. Update and verify
sudo apt update
apt-cache policy docker-ce    # Should show versions from Docker repo

# 6. Clean up — remove the repo (optional, if you don't want Docker yet)
# sudo rm /etc/apt/sources.list.d/docker.list
# sudo rm /usr/share/keyrings/docker-archive-keyring.gpg
# sudo apt update
```

---

## 3.8 — Troubleshooting Exercises

### Exercise 1: "Unable to Locate Package"

**Problem:**
```bash
sudo apt install somepackage
# E: Unable to locate package somepackage
```

**Diagnosis:**
```bash
# 1. Did you update the index?
sudo apt update

# 2. Check for typos — search instead
apt search somepackage

# 3. Is the package in a disabled repo?
grep -r "somepackage" /etc/apt/sources.list.d/

# 4. Is the package architecture-specific?
dpkg --print-architecture    # Check your architecture (amd64, arm64)

# 5. Is it available in universe/multiverse?
sudo add-apt-repository universe
sudo apt update
apt search somepackage
```

**Common causes:**
1. Didn't run `apt update` first
2. Typo in package name
3. Package is in a repo that isn't enabled (universe, EPEL)
4. Package doesn't exist for your distribution version
5. Third-party repo needed but not added

---

### Exercise 2: Broken Dependencies

**Problem:**
```bash
sudo apt install -f
# OR
sudo dpkg --configure -a
```
These are the go-to commands for fixing broken package states.

**Full recovery process:**
```bash
# Step 1: Fix any unconfigured packages
sudo dpkg --configure -a

# Step 2: Fix broken dependencies
sudo apt install -f

# Step 3: Update and try again
sudo apt update
sudo apt upgrade -y

# Step 4: If still broken, clean and rebuild
sudo apt clean
sudo apt update
sudo apt install -f
```

**On DNF/RHEL:**
```bash
# Fix broken dependencies
sudo dnf distro-sync

# Clean and rebuild
sudo dnf clean all
sudo dnf makecache
sudo dnf upgrade -y
```

---

### Exercise 3: GPG Key Errors

**Problem:**
```
W: GPG error: https://repo.example.com stable Release: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY ABC123DEF456
```

**Fix:**
```bash
# Import the missing key
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys ABC123DEF456

# Modern method (preferred):
curl -fsSL https://repo.example.com/gpg-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/example-keyring.gpg
```

---

### Exercise 4: Disk Full During Upgrade

**Problem:** An upgrade fails because `/var` is full.

**Fix:**
```bash
# 1. Check disk space
df -h /var

# 2. Clean the package cache
sudo apt clean
# OR
sudo dnf clean all

# 3. Remove old kernels (Ubuntu)
sudo apt autoremove --purge

# 4. Check again
df -h /var

# 5. Retry the upgrade
sudo apt update && sudo apt upgrade -y
```

---

### Exercise 5: Package Conflict

**Problem:**
```
E: Unmet dependencies. Try 'apt install -f' to fix.
The following packages have unmet dependencies:
  package-a : Depends: libfoo (>= 2.0) but 1.5 is to be installed
```

**Fix:**
```bash
# Option 1: Let apt resolve it
sudo apt install -f

# Option 2: Install the specific dependency version
sudo apt install libfoo=2.0

# Option 3: Remove the conflicting package and reinstall
sudo apt remove package-a
sudo apt install package-a
```

---

## 3.9 — Practical Scenarios

### Scenario 1: Set Up a New Server

**Task:** You've just provisioned a new Ubuntu server. Perform initial package setup:
1. Update all packages to the latest versions
2. Install essential admin tools
3. Remove unnecessary packages
4. Configure automatic security updates

**Try this yourself first.**

<details>
<summary><strong>Solution</strong></summary>

```bash
# 1. Full system update
sudo apt update && sudo apt upgrade -y

# 2. Install essential tools
sudo apt install -y \
  vim \
  curl \
  wget \
  htop \
  tree \
  net-tools \
  dnsutils \
  unzip \
  git \
  tmux \
  ncdu \
  jq \
  fail2ban

# 3. Remove unnecessary packages
sudo apt autoremove -y
sudo apt clean

# 4. Configure automatic security updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
# Select "Yes" when prompted

# Verify auto-updates are configured
cat /etc/apt/apt.conf.d/20auto-upgrades
# Should show:
# APT::Periodic::Update-Package-Lists "1";
# APT::Periodic::Unattended-Upgrade "1";
```

</details>

### Scenario 2: Investigate What Changed

**Task:** An application stopped working after someone ran updates yesterday. Investigate what packages changed.

**Try this yourself first.**

<details>
<summary><strong>Solution</strong></summary>

**On Ubuntu/Debian:**
```bash
# Check apt history logs
cat /var/log/apt/history.log | tail -50

# Search for specific date
grep -A 20 "Start-Date: 2026-09-03" /var/log/apt/history.log

# Check dpkg log for more detail
grep "2026-09-03" /var/log/dpkg.log | grep " install\| upgrade\| remove"
```

**On RHEL/Rocky:**
```bash
# DNF history is much better for this
dnf history
dnf history info 15     # Show details of transaction 15

# Undo the change if needed
sudo dnf history undo 15
```

</details>

---

## 3.10 — Security Best Practices for Package Management

| Practice | Why |
|----------|-----|
| Always use official repos | Third-party repos can inject malicious packages |
| Keep `gpgcheck=1` (DNF) | Ensures package authenticity |
| Enable automatic security updates | Patches vulnerabilities quickly |
| Review what you're installing | Don't blindly answer `y` — read the package list |
| Pin critical packages | Prevent unexpected upgrades from breaking services |
| Remove unused packages | Reduces attack surface |
| Audit installed packages periodically | Find unnecessary or vulnerable software |
| Keep the system updated | Most breaches exploit known, unpatched vulnerabilities |

**✅ Best Practice — Automatic security updates:**

**Ubuntu:**
```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

**Rocky/RHEL:**
```bash
sudo dnf install dnf-automatic
sudo systemctl enable --now dnf-automatic-install.timer
```

---

## 3.11 — Practice Questions

1. **Update and upgrade** your system. How many packages were upgraded? What command shows upgradable packages before upgrading?

2. **Install `tmux`** and then:
   - Find where its binary is located
   - List all files installed by the tmux package
   - Find what package provides `/usr/bin/tmux`

3. **Search for packages** related to "monitoring". Pick one and install it.

4. **Hold/pin** the `curl` package to its current version. Verify it's held. Then unhold it.

5. **Find the 10 largest installed packages** on your system by installed size.

6. **Clean up:** Remove unused dependencies, clean the package cache. How much space was freed?

7. **On Ubuntu:** Add the Docker repository, verify packages are available, then remove the repository.

8. **Package forensics:** Find the last 5 packages that were installed on your system. When were they installed and by whom?

---

## Level 3 Summary

### What I Learned

- Packages bundle software with metadata and dependencies for easy management
- APT (Debian/Ubuntu) and DNF (RHEL/Rocky) are the high-level package managers
- `dpkg` and `rpm` are low-level tools for individual package files
- Repositories are remote servers hosting thousands of packages
- Always `apt update` before installing on Ubuntu; DNF auto-refreshes
- Package pinning prevents unwanted upgrades of critical software
- Third-party repos should be added carefully with GPG verification
- Automatic security updates are essential for servers
- DNF history allows transaction rollback — a powerful safety net

### Commands to Remember

| Command | Purpose | Distro |
|---------|---------|--------|
| `sudo apt update` | Refresh package index | Ubuntu/Debian |
| `sudo apt upgrade -y` | Upgrade all packages | Ubuntu/Debian |
| `sudo apt install -y pkg` | Install a package | Ubuntu/Debian |
| `sudo apt remove pkg` | Remove (keep config) | Ubuntu/Debian |
| `sudo apt purge pkg` | Remove (delete config) | Ubuntu/Debian |
| `sudo apt autoremove` | Remove orphaned dependencies | Ubuntu/Debian |
| `apt search keyword` | Search for packages | Ubuntu/Debian |
| `apt show pkg` | Show package details | Ubuntu/Debian |
| `dpkg -l` | List installed packages | Ubuntu/Debian |
| `dpkg -S /path` | Find which package owns a file | Ubuntu/Debian |
| `dpkg -L pkg` | List files in a package | Ubuntu/Debian |
| `sudo apt-mark hold pkg` | Prevent package from upgrading | Ubuntu/Debian |
| `sudo dnf upgrade -y` | Update and upgrade all | RHEL/Rocky |
| `sudo dnf install -y pkg` | Install a package | RHEL/Rocky |
| `sudo dnf remove pkg` | Remove a package | RHEL/Rocky |
| `dnf search keyword` | Search for packages | RHEL/Rocky |
| `dnf info pkg` | Show package details | RHEL/Rocky |
| `dnf provides "*/cmd"` | Find package that provides a command | RHEL/Rocky |
| `rpm -qa` | List all installed RPMs | RHEL/Rocky |
| `rpm -qf /path` | Find package owning a file | RHEL/Rocky |
| `dnf history` | Show transaction history | RHEL/Rocky |
| `sudo dnf history undo N` | Rollback a transaction | RHEL/Rocky |

### Practical Skills Gained

- Install, update, and remove software on both major Linux families
- Add and manage third-party repositories safely
- Fix broken packages and dependency issues
- Pin packages to prevent unwanted upgrades
- Investigate package changes after issues
- Set up automatic security updates
- Find which package owns any file on the system

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Not running `apt update` before install | Always: `sudo apt update && sudo apt install pkg` |
| Using `dpkg -i` for `.deb` with dependencies | Use `sudo apt install -f` to fix, or use `apt install ./file.deb` |
| Adding untrusted third-party repos | Only use official vendor repos with GPG verification |
| Running `apt upgrade` on production without testing | Test on staging first; pin critical packages |
| Not cleaning package cache on small disks | Run `apt clean` or `dnf clean all` periodically |
| Ignoring held packages during upgrades | Review held packages: `apt-mark showhold` |

### Administrator-Level Knowledge

- **First thing on a new RHEL server:** `sudo dnf install epel-release -y`
- **First thing on a new Ubuntu server:** `sudo apt update && sudo apt upgrade -y`
- Use `dnf history undo` to rollback problematic updates
- Configure `unattended-upgrades` (Ubuntu) or `dnf-automatic` (RHEL) for security patches
- Check `/var/log/apt/history.log` or `dnf history` to audit package changes
- Use `dpkg-query -W --showformat` or `rpm -qa --queryformat` for custom package reports
- Pin database and web server packages in production environments

### Interview Questions

1. **What is the difference between `apt` and `dpkg`?**
2. **What is the difference between `apt remove` and `apt purge`?**
3. **How do you find which package provides a specific file?**
4. **How do you add a third-party repository on Ubuntu? On RHEL?**
5. **What is EPEL and why is it needed?**
6. **How do you prevent a specific package from being upgraded?**
7. **An `apt install` fails with broken dependencies. How do you fix it?**
8. **How do you configure automatic security updates?**
9. **How do you rollback a package update on RHEL/Rocky?**
10. **What is the purpose of GPG keys in package management?**

---

*When you've completed all the labs and practice questions, proceed to [Level 4 — Processes & System Monitoring](level-04-processes-monitoring.md)*
