# Day 5 - Package Management & System Admin

## Topic

**Package Management & System Admin**

Focus areas:

- `apt`, `yum`, and `dnf`
- `dpkg` and `rpm`
- `systemctl`
- Linux services
- `cron` jobs
- Installing Node.js 20 LTS using `nvm`

---

## Learning Outcomes

By the end of Day 5, students should be able to:

- Explain the role of package managers in Linux.
- Compare `apt`, `yum`, and `dnf`.
- Understand the difference between high-level and low-level package tools.
- Explain basic `dpkg` and `rpm` internals.
- Understand how `systemd` manages services.
- Use `systemctl` to start, stop, enable, disable, and check services.
- Understand `systemd` units, targets, and dependencies.
- Write basic cron jobs using correct cron syntax.
- Install and manage a service such as Nginx.
- Install Node.js 20 LTS using `nvm`.

---

# 1. Theory

---

## 1.1 Package Managers in Linux

A **package manager** is a tool used to install, update, remove, and manage software packages in Linux.

Instead of manually downloading software from websites, Linux systems use package managers to retrieve software from trusted repositories.

A package manager usually handles:

- Downloading software packages
- Installing packages
- Removing packages
- Updating installed packages
- Resolving dependencies
- Verifying package integrity
- Managing package repositories

---

## 1.2 Why Package Managers Are Important

Package managers are important because most software depends on other libraries or tools.

For example, if you install Nginx, it may need additional libraries to run correctly. The package manager automatically checks and installs the required dependencies.

Without a package manager, the administrator would need to manually:

- Find required dependencies
- Download correct versions
- Install them in the correct order
- Track updates manually
- Remove packages safely

Package managers make system administration faster, safer, and more consistent.

---

## 1.3 apt vs yum vs dnf

Different Linux distributions use different package managers.

| Package Manager | Used In | Package Format | Example |
|---|---|---|---|
| `apt` | Debian, Ubuntu, Linux Mint | `.deb` | `sudo apt install nginx` |
| `yum` | Older RHEL, CentOS 7, Amazon Linux 2 | `.rpm` | `sudo yum install nginx` |
| `dnf` | Fedora, RHEL 8/9, Rocky Linux, AlmaLinux | `.rpm` | `sudo dnf install nginx` |

---

## 1.4 apt

`apt` stands for **Advanced Package Tool**.

It is used in Debian-based Linux distributions such as Ubuntu and Debian.

Common commands:

```bash
sudo apt update
sudo apt install nginx
sudo apt remove nginx
sudo apt upgrade
sudo apt search nginx
sudo apt show nginx
```

### Important apt Commands

| Command | Purpose |
|---|---|
| `apt update` | Refresh package repository metadata |
| `apt install package` | Install a package |
| `apt remove package` | Remove a package |
| `apt purge package` | Remove package including configuration files |
| `apt upgrade` | Upgrade installed packages |
| `apt search package` | Search for a package |
| `apt show package` | Show package details |

### Example

```bash
sudo apt update
sudo apt install nginx -y
```

---

## 1.5 yum

`yum` stands for **Yellowdog Updater Modified**.

It is used in older Red Hat-based distributions such as CentOS 7 and Amazon Linux 2.

Common commands:

```bash
sudo yum install nginx
sudo yum remove nginx
sudo yum update
sudo yum search nginx
sudo yum info nginx
```

### Important yum Commands

| Command | Purpose |
|---|---|
| `yum install package` | Install package |
| `yum remove package` | Remove package |
| `yum update` | Update system packages |
| `yum search package` | Search package |
| `yum info package` | Show package information |

---

## 1.6 dnf

`dnf` stands for **Dandified YUM**.

It is the modern replacement for `yum` and is used in Fedora, RHEL 8/9, Rocky Linux, and AlmaLinux.

Common commands:

```bash
sudo dnf install nginx
sudo dnf remove nginx
sudo dnf update
sudo dnf search nginx
sudo dnf info nginx
```

`dnf` improves dependency resolution and performance compared to older `yum`.

---

## 1.7 High-Level vs Low-Level Package Tools

Linux package management usually has two layers:

| Layer | Debian/Ubuntu | RHEL/CentOS/Fedora | Purpose |
|---|---|---|---|
| High-level tool | `apt` | `yum` / `dnf` | Handles repositories and dependencies |
| Low-level tool | `dpkg` | `rpm` | Installs or queries local package files |

High-level tools are usually preferred for daily administration because they automatically manage dependencies.

Low-level tools are useful when working directly with package files such as `.deb` or `.rpm`.

---

## 1.8 dpkg Internals

`dpkg` is the low-level package management tool for Debian-based systems.

It works directly with `.deb` packages.

Example `.deb` package:

```text
nginx_1.24.0-1_amd64.deb
```

Common `dpkg` commands:

```bash
dpkg -l
dpkg -l | grep nginx
dpkg -s nginx
dpkg -L nginx
sudo dpkg -i package.deb
sudo dpkg -r package-name
```

### Important dpkg Commands

| Command | Meaning |
|---|---|
| `dpkg -l` | List installed packages |
| `dpkg -s package` | Show package status and details |
| `dpkg -L package` | List files installed by a package |
| `dpkg -i file.deb` | Install a local `.deb` file |
| `dpkg -r package` | Remove a package |

### Important Note

`dpkg` does not automatically resolve dependencies.

For example:

```bash
sudo dpkg -i package.deb
```

If dependencies are missing, installation may fail. To fix missing dependencies, use:

```bash
sudo apt install -f
```

---

## 1.9 rpm Internals

`rpm` stands for **Red Hat Package Manager**.

It is the low-level package tool used by Red Hat-based systems.

It works directly with `.rpm` packages.

Example `.rpm` package:

```text
nginx-1.24.0-1.el9.x86_64.rpm
```

Common `rpm` commands:

```bash
rpm -qa
rpm -qa | grep nginx
rpm -qi nginx
rpm -ql nginx
sudo rpm -ivh package.rpm
sudo rpm -e package-name
```

### Important rpm Commands

| Command | Meaning |
|---|---|
| `rpm -qa` | List all installed packages |
| `rpm -qi package` | Show package information |
| `rpm -ql package` | List files installed by package |
| `rpm -ivh file.rpm` | Install local `.rpm` package |
| `rpm -Uvh file.rpm` | Upgrade package |
| `rpm -e package` | Remove package |

### Important Note

Like `dpkg`, `rpm` does not automatically resolve dependencies. For normal installation, use `yum` or `dnf`.

---

# 2. systemd and Services

---

## 2.1 What is systemd?

`systemd` is the modern Linux init system and service manager.

It is responsible for starting and managing system processes after the Linux kernel boots.

It manages:

- Services
- Startup processes
- Background daemons
- Mount points
- Timers
- Logs
- Targets
- Dependencies between services

The main command used to interact with `systemd` is:

```bash
systemctl
```

---

## 2.2 What is a Service?

A **service** is a background process that usually starts automatically and provides some functionality.

Examples:

| Service | Purpose |
|---|---|
| `nginx` | Web server |
| `ssh` | Remote login |
| `cron` | Scheduled tasks |
| `docker` | Container engine |
| `mysql` | Database server |

Services are usually managed using `systemctl`.

---

## 2.3 systemctl Commands

Common service management commands:

```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl status nginx
sudo systemctl enable nginx
sudo systemctl disable nginx
```

### Explanation

| Command | Purpose |
|---|---|
| `start` | Start service now |
| `stop` | Stop service now |
| `restart` | Stop and start service |
| `reload` | Reload config without full restart, if supported |
| `status` | Check current service status |
| `enable` | Start service automatically at boot |
| `disable` | Prevent service from starting at boot |

---

## 2.4 systemd Units

A **unit** is an object managed by `systemd`.

A unit can represent a service, socket, mount point, timer, device, or target.

Common unit types:

| Unit Type | Extension | Purpose |
|---|---|---|
| Service unit | `.service` | Manages services |
| Socket unit | `.socket` | Manages network or IPC sockets |
| Target unit | `.target` | Groups multiple units |
| Timer unit | `.timer` | Runs jobs on schedule |
| Mount unit | `.mount` | Manages mounted filesystems |

Example service unit:

```text
nginx.service
```

You can check a unit with:

```bash
systemctl status nginx.service
```

Usually, `.service` can be omitted:

```bash
systemctl status nginx
```

---

## 2.5 systemd Targets

A **target** is a group of units that represents a system state.

Targets are similar to old Linux runlevels.

Common targets:

| Target | Purpose |
|---|---|
| `multi-user.target` | Normal command-line multi-user mode |
| `graphical.target` | GUI mode |
| `rescue.target` | Rescue mode |
| `emergency.target` | Minimal emergency shell |
| `network-online.target` | Network is fully online |

Check current default target:

```bash
systemctl get-default
```

Set default target:

```bash
sudo systemctl set-default multi-user.target
```

or:

```bash
sudo systemctl set-default graphical.target
```

---

## 2.6 systemd Dependency Graph

`systemd` starts services based on dependencies.

For example, Nginx may depend on networking being available before it starts.

Common dependency directives:

| Directive | Meaning |
|---|---|
| `Requires=` | Strong dependency. If dependency fails, this unit may fail |
| `Wants=` | Weak dependency. Tries to start dependency, but does not fail if dependency fails |
| `After=` | Start this unit after another unit |
| `Before=` | Start this unit before another unit |

Example:

```ini
[Unit]
Description=Example App
After=network.target
Wants=network-online.target
```

This means the service should start after basic networking is available and should try to wait for network-online behavior.

View dependencies of a service:

```bash
systemctl list-dependencies nginx
```

View reverse dependencies:

```bash
systemctl list-dependencies --reverse nginx
```

---

## 2.7 Viewing Logs with journalctl

`journalctl` is used to view logs collected by `systemd`.

Common commands:

```bash
journalctl
journalctl -u nginx
journalctl -u nginx --no-pager
journalctl -u nginx -f
journalctl -u nginx --since "1 hour ago"
```

| Command | Purpose |
|---|---|
| `journalctl -u nginx` | View logs for Nginx |
| `journalctl -u nginx -f` | Follow live logs |
| `journalctl --since "1 hour ago"` | View recent logs |

---

# 3. Cron Jobs

---

## 3.1 What is cron?

`cron` is a Linux scheduler used to run commands automatically at specific times.

It is commonly used for:

- Backups
- Log rotation
- Health checks
- Cleanup scripts
- Report generation
- Monitoring tasks

Cron jobs are written inside a **crontab** file.

Edit the current user's crontab:

```bash
crontab -e
```

List current cron jobs:

```bash
crontab -l
```

Remove current user's crontab:

```bash
crontab -r
```

---

## 3.2 Cron Syntax

Cron uses five time fields followed by the command.

```text
* * * * * command_to_run
│ │ │ │ │
│ │ │ │ └── Day of week: 0-7
│ │ │ └──── Month: 1-12
│ │ └────── Day of month: 1-31
│ └──────── Hour: 0-23
└────────── Minute: 0-59
```

### Cron Field Meaning

| Field | Allowed Values | Meaning |
|---|---|---|
| Minute | `0-59` | Minute of the hour |
| Hour | `0-23` | Hour of the day |
| Day of month | `1-31` | Date of the month |
| Month | `1-12` | Month of the year |
| Day of week | `0-7` | Day of week, where 0 and 7 are Sunday |

---

## 3.3 Cron Examples

Run every minute:

```cron
* * * * * echo "Hello" >> /tmp/hello.log
```

Run every 5 minutes:

```cron
*/5 * * * * date >> /tmp/date.log
```

Run every day at 2:00 AM:

```cron
0 2 * * * /home/user/backup.sh
```

Run every Monday at 9:30 AM:

```cron
30 9 * * 1 /home/user/report.sh
```

Run on the first day of every month:

```cron
0 0 1 * * /home/user/monthly.sh
```

---

## 3.4 Important Cron Notes

Cron jobs run in a minimal shell environment. This means commands that work in your terminal may fail in cron if paths or environment variables are missing.

Best practices:

- Use full paths where possible.
- Redirect output to a log file.
- Make scripts executable.
- Test commands manually before adding them to cron.
- Check cron logs if the job does not run.

Example with logging:

```cron
*/5 * * * * /usr/bin/df -h >> /tmp/disk_usage.log 2>&1
```

---

# 4. Day 5 Practical Lab

---

## Lab 1: Install Nginx

### Ubuntu/Debian

Update package list:

```bash
sudo apt update
```

Install Nginx:

```bash
sudo apt install -y nginx
```

### RHEL/CentOS/Fedora

Using `dnf`:

```bash
sudo dnf install -y nginx
```

Using `yum`:

```bash
sudo yum install -y nginx
```

---

## Lab 2: Start Nginx as a Service

Start Nginx:

```bash
sudo systemctl start nginx
```

Check status:

```bash
sudo systemctl status nginx --no-pager
```

Enable Nginx to start at boot:

```bash
sudo systemctl enable nginx
```

Check if enabled:

```bash
systemctl is-enabled nginx
```

---

## Lab 3: Verify Nginx is Running

Check listening ports:

```bash
ss -tulpn | grep :80
```

Test with curl:

```bash
curl localhost
```

If Nginx is running correctly, you should see the default Nginx HTML response.

---

## Lab 4: Restart and Stop Nginx

Restart:

```bash
sudo systemctl restart nginx
```

Stop:

```bash
sudo systemctl stop nginx
```

Start again:

```bash
sudo systemctl start nginx
```

---

## Lab 5: Check Nginx Logs

View service logs:

```bash
journalctl -u nginx --no-pager
```

Follow live logs:

```bash
journalctl -u nginx -f
```

View Nginx access log:

```bash
sudo tail -f /var/log/nginx/access.log
```

View Nginx error log:

```bash
sudo tail -f /var/log/nginx/error.log
```

---

## Lab 6: Write a Cron Job That Logs Disk Usage Every 5 Minutes

Create a log directory:

```bash
mkdir -p ~/system-logs
```

Test the command manually:

```bash
df -h >> ~/system-logs/disk_usage.log
```

Check the log:

```bash
cat ~/system-logs/disk_usage.log
```

Open crontab:

```bash
crontab -e
```

Add this line:

```cron
*/5 * * * * /usr/bin/df -h >> /home/$USER/system-logs/disk_usage.log 2>&1
```

However, `$USER` may not always work as expected in cron. A safer version is to use the full username path.

Example:

```cron
*/5 * * * * /usr/bin/df -h >> /home/rohan/system-logs/disk_usage.log 2>&1
```

Check cron jobs:

```bash
crontab -l
```

After 5 minutes, verify:

```bash
cat ~/system-logs/disk_usage.log
```

---

## Lab 7: Better Disk Usage Cron Job with Timestamp

The previous cron job logs disk usage, but it does not clearly show when the command ran.

Create a script:

```bash
nano ~/disk_usage_logger.sh
```

Add:

```bash
#!/bin/bash

LOG_FILE="$HOME/system-logs/disk_usage.log"

mkdir -p "$(dirname "$LOG_FILE")"

echo "===== Disk usage report: $(date) =====" >> "$LOG_FILE"
df -h >> "$LOG_FILE"
echo "" >> "$LOG_FILE"
```

Make it executable:

```bash
chmod +x ~/disk_usage_logger.sh
```

Test manually:

```bash
~/disk_usage_logger.sh
```

Add to cron:

```bash
crontab -e
```

Cron entry:

```cron
*/5 * * * * /home/rohan/disk_usage_logger.sh
```

Replace `rohan` with your actual username.

Verify:

```bash
tail -n 30 ~/system-logs/disk_usage.log
```

---

# 5. Assignment

---

## Assignment 5: Install Node.js 20 LTS Using nvm

### Objective

Install **Node.js 20 LTS** using `nvm` and verify that the correct version is installed.

---

## 5.1 What is nvm?

`nvm` stands for **Node Version Manager**.

It allows users to install and manage multiple versions of Node.js on the same system.

This is useful because different projects may require different Node.js versions.

---

## 5.2 Install Required Dependencies

For Ubuntu/Debian:

```bash
sudo apt update
sudo apt install -y curl
```

For RHEL/CentOS/Fedora:

```bash
sudo dnf install -y curl
```

or:

```bash
sudo yum install -y curl
```

---

## 5.3 Install nvm

Run:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

Load `nvm` into the current shell:

```bash
source ~/.bashrc
```

If you use `zsh`, run:

```bash
source ~/.zshrc
```

Verify `nvm`:

```bash
nvm --version
```

---

## 5.4 Install Node.js 20 LTS

Install Node.js 20:

```bash
nvm install 20
```

Use Node.js 20:

```bash
nvm use 20
```

Set Node.js 20 as default:

```bash
nvm alias default 20
```

---

## 5.5 Verify Installation

Check Node.js version:

```bash
node -v
```

Expected output should start with:

```text
v20
```

Check npm version:

```bash
npm -v
```

Check currently active Node.js version:

```bash
nvm current
```

---

## 5.6 Assignment Submission Format

Students should submit:

```text
Node.js version:
npm version:
nvm version:
Command used to install Node.js:
Screenshot or terminal output:
```

Example:

```text
Node.js version: v20.x.x
npm version: 10.x.x
nvm version: 0.40.3
Command used: nvm install 20
```

---

# 6. Quick Command Summary

## Package Management

```bash
sudo apt update
sudo apt install -y nginx
sudo apt remove nginx
dpkg -l | grep nginx
```

```bash
sudo dnf install -y nginx
sudo dnf remove nginx
rpm -qa | grep nginx
```

---

## Service Management

```bash
sudo systemctl start nginx
sudo systemctl status nginx
sudo systemctl enable nginx
sudo systemctl restart nginx
sudo systemctl stop nginx
journalctl -u nginx --no-pager
```

---

## Cron

```bash
crontab -e
crontab -l
*/5 * * * * /usr/bin/df -h >> /home/rohan/system-logs/disk_usage.log 2>&1
```

---

## Node.js with nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm install 20
nvm use 20
nvm alias default 20
node -v
npm -v
```

---

# 7. Common Beginner Mistakes

- Running `apt install` without `sudo`.
- Forgetting to run `sudo apt update` before installing packages.
- Confusing `start` and `enable` in `systemctl`.
- Thinking `systemctl enable nginx` starts the service immediately.
- Forgetting to use full paths in cron jobs.
- Writing invalid cron syntax.
- Forgetting to make scripts executable with `chmod +x`.
- Installing Node.js with `apt` when the assignment asks for `nvm`.
- Forgetting to reload the shell after installing `nvm`.

---

# 8. End-of-Day Review Questions

1. What is the difference between `apt`, `yum`, and `dnf`?
2. What is the difference between `apt` and `dpkg`?
3. What is the difference between `dnf` and `rpm`?
4. What does `systemctl enable nginx` do?
5. What does `systemctl start nginx` do?
6. What is a `systemd` unit?
7. What is a `systemd` target?
8. What does this cron expression mean?

```cron
*/5 * * * *
```

9. Why should cron jobs use full paths?
10. How do you verify that Node.js 20 is installed?

---

# 9. Expected Practical Output

By the end of the lab, students should have:

- Nginx installed.
- Nginx started and enabled.
- Nginx status checked using `systemctl`.
- Disk usage logging every 5 minutes using cron.
- Node.js 20 LTS installed using `nvm`.
- Verified versions of `node`, `npm`, and `nvm`.
