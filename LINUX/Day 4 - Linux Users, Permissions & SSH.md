# Day 4 - Linux Users, Permissions & SSH

## Learning Goals

By the end of this class, students should be able to:

- Explain Linux users, groups, UIDs, and GIDs.
- Understand the purpose of `/etc/passwd` and `/etc/shadow`.
- Use `chmod` with octal and symbolic permission modes.
- Change file ownership with `chown` and group ownership with `chgrp`.
- Understand `sudo` and the role of `/etc/sudoers`.
- Create a user, assign groups, and set correct permissions.
- Explain SSH and connect to a remote Linux machine securely.
- Understand password-based and key-based SSH authentication.

---

# Part A — Theory in Detail

## 1. Linux Users and Groups

Linux is a multi-user operating system. This means multiple users can exist on the same system, and each user can have different access rights.

A **user** represents an account that can log in or run processes.

A **group** is a collection of users. Groups make permission management easier because permissions can be assigned to a group instead of assigning them to users one by one.

Example:

```bash
whoami
id
```

The `whoami` command shows the current logged-in user.

The `id` command shows the user ID, group ID, and all groups the user belongs to.

Example output:

```bash
uid=1001(student) gid=1001(student) groups=1001(student),27(sudo)
```

This means:

- `uid=1001(student)` means the user ID is `1001` and the username is `student`.
- `gid=1001(student)` means the primary group ID is `1001`.
- `groups=1001(student),27(sudo)` means the user belongs to the `student` and `sudo` groups.

---

## 2. UIDs and GIDs

### UID: User ID

A **UID** is a numeric identifier assigned to each user.

Linux internally identifies users by UID, not by username. The username is mainly for human readability.

Common UID ranges:

| UID Range | Meaning                                        |
| --------- | ---------------------------------------------- |
| `0`       | Root user                                      |
| `1-999`   | System users and service accounts              |
| `1000+`   | Normal human users on many Linux distributions |

Example:

```bash
id student
```

### GID: Group ID

A **GID** is a numeric identifier assigned to each group.

Every user has:

- A **primary group**
- Optional **secondary groups**

The primary group is usually created with the same name as the user.

Example:

```bash
groups student
```

---

## 3. `/etc/passwd`

The `/etc/passwd` file stores basic user account information.

View it using:

```bash
cat /etc/passwd
```

Each line represents one user account.

Example:

```bash
student:x:1001:1001:Student User:/home/student:/bin/bash
```

Field breakdown:

| Field                | Example         | Meaning                             |
| -------------------- | --------------- | ----------------------------------- |
| Username             | `student`       | Login name                          |
| Password placeholder | `x`             | Password is stored in `/etc/shadow` |
| UID                  | `1001`          | User ID                             |
| GID                  | `1001`          | Primary group ID                    |
| Comment/GECOS        | `Student User`  | Full name or description            |
| Home directory       | `/home/student` | User's home folder                  |
| Login shell          | `/bin/bash`     | Shell started after login           |

Important note:

Although the file is named `passwd`, modern Linux systems do **not** store real passwords directly in this file. Password hashes are stored in `/etc/shadow`.

---

## 4. `/etc/shadow`

The `/etc/shadow` file stores encrypted password hashes and password aging information.

View it with root privileges:

```bash
sudo cat /etc/shadow
```

Example line:

```bash
student:$y$j9T$abc...:19800:0:99999:7:::
```

Important fields:

| Field                | Meaning                                     |
| -------------------- | ------------------------------------------- |
| Username             | Account name                                |
| Password hash        | Encrypted password hash                     |
| Last password change | Days since Jan 1, 1970                      |
| Minimum age          | Minimum days before password can be changed |
| Maximum age          | Maximum days password is valid              |
| Warning period       | Days before expiry to warn user             |

Security point:

`/etc/shadow` is readable only by root because it contains sensitive password hashes.

Check permissions:

```bash
ls -l /etc/shadow
```

Typical output:

```bash
-rw-r----- 1 root shadow ... /etc/shadow
```

---

## 5. Linux File Permissions

Every file and directory has three permission categories:

| Category   | Meaning                            |
| ---------- | ---------------------------------- |
| User/Owner | The user who owns the file         |
| Group      | The group associated with the file |
| Others     | Everyone else                      |

Check permissions:

```bash
ls -l
```

Example:

```bash
-rw-r--r-- 1 student student 120 May 24 10:00 notes.txt
```

Breakdown:

```text
- rw- r-- r--
|  |   |   |
|  |   |   └── others permissions
|  |   └────── group permissions
|  └────────── owner permissions
└───────────── file type
```

Permission symbols:

| Symbol | Meaning                |
| ------ | ---------------------- |
| `r`    | Read                   |
| `w`    | Write                  |
| `x`    | Execute                |
| `-`    | Permission not granted |

For files:

- `r` means the file can be read.
- `w` means the file can be modified.
- `x` means the file can be executed as a program or script.

For directories:

- `r` means directory contents can be listed.
- `w` means files can be created, deleted, or renamed inside the directory.
- `x` means the directory can be entered using `cd`.

---

## 6. `chmod` Octal Mode

The `chmod` command changes file or directory permissions.

Octal mode uses numbers:

| Permission  | Number |
| ----------- | -----: |
| Read `r`    |      4 |
| Write `w`   |      2 |
| Execute `x` |      1 |

Add the numbers to create permission sets:

| Octal | Permission | Meaning |
|---|---|---|
| `7` | `rwx` | read, write, execute |
| `6` | `rw-` | read, write |
| `5` | `r-x` | read, execute |
| `4` | `r--` | read only |
| `0` | `---` | no permission |

Example:

```bash
chmod 644 notes.txt
```

Meaning:

```text
6 = owner can read and write
4 = group can read
4 = others can read
```

Result:

```bash
-rw-r--r-- notes.txt
```

Common permission examples:

| Command               | Meaning                                       |
| --------------------- | --------------------------------------------- |
| `chmod 600 file.txt`  | Only owner can read/write                     |
| `chmod 644 file.txt`  | Owner read/write, others read-only            |
| `chmod 700 script.sh` | Only owner can read/write/execute             |
| `chmod 755 script.sh` | Owner full access, others read/execute        |
| `chmod 770 shared/`   | Owner and group full access, others no access |
|                       |                                               |
|                       |                                               |

---

## 7. `chmod` Symbolic Mode

Symbolic mode uses letters instead of numbers.

User categories:

| Symbol | Meaning    |
| ------ | ---------- |
| `u`    | User/owner |
| `g`    | Group      |
| `o`    | Others     |
| `a`    | All users  |

Operators:

| Operator | Meaning              |
| -------- | -------------------- |
| `+`      | Add permission       |
| `-`      | Remove permission    |
| `=`      | Set exact permission |

Examples:

```bash
chmod u+x script.sh
```

Adds execute permission for the owner.

```bash
chmod g+w project.txt
```

Adds write permission for the group.

```bash
chmod o-r secret.txt
```

Removes read permission from others.

```bash
chmod a+r notes.txt
```

Adds read permission for everyone.

```bash
chmod u=rw,g=r,o= notes.txt
```

Sets:

- Owner: read and write
- Group: read only
- Others: no permissions

---

## 8. `chown` — Change File Owner

The `chown` command changes file ownership.

Syntax:

```bash
sudo chown USER FILE
```

Example:

```bash
sudo chown student notes.txt
```

Change owner and group together:

```bash
sudo chown student:developers notes.txt
```

Change ownership recursively for a directory:

```bash
sudo chown -R student:developers /project
```

`-R` means recursive, so it applies to all files and subdirectories.

---

## 9. `chgrp` — Change Group Ownership

The `chgrp` command changes the group owner of a file or directory.

Syntax:

```bash
sudo chgrp GROUP FILE
```

Example:

```bash
sudo chgrp developers notes.txt
```

Recursive example:

```bash
sudo chgrp -R developers /project
```

This is useful when multiple users need shared access to the same directory.

---

## 10. `sudo` and `/etc/sudoers`

### What is `sudo`?

`sudo` means **superuser do**.

It allows a normal user to run commands with administrative privileges.

Example:

```bash
sudo apt update
sudo useradd testuser
sudo useradd -m testuser # make user with home dir 
```

Without `sudo`, normal users cannot perform many administrative tasks.

### Why use `sudo` instead of logging in as root?

Using `sudo` is safer because:

- Users do not need to know the root password.
- Commands are logged.
- Access can be granted only to specific users or groups.
- It reduces the risk of accidental system damage.

### `/etc/sudoers`

The `/etc/sudoers` file controls who can use `sudo`.

Never edit it directly with a normal text editor. Use:

```bash
sudo visudo
```

`visudo` checks syntax before saving. This prevents mistakes that could break sudo access.

Example sudoers rule:

```bash
student ALL=(ALL:ALL) ALL
```

Meaning:

| Part        | Meaning                            |
| ----------- | ---------------------------------- |
| `student`   | User allowed to use sudo           |
| `ALL`       | From any host                      |
| `(ALL:ALL)` | Can run commands as any user/group |
| `ALL`       | Can run all commands               |

On Ubuntu, users in the `sudo` group can usually use sudo:

```bash
%sudo ALL=(ALL:ALL) ALL
```

Add user to sudo group:

```bash
sudo usermod -aG sudo student
```

Important:

The user may need to log out and log back in for new group membership to apply.

---

## 11. SSH Theory

SSH stands for **Secure Shell**.

SSH is used to securely access a remote Linux machine from another machine.

It provides:

- Encrypted remote login
- Remote command execution
- Secure file transfer using `scp` or `sftp`
- Secure tunneling and port forwarding
 
Default SSH port:

```text
22/tcp
```

Main SSH components:

| Component  | Meaning                                              |
| ---------- | ---------------------------------------------------- |
| SSH client | The machine running the `ssh` command                |
| SSH server | The remote machine running the `sshd` service        |
| `sshd`     | SSH daemon that listens for incoming SSH connections |

Basic SSH command:

```bash
ssh user@server_ip
```

Example:

```bash
ssh student@192.168.1.50
```

---

## 12. Password Authentication vs Key Authentication

### Password Authentication

Password authentication asks for the user's password when connecting.

Example:

```bash
ssh student@192.168.1.50
```

Advantages:

- Easy for beginners
- Quick to set up

Disadvantages:

- Vulnerable to brute-force attacks if exposed to the internet
- Depends heavily on password strength
- Less suitable for production servers

### Key-Based Authentication

Key-based authentication uses a cryptographic key pair.

The key pair contains:

| Key         | Location       | Purpose                            |
| ----------- | -------------- | ---------------------------------- |
| Private key | Client machine | Must be kept secret                |
| Public key  | Server machine | Stored in `~/.ssh/authorized_keys` |

Generate key pair:

```bash
ssh-keygen -t ed25519 -C "student@linux-lab"
```

Default files:

```text
~/.ssh/id_ed25519      private key
~/.ssh/id_ed25519.pub  public key
```

Security rule:

Never share your private key.

---

# Part B — Practical Lab

## Lab Goal

In this lab, students will:

1. Create users and groups.
2. Assign users to groups.
3. Create files and directories.
4. Apply permissions using `chmod`.
5. Change ownership using `chown` and `chgrp`.
6. Give sudo access to a user.
7. Connect using SSH.
8. Configure SSH key-based login.

---

## 1. Check Current User Information

```bash
whoami
id
pwd
```

Check users from `/etc/passwd`:

```bash
cat /etc/passwd | tail
```

Check groups:

```bash
cat /etc/group | tail
```

---

## 2. Create a New Group

Create a group named `developers`:

```bash
sudo groupadd developers
```

Verify:

```bash
getent group developers
```

---

## 3. Create a New User

Create a user named `alice` with a home directory:

```bash
sudo useradd -m -s /bin/bash alice
```

Set password:

```bash
sudo passwd alice
```

Verify user:

```bash
id alice
getent passwd alice
```

---

## 4. Assign User to a Group

Add `alice` to the `developers` group:

```bash
sudo usermod -aG developers alice
```

Verify:

```bash
id alice
groups alice
```

Important:

Always use `-aG`, not only `-G`. The `-a` option appends the new group without removing existing groups.

---

## 5. Create a Shared Project Directory

Create a directory:

```bash
sudo mkdir -p /project/dev
```

Change owner and group:

```bash
sudo chown root:developers /project/dev
```

Set permissions:

```bash
sudo chmod 770 /project/dev
```

Verify:

```bash
ls -ld /project/dev
```

Expected style:

```text
drwxrwx--- root developers /project/dev
```

Meaning:

- Root owns the directory.
- Members of `developers` group can read, write, and enter the directory.
- Others have no access.

---

## 6. Test File Creation as the New User

Switch to `alice`:

```bash
su - alice
```

Go to project directory:

```bash
cd /project/dev
```

Create file:

```bash
touch alice-notes.txt
```

Write text:

```bash
echo "Created by Alice" > alice-notes.txt
```

Check file:

```bash
ls -l
cat alice-notes.txt
```

Exit from Alice session:

```bash
exit
```

---

## 7. Practice `chmod` Octal Mode

Create practice file:

```bash
touch permissions-demo.txt
ls -l permissions-demo.txt
```

Apply `644`:

```bash
chmod 644 permissions-demo.txt
ls -l permissions-demo.txt
```

Apply `600`:

```bash
chmod 600 permissions-demo.txt
ls -l permissions-demo.txt
```

Apply `755`:

```bash
chmod 755 permissions-demo.txt
ls -l permissions-demo.txt
```

---

## 8. Practice `chmod` Symbolic Mode

```bash
chmod u+x permissions-demo.txt
chmod g+w permissions-demo.txt
chmod o-r permissions-demo.txt
ls -l permissions-demo.txt
```

Set exact permissions:

```bash
chmod u=rw,g=r,o= permissions-demo.txt
ls -l permissions-demo.txt
```

---

## 9. Practice `chown` and `chgrp`

Change owner:

```bash
sudo chown alice permissions-demo.txt
ls -l permissions-demo.txt
```

Change group:

```bash
sudo chgrp developers permissions-demo.txt
ls -l permissions-demo.txt
```

Change owner and group together:

```bash
sudo chown root:developers permissions-demo.txt
ls -l permissions-demo.txt
```

---

## 10. Give Sudo Access to a User

Add `alice` to sudo group:

```bash
sudo usermod -aG sudo alice
```

Verify:

```bash
groups alice
```

Test:

```bash
su - alice
sudo whoami
```

Expected output:

```text
root
```

Exit:

```bash
exit
```

---

# Part C — SSH Practical Lab

The uploaded SSH lesson explains SSH remote login, SSH key pairs, `authorized_keys`, and Docker-based SSH server testing. This section keeps the SSH flow and adapts it for Day 3.

## 1. Check SSH Client

On the client machine:

```bash
ssh -V
```

---
## 2. Start SSH Server Container

```bash
docker run -d --name ssh-server \
  -p 2222:2222 \
  -e TZ=Asia/Kathmandu \
  -e USER_NAME=student \
  -e USER_PASSWORD=student123 \
  -e PASSWORD_ACCESS=true \
  -e SUDO_ACCESS=true \
  --restart unless-stopped \
  linuxserver/openssh-server:latest
```

Check container:

```bash
docker ps
```

Exec into the container and check SSH Server

```bash
docker exec -it ssh-server bash
```

**Note** : *These commands will not run inside the container because these containers are very minimal version of ubuntu.*

On the server machine: 

```bash
sudo systemctl status ssh --no-pager
```

Check if port 22 is listening:

```bash
ss -tulpn | grep :22
```

If SSH server is not installed:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```


Check port mapping:

```bash
docker port ssh-server
```

Expected:

```text
2222/tcp -> 0.0.0.0:2222
```

---

## 3. Connect with Password

```bash
ssh student@localhost -p 2222
```

Password:

```text
student123
```

Exit:

```bash
exit
```

---

## 4. Generate SSH Key and Copy It

```bash
ssh-keygen -t ed25519 -C "student@docker-lab"
ssh-copy-id -p 2222 student@localhost
```

Test login:

```bash
ssh -p 2222 student@localhost
```

---

## 5. Troubleshooting Docker SSH

Check if host port is listening:

```bash
sudo ss -tulpn | grep :2222
```

Check Docker port mapping:

```bash
docker port ssh-server
```

Check logs:

```bash
docker logs ssh-server --tail=200
```

Common issue:

If the container listens on `2222` internally, use:

```bash
-p 2222:2222
```

Do not use `-p 2222:22` unless the container is actually listening on port `22`.

---