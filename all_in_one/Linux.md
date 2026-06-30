# Linux — Zero to Hero

> A complete, beginner-to-advanced guide to Linux for DevOps engineers, system administrators, and developers.

---

## Table of Contents

1. [Introduction & Theory](#1-introduction--theory)
2. [Installation & Setup](#2-installation--setup)
3. [Core Concepts](#3-core-concepts)
4. [Hands-on Tasks](#4-hands-on-tasks)
5. [Projects](#5-projects)
6. [Best Practices & Common Pitfalls](#6-best-practices--common-pitfalls)
7. [Interview Questions](#7-interview-questions)
8. [Quizzes](#8-quizzes)
9. [Further Resources](#9-further-resources)

---

## 1. Introduction & Theory

### 1.1 What is Linux?

Linux is a family of **free and open-source, Unix-like operating systems** built around the **Linux kernel**, first released by Linus Torvalds in 1991. Strictly speaking, "Linux" refers to the kernel — the core program that manages hardware, memory, processes, and devices. A usable system combines the kernel with system utilities (often from the GNU project), libraries, a shell, and applications. This combination is packaged and distributed as a **Linux distribution** (distro).

Linux powers the overwhelming majority of the world's infrastructure:

- **Servers & cloud:** ~90%+ of public cloud workloads and nearly all of the top supercomputers.
- **Containers:** Docker and Kubernetes are fundamentally Linux technologies.
- **Embedded & mobile:** Android is built on the Linux kernel; routers, IoT, automotive systems use Linux.
- **Development:** Most CI/CD pipelines, build servers, and DevOps tooling run on Linux.

### 1.2 Why Linux matters for DevOps

Almost every DevOps tool (Docker, Kubernetes, Jenkins, Terraform, Ansible) assumes Linux fundamentals: the filesystem hierarchy, permissions, processes, networking, and the shell. Mastery of Linux is the **single highest-leverage skill** in DevOps because it underpins everything else.

| Reason | Explanation |
|--------|-------------|
| Ubiquity | Production servers are overwhelmingly Linux. |
| Automation | The shell + scripting enable repeatable automation. |
| Transparency | Open source means you can inspect and debug anything. |
| Stability | Long uptimes, predictable behavior, fine-grained control. |
| Cost | No licensing fees for most distributions. |

### 1.3 Architecture overview

Linux uses a **layered architecture**:

```
+--------------------------------------------------+
|              User Applications                   |  (bash, nginx, python, vim)
+--------------------------------------------------+
|        System Libraries (glibc, libc)            |  (standard C library, APIs)
+--------------------------------------------------+
|              System Call Interface               |  (open, read, write, fork, exec)
+--------------------------------------------------+
|                 Linux Kernel                     |
|  - Process scheduler   - Memory management       |
|  - Virtual filesystem  - Networking stack        |
|  - Device drivers      - Security (SELinux)      |
+--------------------------------------------------+
|                   Hardware                       |  (CPU, RAM, disk, network)
+--------------------------------------------------+
```

Key kernel responsibilities:

- **Process management:** Creating, scheduling, and terminating processes; context switching.
- **Memory management:** Virtual memory, paging, swapping, the page cache.
- **Filesystem:** A unified Virtual File System (VFS) abstraction over ext4, XFS, Btrfs, etc.
- **Device drivers:** Interface between hardware and software.
- **Networking:** TCP/IP stack, sockets, routing, firewalling (netfilter).
- **Security:** Permissions, capabilities, namespaces, cgroups, SELinux/AppArmor.

### 1.4 User space vs. kernel space

- **Kernel space:** Privileged execution mode where the kernel runs with full hardware access (ring 0).
- **User space:** Restricted mode where applications run (ring 3). Applications request privileged operations through **system calls**, which transition into kernel space safely.

This separation is what makes Linux robust: a crashing application cannot directly corrupt the kernel or other processes.

### 1.5 Popular distributions

| Family | Distros | Package Manager | Typical Use |
|--------|---------|-----------------|-------------|
| Debian | Debian, Ubuntu, Linux Mint | `apt` / `dpkg` | Servers, desktops, cloud |
| Red Hat | RHEL, CentOS Stream, Fedora, Rocky, Alma | `dnf` / `yum` / `rpm` | Enterprise servers |
| SUSE | openSUSE, SLES | `zypper` | Enterprise |
| Arch | Arch, Manjaro | `pacman` | Rolling release, power users |
| Alpine | Alpine Linux | `apk` | Containers (tiny footprint) |

For DevOps, **Ubuntu Server** and **RHEL-family (Rocky/Alma)** are the most common, while **Alpine** dominates container base images.

### 1.6 When to use Linux

- Hosting web servers, databases, application backends.
- Running containers and orchestration platforms.
- Building CI/CD pipelines and automation.
- Edge/IoT and embedded systems.
- Any time you need fine-grained, scriptable control over a system.

When **not** to choose Linux: scenarios requiring Windows-only software (certain enterprise desktop apps, specific GUI tooling), or hardware lacking Linux driver support.

---

## 2. Installation & Setup

### 2.1 Ways to run Linux

1. **Virtual Machine (recommended for learning):** VirtualBox, VMware, or KVM. Safe, isolated, snapshot-able.
2. **WSL2 (Windows Subsystem for Linux):** Run Ubuntu directly inside Windows. Great for development.
3. **Cloud instance:** AWS EC2, Azure VM, GCP Compute Engine — closest to production.
4. **Bare metal:** Install directly on hardware (dual-boot or dedicated machine).
5. **Containers:** `docker run -it ubuntu bash` for a throwaway environment.

### 2.2 Installing Ubuntu Server in VirtualBox (step-by-step)

```bash
# 1. Download the ISO
#    https://ubuntu.com/download/server  (e.g., ubuntu-24.04-live-server-amd64.iso)

# 2. In VirtualBox:
#    - Create New VM: Type=Linux, Version=Ubuntu (64-bit)
#    - RAM: 2048 MB minimum (4096 recommended)
#    - Disk: 20 GB dynamically allocated
#    - Attach the ISO under Settings > Storage > Optical Drive

# 3. Boot the VM and follow the installer:
#    - Choose language, keyboard
#    - Configure network (DHCP is fine for learning)
#    - Set up storage (use entire disk)
#    - Create a user account
#    - Enable "Install OpenSSH server" when prompted
```

### 2.3 Enabling WSL2 on Windows

```powershell
# Run in an elevated PowerShell
wsl --install                 # installs WSL2 + Ubuntu by default
wsl --list --online           # see available distros
wsl --install -d Ubuntu-24.04 # install a specific distro
wsl --set-default-version 2   # ensure WSL2 (not WSL1)
```

### 2.4 First-boot setup checklist

```bash
# Update the package index and upgrade installed packages
sudo apt update && sudo apt upgrade -y

# Install essential tooling
sudo apt install -y vim git curl wget htop net-tools build-essential unzip tree

# Verify versions
uname -a            # kernel and architecture
lsb_release -a      # distribution details
whoami              # current user
```

Expected `uname -a` output (example):

```
Linux ubuntu-server 6.8.0-31-generic #31-Ubuntu SMP x86_64 GNU/Linux
```

### 2.5 Configuring SSH access

```bash
# On the server: ensure SSH is running
sudo systemctl status ssh
sudo systemctl enable --now ssh

# On your client: generate a key pair (if you don't have one)
ssh-keygen -t ed25519 -C "you@example.com"

# Copy your public key to the server
ssh-copy-id user@server-ip

# Connect
ssh user@server-ip
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 The shell

The **shell** is a command interpreter. The most common is **Bash** (Bourne Again Shell). When you open a terminal, you get a prompt:

```
user@hostname:~$
```

- `user` — your username
- `hostname` — the machine name
- `~` — current directory (home)
- `$` — regular user (`#` indicates root)

#### 3.1.2 Filesystem Hierarchy Standard (FHS)

Linux organizes everything under a single root `/`:

| Path | Purpose |
|------|---------|
| `/` | Root of the filesystem |
| `/bin`, `/usr/bin` | Essential user binaries (ls, cp, cat) |
| `/sbin`, `/usr/sbin` | System administration binaries |
| `/etc` | System-wide configuration files |
| `/home` | User home directories |
| `/root` | Root user's home |
| `/var` | Variable data: logs, mail, spool |
| `/var/log` | Log files |
| `/tmp` | Temporary files (cleared on reboot) |
| `/opt` | Optional/third-party software |
| `/usr` | User programs and data |
| `/lib`, `/usr/lib` | Shared libraries |
| `/dev` | Device files (disks, terminals) |
| `/proc` | Virtual filesystem exposing kernel/process info |
| `/sys` | Virtual filesystem for device/kernel parameters |
| `/mnt`, `/media` | Mount points for external filesystems |
| `/boot` | Kernel and bootloader files |

#### 3.1.3 Navigation commands

```bash
pwd                 # print working directory
ls                  # list files
ls -l               # long format (permissions, owner, size, date)
ls -la              # include hidden files (dotfiles)
ls -lh              # human-readable sizes
cd /etc             # change directory (absolute path)
cd ..               # go up one level
cd ~                # go to home directory
cd -                # go to previous directory
tree                # show directory tree
```

#### 3.1.4 Working with files and directories

```bash
touch file.txt              # create empty file / update timestamp
mkdir mydir                 # create directory
mkdir -p a/b/c              # create nested directories
cp file.txt copy.txt        # copy file
cp -r dir1 dir2             # copy directory recursively
mv file.txt /tmp/           # move (or rename) file
mv old.txt new.txt          # rename
rm file.txt                 # delete file
rm -r mydir                 # delete directory recursively
rm -rf mydir                # force delete (DANGEROUS)
rmdir emptydir              # remove empty directory
```

#### 3.1.5 Viewing file content

```bash
cat file.txt        # print whole file
less file.txt       # page through (q to quit)
head -n 20 file.txt # first 20 lines
tail -n 20 file.txt # last 20 lines
tail -f app.log     # follow a log in real time
wc -l file.txt      # count lines
nl file.txt         # number lines
```

#### 3.1.6 Getting help

```bash
man ls              # manual page for ls
ls --help           # quick help
type ls             # what kind of command is it
which python3       # path of an executable
apropos network     # search man pages by keyword
```

### 3.2 INTERMEDIATE

#### 3.2.1 File permissions

Every file has an **owner**, a **group**, and a permission set for **owner / group / others**:

```
-rwxr-xr--  1 alice devs  4096 Jun 30 10:00 script.sh
│└┬┘└┬┘└┬┘
│ │  │  └── others: read only (r--)
│ │  └───── group:  read + execute (r-x)
│ └──────── owner:  read + write + execute (rwx)
└────────── file type (- file, d directory, l link)
```

Permission values:

| Symbol | Value | Meaning |
|--------|-------|---------|
| `r` | 4 | read |
| `w` | 2 | write |
| `x` | 1 | execute |

```bash
chmod 755 script.sh     # rwx r-x r-x
chmod u+x script.sh     # add execute for owner
chmod g-w file.txt      # remove write for group
chmod -R 644 dir/       # recursive
chown alice file.txt    # change owner
chown alice:devs file   # change owner and group
chgrp devs file.txt     # change group
umask                   # default permission mask
```

#### 3.2.2 Users and groups

```bash
sudo adduser bob              # create user (interactive)
sudo useradd -m -s /bin/bash bob   # create user (scripted)
sudo passwd bob              # set password
sudo usermod -aG sudo bob    # add bob to sudo group
sudo deluser bob             # delete user
id bob                       # show uid, gid, groups
groups bob                   # list groups
cat /etc/passwd              # user accounts
cat /etc/group               # groups
cat /etc/shadow              # hashed passwords (root only)
su - bob                     # switch user
sudo -i                      # become root
```

#### 3.2.3 Process management

```bash
ps aux                  # all running processes
ps -ef                  # alternative format
top                     # live process viewer
htop                    # enhanced interactive viewer
pgrep nginx             # find PIDs by name
kill 1234               # send SIGTERM to PID 1234
kill -9 1234            # SIGKILL (force)
killall nginx           # kill by name
nice -n 10 ./task.sh    # start with lower priority
renice -n 5 -p 1234     # change priority of running process
jobs                    # background jobs in this shell
bg                      # resume job in background
fg                      # bring job to foreground
nohup ./long.sh &       # run detached from terminal
```

Signals you should know:

| Signal | Number | Meaning |
|--------|--------|---------|
| SIGTERM | 15 | Graceful termination (default) |
| SIGKILL | 9 | Force kill (cannot be caught) |
| SIGHUP | 1 | Hangup / reload config |
| SIGINT | 2 | Interrupt (Ctrl+C) |
| SIGSTOP | 19 | Pause process |
| SIGCONT | 18 | Resume process |

#### 3.2.4 I/O redirection and pipes

```bash
command > file          # redirect stdout (overwrite)
command >> file         # redirect stdout (append)
command 2> errors.log   # redirect stderr
command > out 2>&1      # redirect both stdout and stderr
command &> all.log      # shorthand for both
command < input.txt     # read stdin from file
cmd1 | cmd2             # pipe stdout of cmd1 into cmd2
cmd | tee file          # write to file AND stdout
```

Example:

```bash
ps aux | grep nginx | awk '{print $2}' | head -5
```

#### 3.2.5 Text processing: grep, sed, awk

```bash
# grep — search text
grep "error" app.log
grep -i "error" app.log         # case-insensitive
grep -r "TODO" src/             # recursive
grep -n "main" file.c           # show line numbers
grep -v "debug" app.log         # invert match
grep -E "warn|error" app.log    # extended regex

# sed — stream editor
sed 's/foo/bar/' file.txt       # replace first occurrence per line
sed 's/foo/bar/g' file.txt      # replace all
sed -i 's/foo/bar/g' file.txt   # edit file in place
sed -n '10,20p' file.txt        # print lines 10-20
sed '/^#/d' config              # delete comment lines

# awk — field processing
awk '{print $1}' file           # first column
awk -F: '{print $1}' /etc/passwd  # custom delimiter
awk '$3 > 1000 {print $1}' data # conditional
awk '{sum+=$1} END {print sum}' nums.txt  # sum a column
```

#### 3.2.6 Finding files

```bash
find / -name "*.conf"               # by name
find /var/log -type f -size +100M   # files over 100MB
find . -mtime -7                    # modified in last 7 days
find . -name "*.tmp" -delete        # find and delete
find . -type f -exec chmod 644 {} \;  # run command per match
locate nginx.conf                   # fast DB-based search (updatedb)
which docker                        # locate an executable in PATH
```

#### 3.2.7 Archiving and compression

```bash
tar -cvf archive.tar dir/       # create tar
tar -xvf archive.tar            # extract tar
tar -czvf archive.tar.gz dir/   # create gzip-compressed tar
tar -xzvf archive.tar.gz        # extract gzip tar
tar -cjvf archive.tar.bz2 dir/  # bzip2 compression
gzip file.txt                   # compress -> file.txt.gz
gunzip file.txt.gz              # decompress
zip -r archive.zip dir/         # zip
unzip archive.zip               # unzip
```

#### 3.2.8 Package management

```bash
# Debian/Ubuntu (apt)
sudo apt update                 # refresh package index
sudo apt upgrade                # upgrade installed packages
sudo apt install nginx          # install
sudo apt remove nginx           # remove (keep config)
sudo apt purge nginx            # remove + config
sudo apt autoremove             # remove orphaned deps
apt search keyword              # search
apt show nginx                  # package details
dpkg -l                         # list installed
dpkg -i package.deb             # install local .deb

# RHEL/Fedora (dnf)
sudo dnf install nginx
sudo dnf remove nginx
sudo dnf update
dnf search keyword
rpm -qa                         # list installed
```

### 3.3 ADVANCED

#### 3.3.1 systemd and service management

`systemd` is the init system and service manager on most modern distros (PID 1).

```bash
systemctl status nginx          # service status
sudo systemctl start nginx      # start
sudo systemctl stop nginx       # stop
sudo systemctl restart nginx    # restart
sudo systemctl reload nginx     # reload config (no downtime)
sudo systemctl enable nginx     # start at boot
sudo systemctl disable nginx    # don't start at boot
sudo systemctl enable --now nginx  # enable + start
systemctl list-units --type=service   # list services
systemctl is-active nginx       # active?
systemctl is-enabled nginx      # enabled?
journalctl -u nginx             # service logs
journalctl -u nginx -f          # follow logs
journalctl --since "1 hour ago"
journalctl -p err -b            # errors since boot
```

A custom unit file (`/etc/systemd/system/myapp.service`):

```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload    # reload unit definitions
sudo systemctl enable --now myapp
```

#### 3.3.2 Networking

```bash
ip addr                         # show interfaces and IPs
ip route                        # routing table
ip link set eth0 up             # bring interface up
ping -c 4 google.com            # connectivity test
traceroute google.com           # path to host
ss -tulpn                       # listening sockets (modern)
netstat -tulpn                  # listening sockets (legacy)
curl -I https://example.com     # HTTP headers
curl ifconfig.me                # public IP
wget https://example.com/file   # download
dig example.com                 # DNS lookup
nslookup example.com            # DNS lookup
nc -zv host 22                  # test if port open
cat /etc/hosts                  # static host entries
cat /etc/resolv.conf            # DNS resolvers
```

#### 3.3.3 Firewall (ufw / firewalld / iptables)

```bash
# ufw (Ubuntu)
sudo ufw enable
sudo ufw allow 22/tcp
sudo ufw allow 80,443/tcp
sudo ufw deny 3306
sudo ufw status verbose
sudo ufw delete allow 80

# firewalld (RHEL)
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

#### 3.3.4 Disk and filesystem management

```bash
df -h                   # disk usage by filesystem
du -sh /var/log         # size of a directory
du -ah . | sort -rh | head  # largest items
lsblk                   # block devices tree
fdisk -l                # partition tables
mount /dev/sdb1 /mnt    # mount a filesystem
umount /mnt             # unmount
blkid                   # show UUIDs
free -h                 # memory usage
cat /etc/fstab          # persistent mounts
```

Adding a persistent mount in `/etc/fstab`:

```
UUID=xxxx-xxxx  /data  ext4  defaults  0  2
```

#### 3.3.5 Environment variables and the PATH

```bash
echo $PATH              # executable search path
export MY_VAR=value     # set for current session + children
env                     # list environment variables
unset MY_VAR            # remove variable
printenv HOME           # print one variable
```

Persist variables in `~/.bashrc`, `~/.profile`, or `/etc/environment`.

#### 3.3.6 Cron and scheduled tasks

```bash
crontab -e              # edit current user's crontab
crontab -l              # list jobs
```

Crontab syntax:

```
# ┌───── minute (0-59)
# │ ┌───── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌───── month (1-12)
# │ │ │ │ ┌───── day of week (0-6, Sun=0)
# │ │ │ │ │
  0 2 * * *  /opt/backup.sh        # every day at 02:00
  */15 * * * * /opt/check.sh       # every 15 minutes
  0 0 * * 0  /opt/weekly.sh        # every Sunday midnight
```

For modern systems, `systemd timers` are an alternative to cron.

#### 3.3.7 Shell scripting

```bash
#!/usr/bin/env bash
set -euo pipefail        # strict mode: exit on error, unset var, pipe failure

# Variables
NAME="World"
COUNT=5

# Conditionals
if [[ -f /etc/passwd ]]; then
  echo "passwd exists"
fi

# Loops
for i in {1..5}; do
  echo "iteration $i"
done

while read -r line; do
  echo "Line: $line"
done < input.txt

# Functions
greet() {
  local who="$1"
  echo "Hello, $who"
}
greet "$NAME"

# Command substitution
NOW=$(date +%F)
echo "Today is $NOW"

# Exit codes
if grep -q "error" app.log; then
  echo "Found errors"; exit 1
fi
```

Run and debug:

```bash
chmod +x script.sh
./script.sh
bash -x script.sh       # trace execution
shellcheck script.sh    # static analysis (lint)
```

#### 3.3.8 Performance and troubleshooting

```bash
uptime                  # load average
vmstat 1                # virtual memory stats
iostat -x 1             # disk I/O stats
sar -u 1 5              # CPU usage samples
free -m                 # memory in MB
lscpu                   # CPU details
dmesg | tail            # kernel ring buffer (hardware/driver issues)
strace -p 1234          # trace syscalls of a process
lsof -i :80             # what is using port 80
lsof -p 1234            # files opened by a process
```

#### 3.3.9 SELinux / AppArmor (mandatory access control)

```bash
# SELinux (RHEL)
getenforce                      # Enforcing / Permissive / Disabled
sudo setenforce 0               # temporarily permissive
sestatus                        # detailed status
ls -Z file                      # security context

# AppArmor (Ubuntu)
sudo aa-status
sudo aa-complain /etc/apparmor.d/profile
sudo aa-enforce /etc/apparmor.d/profile
```

---

## 4. Hands-on Tasks

> Each task lists steps, the exact command, and expected output.

### Task 1: Navigate and inspect the filesystem

```bash
cd /
ls -la
cd /var/log
ls -lh
```

**Expected:** You see directories like `bin`, `etc`, `home`, `var`; in `/var/log` you see files such as `syslog`, `auth.log`.

### Task 2: Create a directory structure and files

```bash
mkdir -p ~/lab/{src,bin,docs}
touch ~/lab/src/main.py ~/lab/docs/README.md
tree ~/lab
```

**Expected:**

```
/home/user/lab
├── bin
├── docs
│   └── README.md
└── src
    └── main.py
```

### Task 3: Set permissions on a script

```bash
echo '#!/bin/bash' > ~/lab/bin/run.sh
echo 'echo "Hello from script"' >> ~/lab/bin/run.sh
chmod 755 ~/lab/bin/run.sh
~/lab/bin/run.sh
```

**Expected:** `Hello from script`

### Task 4: Create a user and assign sudo

```bash
sudo adduser devops
sudo usermod -aG sudo devops
id devops
```

**Expected:** `id devops` shows membership including the `sudo` group.

### Task 5: Find and kill a process

```bash
sleep 600 &
pgrep sleep
kill $(pgrep sleep)
pgrep sleep || echo "process gone"
```

**Expected:** Final line prints `process gone`.

### Task 6: Pipe and filter text

```bash
ps aux --sort=-%mem | head -6
cat /etc/passwd | awk -F: '{print $1}' | sort | head
```

**Expected:** Top memory-consuming processes; sorted list of usernames.

### Task 7: Search logs with grep

```bash
sudo grep -i "fail" /var/log/auth.log | tail -5
```

**Expected:** Last five lines containing "fail" (failed logins, sudo failures, etc.).

### Task 8: Schedule a cron job

```bash
( crontab -l 2>/dev/null; echo "* * * * * date >> ~/cron-test.log" ) | crontab -
sleep 65
cat ~/cron-test.log
crontab -r        # cleanup
```

**Expected:** `~/cron-test.log` contains at least one timestamp line.

### Task 9: Create and manage a systemd service

```bash
sudo tee /etc/systemd/system/hello.service >/dev/null <<'EOF'
[Unit]
Description=Hello loop
[Service]
ExecStart=/bin/bash -c 'while true; do echo hello; sleep 10; done'
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now hello
systemctl status hello
journalctl -u hello -n 5
```

**Expected:** Service `active (running)`; logs show repeated `hello`.

### Task 10: Inspect networking

```bash
ip addr show
ss -tulpn | head
curl -s ifconfig.me; echo
```

**Expected:** Interface IPs, listening ports, and your public IP.

---

## 5. Projects

### Project 1 (Beginner): Automated System Backup Script

**Goal:** Write a Bash script that backs up a directory, compresses it, timestamps it, and rotates old backups.

```bash
#!/usr/bin/env bash
set -euo pipefail

SRC="${1:-$HOME/lab}"
DEST="${2:-$HOME/backups}"
RETENTION_DAYS=7

mkdir -p "$DEST"
STAMP=$(date +%Y%m%d_%H%M%S)
ARCHIVE="$DEST/backup_$STAMP.tar.gz"

echo "Backing up $SRC -> $ARCHIVE"
tar -czf "$ARCHIVE" -C "$(dirname "$SRC")" "$(basename "$SRC")"

echo "Removing backups older than $RETENTION_DAYS days"
find "$DEST" -name "backup_*.tar.gz" -mtime +"$RETENTION_DAYS" -delete

echo "Done. Current backups:"
ls -lh "$DEST"
```

**Extensions:** Add email notification, log to a file, schedule via cron, upload to S3.

### Project 2 (Intermediate): System Health Monitoring Dashboard

**Goal:** A script that collects CPU, memory, disk, and top processes, then writes an HTML report served by a simple web server.

```bash
#!/usr/bin/env bash
set -euo pipefail
OUT=/var/www/html/health.html

{
  echo "<html><body><h1>Health Report $(date)</h1><pre>"
  echo "=== Uptime/Load ==="; uptime
  echo "=== Memory ===";      free -h
  echo "=== Disk ===";        df -h
  echo "=== Top Processes ==="; ps aux --sort=-%cpu | head -6
  echo "</pre></body></html>"
} > "$OUT"
```

Schedule with cron every minute and serve with `python3 -m http.server` or nginx. **Extensions:** alert when disk > 80%, add CPU temperature, push metrics to a file consumed by Grafana.

### Project 3 (Advanced): Hardened Multi-User Web Server

**Goal:** Provision a secure Ubuntu server hosting a website with proper users, firewall, SSH hardening, and a systemd-managed app.

Steps:

1. Create a dedicated non-root user; disable root SSH login (`PermitRootLogin no` in `/etc/ssh/sshd_config`).
2. Enforce key-based auth (`PasswordAuthentication no`).
3. Configure `ufw` to allow only 22, 80, 443.
4. Install and configure nginx; deploy a static or proxied app.
5. Run the backend as a systemd service with `Restart=on-failure` and a dedicated unprivileged user.
6. Enable automatic security updates (`unattended-upgrades`).
7. Set up `fail2ban` to block brute-force SSH attempts.
8. Add a cron-based backup (Project 1) and a health monitor (Project 2).

**Deliverable:** A documented runbook plus the scripts/config that reproduce the setup.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Principle of least privilege:** Don't run as root; use `sudo` for specific commands. Run services as dedicated users.
- **Always `set -euo pipefail`** in production Bash scripts and quote variables (`"$var"`).
- **Use absolute paths** in cron jobs and systemd units; their environment differs from your shell.
- **Back up before destructive operations**, and test restores.
- **Prefer `ss`, `ip`, `journalctl`** over the deprecated `netstat`, `ifconfig`, syslog-only workflows.
- **Document and version-control** your configuration (Infrastructure as Code).
- **Keep systems patched** with automatic security updates.
- **Use `shellcheck`** to lint scripts and `bash -n` to check syntax.
- **Read logs first** (`journalctl`, `/var/log`) when troubleshooting.

### Common Pitfalls

| Pitfall | Why it hurts | Fix |
|---------|-------------|-----|
| `rm -rf /` or unquoted `rm -rf $VAR/` | Catastrophic data loss | Quote vars; double-check paths; use `--preserve-root` |
| Editing files as root carelessly | Breaks boot/services | Make backups; use `visudo` for sudoers |
| Forgetting `chmod +x` | "Permission denied" running scripts | `chmod +x script.sh` |
| Relying on `$PATH` in cron | Commands not found | Use absolute paths |
| Not quoting variables | Word-splitting/glob bugs | Always `"$var"` |
| Ignoring SELinux/AppArmor | Mysterious "permission denied" | Check `getenforce`/`aa-status`, audit logs |
| Filling `/` with logs | Services crash | Monitor with `df -h`; set up log rotation |
| Killing with `-9` first | Corrupts state, skips cleanup | Try SIGTERM (default) before SIGKILL |

---

## 7. Interview Questions

### Beginner

1. **What is the difference between Linux and a Linux distribution?**
   *Linux is the kernel; a distribution bundles the kernel with utilities, libraries, a package manager, and applications (e.g., Ubuntu, RHEL).*

2. **What does `chmod 755` mean?**
   *Owner gets read+write+execute (7), group and others get read+execute (5). Common for executable scripts/directories.*

3. **How do you view the last 20 lines of a log and follow it live?**
   *`tail -n 20 file.log` and `tail -f file.log`.*

4. **What is the difference between absolute and relative paths?**
   *Absolute starts from root `/` (e.g., `/etc/hosts`); relative is from the current directory (e.g., `./config`).*

5. **How do you find which directory you are in?**
   *`pwd`.*

### Intermediate

6. **Explain the difference between a hard link and a symbolic link.**
   *A hard link is another directory entry pointing to the same inode (same data; survives deletion of the original). A symlink is a pointer to a path (breaks if the target is removed; can cross filesystems).*

7. **How does the Linux boot process work at a high level?**
   *BIOS/UEFI → bootloader (GRUB) → kernel loads + initramfs → `systemd` (PID 1) → target/services → login.*

8. **What is the difference between `SIGTERM` and `SIGKILL`?**
   *SIGTERM (15) requests graceful shutdown and can be caught/handled; SIGKILL (9) forcibly terminates and cannot be caught or ignored.*

9. **How do you check what process is listening on port 8080?**
   *`sudo ss -tulpn | grep :8080` or `sudo lsof -i :8080`.*

10. **What does `set -euo pipefail` do in a script?**
    *`-e` exit on error, `-u` error on unset variables, `-o pipefail` makes a pipeline fail if any stage fails.*

### Advanced

11. **Explain namespaces and cgroups and their role in containers.**
    *Namespaces isolate resources (PID, network, mount, UTS, IPC, user) so processes see a private view; cgroups limit/account resource usage (CPU, memory, I/O). Together they underpin containerization.*

12. **A server has high load average but low CPU usage — what's happening?**
    *Likely I/O wait (processes blocked on disk/network) or many processes in uninterruptible sleep (D state). Investigate with `iostat`, `vmstat`, `top` (wa%), and `ps` for D-state processes.*

13. **How would you debug a service that fails to start under systemd?**
    *`systemctl status svc`, `journalctl -u svc -n 50`, validate the unit file, check `ExecStart` path/permissions, user, working directory, dependencies (`After=`), and run the command manually.*

14. **What is the difference between the page cache and swap?**
    *Page cache holds file data in free RAM to speed up I/O (reclaimable). Swap is disk space used to evict anonymous memory pages when RAM is under pressure (much slower).*

15. **How do you securely harden SSH on a production server?**
    *Disable root login and password auth, use key-based auth, change/limit access, use `fail2ban`, restrict users (`AllowUsers`), keep updated, and firewall the port.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Which command shows the current working directory?
- A) `cwd`  B) `pwd`  C) `dir`  D) `path`

**Q2.** What permission value gives read + write + execute?
- A) 5  B) 6  C) 7  D) 4

**Q3.** Which file stores user account information?
- A) `/etc/shadow`  B) `/etc/passwd`  C) `/etc/group`  D) `/etc/users`

**Q4.** Which signal cannot be caught or ignored?
- A) SIGTERM  B) SIGHUP  C) SIGINT  D) SIGKILL

**Q5.** Which command follows a log file in real time?
- A) `cat -f`  B) `tail -f`  C) `head -f`  D) `less -f`

**Q6.** What does `2>&1` do?
- A) Redirect stdin to stdout  B) Redirect stderr to stdout  C) Redirect stdout to a file  D) Duplicate a file descriptor 2 times

**Q7.** Which init system manages services on most modern distros?
- A) SysVinit  B) Upstart  C) systemd  D) OpenRC

**Q8.** Which command lists listening TCP/UDP sockets?
- A) `ss -tulpn`  B) `ps aux`  C) `df -h`  D) `du -sh`

**Q9.** Where are system-wide configuration files stored?
- A) `/var`  B) `/etc`  C) `/opt`  D) `/usr`

**Q10.** Which package manager is used by Ubuntu?
- A) `yum`  B) `dnf`  C) `apt`  D) `pacman`

### Short Answer

**S1.** Write a command to recursively change ownership of `/srv/app` to user `web` and group `web`.

**S2.** What command shows the kernel version and architecture?

**S3.** Write a cron entry that runs `/opt/clean.sh` every day at 3:30 AM.

**S4.** How do you make a Bash script executable and run it?

**S5.** What command would you use to display the top memory-consuming processes?

### Answer Key

**Multiple Choice:** Q1-B, Q2-C, Q3-B, Q4-D, Q5-B, Q6-B, Q7-C, Q8-A, Q9-B, Q10-C

**Short Answer:**
- **S1.** `sudo chown -R web:web /srv/app`
- **S2.** `uname -a` (or `uname -r` for version, `uname -m` for architecture)
- **S3.** `30 3 * * * /opt/clean.sh`
- **S4.** `chmod +x script.sh && ./script.sh`
- **S5.** `ps aux --sort=-%mem | head` (or `top`/`htop`)

---

## 9. Further Resources

### Official Documentation
- The Linux Documentation Project — https://tldp.org
- man7.org manual pages — https://man7.org/linux/man-pages/
- Ubuntu Server Guide — https://ubuntu.com/server/docs
- Red Hat Documentation — https://access.redhat.com/documentation

### Books
- *The Linux Command Line* — William Shotts (free PDF at linuxcommand.org)
- *How Linux Works* — Brian Ward
- *UNIX and Linux System Administration Handbook* — Nemeth et al.
- *Linux Bible* — Christopher Negus

### Interactive Practice
- OverTheWire: Bandit — https://overthewire.org/wargames/bandit/ (great for shell skills)
- Linux Journey — https://linuxjourney.com
- KillerCoda / Katacoda-style scenarios
- explainshell.com — paste any command to see what each part does

### Certifications
- CompTIA Linux+
- LPIC-1 / LPIC-2 (Linux Professional Institute)
- Red Hat Certified System Administrator (RHCSA) / Engineer (RHCE)

### Cheatsheets & Tools
- `tldr` pages (`tldr tar`) — simplified examples
- ShellCheck — https://www.shellcheck.net
- DevHints Bash cheatsheet — https://devhints.io/bash

---

*End of Linux — Zero to Hero.*
