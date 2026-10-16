# Linux File Permissions & Essential Commands

> **DevOps – Lecture 1 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.

---

## Table of Contents
1. [Why Permissions Matter](#1-why-permissions-matter)
2. [Reading Permissions: `ls -l`](#2-reading-permissions-ls--l)
3. [The Three Permissions: r, w, x](#3-the-three-permissions-r-w-x)
4. [Numeric (Octal) Permissions](#4-numeric-octal-permissions)
5. [Changing Permissions: `chmod`](#5-changing-permissions-chmod)
6. [Changing Ownership: `chown` & `chgrp`](#6-changing-ownership-chown--chgrp)
7. [Special Permissions: setuid, setgid, sticky bit](#7-special-permissions-setuid-setgid-sticky-bit)
8. [Essential Commands](#8-essential-commands)
9. [Linux File System Structure](#9-linux-file-system-structure)
10. [Hands-on Practice Lab](#10-hands-on-practice-lab)
11. [Quick Revision Cheat Sheet](#11-quick-revision-cheat-sheet)
12. [Likely Exam / Interview Questions](#12-likely-exam--interview-questions)

---

## 1. Why Permissions Matter

Linux is a **multi-user** OS. Many users and services (web server, database, cron jobs) share one machine, so the system needs a way to answer:

> *"Is **this user** allowed to do **this action** on **this file**?"*

Linux answers by checking two things for every file and directory:
- **Who** is asking? The **owner** (u), the **group** (g), or **others** (o).
- **What** do they want to do? **Read** (r), **write** (w), or **execute** (x).

**Analogy:** think of a file as a room in an office building.
- **Owner** = the person whose office it is
- **Group** = their team
- **Others** = everyone else in the building
- **r / w / x** = can look inside / can rearrange things / can use the equipment

**Why DevOps cares:** wrong permissions cause very common real-world failures:
- `Permission denied` when a deploy script runs
- SSH refusing a private key because it is "too open"
- A web server unable to read its files
- Security holes such as world-writable config files

---

## 2. Reading Permissions: `ls -l`

```bash
$ ls -l
-rwxr-xr-- 1 aarham devs 4096 Oct 5 10:00 deploy.sh
drwxr-x--- 2 aarham devs 4096 Oct 5 10:00 configs
```

Breaking down `-rwxr-xr--`:

```
  -     rwx     r-x     r--
  │      │       │       │
  │      │       │       └── Others : read only
  │      │       └────────── Group  : read + execute
  │      └────────────────── Owner  : read + write + execute
  └───────────────────────── File type
```

**File type character (first position):**

| Char | Meaning |
|------|---------|
| `-`  | Regular file |
| `d`  | Directory |
| `l`  | Symbolic link (shortcut) |
| `c` / `b` | Character / block device (e.g., `/dev/tty`, `/dev/sda`) |

The rest of the line: `1` = link count, `aarham` = **owner**, `devs` = **group**, `4096` = size in bytes, then the modification time and the name.

---

## 3. The Three Permissions: r, w, x

The **same letter means different things on a file and on a directory**. This is the most commonly misunderstood part of the topic.

| Permission | On a **file** | On a **directory** |
|-----------|---------------|--------------------|
| **r** (read)    | View the contents (`cat`, `less`) | List the names inside it (`ls`) |
| **w** (write)   | Modify the contents | Create, delete, or rename files **inside** it |
| **x** (execute) | Run it as a program or script | **Enter** it (`cd`) and access files inside |

### Key insights
- **A directory without `x` is effectively locked.** Even if you know a file's name, you can't reach it.
- **Directory `w` needs `x` to be useful.** To create or delete files inside a directory you need both `w` and `x` on it.
- **➕ Extra: deleting a file depends on the *directory's* permissions, not the file's.** If you have `w+x` on a directory, you can delete a file in it even when that file is read-only to you. (The sticky bit, covered in section 7, exists to stop this.)
- A script needs `x` to run as `./script.sh`. Without `x`, you can still run it with `bash script.sh`, because then `bash` is the program being executed and it only needs to *read* the script.

---

## 4. Numeric (Octal) Permissions

Each permission has a number:

| Permission | Value |
|-----------|-------|
| r | **4** |
| w | **2** |
| x | **1** |
| – (none) | **0** |

You **add** them to get one digit (0–7) per category:

| Digit | Binary | Letters | Meaning |
|-------|--------|---------|---------|
| 7 | 111 | `rwx` | 4+2+1 – everything |
| 6 | 110 | `rw-` | 4+2 – read + write |
| 5 | 101 | `r-x` | 4+1 – read + execute |
| 4 | 100 | `r--` | read only |
| 3 | 011 | `-wx` | write + execute (rare) |
| 2 | 010 | `-w-` | write only (rare) |
| 1 | 001 | `--x` | execute only |
| 0 | 000 | `---` | nothing |

> 💡 **Why 4, 2, 1?** Each permission is one **bit**: r = 2², w = 2¹, x = 2⁰. Three bits give 8 possible values (0–7), which is why the system is called *octal*.

Three digits = **owner, group, others**, in that order:

```
chmod 754 file
       │││
       ││└── others = 4 = r--
       │└─── group  = 5 = r-x
       └──── owner  = 7 = rwx
```

### Common permission values to memorize

| Octal | Symbolic | Typical use |
|-------|----------|-------------|
| **755** | `rwxr-xr-x` | Scripts, programs, directories |
| **644** | `rw-r--r--` | Normal files (configs, HTML, code) |
| **700** | `rwx------` | Private directory (e.g., `~/.ssh`) |
| **600** | `rw-------` | Private files (**SSH private keys**, secrets) |
| **777** | `rwxrwxrwx` | Everyone can do everything. ⚠️ **Almost never correct** |

### Converting between forms

**Symbolic → numeric:** `rw-r-----` → `rw-`=6, `r--`=4, `---`=0 → **640**

**Numeric → symbolic:** `751` → 7=`rwx`, 5=`r-x`, 1=`--x` → **`rwxr-x--x`**

---

## 5. Changing Permissions: `chmod`

`chmod` = **ch**ange **mod**e. There are two ways to use it.

### A) Symbolic mode: *who* + *operator* + *permission*

| Who | Operator | Permission |
|-----|----------|------------|
| `u` = user/owner | `+` add | `r` |
| `g` = group | `-` remove | `w` |
| `o` = others | `=` set exactly | `x` |
| `a` = all (u+g+o) | | |

```bash
chmod u+x deploy.sh        # give the owner execute permission
chmod g-w notes.txt        # remove write from the group
chmod o=r report.pdf       # others get exactly read (anything else is removed)
chmod a+r index.html       # everyone can read
chmod u+x,g-w,o= file      # several changes at once (o= means "others get nothing")
chmod ug+rw shared.txt     # owner and group both get read + write
```

**Best for:** small changes ("just add execute") that leave the other bits alone.

### B) Numeric mode: sets **all** permissions at once

```bash
chmod 755 deploy.sh        # rwxr-xr-x
chmod 644 config.yaml      # rw-r--r--
chmod 600 ~/.ssh/id_rsa    # rw------- (SSH requires this for private keys)
```

**Best for:** setting a known final state.

### Useful flag
```bash
chmod -R 755 my_folder/    # -R = recursive: apply to the folder and everything inside
```
⚠️ Be careful with `-R`. It gives *files* execute permission too, which you usually don't want.

### Symbolic vs. numeric

| | Symbolic | Numeric |
|-|----------|---------|
| Changes | Only the bits you mention | Replaces everything |
| Example | `chmod u+x f` | `chmod 755 f` |
| Readability | Easy for beginners | Faster once memorized |

---

## 6. Changing Ownership: `chown` & `chgrp`

Every file has **one owner** and **one group**.

```bash
chown alice file.txt            # change the owner to alice
chown alice:devs file.txt       # change the owner AND the group
chown :devs file.txt            # change only the group
chown -R www-data:www-data /var/www/html   # recursive (common for web servers)

chgrp devs file.txt             # change only the group
```

- `chown` = **ch**ange **own**er
- `chgrp` = **ch**ange **gr**ou**p**
- ➕ **Extra:** changing a file's owner usually needs **root** (`sudo chown ...`). A normal user can't give their files away to someone else.

**➕ Extra – related user/group commands:**
```bash
whoami                 # who am I?
id                     # my user ID, group ID, and all my groups
groups                 # which groups I belong to
sudo usermod -aG devs alice   # add alice to the devs group
```

---

## 7. Special Permissions: setuid, setgid, sticky bit

On top of `rwx` there are three special bits. They're written as a **4th digit at the front** of the numeric mode.

| Special bit | Octal | Shows up in `ls -l` as | Applies to |
|-------------|-------|------------------------|------------|
| **setuid** | **4**000 | `s` in the **owner's** x spot: `rwsr-xr-x` | Executable files |
| **setgid** | **2**000 | `s` in the **group's** x spot: `rwxr-sr-x` | Files & directories |
| **sticky** | **1**000 | `t` in the **others'** x spot: `rwxrwxrwt` | Directories |

### 🔹 setuid (Set User ID)
**What it does:** whoever runs the program, it runs **with the file owner's privileges**.

**Classic example: `passwd`**
```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root ... /usr/bin/passwd
```
Passwords are stored in `/etc/shadow`, which only root can write. A normal user still needs to change their own password. `passwd` is owned by root and has setuid set, so when you run it, it runs **as root** just long enough to update the file.

```bash
chmod u+s program     # or: chmod 4755 program
```
⚠️ **Security risk:** a buggy setuid-root program can give anyone root access. Use it rarely.

### 🔹 setgid (Set Group ID)
- **On a file:** runs with the **file's group** privileges.
- **➕ Extra – on a directory (the more common use):** new files created inside **inherit the directory's group**, not the creator's default group. This is very useful for **shared team folders**:

```bash
sudo mkdir /projects/team
sudo chgrp devs /projects/team
sudo chmod 2775 /projects/team   # setgid + rwxrwxr-x
# Every file anyone creates in here now belongs to the 'devs' group
```

### 🔹 Sticky bit
**What it does:** in a directory where everyone can write, **only the file's owner** (or the directory owner, or root) can **delete or rename** a file.

**Classic example: `/tmp`**
```bash
$ ls -ld /tmp
drwxrwxrwt 10 root root ... /tmp
```
`/tmp` is `1777`: everyone can create files there (`777`), but the sticky bit (`1`) stops you from deleting other people's files.

```bash
chmod +t shared_dir     # or: chmod 1777 shared_dir
```

> **Remember:** without the sticky bit, anyone with `w` on a directory could delete *any* file in it (see section 3).

### ➕ Extra: lowercase vs. uppercase `s` / `t`
- `s` / `t` (lowercase) = the special bit **and** the underlying `x` are both set.
- `S` / `T` (uppercase) = the special bit is set **but `x` is missing**. This is usually a mistake.

---

## 8. Essential Commands

### 8.1 Navigating the file system

| Command | What it does | Example |
|---------|--------------|---------|
| `pwd` | **P**rint **w**orking **d**irectory (where am I?) | `pwd` → `/home/aarham` |
| `ls` | List directory contents | `ls -al` |
| `cd` | **C**hange **d**irectory | `cd /var/log` |

**Useful `ls` flags:**
```bash
ls -l     # long format: permissions, owner, size, date
ls -a     # show hidden files (names starting with .)
ls -al    # both
ls -lh    # human-readable sizes (4.0K, 2.1M)
ls -ld dir   # info about the directory itself, not its contents
```

**`cd` shortcuts:**
```bash
cd ~      # home directory (also just: cd)
cd ..     # up one level
cd -      # back to the previous directory
cd /      # root of the file system
```

### 8.2 Creating, copying, moving, deleting

| Command | What it does | Example |
|---------|--------------|---------|
| `touch` | Create an empty file, or update the timestamp of an existing one | `touch app.log` |
| `mkdir` ➕ | Make a directory | `mkdir -p a/b/c` (`-p` creates the parent directories too) |
| `cp` | Copy | `cp src.txt dst.txt` |
| `mv` | Move **or rename** (same command) | `mv old.txt new.txt` |
| `rm` ➕ | Remove | `rm file`, `rm -r dir` |

```bash
cp -r project/ backup/      # -r needed to copy directories
mv file.txt /tmp/           # move
mv file.txt renamed.txt     # rename (a move within the same folder)
rm -rf folder/              # ⚠️ force-delete recursively, no undo, no recycle bin
```

### 8.3 Viewing & editing files

| Command | Use it when |
|---------|-------------|
| `cat file` | The file is short. Prints everything at once. (Name comes from con**cat**enate: `cat a b > c` joins files.) |
| `less file` | The file is long. Scroll with arrows, search with `/word`, quit with `q`. |
| `more file` | An older, simpler pager (forward only). Prefer `less` ("less is more"). |
| `head -n 20 file` ➕ | Show the first 20 lines |
| `tail -n 20 file` ➕ | Show the last 20 lines |
| `tail -f app.log` ➕ | **Follow** a log live. Used constantly in DevOps. |

**Editors:**
- **`nano`**: beginner-friendly. Shortcuts are shown at the bottom (`Ctrl+O` save, `Ctrl+X` exit).
- **`vi` / `vim`**: available on almost every server, so worth learning the basics:

| vi key | Action |
|--------|--------|
| `i` | Enter **insert** mode (start typing) |
| `Esc` | Back to **normal** mode |
| `:w` | Save |
| `:q` | Quit |
| `:wq` | Save and quit |
| `:q!` | Quit **without** saving |

### 8.4 Processes & system monitoring

A **process** is a running program. Each one has a unique **PID** (Process ID).

| Command | What it does |
|---------|--------------|
| `ps` | Snapshot of processes in the current terminal |
| `ps aux` | **All** processes on the system, with user, CPU %, and memory % |
| `ps aux \| grep nginx` | Find a specific process |
| `top` | Live, auto-refreshing view of CPU, memory, and processes (`q` to quit) |
| `htop` | Friendlier, colourful `top` (may need installing) |
| `kill PID` | Ask a process to stop |
| `kill -9 PID` | **Force**-kill a process that won't stop |

**➕ Extra – what `kill` really does:** `kill` sends a **signal** to a process.
- `kill PID` sends **SIGTERM (15)**, which asks politely. The program can clean up and save its state.
- `kill -9 PID` sends **SIGKILL (9)**, which the kernel enforces immediately. Nothing gets cleaned up. Use it as a last resort.

### 8.5 Remote access & services

**SSH: log into a remote server securely**
```bash
ssh user@192.168.1.10
ssh -i ~/.ssh/mykey.pem ubuntu@ec2-xx-xx.compute.amazonaws.com   # with a key file (common on AWS)
```
> 🔗 **Ties back to permissions:** SSH **refuses** a private key that others can read:
> `WARNING: UNPROTECTED PRIVATE KEY FILE! Permissions 0644 ... are too open.`
> Fix: `chmod 600 mykey.pem` (and `chmod 700 ~/.ssh`).

**SCP: copy files over SSH**
```bash
scp app.tar.gz user@server:/home/user/           # local  → remote
scp user@server:/var/log/app.log ./              # remote → local
scp -r ./site user@server:/var/www/              # copy a whole directory
```

**systemctl: manage services (background programs, also called daemons)**
```bash
sudo systemctl start nginx      # start now
sudo systemctl stop nginx       # stop now
sudo systemctl restart nginx    # stop + start (e.g., after a config change)
systemctl status nginx          # is it running? recent logs?
sudo systemctl enable nginx     # ➕ start automatically on boot
sudo systemctl disable nginx    # ➕ don't start on boot
```
> `start` affects **now**; `enable` affects **future boots**. You often want both, and `sudo systemctl enable --now nginx` does both in one step.

---

## 9. Linux File System Structure

Everything starts at the **root**, `/`. There are no `C:\` or `D:\` drives. Every disk and device is mounted somewhere inside this one tree.

```
/
├── bin/     → Essential user commands (ls, cp, cat, …)
├── sbin/    → ➕ System/admin commands (reboot, fdisk, …)
├── etc/     → Configuration files (text-based)
├── home/    → Users' personal folders (/home/aarham)
├── root/    → ➕ Home directory of the root user
├── var/     → Variable data that changes: logs, caches, databases
│   └── log/ → System and application logs  ← you'll live here when debugging
├── tmp/     → Temporary files (often cleared on reboot, sticky bit set)
├── usr/     → ➕ Installed software, libraries, docs
├── opt/     → ➕ Optional/third-party software
├── dev/     → ➕ Device files (disks, terminals)
└── proc/    → ➕ Virtual files with live kernel/process info
```

| Directory | Remember it as | DevOps example |
|-----------|----------------|----------------|
| `/bin` | **Bin**aries (programs) | `/bin/bash` |
| `/etc` | The settings folder | `/etc/nginx/nginx.conf`, `/etc/passwd`, `/etc/hosts` |
| `/home` | Users' houses | `/home/ubuntu` on an EC2 instance |
| `/var` | **Var**ies over time | `/var/log/syslog`, `/var/www/html` |
| `/tmp` | Scratch space | Temporary downloads, build files |

> ➕ On modern distros (Ubuntu 20+, Fedora), `/bin` is usually a **symlink** to `/usr/bin`.

**Absolute vs. relative paths:**
- **Absolute**: starts with `/` and works from anywhere: `/var/log/syslog`
- **Relative**: starts from your current directory: `../logs/app.log`

---

## 10. Hands-on Practice Lab

Run these in order and **check the output of `ls -l` after each step**.

```bash
# 1. Setup
mkdir ~/perm_lab && cd ~/perm_lab
touch notes.txt
echo 'echo "Hello from script!"' > hello.sh
ls -l

# 2. Try to run the script. It should fail with "Permission denied"
./hello.sh

# 3. Make it executable and try again
chmod u+x hello.sh
ls -l hello.sh          # -rwxr--r--
./hello.sh              # works

# 4. Practice numeric mode
chmod 600 notes.txt     # rw-------
chmod 644 notes.txt     # rw-r--r--
chmod 000 notes.txt     # ---------
cat notes.txt           # Permission denied, even for you
chmod 644 notes.txt     # restore

# 5. Directory x vs. r
mkdir secret
touch secret/file.txt
chmod 644 secret        # r but no x
ls secret               # may list names, with errors
cd secret               # Permission denied (no x)
chmod 755 secret        # restore

# 6. Sticky bit
mkdir shared
chmod 1777 shared
ls -ld shared           # drwxrwxrwt

# 7. Find a setuid program on your system
ls -l /usr/bin/passwd   # look for the 's'

# 8. Clean up
cd ~ && rm -rf ~/perm_lab
```

**Challenge questions to answer while doing the lab:**
1. After `chmod 000 notes.txt`, could you still **delete** it? Why? *(Hint: section 3.)*
2. What's the octal value of `rwxr-x---`?
3. Why does `bash hello.sh` work even without `x`?

---

## 11. Quick Revision Cheat Sheet

```
PERMISSIONS     r=4  w=2  x=1        order: owner | group | others
FILE            r=view   w=edit           x=run
DIRECTORY       r=list   w=create/delete  x=enter (cd)

COMMON          755 rwxr-xr-x  scripts/dirs
                644 rw-r--r--  normal files
                600 rw-------  SSH keys/secrets
                700 rwx------  private dirs (~/.ssh)

SPECIAL         4xxx setuid  → runs as file owner   (passwd)
                2xxx setgid  → runs as group / dir files inherit group
                1xxx sticky  → only owner can delete (/tmp = 1777)

chmod u+x f     add execute for owner         chmod -R  recursive
chown u:g f     change owner and group        chgrp g f change group

NAVIGATE        pwd  ls -al  cd ~ / .. / -
FILES           touch  mkdir -p  cp -r  mv  rm -r
VIEW            cat  less  head  tail -f
PROCESS         ps aux  top/htop  kill PID  kill -9 PID
REMOTE          ssh user@host   scp src user@host:dst
SERVICES        systemctl start|stop|restart|status|enable <svc>

DIRS            /bin cmds  /etc config  /home users  /var logs+data  /tmp temp
```

---

## 12. Likely Exam / Interview Questions

<details>
<summary><b>Q1. What does <code>chmod 755</code> mean?</b></summary>

Owner = 7 (rwx), group = 5 (r-x), others = 5 (r-x). The owner can do everything; everyone else can read and execute but not modify.
</details>

<details>
<summary><b>Q2. Difference between <code>r</code> and <code>x</code> on a directory?</b></summary>

`r` lets you **list** the names of the files inside. `x` lets you **enter** the directory and access the files inside. You need `x` to `cd` into it.
</details>

<details>
<summary><b>Q3. What is the sticky bit and where is it used?</b></summary>

A directory permission (octal 1000, shown as `t`) that lets only a file's owner delete or rename it, even when the directory is world-writable. It's used on `/tmp` (`1777`).
</details>

<details>
<summary><b>Q4. Why does <code>passwd</code> have setuid set?</b></summary>

Normal users need to update `/etc/shadow`, which only root can write. Setuid makes `passwd` run with its owner's (root's) privileges.
</details>

<details>
<summary><b>Q5. SSH says "permissions are too open" for my key. Fix?</b></summary>

`chmod 600 keyfile`. A private key must be readable only by its owner.
</details>

<details>
<summary><b>Q6. <code>kill</code> vs. <code>kill -9</code>?</b></summary>

`kill` sends SIGTERM, a polite request the process can handle so it shuts down cleanly. `kill -9` sends SIGKILL, which the process can't catch or ignore: it stops immediately with no cleanup.
</details>

<details>
<summary><b>Q7. <code>systemctl start</code> vs. <code>systemctl enable</code>?</b></summary>

`start` runs the service **now**. `enable` makes it start **automatically at boot**.
</details>

<details>
<summary><b>Q8. Convert <code>rw-r-----</code> to octal.</b></summary>

rw- = 6, r-- = 4, --- = 0 → **640**
</details>

<details>
<summary><b>Q9. Where would you look for an app's logs and its config?</b></summary>

Logs: `/var/log/`. Config: `/etc/`.
</details>

<details>
<summary><b>Q10. Why is <code>chmod 777</code> considered bad practice?</b></summary>

Any user on the system can modify or replace the file. For a script, that means anyone can inject commands that run whenever it's executed. Give only the permissions that are actually needed (the *principle of least privilege*).
</details>
