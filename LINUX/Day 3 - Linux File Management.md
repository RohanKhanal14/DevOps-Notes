# Day 3 — Linux File Management

## Learning Outcomes

By the end of Day 3, students should be able to:

- Explain inodes and the difference between hard links and symbolic links.
- Understand standard input, standard output, and standard error.
- Use redirection operators: `>`, `>>`, `<`, `2>`, and `2>&1`.
- Use pipes (`|`) to connect commands.
- Use wildcards and globbing patterns to match files.
- Practice core file commands: `cp`, `mv`, `rm`, `mkdir`, `touch`, `find`, and `grep`.
- Build a one-liner to find `.log` files larger than 100 MB.
- Write a cleanup script to delete log files older than 7 days.

---

# 1. Theory

## 1.1 Inodes

An **inode** is a data structure used by Linux filesystems to store metadata about a file.

The inode stores information such as:

- File type
- File permissions
- File owner and group
- File size
- File timestamps
- Number of hard links
- Pointers to the actual data blocks on disk

Important point:

> The inode does not store the file name. The file name is stored in the directory entry, which maps the name to an inode number.

### Check inode number

```bash
ls -i file.txt
```

Example:

```bash
123456 file.txt
```

Here, `123456` is the inode number.

You can also view detailed inode metadata using:

```bash
stat file.txt
```

---

## 1.2 Hard Links vs Soft Links

Linux supports two common types of links:

- Hard link
- Soft link, also called symbolic link or symlink

### Hard Link

A **hard link** is another filename that points to the same inode as the original file.

Create a hard link:

```bash
ln original.txt hardlink.txt
```

Check inode numbers:

```bash
ls -li original.txt hardlink.txt
```

If both files have the same inode number, they are hard links to the same file content.

Characteristics of hard links:

- Same inode as the original file
- Works only within the same filesystem
- Usually cannot be created for directories
- Still works even if the original filename is deleted
- The actual data is removed only when all hard links are deleted

Example:

```bash
echo "Linux class" > original.txt
ln original.txt hardlink.txt
rm original.txt
cat hardlink.txt
```

Output:

```text
Linux class
```

The data is still available because `hardlink.txt` points to the same inode.

---

### Soft Link / Symbolic Link

A **soft link** is a shortcut that points to another file path.

Create a symbolic link:

```bash
ln -s original.txt softlink.txt
```

Check details:

```bash
ls -l softlink.txt
```

Example output:

```text
softlink.txt -> original.txt
```

Characteristics of soft links:

- Has a different inode from the original file
- Points to a file path, not directly to the data
- Can link to directories
- Can link across different filesystems
- Breaks if the original file is deleted or moved

Example:

```bash
echo "Linux class" > original.txt
ln -s original.txt softlink.txt
rm original.txt
cat softlink.txt
```

Possible output:

```text
cat: softlink.txt: No such file or directory
```

The symlink is now broken because the original path no longer exists.

---

## 1.3 Hard Link vs Soft Link Summary

| Feature                         | Hard Link        | Soft Link / Symlink |
| ------------------------------- | ---------------- | ------------------- |
| Points to                       | Same inode       | File path           |
| Inode number                    | Same as original | Different inode     |
| Works after original is deleted | Yes              | No                  |
| Can link directories            | Usually no       | Yes                 |
| Can cross filesystems           | No               | Yes                 |
| Created with                    | `ln file link`   | `ln -s file link`   |

---

## 1.4 stdin, stdout, and stderr

Linux commands normally work with three standard streams.

| Stream | Number | Meaning         | Default Location |
| ------ | -----: | --------------- | ---------------- |
| stdin  |    `0` | Standard input  | Keyboard         |
| stdout |    `1` | Standard output | Terminal screen  |
| stderr |    `2` | Standard error  | Terminal screen  |

Example:

```bash
ls /etc
```

The normal output goes to **stdout**.

Example with error:

```bash
ls /wrong-folder
```

The error message goes to **stderr**.

Understanding these streams is important because redirection allows us to control where input, output, and errors go.

---

## 1.5 Redirection

Redirection is used to send command input or output to files instead of the terminal.

### Redirect stdout with `>`

The `>` operator writes command output to a file.

```bash
ls /etc > etc-list.txt
```

Important:

> `>` overwrites the file if it already exists.

---

### Append stdout with `>>`

The `>>` operator appends output to the end of a file.

```bash
date >> activity.log
```

This is useful for logs because it does not remove existing content.

---

### Redirect stdin with `<`

The `<` operator gives input to a command from a file.

```bash
wc -l < names.txt
```

This counts the number of lines in `names.txt`.

---

### Redirect stderr with `2>`

Errors can be redirected separately using `2>`.

```bash
ls /wrong-folder 2> error.log
```

Now the error message is saved in `error.log` instead of being shown on the screen.

---

### Redirect stdout and stderr together

To save both normal output and errors into one file:

```bash
find /etc -name "*.conf" > output.log 2>&1
```

Modern Bash also supports:

```bash
find /etc -name "*.conf" &> output.log
```

---

## 1.6 Pipes `|`

A pipe sends the output of one command as input to another command.

Basic structure:

```bash
command1 | command2
```

Example:

```bash
ls /etc | less
```

Here:

- `ls /etc` produces output.
- `less` receives that output and displays it page by page.

Another example:

```bash
cat /etc/passwd | grep root
```

Better version:

```bash
grep root /etc/passwd
```

Pipes are very useful when combining commands:

```bash
find /var/log -type f | grep "\.log$"
```

---

## 1.7 Wildcards and Globbing

Globbing means using wildcard patterns to match filenames. These patterns are expanded by the shell before the command runs.

Common glob patterns:

| Pattern  | Meaning                               | Example         |
| -------- | ------------------------------------- | --------------- |
| `*`      | Matches zero or more characters       | `*.log`         |
| `?`      | Matches exactly one character         | `file?.txt`     |
| `[abc]`  | Matches one character from the list   | `file[123].txt` |
| `[a-z]`  | Matches one character from a range    | `[a-d]*`        |
| `[!abc]` | Matches one character not in the list | `[!0-9]*`       |

Examples:

```bash
ls *.txt
ls *.log
ls file?.txt
ls file[1-3].txt
ls [a-d]*
```

Important note:

> Globbing is done by the shell, not by the command itself.

For example:

```bash
echo *.txt
```

The shell expands `*.txt` into matching filenames before running `echo`.

---

# 2. Day 3 Practical Lab

## 2.1 Lab Setup

Create a working directory:

```bash
mkdir -p ~/day3_lab
cd ~/day3_lab
pwd
```

Create sample files:

```bash
touch file1.txt file2.txt file3.log file4.log notes.md app.log error.log
```

Check files:

```bash
ls -l
```

---

## 2.2 Practice `mkdir`

Create directories:

```bash
mkdir docs logs backup
```

Create nested directories:

```bash
mkdir -p projects/app/src projects/app/config projects/app/logs
```

Check structure:

```bash
ls -R
```

---

## 2.3 Practice `touch`

Create empty files:

```bash
touch report.txt summary.txt
```

Update timestamp of a file:

```bash
touch report.txt
stat report.txt
```

---

## 2.4 Practice `cp`

Copy a file:

```bash
cp file1.txt docs/
```

Copy and rename:

```bash
cp file2.txt docs/file2_backup.txt
```

Copy all `.log` files into `logs`:

```bash
cp *.log logs/
```

Copy a directory recursively:

```bash
cp -r projects backup/
```

Safe copy with confirmation and verbose output:

```bash
cp -iv file1.txt docs/
```

Useful options:

```bash
cp -i source destination   # ask before overwrite
cp -v source destination   # show copied files
cp -r dir1 dir2            # copy directories recursively
```

---

## 2.5 Practice `mv`

Rename a file:

```bash
mv notes.md notes_day3.md
```

Move a file:

```bash
mv report.txt docs/
```

Move and rename at the same time:

```bash
mv summary.txt docs/final_summary.txt
```

Rename a directory:

```bash
mv docs documents
```

---

## 2.6 Practice `rm`

Create test files:

```bash
touch delete1.tmp delete2.tmp delete3.tmp
```

Remove one file:

```bash
rm delete1.tmp
```

Remove files with confirmation:

```bash
rm -i delete2.tmp
```

Remove multiple files using globbing:

```bash
rm -i *.tmp
```

Remove a directory and its contents:

```bash
rm -r backup
```

Dangerous command:

```bash
rm -rf directory_name
```

Safety habit before using `rm`:

```bash
pwd
ls -la
```

---

## 2.7 Practice `find`

Find all `.log` files in the current directory:

```bash
find . -type f -name "*.log"
```

Find all `.txt` files:

```bash
find . -type f -name "*.txt"
```

Find files modified more than 7 days ago:

```bash
find . -type f -mtime +7
```

Find files larger than 100 MB:

```bash
find . -type f -size +100M
```

Find `.log` files larger than 100 MB:

```bash
find . -type f -name "*.log" -size +100M
```

---

## 2.8 Practice `grep`

Create sample log content:

```bash
cat > app.log <<'EOF_LOG'
INFO Application started
ERROR Database connection failed
INFO Retrying connection
WARNING Disk usage high
ERROR Timeout occurred
EOF_LOG
```

Search for `ERROR`:

```bash
grep "ERROR" app.log
```

Case-insensitive search:

```bash
grep -i "error" app.log
```

Show line numbers:

```bash
grep -n "ERROR" app.log
```

Search recursively:

```bash
grep -R "ERROR" .
```

Search only in `.log` files:

```bash
grep "ERROR" *.log
```

---

## 2.9 Practice Pipes `|`

Use pipe with `grep`:

```bash
ls -l | grep ".log"
```

Find log files and count them:

```bash
find . -type f -name "*.log" | wc -l
```

Find errors in all log files:

```bash
grep -R "ERROR" . | less
```

List files sorted by size:

```bash
ls -lh | sort -k5 -h
```

---

# 3. Bash One-Liner

## Task

Build a bash one-liner to find all `.log` files over 100 MB.

## Recommended command

```bash
find /var/log -type f -name "*.log" -size +100M
```

Explanation:

| Part            | Meaning                           |
| --------------- | --------------------------------- |
| `find`          | Search for files and directories  |
| `/var/log`      | Start searching inside `/var/log` |
| `-type f`       | Match only regular files          |
| `-name "*.log"` | Match files ending with `.log`    |
| `-size +100M`   | Match files larger than 100 MB    |

More detailed version:

```bash
find /var/log -type f -name "*.log" -size +100M -exec ls -lh {} \;
```

This shows file size and details.

Alternative version with pipe:

```bash
find /var/log -type f -name "*.log" -size +100M | xargs -r ls -lh
```

---

# 4. Assignment

## Assignment 3: Cleanup Script

Write a cleanup script to delete log files older than 7 days.

### Requirement

The script should:

- Search for `.log` files.
- Delete only files older than 7 days.
- Use a safe target directory.
- Print which files will be deleted.
- Keep the script understandable for beginners.

---

## 4.1 Safe Version: Preview First

Before deleting anything, always preview the files:

```bash
find /var/log -type f -name "*.log" -mtime +7 -print
```

Explanation:

| Part            | Meaning                       |
| --------------- | ----------------------------- |
| `/var/log`      | Directory to search           |
| `-type f`       | Regular files only            |
| `-name "*.log"` | Only `.log` files             |
| `-mtime +7`     | Modified more than 7 days ago |
| `-print`        | Show matching files           |

---

## 4.2 Cleanup Script

Create the script:

```bash
nano cleanup_logs.sh
```

Add this content:

```bash
#!/bin/bash  --------> shebang

# cleanup_logs.sh
# Deletes .log files older than 7 days from a target directory.

LOG_DIR="/var/log"
DAYS=7

if [ ! -d "$LOG_DIR" ]; then
    echo "Error: Directory $LOG_DIR does not exist."
    exit 1
fi

echo "Searching for .log files older than $DAYS days in $LOG_DIR..."

find "$LOG_DIR" -type f -name "*.log" -mtime +$DAYS -print

echo "Deleting old log files..."

find "$LOG_DIR" -type f -name "*.log" -mtime +$DAYS -delete

echo "Cleanup completed."
```

Make it executable:

```bash
chmod +x cleanup_logs.sh
```

Run it:

```bash
sudo ./cleanup_logs.sh
```

---

## 4.3 Safer Script with Confirmation

This version asks before deleting:

```bash
#!/bin/bash

LOG_DIR="/var/log"
DAYS=7

if [ ! -d "$LOG_DIR" ]; then
    echo "Error: Directory $LOG_DIR does not exist."
    exit 1
fi

echo "The following .log files are older than $DAYS days:"
find "$LOG_DIR" -type f -name "*.log" -mtime +$DAYS -print

read -p "Do you want to delete these files? (y/n): " ANSWER

if [ "$ANSWER" = "y" ]; then
    find "$LOG_DIR" -type f -name "*.log" -mtime +$DAYS -delete
    echo "Old log files deleted."
else
    echo "Cleanup cancelled."
fi
```

---

# 5. Extra Practice Questions

1. What is the difference between a hard link and a soft link?
2. Which command shows the inode number of a file?
3. What is the difference between `>` and `>>`?
4. What is the purpose of `2>`?
5. What does this command do?

```bash
find /var/log -type f -name "*.log" -size +100M
```

6. Why should you preview files before using `rm` or `find -delete`?
7. What is the difference between `stdout` and `stderr`?
8. What does the wildcard `*.log` match?

---

# 6. Quick Command Summary

| Command | Purpose |
|---|---|
| `touch file.txt` | Create an empty file or update timestamp |
| `mkdir dir` | Create a directory |
| `mkdir -p a/b/c` | Create nested directories |
| `cp file dir/` | Copy file |
| `cp -r dir1 dir2` | Copy directory recursively |
| `mv old new` | Rename or move file |
| `rm file` | Delete file |
| `rm -i file` | Delete with confirmation |
| `find . -name "*.log"` | Find `.log` files |
| `grep "ERROR" file.log` | Search text in a file |
| `command > file` | Redirect stdout and overwrite |
| `command >> file` | Redirect stdout and append |
| `command 2> file` | Redirect stderr |
| `command1 \| command2` | Pipe output from command1 to command2 |
| `ln file hardlink` | Create hard link |
| `ln -s file symlink` | Create soft link |

---

