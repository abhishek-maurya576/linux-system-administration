# Level 5 — Services & systemd

> **Goal:** Understand how Linux manages services using systemd, control services with `systemctl`, read logs with `journalctl`, create custom services, troubleshoot failed services, and manage boot targets.

> **Lab Environment:** VirtualBox VM recommended (WSL2 has limited systemd support). Ubuntu Server or Rocky Linux.

> **Prerequisite:** Complete Levels 1–4

---

## 5.1 — What Is a Service?

### Minimum Theory

A **service** (also called a **daemon**) is a program that runs in the background, waiting to do work. Services start at boot and keep running until the system shuts down (or they crash).

**Examples of services you'll manage daily:**

| Service | Purpose | Package |
|---------|---------|---------|
| `sshd` | SSH server — remote access | `openssh-server` |
| `nginx` | Web server / reverse proxy | `nginx` |
| `apache2` / `httpd` | Web server | `apache2` (Ubuntu) / `httpd` (RHEL) |
| `cron` / `crond` | Scheduled task runner | `cron` (Ubuntu) / `cronie` (RHEL) |
| `rsyslog` | System logging | `rsyslog` |
| `NetworkManager` | Network management | `NetworkManager` |
| `firewalld` | Firewall management | `firewalld` |
| `mysql` / `mariadb` | Database server | `mysql-server` / `mariadb-server` |
| `docker` | Container runtime | `docker-ce` |
| `postfix` | Mail server | `postfix` |

**Why services matter for admins:**
- If SSH service stops → you lose remote access
- If web server service stops → website goes down
- If cron service stops → scheduled tasks don't run
- If logging service stops → you lose visibility into problems

---

## 5.2 — What Is systemd?

### The Init System

When Linux boots, the kernel starts **one** userspace process: the **init system** (PID 1). This init system is responsible for starting all other services and managing the system.

**📌 Important — systemd is the init system on virtually all modern Linux distributions:**

| Distribution | Init System |
|-------------|-------------|
| Ubuntu 16.04+ | systemd |
| Debian 8+ | systemd |
| RHEL/CentOS 7+ | systemd |
| Rocky/AlmaLinux 8+ | systemd |
| Fedora 15+ | systemd |
| Older systems | SysVinit or Upstart (legacy) |

```bash
# Verify your system uses systemd
ps -p 1 -o comm=
# Output: systemd

# Check systemd version
systemctl --version
```

### What systemd Does

systemd is much more than just a service manager. It handles:

| Feature | Purpose |
|---------|---------|
| **Service management** | Start, stop, restart, enable services |
| **Boot process** | Parallel service startup for fast boots |
| **Logging** | Centralized journal (`journalctl`) |
| **Timers** | Modern replacement for cron jobs |
| **Mounts** | Mount/automount filesystems |
| **Network** | `systemd-networkd` for network config |
| **Targets** | Group services (replaces runlevels) |
| **Socket activation** | Start services on-demand when connections arrive |

### systemd Units

Everything systemd manages is called a **unit**. Different unit types:

| Unit Type | Extension | Purpose | Example |
|-----------|-----------|---------|---------|
| **Service** | `.service` | Daemons/services | `nginx.service` |
| **Socket** | `.socket` | Network sockets, IPC | `sshd.socket` |
| **Timer** | `.timer` | Scheduled tasks (cron replacement) | `logrotate.timer` |
| **Mount** | `.mount` | Filesystem mount points | `home.mount` |
| **Target** | `.target` | Group of units (like runlevels) | `multi-user.target` |
| **Path** | `.path` | Watch files/directories | `cups.path` |
| **Device** | `.device` | Hardware devices | |
| **Slice** | `.slice` | Resource management (cgroups) | `user.slice` |

**📌 Important:** When you type `systemctl restart nginx`, systemd looks for `nginx.service`. You can omit `.service` for convenience — systemd assumes it.

---

## 5.3 — `systemctl` — Managing Services

### Checking Service Status

```bash
systemctl status nginx
```

**Expected output:**
```
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2026-09-01 09:00:00 UTC; 3 days ago
       Docs: man:nginx(8)
    Process: 580 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 600 (nginx)
      Tasks: 3 (limit: 4657)
     Memory: 8.5M
        CPU: 1.234s
     CGroup: /system.slice/nginx.service
             ├─600 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─601 "nginx: worker process"
             └─602 "nginx: worker process"

Sep 01 09:00:00 server01 systemd[1]: Starting A high performance web server...
Sep 01 09:00:00 server01 systemd[1]: Started A high performance web server.
```

**Reading the status output:**

| Field | Meaning |
|-------|---------|
| `Loaded: loaded (... enabled ...)` | Unit file is loaded and service is **enabled** (starts at boot) |
| `Active: active (running)` | Service is currently running |
| `Main PID: 600` | The main process ID |
| `Tasks: 3` | Number of threads/processes |
| `Memory: 8.5M` | RAM used by the service |
| `CGroup` | Process tree within its cgroup |
| Log lines at bottom | Recent journal entries for the service |

**Possible Active states:**

| State | Meaning |
|-------|---------|
| `active (running)` | Running normally |
| `active (exited)` | Ran successfully and exited (one-shot tasks) |
| `active (waiting)` | Running but waiting for an event |
| `inactive (dead)` | Not running |
| `failed` | Crashed or exited with an error |
| `activating` | Starting up |
| `deactivating` | Shutting down |

---

### Starting and Stopping Services

```bash
# Start a service
sudo systemctl start nginx

# Stop a service
sudo systemctl stop nginx

# Restart a service (stop + start)
sudo systemctl restart nginx

# Reload configuration without restarting (zero downtime)
sudo systemctl reload nginx

# Reload if supported, otherwise restart
sudo systemctl reload-or-restart nginx
```

**📌 Important — `restart` vs `reload`:**

| Action | What Happens | Downtime |
|--------|-------------|----------|
| `restart` | Service is stopped, then started | Brief downtime (connections dropped) |
| `reload` | Service re-reads config files without stopping | No downtime (existing connections preserved) |

**✅ Best Practice:** Always use `reload` when possible (e.g., after editing Nginx config). Use `restart` only when `reload` isn't supported or when you need a clean state.

```bash
# Check if a service supports reload
systemctl show nginx --property=CanReload
# CanReload=yes
```

**⚠️ Common Mistake:** Restarting a production web server when `reload` would work. Restarting drops all active connections.

---

### Enabling and Disabling Services

```bash
# Enable service to start automatically at boot
sudo systemctl enable nginx

# Disable service from starting at boot
sudo systemctl disable nginx

# Enable AND start immediately
sudo systemctl enable --now nginx

# Disable AND stop immediately
sudo systemctl disable --now nginx
```

**What `enable` actually does:** Creates a symlink from the target directory to the service's unit file:

```bash
sudo systemctl enable nginx
# Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service →
#   /lib/systemd/system/nginx.service
```

**What `disable` does:** Removes that symlink.

**Check if a service is enabled:**
```bash
systemctl is-enabled nginx
# enabled / disabled / static / masked
```

| State | Meaning |
|-------|---------|
| `enabled` | Starts at boot |
| `disabled` | Does not start at boot |
| `static` | Cannot be enabled directly (started by other units) |
| `masked` | Completely blocked — cannot be started at all |

### Masking Services

```bash
# Mask a service (prevent it from being started at all)
sudo systemctl mask nginx

# Unmask
sudo systemctl unmask nginx
```

**Why mask?** To completely prevent a service from running, even if another service tries to start it. Used for services that conflict or that you want permanently disabled.

**💡 Admin Tip:** Mask services you never want running rather than just disabling them. `disable` still allows manual starting; `mask` blocks everything.

---

### Listing Services

```bash
# List all active services
systemctl list-units --type=service

# List ALL services (including inactive)
systemctl list-units --type=service --all

# List only failed services
systemctl list-units --type=service --state=failed
systemctl --failed

# List enabled services
systemctl list-unit-files --type=service --state=enabled

# List all unit files
systemctl list-unit-files
```

**Quick checks:**

```bash
# Is the service running?
systemctl is-active nginx
# active / inactive

# Has the service failed?
systemctl is-failed nginx
# active / failed

# Is it enabled to start at boot?
systemctl is-enabled nginx
# enabled / disabled
```

---

## 5.4 — Service Unit Files

### Where Unit Files Live

| Location | Purpose | Priority |
|----------|---------|----------|
| `/lib/systemd/system/` | Package-installed unit files (defaults) | Lowest |
| `/usr/lib/systemd/system/` | Same as above (RHEL style) | Lowest |
| `/etc/systemd/system/` | Admin-created/modified unit files | **Highest** (overrides above) |
| `/run/systemd/system/` | Runtime-generated unit files | Medium |

**📌 Important:** Never edit files in `/lib/systemd/system/`. They get overwritten on package updates. Always copy to `/etc/systemd/system/` or use drop-in overrides.

### Reading a Unit File

```bash
systemctl cat nginx.service
```

**Example unit file (`/lib/systemd/system/nginx.service`):**

```ini
[Unit]
Description=A high performance web server and a reverse proxy server
After=network.target
Documentation=man:nginx(8)

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

### Unit File Sections Explained

#### `[Unit]` Section — Metadata and Dependencies

| Directive | Meaning | Example |
|-----------|---------|---------|
| `Description=` | Human-readable description | `Description=My Web App` |
| `After=` | Start this AFTER the listed units | `After=network.target` |
| `Before=` | Start this BEFORE the listed units | `Before=httpd.service` |
| `Requires=` | Hard dependency — if dep fails, this fails too | `Requires=mysql.service` |
| `Wants=` | Soft dependency — if dep fails, this still starts | `Wants=redis.service` |
| `BindsTo=` | Like Requires, plus stops when dep stops | `BindsTo=dev-sda1.device` |
| `Conflicts=` | Cannot run alongside this unit | `Conflicts=sendmail.service` |
| `Documentation=` | Links to docs | `Documentation=man:nginx(8)` |

**📌 Important — `After` vs `Requires`:**
- `After=` controls **ordering** (when to start)
- `Requires=` controls **dependency** (if dep fails, don't start)
- You usually need BOTH: `After=network.target` + `Requires=network.target`

#### `[Service]` Section — How to Run the Service

| Directive | Meaning | Example |
|-----------|---------|---------|
| `Type=` | Service startup type (see below) | `Type=simple` |
| `ExecStart=` | Command to start the service | `ExecStart=/usr/bin/myapp` |
| `ExecStop=` | Command to stop the service | `ExecStop=/bin/kill $MAINPID` |
| `ExecReload=` | Command to reload config | `ExecReload=/bin/kill -HUP $MAINPID` |
| `ExecStartPre=` | Command to run before start | `ExecStartPre=/usr/bin/check-config` |
| `ExecStartPost=` | Command to run after start | `ExecStartPost=/usr/bin/notify` |
| `Restart=` | When to auto-restart (see below) | `Restart=on-failure` |
| `RestartSec=` | Seconds to wait before restart | `RestartSec=5` |
| `User=` | Run as this user | `User=www-data` |
| `Group=` | Run as this group | `Group=www-data` |
| `WorkingDirectory=` | Set working directory | `WorkingDirectory=/opt/myapp` |
| `Environment=` | Set environment variables | `Environment=PORT=8080` |
| `EnvironmentFile=` | Load env vars from file | `EnvironmentFile=/etc/default/myapp` |
| `StandardOutput=` | Where stdout goes | `StandardOutput=journal` |
| `StandardError=` | Where stderr goes | `StandardError=journal` |
| `PIDFile=` | Path to PID file | `PIDFile=/run/myapp.pid` |
| `TimeoutStartSec=` | Max time to start | `TimeoutStartSec=30` |
| `TimeoutStopSec=` | Max time to stop | `TimeoutStopSec=30` |

**Service Types:**

| Type | Behavior | Use For |
|------|----------|---------|
| `simple` | Process started by `ExecStart` IS the service (default) | Most services, Node.js, Python apps |
| `forking` | Process forks and parent exits. systemd tracks child via PIDFile | Apache, Nginx, traditional daemons |
| `oneshot` | Process runs and exits. Service is "active" after exit | Setup scripts, initialization tasks |
| `notify` | Like simple, but service sends notification when ready | Services using sd_notify() |
| `idle` | Like simple, but delays until other jobs finish | Low-priority boot services |

**Restart policies:**

| Value | Restart When |
|-------|-------------|
| `no` | Never auto-restart (default) |
| `on-success` | Only on clean exit (exit code 0) |
| `on-failure` | On non-zero exit, signal, timeout, watchdog |
| `on-abnormal` | On signal, timeout, watchdog |
| `on-abort` | On uncaught signal |
| `on-watchdog` | On watchdog timeout |
| `always` | Always restart, no matter what |

**✅ Best Practice:** For production services, always set `Restart=on-failure` or `Restart=always` with a `RestartSec=5`:
```ini
Restart=on-failure
RestartSec=5
StartLimitBurst=3
StartLimitIntervalSec=60
```
This restarts on failure, waits 5 seconds, and stops retrying after 3 failures within 60 seconds.

#### `[Install]` Section — Boot Integration

| Directive | Meaning | Example |
|-----------|---------|---------|
| `WantedBy=` | Which target pulls this service in | `WantedBy=multi-user.target` |
| `RequiredBy=` | Hard dependency from target | `RequiredBy=multi-user.target` |
| `Alias=` | Alternative name for the unit | `Alias=myapp.service` |

**`WantedBy=multi-user.target`** means: enable this service when the system reaches multi-user mode (normal server boot).

---

## 5.5 — Creating a Custom systemd Service

### Step-by-Step: Create a Service for a Python App

**Step 1: Create the application**

```bash
sudo mkdir -p /opt/myapp
sudo tee /opt/myapp/app.py > /dev/null << 'EOF'
#!/usr/bin/env python3
"""Simple web server for systemd service demo."""
import http.server
import socketserver
import os

PORT = int(os.environ.get('PORT', 8080))

class Handler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/plain')
        self.end_headers()
        self.wfile.write(b'Hello from my systemd service!\n')

with socketserver.TCPServer(("", PORT), Handler) as httpd:
    print(f"Serving on port {PORT}")
    httpd.serve_forever()
EOF

sudo chmod +x /opt/myapp/app.py
```

**Step 2: Create a dedicated system user**

```bash
sudo useradd -r -s /sbin/nologin myapp
sudo chown -R myapp:myapp /opt/myapp
```

**Step 3: Create the unit file**

```bash
sudo tee /etc/systemd/system/myapp.service > /dev/null << 'EOF'
[Unit]
Description=My Custom Python Web Application
After=network.target
Documentation=https://example.com/myapp/docs

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment=PORT=8080

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/opt/myapp

[Install]
WantedBy=multi-user.target
EOF
```

**Step 4: Reload systemd and start the service**

```bash
# Reload systemd to pick up the new unit file
sudo systemctl daemon-reload

# Start the service
sudo systemctl start myapp

# Check status
sudo systemctl status myapp

# Test it
curl http://localhost:8080
# Hello from my systemd service!

# Enable at boot
sudo systemctl enable myapp
```

**Step 5: Test restart behavior**

```bash
# Kill the process — systemd should restart it
sudo kill $(pgrep -f "app.py")

# Wait a few seconds and check
sleep 6
systemctl status myapp
# Should show: active (running) — systemd restarted it!
```

**📌 Important:** After any change to a unit file, you MUST run:
```bash
sudo systemctl daemon-reload
```
Without this, systemd uses the old cached version of the unit file.

**⚠️ Common Mistake:** Editing a unit file and then restarting the service without `daemon-reload`. The old config is still active.

---

### Using Drop-In Overrides

Instead of modifying unit files directly, use drop-in directories to override specific settings:

```bash
# Create an override for nginx
sudo systemctl edit nginx
```

This opens an editor and creates `/etc/systemd/system/nginx.service.d/override.conf`:

```ini
[Service]
# Increase timeout
TimeoutStartSec=60
# Set memory limit
MemoryMax=512M
```

```bash
# Apply the override
sudo systemctl daemon-reload
sudo systemctl restart nginx

# Verify the override is active
systemctl show nginx --property=TimeoutStartSec
systemctl show nginx --property=MemoryMax

# View the full composed configuration
systemctl cat nginx
```

**✅ Best Practice:** Always use `systemctl edit` for overrides rather than editing original unit files. Overrides survive package updates.

To edit the full unit file (not just overrides):
```bash
sudo systemctl edit --full nginx
```

---

## 5.6 — `journalctl` — Reading Logs

### What Is the Journal?

systemd maintains its own logging system called the **journal**. It captures:
- Service stdout/stderr
- Kernel messages
- System messages
- Boot messages

### Basic Usage

```bash
# Show all logs (paged)
journalctl

# Show logs in reverse order (newest first)
journalctl -r

# Show logs and follow (like tail -f)
journalctl -f

# Show only the last 50 lines
journalctl -n 50

# Show logs without paging (for scripting)
journalctl --no-pager
```

### Filtering by Service

```bash
# Logs for a specific service
journalctl -u nginx

# Follow a service's logs in real-time
journalctl -u nginx -f

# Last 20 log entries for a service
journalctl -u nginx -n 20

# Logs for multiple services
journalctl -u nginx -u php-fpm
```

**💡 Admin Tip:** When troubleshooting a service, this is your go-to workflow:
```bash
# Terminal 1: Watch the logs
journalctl -u myservice -f

# Terminal 2: Restart the service and watch for errors
sudo systemctl restart myservice
```

### Filtering by Time

```bash
# Logs since a specific time
journalctl --since "2026-09-04 10:00:00"

# Logs from the last hour
journalctl --since "1 hour ago"

# Logs from today
journalctl --since today

# Logs between two times
journalctl --since "2026-09-04 09:00" --until "2026-09-04 12:00"

# Logs from the current boot
journalctl -b

# Logs from the previous boot
journalctl -b -1

# List all recorded boots
journalctl --list-boots
```

### Filtering by Priority

```bash
# Show only errors and above
journalctl -p err

# Show only warnings and above
journalctl -p warning
```

**Log priority levels (syslog):**

| Priority | Code | Meaning |
|----------|------|---------|
| 0 | `emerg` | System is unusable |
| 1 | `alert` | Immediate action required |
| 2 | `crit` | Critical conditions |
| 3 | `err` | Error conditions |
| 4 | `warning` | Warning conditions |
| 5 | `notice` | Normal but significant |
| 6 | `info` | Informational |
| 7 | `debug` | Debug messages |

### Filtering by Other Criteria

```bash
# Kernel messages only
journalctl -k

# Messages from a specific PID
journalctl _PID=600

# Messages from a specific user
journalctl _UID=1000

# Messages from a specific executable
journalctl _EXE=/usr/sbin/nginx
```

### Output Formats

```bash
# JSON output (for scripting)
journalctl -u nginx -o json-pretty -n 5

# Short output with timestamps (default)
journalctl -u nginx -o short-iso

# Verbose output (all fields)
journalctl -u nginx -o verbose -n 5
```

### Journal Disk Usage

```bash
# Check journal disk usage
journalctl --disk-usage

# Limit journal size
sudo journalctl --vacuum-size=500M

# Remove logs older than 30 days
sudo journalctl --vacuum-time=30d
```

**Permanent journal size limit in `/etc/systemd/journald.conf`:**

```ini
[Journal]
SystemMaxUse=500M
MaxRetentionSec=30day
```

```bash
# Apply changes
sudo systemctl restart systemd-journald
```

**🎯 Interview Point:** "How do you view logs for a specific service?"  
> `journalctl -u service_name`. Add `-f` to follow in real-time, `--since "1 hour ago"` for time filtering, or `-p err` for error-level only.

---

## 5.7 — Boot Targets (Runlevels)

### What Are Targets?

Targets are groups of units that define system states. They replace the old SysVinit runlevels.

| Target | Old Runlevel | Description |
|--------|-------------|-------------|
| `poweroff.target` | 0 | Shut down the system |
| `rescue.target` | 1 | Single-user mode (root only, no network) |
| `multi-user.target` | 3 | Multi-user, with networking, no GUI |
| `graphical.target` | 5 | Multi-user, with networking and GUI |
| `reboot.target` | 6 | Reboot the system |
| `emergency.target` | — | Emergency shell (minimal boot) |

### Managing Targets

```bash
# Check current default target
systemctl get-default
# multi-user.target (servers) or graphical.target (desktops)

# Set default target to multi-user (no GUI — server mode)
sudo systemctl set-default multi-user.target

# Set default target to graphical (GUI — desktop mode)
sudo systemctl set-default graphical.target

# Switch to a target immediately (without reboot)
sudo systemctl isolate multi-user.target   # Drop to text mode
sudo systemctl isolate graphical.target    # Switch to GUI

# Emergency mode (troubleshooting)
sudo systemctl isolate rescue.target
```

**💡 Admin Tip:** On servers, always set the default to `multi-user.target`. A GUI wastes resources and increases the attack surface:
```bash
sudo systemctl set-default multi-user.target
```

### System Power Management

```bash
# Reboot
sudo systemctl reboot
# OR
sudo reboot

# Shut down
sudo systemctl poweroff
# OR
sudo shutdown -h now

# Scheduled shutdown (10 minutes)
sudo shutdown -h +10 "Server going down for maintenance"

# Cancel scheduled shutdown
sudo shutdown -c

# Suspend (laptop)
sudo systemctl suspend

# Hibernate
sudo systemctl hibernate
```

---

## 5.8 — systemd Timers (Cron Replacement)

### Why Timers?

systemd timers are a modern alternative to cron with advantages:
- Better logging (via journal)
- Dependency management
- Can run missed events after downtime
- Per-timer resource controls

### Creating a Timer

**Step 1: Create the service (what to run):**

```bash
sudo tee /etc/systemd/system/disk-check.service > /dev/null << 'EOF'
[Unit]
Description=Check disk usage and log warning

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'df -h / | tail -1 | awk "{if (\$5+0 > 80) print \"WARNING: Disk usage is \" \$5}" >> /var/log/disk-check.log'
EOF
```

**Step 2: Create the timer (when to run):**

```bash
sudo tee /etc/systemd/system/disk-check.timer > /dev/null << 'EOF'
[Unit]
Description=Run disk check every hour

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

**Step 3: Enable and start the timer:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now disk-check.timer
```

**Step 4: Verify:**

```bash
# List all active timers
systemctl list-timers

# Check timer status
systemctl status disk-check.timer

# Manually trigger the service
sudo systemctl start disk-check.service
```

### Timer Calendar Expressions

| Expression | Meaning |
|-----------|---------|
| `hourly` | Every hour at :00 |
| `daily` | Every day at 00:00 |
| `weekly` | Every Monday at 00:00 |
| `monthly` | 1st of every month at 00:00 |
| `*-*-* 06:00:00` | Every day at 6:00 AM |
| `Mon *-*-* 09:00:00` | Every Monday at 9:00 AM |
| `*-*-* *:00/15:00` | Every 15 minutes |
| `*-*-01 00:00:00` | First day of every month |

**Test calendar expressions:**
```bash
systemd-analyze calendar "Mon *-*-* 09:00:00"
systemd-analyze calendar hourly
```

### Monotonic Timers (Relative)

```bash
[Timer]
OnBootSec=5min         # 5 minutes after boot
OnUnitActiveSec=1h     # 1 hour after last activation
OnStartupSec=10min     # 10 minutes after systemd start
```

---

## 5.9 — Service Dependencies and Ordering

### Viewing Dependencies

```bash
# Show what a service depends on
systemctl list-dependencies nginx

# Show what depends ON a service
systemctl list-dependencies --reverse nginx

# Show the full boot dependency tree
systemctl list-dependencies multi-user.target
```

### Boot Analysis

```bash
# How long did the boot take?
systemd-analyze

# Blame — which services took the longest?
systemd-analyze blame | head -15

# Critical chain — the boot path
systemd-analyze critical-chain

# Plot the boot process (generates SVG)
systemd-analyze plot > /tmp/boot-chart.svg
```

**💡 Admin Tip:** If the server takes too long to boot, use `systemd-analyze blame` to find slow services. Consider disabling or optimizing them.

---

## 5.10 — Hands-On Labs

### Lab 1: Basic Service Management

```bash
# 1. Install nginx (if not installed)
sudo apt install -y nginx    # Ubuntu
# sudo dnf install -y nginx  # Rocky

# 2. Check service status
systemctl status nginx

# 3. Stop the service
sudo systemctl stop nginx
systemctl status nginx    # Should show: inactive (dead)

# 4. Start the service
sudo systemctl start nginx
systemctl is-active nginx    # Should show: active

# 5. Restart the service
sudo systemctl restart nginx

# 6. Reload configuration
sudo systemctl reload nginx

# 7. Check if enabled at boot
systemctl is-enabled nginx

# 8. Disable from boot
sudo systemctl disable nginx
systemctl is-enabled nginx    # disabled

# 9. Re-enable
sudo systemctl enable nginx
systemctl is-enabled nginx    # enabled

# 10. List all running services
systemctl list-units --type=service --state=running
```

### Lab 2: Reading Logs with journalctl

```bash
# 1. View nginx logs
journalctl -u nginx -n 20

# 2. Follow nginx logs in real-time (open a second terminal for step 3)
journalctl -u nginx -f

# 3. In another terminal, restart nginx and watch the logs
sudo systemctl restart nginx

# 4. View logs from the current boot only
journalctl -b -u nginx

# 5. View only error messages system-wide
journalctl -p err --since today

# 6. View logs from the last 30 minutes
journalctl --since "30 minutes ago"

# 7. Check journal disk usage
journalctl --disk-usage

# 8. View kernel messages
journalctl -k --since today
```

### Lab 3: Create a Custom Service

```bash
# 1. Create a simple script
sudo mkdir -p /opt/healthcheck
sudo tee /opt/healthcheck/check.sh > /dev/null << 'SCRIPT'
#!/bin/bash
# Simple health check script
LOG="/var/log/healthcheck.log"
echo "$(date '+%Y-%m-%d %H:%M:%S') - Health check started" >> "$LOG"
echo "  Disk: $(df -h / | tail -1 | awk '{print $5}') used" >> "$LOG"
echo "  Memory: $(free -h | awk '/^Mem:/ {print $3 "/" $2}')" >> "$LOG"
echo "  Load: $(cat /proc/loadavg | awk '{print $1, $2, $3}')" >> "$LOG"
echo "  Uptime: $(uptime -p)" >> "$LOG"
echo "---" >> "$LOG"
SCRIPT
sudo chmod +x /opt/healthcheck/check.sh

# 2. Create the service unit
sudo tee /etc/systemd/system/healthcheck.service > /dev/null << 'EOF'
[Unit]
Description=System Health Check

[Service]
Type=oneshot
ExecStart=/opt/healthcheck/check.sh
EOF

# 3. Create the timer unit (run every 5 minutes)
sudo tee /etc/systemd/system/healthcheck.timer > /dev/null << 'EOF'
[Unit]
Description=Run health check every 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
EOF

# 4. Reload, enable, and start
sudo systemctl daemon-reload
sudo systemctl enable --now healthcheck.timer

# 5. Verify
systemctl list-timers | grep health
systemctl status healthcheck.timer

# 6. Run it manually once
sudo systemctl start healthcheck.service

# 7. Check the log
cat /var/log/healthcheck.log

# 8. Wait 5 minutes, check again
sleep 300
cat /var/log/healthcheck.log
```

### Lab 4: Service Investigation

```bash
# 1. View the nginx unit file
systemctl cat nginx.service

# 2. Show all nginx properties
systemctl show nginx | head -30

# 3. Show specific properties
systemctl show nginx --property=MainPID,ActiveState,SubState

# 4. View dependencies
systemctl list-dependencies nginx

# 5. Boot analysis
systemd-analyze
systemd-analyze blame | head -10

# 6. Check default target
systemctl get-default
```

---

## 5.11 — Troubleshooting Failed Services

### Systematic Troubleshooting Workflow

```
1. Check status  →  2. Read logs  →  3. Check config  →  4. Check deps  →  5. Fix  →  6. Verify
```

### Exercise 1: Service Failed to Start

**Problem:** `systemctl start myapp` fails.

```bash
# Step 1: Check status
systemctl status myapp
# Look for: Active: failed, error messages

# Step 2: Read the full logs
journalctl -u myapp -n 50 --no-pager

# Step 3: Check the unit file for errors
systemctl cat myapp

# Step 4: Verify the ExecStart binary exists and is executable
ls -la /path/to/binary
file /path/to/binary

# Step 5: Try running the command manually as the service user
sudo -u myapp_user /path/to/binary

# Step 6: Check permissions on working directory, log files, etc.
ls -la /opt/myapp/
ls -la /var/log/myapp/

# Step 7: Check if a port is already in use
sudo ss -tlnp | grep :8080

# Step 8: After fixing, reload and restart
sudo systemctl daemon-reload
sudo systemctl restart myapp
systemctl status myapp
```

### Exercise 2: Service Keeps Crashing

**Problem:** Service starts but crashes repeatedly.

```bash
# Check how many times it has restarted
systemctl show myapp --property=NRestarts

# View recent crash logs
journalctl -u myapp --since "1 hour ago" -p err

# Check the restart configuration
systemctl show myapp --property=Restart,RestartSec,StartLimitBurst

# Check resource limits
systemctl show myapp --property=MemoryMax,CPUQuota

# Check the process exit code from logs
journalctl -u myapp | grep -i "exit\|fail\|error\|code"
```

### Exercise 3: Service Starts Too Slowly

**Problem:** Service takes a long time to start and times out.

```bash
# Check the timeout setting
systemctl show myapp --property=TimeoutStartSec

# Fix: Increase the timeout
sudo systemctl edit myapp
# Add:
# [Service]
# TimeoutStartSec=120

sudo systemctl daemon-reload
sudo systemctl restart myapp
```

### Common Service Failure Causes

| Symptom | Common Cause | Fix |
|---------|-------------|-----|
| `203/EXEC` | Binary not found or not executable | Check `ExecStart` path, `chmod +x` |
| `200/CHDIR` | WorkingDirectory doesn't exist | Create the directory |
| `217/USER` | User specified in unit doesn't exist | Create the user with `useradd` |
| `exit-code 1` | Application error | Check app logs, run manually |
| `signal SEGV` | Application crash (segfault) | Check app for bugs, update |
| `Port already in use` | Another service on the same port | `ss -tlnp` to find conflict |
| `Permission denied` | Wrong file/dir permissions | Fix ownership with `chown` |
| `timeout` | Service takes too long to start | Increase `TimeoutStartSec` |

---

## 5.12 — Practical Scenarios

### Scenario 1: Deploy and Manage a Web Application Service

**Task:** Create a systemd service for a Node.js (or Python) application that:
- Runs as a dedicated user
- Restarts automatically on failure
- Starts at boot
- Has proper logging

**Try this yourself using the custom service template from Lab 3.**

### Scenario 2: Service Won't Start After Config Change

**Problem:** You edited the Nginx config and now it won't start.

<details>
<summary><strong>Solution</strong></summary>

```bash
# Step 1: Check status
systemctl status nginx
# Active: failed

# Step 2: Read the logs
journalctl -u nginx -n 20
# nginx: [emerg] unknown directive "servr_name" in /etc/nginx/sites-enabled/default:3

# Step 3: The log tells you exactly what's wrong — typo in config!

# Step 4: Test nginx config
sudo nginx -t
# nginx: [emerg] unknown directive "servr_name"...
# nginx: configuration file test failed

# Step 5: Fix the typo
sudo vim /etc/nginx/sites-enabled/default
# Fix: servr_name → server_name

# Step 6: Test again
sudo nginx -t
# nginx: configuration file test is successful

# Step 7: Start the service
sudo systemctl start nginx
systemctl status nginx
```

</details>

### Scenario 3: Find Why Boot Is Slow

**Task:** The server takes 3 minutes to boot. Find the culprit.

<details>
<summary><strong>Solution</strong></summary>

```bash
# Step 1: Check total boot time
systemd-analyze
# Startup finished in 2.1s (kernel) + 2min 45s (userspace) = 2min 47s

# Step 2: Find slow services
systemd-analyze blame | head -10
# 2min 30s    slow-network-check.service
#      10s    docker.service
#       5s    nginx.service

# Step 3: Investigate the slow service
systemctl status slow-network-check.service
journalctl -u slow-network-check.service -b

# Step 4: Fix or disable if not needed
sudo systemctl disable slow-network-check.service

# Step 5: Verify on next boot
sudo reboot
systemd-analyze
```

</details>

---

## 5.13 — Practice Questions

1. **Install `nginx`**, start it, verify it's running, then enable it at boot. Show the exact commands.

2. **Create a custom oneshot service** that writes the current date and system load to `/var/log/system-snapshot.log`. Create a timer that runs it every 10 minutes.

3. **A service failed.** Use `journalctl` to find all error-level messages for `sshd` from the last 2 hours.

4. **Find which services** took the longest to start during boot. Which command do you use?

5. **Mask the `bluetooth` service** (or any service you don't need). Verify it cannot be started. Then unmask it.

6. **What is the difference between** `systemctl restart` and `systemctl reload`? When would you use each?

7. **Create a drop-in override** for nginx that sets `TimeoutStopSec=30` without modifying the original unit file.

8. **List all failed services** on the system. For each failed service, show how to investigate why it failed.

---

## Level 5 Summary

### What I Learned

- systemd is the init system (PID 1) managing services, boot, and more
- Services are background daemons; everything managed by systemd is a "unit"
- `systemctl` is the primary tool for service management
- `journalctl` provides centralized, filterable log access
- Unit files in `/etc/systemd/system/` override package defaults
- `daemon-reload` is required after any unit file change
- `enable` creates a symlink for boot startup; `disable` removes it
- `mask` completely prevents a service from running
- systemd timers are a modern cron replacement with better logging
- Boot targets replace old runlevels (multi-user.target = server mode)

### Commands to Remember

| Command | Purpose |
|---------|---------|
| `systemctl status service` | Check service status and recent logs |
| `sudo systemctl start service` | Start a service |
| `sudo systemctl stop service` | Stop a service |
| `sudo systemctl restart service` | Restart a service |
| `sudo systemctl reload service` | Reload config without restart |
| `sudo systemctl enable service` | Enable at boot |
| `sudo systemctl enable --now service` | Enable AND start |
| `sudo systemctl disable service` | Disable from boot |
| `sudo systemctl mask service` | Completely block a service |
| `systemctl is-active service` | Check if running |
| `systemctl is-enabled service` | Check if enabled at boot |
| `systemctl --failed` | List all failed services |
| `systemctl list-units --type=service` | List active services |
| `systemctl list-timers` | List active timers |
| `systemctl cat service` | View the unit file |
| `sudo systemctl edit service` | Create drop-in override |
| `sudo systemctl daemon-reload` | Reload unit files after changes |
| `journalctl -u service` | View service logs |
| `journalctl -u service -f` | Follow service logs live |
| `journalctl -p err` | Show error-level messages |
| `journalctl --since "1 hour ago"` | Time-filtered logs |
| `journalctl -b` | Current boot logs |
| `systemd-analyze blame` | Find slow boot services |
| `systemctl get-default` | Show default boot target |
| `sudo systemctl set-default target` | Change default boot target |

### Practical Skills Gained

- Start, stop, restart, enable, and disable services
- Read and interpret service status output
- Use journalctl for targeted log investigation
- Create custom systemd services from scratch
- Create systemd timers for scheduled tasks
- Override service settings safely with drop-ins
- Troubleshoot failed services systematically
- Analyze and optimize boot time

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Forgetting `daemon-reload` after editing a unit file | Always: `sudo systemctl daemon-reload` |
| Editing files in `/lib/systemd/system/` | Use `/etc/systemd/system/` or `systemctl edit` |
| Using `restart` when `reload` is available | Check with `systemctl show --property=CanReload` |
| Not checking logs after a failure | Always: `journalctl -u service -n 50` |
| Setting `Restart=always` without `StartLimitBurst` | Service can restart-loop infinitely |
| Running services as root | Create a dedicated user with `useradd -r` |

### Administrator-Level Knowledge

- **Troubleshooting flow:** `status` → `journalctl -u X` → `cat unit file` → test manually → fix → `daemon-reload` → restart
- **Production services:** Always set `Restart=on-failure`, `RestartSec=5`, dedicated user
- **Security directives:** Use `NoNewPrivileges=true`, `ProtectSystem=strict`, `ProtectHome=true`
- **Timers vs cron:** Timers have better logging, dependency awareness, and missed-event handling
- **Boot optimization:** `systemd-analyze blame` to find and disable/optimize slow services
- **Server mode:** `systemctl set-default multi-user.target` (no GUI)

### Interview Questions

1. **What is systemd and what is its role?**
2. **Explain the difference between `start`, `enable`, `restart`, and `reload`.**
3. **How do you troubleshoot a failed service?**
4. **What are the main sections of a systemd unit file?**
5. **How do you create a custom systemd service?**
6. **What is the difference between `Requires=` and `Wants=` in a unit file?**
7. **How do you view logs for a specific service from the last hour?**
8. **What is the difference between `disable` and `mask`?**
9. **How do systemd timers compare to cron?**
10. **How do you analyze and reduce boot time?**

---

*When you've completed all the labs and practice questions, proceed to [Level 6 — Storage & Filesystems](level-06-storage-filesystems.md)*
