# Linux Practical Notes — Beginner to System Administrator

> **A comprehensive, hands-on Linux administration course designed to guide learners from first terminal commands to production-grade system administration and DevOps practices.**

---

## Overview

This repository contains a structured, practical, command-first curriculum for mastering Linux system administration. Unlike purely theoretical guides, every level is built around real-world scenarios, complete executable commands, step-by-step labs, troubleshooting exercises, and interview preparation questions.

Whether you are preparing for certifications (RHCSA, LFCS), shifting into DevOps/Cloud engineering, or managing production servers, these notes serve as both an intensive learning roadmap and an ongoing field reference.

---

## Course Roadmap

| Level | Module | Core Focus Areas | Status |
|:---:|---|---|:---:|
| **01** | [Linux Fundamentals](level-01-linux-fundamentals.md) | Terminal, filesystem hierarchy, file ops, pipes, redirection, search | **Available** |
| **02** | [Users, Groups & Permissions](level-02-users-groups-permissions.md) | User management, chmod, chown, ACLs, sudoers, security basics | **Available** |
| **03** | [Package Management](level-03-package-management.md) | APT, DNF/YUM, repositories, dependencies, package pinning | **Available** |
| **04** | [Processes & System Monitoring](level-04-processes-monitoring.md) | ps, top, htop, signals, kill, /proc, load analysis, background jobs | **Available** |
| **05** | [Services & systemd](level-05-services-systemd.md) | systemctl, journalctl, custom service units, timers, boot targets | **Available** |
| **06** | Storage & Filesystems | Disks, partitions, fdisk/parted, LVM, mkfs, /etc/fstab, RAID | *In Progress* |
| **07** | Networking & Connectivity | IP assignment, routing, netplan, nmcli, ss, curl, dig, troubleshooting | *Upcoming* |
| **08** | SSH & Remote Administration | SSH keys, config optimization, SCP, rsync, tunneling, hardening | *Upcoming* |
| **09** | Shell Scripting & Automation | Bash scripting, variables, logic, loops, error handling, cron jobs | *Upcoming* |
| **10** | System Logs & Troubleshooting | /var/log, journalctl deep dive, logrotate, incident triage | *Upcoming* |
| **11** | Security & Hardening | UFW/firewalld, SELinux, AppArmor, auditd, Fail2ban, CIS benchmarks | *Upcoming* |
| **12** | Web & Application Servers | Nginx, Apache, reverse proxies, SSL/TLS certificates with Let's Encrypt | *Upcoming* |
| **13** | DNS, DHCP & Core Infrastructure | BIND, local DNS resolver, DHCP configuration, network names | *Upcoming* |
| **14** | Backup, Restore & Recovery | tar, rsync snapshots, automated backups, disaster recovery drills | *Upcoming* |
| **15** | Advanced Linux Administration | Kernel modules, sysctl tuning, GRUB recovery, performance profiling | *Upcoming* |
| **16** | Containers & Modern Admin | Docker fundamentals, container networking, storage, orchestration overview | *Upcoming* |

---

## Progression Model

```
[Level 01 - 02]  Beginner        --> Terminal mastery, navigation, permissions, sudo
[Level 03 - 05]  Linux User      --> Packages, process management, services & systemd
[Level 06 - 09]  Power User      --> Storage, LVM, networking, SSH, Bash automation
[Level 10 - 13]  Junior Admin    --> Logs, incident triage, firewall, web servers, DNS
[Level 14 - 16]  SysAdmin/DevOps --> Disaster recovery, kernel tuning, containers
```

---

## Recommended Lab Setup

To safely practice system-level administration commands (especially storage, networking, and systemd), use one of the recommended sandbox environments:

### Option A: WSL2 (Windows Subsystem for Linux) — Quickest Start
*Best for: Levels 01–05, 09 (Bash Scripting)*

```powershell
# Run PowerShell as Administrator:
wsl --install -d Ubuntu-22.04
```

* **Advantages:** Instant startup, lightweight, deep Windows filesystem integration.
* **Limitations:** Requires systemd enablement in `/etc/wsl.conf`; cannot modify raw block devices or kernel modules.

### Option B: Dedicated Virtual Machine (VirtualBox / VMware / KVM) — Full Labs
*Best for: Levels 06, 07, 08, 10–16 (Storage, LVM, Network Interfaces, Firewalls)*

1. Download and install [VirtualBox](https://www.virtualbox.org/).
2. Download an enterprise Linux distribution ISO:
   * **Ubuntu Server 22.04 LTS**: [ubuntu.com/download/server](https://ubuntu.com/download/server)
   * **Rocky Linux 9 / AlmaLinux 9**: [rockylinux.org](https://rockylinux.org/) or [almalinux.org](https://almalinux.org/)
3. Recommended VM Specifications:
   * 2 vCPUs
   * 2 GB to 4 GB RAM
   * 25 GB Virtual Hard Disk (dynamically allocated)
   * Bridged or Host-Only Adapter for networking labs

> [!WARNING]
> **Safety Warning:** Destructive commands such as `rm -rf`, `dd`, `mkfs`, `fdisk`, and iptables flushes must only be run inside a dedicated virtual machine or disposable container. Never practice destructive commands on host production systems.

---

## Structure of Each Module

Every level module is self-contained and follows this systematic structure:

1. **Core Concept & Theory:** Concise technical explanation connecting concepts to production environments.
2. **Why It Matters in Production:** Practical use cases encountered by sysadmins and DevOps engineers.
3. **Command Reference & Syntax:** Exact syntax breakdown, frequently used flags, and options.
4. **Realistic Examples:** Command execution demonstrations with expected terminal outputs.
5. **Hands-On Guided Labs:** Step-by-step challenges to solidify practical muscle memory.
6. **Troubleshooting & Fixes:** Common mistakes, syntax pitfalls, and diagnostic workflows.
7. **Practical Real-World Scenarios:** Production incident simulations with root cause analysis.
8. **Knowledge Check & Interview Questions:** Curated technical questions asked during junior-to-senior sysadmin interviews.

---

## Marker Legend

Throughout the modules, the following standard callouts highlight critical insights:

| Marker | Meaning |
|---|---|
| 📌 **Important** | Fundamental administrative concept or essential command syntax |
| ⚠️ **Common Mistake** | Frequent configuration pitfall or syntax error |
| ✅ **Best Practice** | Production-grade standard recommended by enterprise environments |
| 🔴 **DANGER** | High-risk command that can lead to data loss or system unavailability |
| 💡 **Admin Tip** | Real-world sysadmin workflow, shortcut, or operational tip |
| 🎯 **Interview Point** | Common technical interview topic or screening question |
| 🔧 **Troubleshooting** | Step-by-step diagnostic technique or error resolution |

---

## Contributing Guidelines

Contributions, corrections, and improvements are welcome! If you would like to contribute new labs, improve explanations, or add real-world scenarios, please follow these steps:

### How to Contribute

1. **Fork the Repository:** Click the **Fork** button at the top right of this page.
2. **Clone your Fork:**
   ```bash
   git clone https://github.com/<your-username>/<repo-name>.git
   cd <repo-name>
   ```
3. **Create a Feature Branch:**
   ```bash
   git checkout -b feature/level-update
   # or
   git checkout -b fix/typo-correction
   ```
4. **Make Your Changes:**
   * Keep explanations concise, technical, and directly connected to practical tasks.
   * Provide full, working commands with realistic outputs (do not truncate code or leave placeholders).
   * Verify all commands on a clean Linux installation (Ubuntu 22.04 or Rocky Linux 9) before submitting.
   * Maintain consistent formatting with existing level files.
5. **Commit and Push:**
   ```bash
   git add .
   git commit -m "docs(level-03): add troubleshooting case for broken apt dependencies"
   git push origin feature/level-update
   ```
6. **Open a Pull Request:** Submit a Pull Request targeting the `main` branch with a clear description of what was changed and why.

---

## Reporting Errors & Feedback

Accuracy is essential for technical reference material. If you spot any mistakes:

* Incorrect command syntax or broken options
* Outdated package names or repository URLs
* Typos, formatting inconsistencies, or unclear explanations
* Missing real-world scenarios or edge cases

Please report them through either of the following channels:

* **GitHub Issues:** Open an issue in this repository describing the problem and the specific level file.
* **Direct Contact:** Contact the repository maintainer directly:
  * **Maintainer:** Abhishek Maurya
  * **Email:** [maurya972137@gmail.com](mailto:maurya972137@gmail.com)

All reported errors are reviewed and corrected promptly.

---

## Author & Maintainer

* **Author:** Abhishek Maurya
* **Email:** [maurya972137@gmail.com](mailto:maurya972137@gmail.com)
* **Focus:** Linux Systems, Backend Architecture & DevOps Engineering

---

## License

This project is licensed under the [MIT License](LICENSE) (or Creative Commons Attribution 4.0 International) for open educational and practical usage. Feel free to use these notes for personal learning, internal team training, and reference.
