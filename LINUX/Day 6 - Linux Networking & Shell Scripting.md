# Day 6 - Linux Networking & Shell Scripting

## Topic

**Linux Networking & Shell Scripting**

Focus areas:

- Linux networking basics
- IP address, subnet, gateway, and DNS
- `/etc/hosts` and `/etc/resolv.conf`
- Network troubleshooting tools
- Bash scripting fundamentals
- Variables, environment variables, exit codes
- Conditions, loops, functions, and arrays
- Web server healthcheck scripting
- Deployment automation script

---

## Learning Outcomes

By the end of Day 6, students should be able to:

- Explain basic Linux networking concepts.
- Understand IP address, subnet, gateway, and DNS.
- Read and understand `/etc/hosts` and `/etc/resolv.conf`.
- Use networking tools such as `ping`, `curl`, `wget`, `ss`, `netstat`, `traceroute`, and `nmap`.
- Write basic Bash scripts using variables and environment variables.
- Understand `$?` and exit codes.
- Use `if/else`, `for` loops, `while` loops, arrays, and functions.
- Write a web server healthcheck script.
- Use `curl` to test HTTP endpoints and status codes.
- Write a basic deployment script that pulls code, installs dependencies, and restarts a service.

---

# 1. Theory: Linux Networking Basics

## 1.1 What is Networking?

Networking is the process of connecting computers, servers, and devices so they can communicate with each other.

In Linux administration, networking knowledge is important because most Linux systems are used as web servers, database servers, application servers, DNS servers, monitoring servers, DevOps automation servers, and cloud instances.

A Linux administrator / devops engineer should be able to check whether a server has internet access, whether a service is listening on a port, and whether another server is reachable.

---

## 1.2 IP Address

An **IP address** is a unique address assigned to a device on a network.

Example:

```text
192.168.1.10
```

An IP address helps identify a machine in a network.

There are two common types:

| Type | Example        | Description               |
| ---- | -------------- | ------------------------- |
| IPv4 | `192.168.1.10` | Most commonly used format |
| IPv6 | `2001:db8::1`  | Newer address format      |
|      |                |                           |

Check IP address:

```bash
ip addr
hostname -I
```

---

## 1.3 Private and Public IP Addresses

### Private IP

Private IP addresses are used inside local networks.

Examples:

```text
192.168.x.x
10.x.x.x
172.16.x.x - 172.31.x.x
```

Private IPs are used inside homes, offices, data centers, and cloud private networks.

### Public IP

A public IP address is reachable from the internet.

Example:

```text
8.8.8.8
```

Cloud servers often have both private IPs for internal communication and public IPs for internet access.

---

## 1.4 Subnet

A **subnet** is a smaller logical network inside a larger network.

Example:

```text
192.168.1.0/24
```

This means the network range is usually:

```text
192.168.1.1 to 192.168.1.254
```

The `/24` is called CIDR notation. It defines how many bits are used for the network part.

Common subnet examples:

| CIDR  | Subnet Mask     | Approx. Usable Hosts |
| ----- | --------------- | -------------------- |
| `/24` | `255.255.255.0` | 254                  |
| `/16` | `255.255.0.0`   | 65,534               |
| `/8`  | `255.0.0.0`     | 16 million+          |

Check route and subnet information:

```bash
ip route
```

---

## 1.5 Gateway

A **gateway** is the router or device that connects your machine to other networks.

For example, if your Linux server wants to access the internet, traffic usually goes through the default gateway.

Check default gateway:

```bash
ip route
```

Example output:

```text
default via 192.168.1.1 dev eth0
```

This means:

- `192.168.1.1` is the gateway.
- `eth0` is the network interface.

---

## 1.6 DNS

**DNS** stands for **Domain Name System**.

DNS converts domain names into IP addresses.

Example:

```text
google.com -> 142.250.x.x
```

Without DNS, users would need to remember IP addresses instead of domain names.

Test DNS resolution:

```bash
nslookup google.com
dig google.com
```

If `dig` is not installed:

```bash
sudo apt install -y dnsutils
```

---

## 1.7 `/etc/hosts`

The `/etc/hosts` file is used for local hostname resolution.

It maps hostnames to IP addresses manually.

View the file:

```bash
cat /etc/hosts
```

Example:

```text
127.0.0.1       localhost
192.168.1.50    myserver.local
```

After adding this entry, you can use:

```bash
ping myserver.local
```

instead of:

```bash
ping 192.168.1.50
```

### Important Use Cases

`/etc/hosts` is useful for:

- Local testing
- Temporary DNS override
- Internal lab environments
- Mapping names before real DNS is configured

---

## 1.8 `/etc/resolv.conf`

The `/etc/resolv.conf` file contains DNS resolver settings.

View it:

```bash
cat /etc/resolv.conf
```

Example:

```text
nameserver 8.8.8.8
nameserver 1.1.1.1
```

This means the system uses these DNS servers to resolve domain names.

### Important Note

On modern Linux systems, `/etc/resolv.conf` may be managed automatically by tools such as NetworkManager, `systemd-resolved`, cloud-init, or DHCP clients. Manual changes may be overwritten.

---

# 2. Network Tools

## 2.1 ping

`ping` checks whether a host is reachable.

```bash
ping google.com
ping -c 4 google.com
```

Use cases:

- Check internet connectivity
- Check server reachability
- Check DNS resolution

If ping by IP works but ping by domain fails, DNS may be the problem.

```bash
ping -c 4 8.8.8.8
ping -c 4 google.com
```

---

## 2.2 curl

`curl` is used to send requests to URLs.

It is commonly used to test APIs, websites, and HTTP status codes.

```bash
curl http://localhost
curl -I http://localhost
curl -o /dev/null -s -w "%{http_code}\n" http://localhost
```

Common HTTP status codes:

| Code  | Meaning               |
| ----- | --------------------- |
| `200` | OK                    |
| `301` | Redirect              |
| `302` | Temporary redirect    |
| `400` | Bad request           |
| `401` | Unauthorized          |
| `403` | Forbidden             |
| `404` | Not found             |
| `500` | Internal server error |
| `502` | Bad gateway           |
| `503` | Service unavailable   |

---

## 2.3 wget

`wget` is used to download files from the internet.

```bash
wget https://example.com/file.zip
wget -O myfile.html https://example.com
wget --spider https://example.com
```

Difference between `curl` and `wget`:

| Tool   | Best For                                  |
| ------ | ----------------------------------------- |
| `curl` | Testing APIs and HTTP endpoints           |
| `wget` | Downloading files recursively or directly |

---

## 2.4 ss

`ss` is used to inspect listening ports and network connections.

```bash
ss -tulpn
ss -tulpn | grep :80
```

Options:

| Option | Meaning         |
| ------ | --------------- |
| `-t`   | TCP             |
| `-u`   | UDP             |
| `-l`   | Listening ports |
| `-p`   | Process name    |
| `-n`   | Numeric output  |

---

## 2.5 netstat

`netstat` is an older tool used to show network connections and listening ports.

Install it:

```bash
sudo apt install -y net-tools
```

Use:

```bash
netstat -tulpn
```

Modern replacement:

```bash
ss -tulpn
```

---

## 2.6 traceroute

`traceroute` shows the route packets take to reach a destination.

```bash
sudo apt install -y traceroute
traceroute google.com
```

It helps troubleshoot network path issues between your machine and a remote host.

---

## 2.7 nmap Basics

`nmap` is a network scanning tool.

```bash
sudo apt install -y nmap
nmap 192.168.1.10
nmap -p 80 192.168.1.10
nmap -p 22,80,443 192.168.1.10
```

### Important Warning

Only scan systems that you own or have permission to test. Unauthorized scanning can be considered suspicious or illegal in many environments.

---

# 3. Bash Scripting Theory

The uploaded shell scripting material explains that a shell script is a text file containing executable commands. When the script runs, each command is executed in order. Shell scripts are useful for automating repeated tasks and creating consistent workflows.

---

## 3.1 What is Bash?

**Bash** stands for **Bourne Again Shell**.

It is one of the most common shells in Linux.

```bash
echo $SHELL
bash --version
```

---

## 3.2 What is a Shell Script?

A shell script is a file that contains Linux commands.

```bash
#!/bin/bash
echo "Hello, World!"
```

The first line is called a **shebang**:

```bash
#!/bin/bash
```

It tells Linux to run the script using Bash.

---

## 3.3 Creating and Running a Script

Create a script:

```bash
nano hello.sh
```

Add:

```bash
#!/bin/bash
echo "Hello, World!"
```

Make it executable and run it:

```bash
chmod +x hello.sh
./hello.sh
```

If you get `Permission denied`, the script is not executable. Fix it with:

```bash
chmod +x hello.sh
```

---

## 3.4 Bash Variables

A variable stores a value.

```bash
#!/bin/bash

NAME="Rohan"
echo "Hello $NAME"
```

Rules:

- No spaces around `=`.
- Use `$VARIABLE` to read the value.
- Use quotes when values contain spaces.

Correct:

```bash
APP_NAME="my app"
```

Wrong:

```bash
APP_NAME = "my app"
```

---

## 3.5 Command Substitution

Command substitution stores the output of a command in a variable.

```bash
CURRENT_DIR=$(pwd)
echo "Current directory is $CURRENT_DIR"
```

Date example:

```bash
TODAY=$(date +%F)
echo "Today is $TODAY"
```

---

## 3.6 Environment Variables

Environment variables are variables available to processes and child processes.

```bash
env
echo $HOME
echo $USER
echo $PATH
```

Common examples:

| Variable | Meaning                           |
| -------- | --------------------------------- |
| `HOME`   | User home directory               |
| `USER`   | Current username                  |
| `PATH`   | Directories searched for commands |
| `SHELL`  | Current shell                     |
| `PWD`    | Current working directory         |

Create a temporary environment variable:

```bash
export APP_ENV=production
echo $APP_ENV
```

---

## 3.7 Exit Codes and `$?`

Every Linux command returns an exit code.

| Exit Code | Meaning          |
| --------- | ---------------- |
| `0`       | Success          |
| Non-zero  | Error or failure |

Check the previous command's exit code:

```bash
echo $?
```

Example:

```bash
ls /etc
echo $?
```

Example failure:

```bash
ls /wrong-path
echo $?
```

The shell scripting material explains that `$?` is used to check whether the previous command completed successfully.

---

## 3.8 Using `exit`

A script can return an exit code using `exit`.

```bash
#!/bin/bash

echo "Something failed"
exit 1
```

Use `exit 0` for success and non-zero values for failure.

---

# 4. Conditions in Bash

## 4.1 if Statement

```bash
if [ condition ]; then
  command
fi
```

Example:

```bash
#!/bin/bash

if [ -f /etc/hosts ]; then
  echo "/etc/hosts exists"
fi
```

---

## 4.2 if/else

```bash
#!/bin/bash

if [ -f /etc/passwd ]; then
  echo "File exists"
else
  echo "File does not exist"
fi
```

---

## 4.3 if/elif/else

```bash
#!/bin/bash

STATUS_CODE=200

if [ "$STATUS_CODE" -eq 200 ]; then
  echo "OK"
elif [ "$STATUS_CODE" -eq 404 ]; then
  echo "Not Found"
else
  echo "Other status"
fi
```

---

## 4.4 Common File Tests

| Test      | Meaning                           |
| --------- | --------------------------------- |
| `-f file` | File exists and is a regular file |
| `-d dir`  | Directory exists                  |
| `-x file` | File exists and is executable     |
| `-r file` | File is readable                  |
| `-w file` | File is writable                  |

```bash
if [ -d /var/log ]; then
  echo "Log directory exists"
fi
```

---

## 4.5 String Comparisons

| Operator | Meaning          |
| -------- | ---------------- |
| `=`      | Equal            |
| `!=`     | Not equal        |
| `-z`     | Empty string     |
| `-n`     | Non-empty string |

```bash
NAME="admin"

if [ "$NAME" = "admin" ]; then
  echo "Welcome admin"
fi
```

---

## 4.6 Numeric Comparisons

| Operator | Meaning               |
| -------- | --------------------- |
| `-eq`    | Equal                 |
| `-ne`    | Not equal             |
| `-gt`    | Greater than          |
| `-lt`    | Less than             |
| `-ge`    | Greater than or equal |
| `-le`    | Less than or equal    |
| &&       | and                   |
| \|\|     | or                    |
| !        | not                   |

```bash
COUNT=10

if [ "$COUNT" -gt 5 ]; then
  echo "Count is greater than 5"
fi
```

---

# 5. Loops in Bash

## 5.1 for Loop

```bash
#!/bin/bash
name=( "ram","sita", "hari")
for NAME in "${name[0]}" ; do
  echo "Hello $NAME"
done
```

Loop through files:

```bash
for FILE in *.log; do
  echo "Found log file: $FILE"
done
```

---

## 5.2 while Loop

```bash
#!/bin/bash

COUNT=1

while [ "$COUNT" -le 5 ]; do
  echo "Count: $COUNT"
  COUNT=$((COUNT + 1))
done
```

---

## 5.3 Loop with Command Output

```bash
#!/bin/bash

for USER in $(cut -d: -f1 /etc/passwd); do
  echo "User: $USER"
done
```

---

# 6. Bash Arrays

## 6.1 Create an Array

```bash
SERVERS=("web01" "web02" "web03")
echo "${SERVERS[0]}"
echo "${SERVERS[@]}"
```

Loop through array:

```bash
for SERVER in "${SERVERS[@]}"; do
  echo "Checking $SERVER"
done
```

---

## 6.2 Endpoint Array Example

```bash
URLS=("http://localhost" "https://example.com")

for URL in "${URLS[@]}"; do
  STATUS=$(curl -o /dev/null -s -w "%{http_code}" "$URL")
  echo "$URL returned $STATUS"
done
```

---

# 7. Bash Functions

## 7.1 What is a Function?

A function is a reusable block of code.

```bash
say_hello() {
  echo "Hello"
}

say_hello
```

---

## 7.2 Function with Arguments

```bash
greet_user() {
  echo "Hello $1"
}

greet_user "Rohan"
```

---

## 7.3 Function Returning Exit Code

```bash
check_file() {
  if [ -f "$1" ]; then
    return 0
  else
    return 1
  fi
}

if check_file "/etc/hosts"; then
  echo "File exists"
else
  echo "File missing"
fi
```

---

# 8. Day 6 Practical Lab

## Lab 1: Check Basic Network Information

```bash
ip addr
ip route
cat /etc/resolv.conf
cat /etc/hosts
```

---

## Lab 2: Test Connectivity

```bash
ping -c 4 8.8.8.8
ping -c 4 google.com
```

If IP ping works but domain ping fails, DNS resolution may be broken.

---

## Lab 3: Use curl to Test HTTP Endpoints

Install curl if needed:

```bash
sudo apt update
sudo apt install -y curl
```

Test endpoint:

```bash
curl -I https://example.com
```

Show only HTTP status code:

```bash
curl -o /dev/null -s -w "%{http_code}\n" https://example.com
```

Save status code in a variable:

```bash
STATUS=$(curl -o /dev/null -s -w "%{http_code}" https://example.com)
echo "$STATUS"
```

---

## Lab 4: Check Listening Ports

```bash
ss -tulpn
ss -tulpn | grep :80
ss -tulpn | grep :22
```

---

## Lab 5: Write a Bash Healthcheck Script for a Web Server

Create the script:

```bash
nano web_healthcheck.sh
```

Add:

```bash
#!/bin/bash

URL="http://localhost"
LOG_FILE="$HOME/web_healthcheck.log"

STATUS_CODE=$(curl -o /dev/null -s -w "%{http_code}" "$URL")

echo "[$(date)] URL=$URL STATUS=$STATUS_CODE" >> "$LOG_FILE"

if [ "$STATUS_CODE" -eq 200 ]; then
  echo "Web server is healthy"
  exit 0
else
  echo "Web server is unhealthy. Status code: $STATUS_CODE"
  exit 1
fi
```

Make executable:

```bash
chmod +x web_healthcheck.sh
```

Run:

```bash
./web_healthcheck.sh
echo $?
cat ~/web_healthcheck.log
```

---

## Lab 6: Healthcheck Script with Multiple URLs

Create:

```bash
nano multi_healthcheck.sh
```

Add:

```bash
#!/bin/bash

URLS=("http://localhost" "https://example.com")
LOG_FILE="$HOME/multi_healthcheck.log"

check_url() {
  local URL="$1"
  local STATUS_CODE

  STATUS_CODE=$(curl -o /dev/null -s -w "%{http_code}" "$URL")

  echo "[$(date)] URL=$URL STATUS=$STATUS_CODE" >> "$LOG_FILE"

  if [ "$STATUS_CODE" -eq 200 ]; then
    echo "$URL is healthy"
    return 0
  else
    echo "$URL is unhealthy. Status code: $STATUS_CODE"
    return 1
  fi
}

for URL in "${URLS[@]}"; do
  check_url "$URL"
done
```

Make executable and run:

```bash
chmod +x multi_healthcheck.sh
./multi_healthcheck.sh
```

---

# 9. Assignment 6

## Assignment: Write a Deployment Script

### Objective

Write a deployment script that:

1. Goes to an application directory.
2. Pulls the latest code from Git.
3. Installs dependencies.
4. Restarts a Linux service.
5. Logs deployment output.

---

## Example Deployment Script

Create the script:

```bash
nano deploy.sh
```

Add:

```bash
#!/bin/bash

APP_DIR="/var/www/myapp"
SERVICE_NAME="myapp"
LOG_FILE="/var/log/myapp-deploy.log"

echo "===== Deployment started at $(date) =====" | tee -a "$LOG_FILE"

if [ ! -d "$APP_DIR" ]; then
  echo "ERROR: Application directory does not exist: $APP_DIR" | tee -a "$LOG_FILE"
  exit 1
fi

cd "$APP_DIR" || {
  echo "ERROR: Failed to enter application directory" | tee -a "$LOG_FILE"
  exit 1
}

echo "Pulling latest code..." | tee -a "$LOG_FILE"
git pull origin main | tee -a "$LOG_FILE"

if [ "${PIPESTATUS[0]}" -ne 0 ]; then
  echo "ERROR: git pull failed" | tee -a "$LOG_FILE"
  exit 1
fi

echo "Installing dependencies..." | tee -a "$LOG_FILE"

if [ -f "package.json" ]; then
  npm install | tee -a "$LOG_FILE"
elif [ -f "requirements.txt" ]; then
  pip install -r requirements.txt | tee -a "$LOG_FILE"
else
  echo "No known dependency file found. Skipping dependency installation." | tee -a "$LOG_FILE"
fi

echo "Restarting service: $SERVICE_NAME" | tee -a "$LOG_FILE"
sudo systemctl restart "$SERVICE_NAME"

if [ $? -eq 0 ]; then
  echo "Service restarted successfully" | tee -a "$LOG_FILE"
else
  echo "ERROR: Service restart failed" | tee -a "$LOG_FILE"
  exit 1
fi

sudo systemctl status "$SERVICE_NAME" --no-pager | tee -a "$LOG_FILE"

echo "===== Deployment finished at $(date) =====" | tee -a "$LOG_FILE"
```

Make executable:

```bash
chmod +x deploy.sh
```

Run:

```bash
./deploy.sh
```

---

## Assignment Submission Format

Students should submit:

```text
Script name:
Application directory used:
Service name used:
Commands included:
Screenshot or terminal output:
Final service status:
```

---

# 10. Quick Command Summary

## Networking

```bash
ip addr
ip route
cat /etc/hosts
cat /etc/resolv.conf
ping -c 4 google.com
curl -I https://example.com
curl -o /dev/null -s -w "%{http_code}\n" https://example.com
ss -tulpn
netstat -tulpn
traceroute google.com
nmap -p 80 localhost
```

## Bash Scripting

```bash
#!/bin/bash
NAME="Rohan"
echo "$NAME"
echo $?
exit 0
```

## Conditions

```bash
if [ -f /etc/hosts ]; then
  echo "File exists"
else
  echo "File missing"
fi
```

## Loops

```bash
for FILE in *.log; do
  echo "$FILE"
done
```

```bash
COUNT=1
while [ "$COUNT" -le 5 ]; do
  echo "$COUNT"
  COUNT=$((COUNT + 1))
done
```

## Functions

```bash
check_service() {
  systemctl status "$1" --no-pager
}

check_service nginx
```

---

# 11. Common Beginner Mistakes

- Forgetting the shebang line `#!/bin/bash`.
- Forgetting to make a script executable with `chmod +x`.
- Adding spaces around `=` when assigning variables.
- Not quoting variables such as `"$APP_DIR"`.
- Confusing `$?` with `$1`.
- Forgetting that exit code `0` means success.
- Running `curl` without checking the status code.
- Using `netstat` on a system where it is not installed.
- Running `nmap` on systems without permission.
- Forgetting to restart a service after deployment.
- Using relative paths in scripts that run from cron or automation tools.

---

# 12. End-of-Day Review Questions

1. What is the purpose of an IP address?
2. What is a subnet?
3. What is a default gateway?
4. What does DNS do?
5. What is the purpose of `/etc/hosts`?
6. What is the purpose of `/etc/resolv.conf`?
7. What is the difference between `curl` and `wget`?
8. What command checks listening ports?
9. What does `$?` show?
10. What does exit code `0` mean?
11. What is the difference between a variable and an environment variable?
12. How do you create a function in Bash?
13. What is the difference between a `for` loop and a `while` loop?
14. How can `curl` be used to check if a website is healthy?
15. What should a deployment script do?

---

# 13. Expected Practical Output

By the end of the lab, students should have:

- Checked IP, route, DNS, and hosts configuration.
- Used `ping`, `curl`, and `ss`.
- Written a web server healthcheck script.
- Used `curl` to check HTTP status codes.
- Used variables, conditions, functions, loops, and arrays in Bash.
- Written a deployment script that pulls code, installs dependencies, and restarts a service.
