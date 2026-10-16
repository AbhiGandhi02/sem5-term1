# Shell Scripting

> **DevOps – Lecture 2 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 1 – Linux Permissions & Commands](01-Linux-Permissions-and-Commands.md). You'll need `chmod +x` and the basic commands from there.

---

## Table of Contents
1. [What Is Shell Scripting & Why DevOps Needs It](#1-what-is-shell-scripting--why-devops-needs-it)
2. [Your First Script](#2-your-first-script)
3. [Running a Script: `./script.sh` vs. `sh script.sh`](#3-running-a-script-scriptsh-vs-sh-scriptsh)
4. [Variables](#4-variables)
5. [Command Substitution: Storing a Command's Output](#5-command-substitution-storing-a-commands-output)
6. [Input: `read` and Script Arguments](#6-input-read-and-script-arguments)
7. [Conditionals: `if` / `elif` / `else`](#7-conditionals-if--elif--else)
8. [Loops: `for`, `while`, `until`](#8-loops-for-while-until)
9. [Functions](#9-functions)
10. [Exit Codes](#10-exit-codes)
11. [Class Exercises – Solved](#11-class-exercises--solved)
12. [Writing Safer Scripts & Debugging](#12-writing-safer-scripts--debugging)
13. [Hands-on Practice Lab](#13-hands-on-practice-lab)
14. [Quick Revision Cheat Sheet](#14-quick-revision-cheat-sheet)
15. [Likely Exam / Interview Questions](#15-likely-exam--interview-questions)

---

## 1. What Is Shell Scripting & Why DevOps Needs It

- The **shell** (e.g., `bash`) is the program that reads the commands you type in the terminal and runs them.
- A **shell script** is a text file containing those same commands, plus programming logic (variables, `if`, loops, functions), that the shell runs **top to bottom, automatically**.

> **In one line:** shell scripting = **Linux commands + programming logic = automation.**

**Analogy:** typing commands one by one is like cooking while reading each step aloud. A script is the **written recipe**: anyone (or any server, or a cron job) can follow it exactly, every time.

**Why DevOps relies on it:**

| Task | Without a script | With a script |
|------|------------------|---------------|
| Set up 50 servers | Type the same 30 commands 50 times | Run one script 50 times (or in a loop) |
| Check disk space daily | Log in and run `df -h` manually | A script runs it and alerts you when it's above 80% |
| Clean old logs | Remember to do it | A script deletes logs older than 7 days, every night |
| Deploy an app | A manual checklist, easy to skip a step | The same steps, same order, every time |

Shell scripts are also everywhere in DevOps tooling: CI/CD pipeline steps (GitHub Actions, Jenkins), Docker `ENTRYPOINT` scripts, EC2 **user-data** scripts, and so on.

---

## 2. Your First Script

```bash
#!/bin/bash
# hello.sh – my first script

echo "Hello, World!"
```

### Anatomy

| Line | Meaning |
|------|---------|
| `#!/bin/bash` | The **shebang**. It must be the **very first line**. It tells the OS which interpreter should run this file (here, `/bin/bash`). |
| `# ...` | A **comment**. The shell ignores it. Use comments to explain *why*. |
| `echo "..."` | A normal command, exactly as you'd type it in the terminal. |

> ➕ **Shebang details:** `#!` is pronounced "she-bang" (hash + bang). Without it, `./script.sh` runs the file with your *current* shell, which may not be bash. `#!/usr/bin/env bash` is a more portable alternative: it finds `bash` wherever it's installed.

### The full workflow

```bash
nano hello.sh         # 1. write the script (or vi hello.sh)
chmod +x hello.sh     # 2. make it executable  (see Lecture 1)
./hello.sh            # 3. run it
```

---

## 3. Running a Script: `./script.sh` vs. `sh script.sh`

| Method | Needs `x` permission? | Which shell runs it? |
|--------|----------------------|----------------------|
| `./script.sh` | ✅ **Yes** (`chmod +x`) | The one in the **shebang** |
| `sh script.sh` | ❌ No (`sh` only needs to *read* it) | **`sh`**, and the shebang is ignored |
| `bash script.sh` ➕ | ❌ No | **`bash`** |

**Why `./`?** For safety, Linux does **not** look in the current directory for commands. `./` explicitly says "the file in *this* directory." Typing just `hello.sh` gives `command not found`.

**Common error:**
```
$ ./hello.sh
bash: ./hello.sh: Permission denied
```
→ Fix: `chmod +x hello.sh`

> ⚠️ **➕ Extra – `sh` is not always `bash`.** On Ubuntu, `sh` is actually **`dash`**, a smaller shell. Bash-only features such as `[[ ]]`, arrays, and `{1..5}` **break** under `sh`. If you wrote a bash script, run it with `./script.sh` or `bash script.sh`, not `sh script.sh`.

---

## 4. Variables

A variable is a **named box that stores a value** you can reuse later.

### Rules

```bash
NAME="Aarham"        # ✅ correct
NAME = "Aarham"      # ❌ WRONG – no spaces around =
```
> 🔑 **The #1 beginner mistake:** with spaces, bash thinks `NAME` is a *command* and `=` and `"Aarham"` are its arguments. You get `NAME: command not found`.

- **Set:** `VAR=value` (no `$`)
- **Use:** `$VAR` or `${VAR}` (with `$`)
- Names can contain letters, digits, and `_`, but can't start with a digit. They're case-sensitive (`name` ≠ `NAME`).
- All values are stored as **strings**. Bash has no `int` or `float` types.

### Example from class: name, roll number, comment

```bash
#!/bin/bash
NAME="Aarham"
ROLL_NO=42
COMMENT="Learning shell scripting!"

echo "Name: $NAME"
echo "Roll No: $ROLL_NO"
echo "Comment: $COMMENT"
```

### Quotes matter

| Quote | Behaviour | Example | Output |
|-------|-----------|---------|--------|
| `"double"` | Variables **are** expanded | `echo "Hi $NAME"` | `Hi Aarham` |
| `'single'` | Everything is **literal** | `echo 'Hi $NAME'` | `Hi $NAME` |
| none | Expanded, but split on spaces ⚠️ | `echo Hi $NAME` | `Hi Aarham` |

> 💡 **Rule of thumb:** always wrap variables in double quotes (`"$VAR"`). If the value contains spaces (like a file name `my file.txt`), leaving out the quotes breaks things.

### `${VAR}`: when braces are needed
```bash
FILE="report"
echo "$FILE_2024.txt"     # ❌ looks for a variable named FILE_2024
echo "${FILE}_2024.txt"   # ✅ report_2024.txt
```

### ➕ Extra: arithmetic
```bash
A=10
B=3
echo $((A + B))     # 13
echo $((A * B))     # 30
echo $((A / B))     # 3  (integer division only)
echo $((A % B))     # 1  (remainder)
COUNT=$((COUNT + 1))   # increment
```

### ➕ Extra: built-in environment variables
```bash
echo "$USER"      # current username
echo "$HOME"      # home directory
echo "$PWD"       # current directory
echo "$HOSTNAME"  # machine name
echo "$PATH"      # where the shell looks for commands
```

---

## 5. Command Substitution: Storing a Command's Output

This is how a script **captures the output of a command** into a variable. Class exercises like "store today's date in a variable" depend on it.

```bash
TODAY=$(date)
echo "Today is: $TODAY"
```

| Syntax | Notes |
|--------|-------|
| `$(command)` | ✅ Modern, readable, can be nested |
| `` `command` `` | Old backtick style. Works, but harder to read |

**Useful examples:**
```bash
DAY=$(date +%A)              # Monday
DATE=$(date +%Y-%m-%d)       # 2026-10-05
ME=$(whoami)                 # aarham
HOST=$(hostname)             # ip-172-31-5-10
FILES=$(ls | wc -l)          # number of files in the current directory
DISK=$(df -h / | tail -1 | awk '{print $5}')   # root disk usage, e.g. 42%
```

### Redirection: sending output to a file
```bash
echo "hello" > file.txt     # >  overwrite the file
echo "world" >> file.txt    # >> append to the file
ps aux > processes.txt      # save the process list to a file
command 2> errors.log       # ➕ 2> redirects errors (stderr)
command > all.log 2>&1      # ➕ output and errors together
```

---

## 6. Input: `read` and Script Arguments

### A) Asking the user with `read`
```bash
#!/bin/bash
read -p "Enter your name: " NAME
echo "Welcome, $NAME!"
```
- `-p` shows a prompt on the same line.
- ➕ `read -s` hides the input (for passwords).

### B) ➕ Extra: passing arguments when running the script
```bash
./greet.sh Aarham 21
```
Inside the script:

| Variable | Meaning | Value here |
|----------|---------|------------|
| `$0` | Script name | `./greet.sh` |
| `$1`, `$2`, … | 1st, 2nd, … argument | `Aarham`, `21` |
| `$#` | **Number** of arguments | `2` |
| `$@` | **All** arguments (as separate words) | `Aarham 21` |
| `$?` | Exit code of the **last** command | (see section 10) |

```bash
#!/bin/bash
echo "Hello $1, you are $2 years old"
echo "You passed $# arguments"
```
> Arguments are preferred over `read` in automation, because CI pipelines and cron jobs can't type answers at a prompt.

---

## 7. Conditionals: `if` / `elif` / `else`

### Syntax
```bash
if [ condition ]; then
    # runs if condition is true
elif [ other_condition ]; then
    # runs if the first was false and this is true
else
    # runs if nothing above was true
fi          # "if" backwards ends the block
```

> ⚠️ **The spaces inside `[ ]` are required.** `[ $A -gt 5 ]` ✅  `[$A -gt 5]` ❌
> Why: `[` is actually a **command** (another name for `test`), and commands need spaces around their arguments.

### Comparison operators

**Numbers:**

| Operator | Meaning | Memory aid |
|----------|---------|------------|
| `-eq` | equal | **eq**ual |
| `-ne` | not equal | **n**ot **e**qual |
| `-gt` | greater than | **g**reater **t**han |
| `-ge` | greater or equal | **g**reater/**e**qual |
| `-lt` | less than | **l**ess **t**han |
| `-le` | less or equal | **l**ess/**e**qual |

**Strings:**

| Operator | Meaning |
|----------|---------|
| `"$A" = "$B"` | equal (`==` also works in bash) |
| `"$A" != "$B"` | not equal |
| `-z "$A"` | string is **empty** (zero length) |
| `-n "$A"` | string is **not** empty |

> ⚠️ Inside `[ ]`, **don't** use `>` or `<` for numbers. `>` means *redirect to a file*. Use `-gt` and `-lt`.

**➕ Files** (very common in DevOps scripts):

| Operator | True if… |
|----------|----------|
| `-e path` | exists (file or directory) |
| `-f path` | exists and is a **regular file** |
| `-d path` | exists and is a **directory** |
| `-r` / `-w` / `-x path` | readable / writable / executable (ties back to Lecture 1) |
| `-s path` | exists and is **not empty** |

**Combining:** `&&` (AND), `||` (OR), `!` (NOT)
```bash
if [ "$AGE" -ge 18 ] && [ "$AGE" -le 60 ]; then echo "Working age"; fi
```

### Examples

```bash
#!/bin/bash
read -p "Enter a number: " NUM

if [ "$NUM" -gt 0 ]; then
    echo "Positive"
elif [ "$NUM" -lt 0 ]; then
    echo "Negative"
else
    echo "Zero"
fi
```

```bash
#!/bin/bash
# Even or odd
read -p "Enter a number: " N
if [ $((N % 2)) -eq 0 ]; then
    echo "$N is even"
else
    echo "$N is odd"
fi
```

```bash
#!/bin/bash
# DevOps-style: create a directory only if it doesn't exist
DIR="/tmp/backups"
if [ ! -d "$DIR" ]; then
    mkdir -p "$DIR"
    echo "Created $DIR"
else
    echo "$DIR already exists"
fi
```

### ➕ Extra: `[[ ]]` (bash's improved test)
`[[ ]]` handles unquoted variables more safely and supports `&&`, `||`, and patterns inside the brackets:
```bash
if [[ $NAME == A* && $AGE -gt 18 ]]; then echo "Matches"; fi
```
It's bash-only, so it won't work under `sh` (see section 3).

### ➕ Extra: `case` (a cleaner alternative to long `elif` chains)
```bash
read -p "Action (start/stop/status): " ACTION
case "$ACTION" in
    start)  echo "Starting..." ;;
    stop)   echo "Stopping..." ;;
    status) echo "Checking..." ;;
    *)      echo "Unknown action" ;;     # * = default
esac
```

---

## 8. Loops: `for`, `while`, `until`

Loops **repeat a block of code**, either over a list of items or while a condition holds.

### `for`: loop over a list
```bash
# Over explicit items
for FRUIT in apple banana mango; do
    echo "I like $FRUIT"
done

# Over a range
for i in {1..5}; do
    echo "Number $i"
done

# C-style
for ((i = 1; i <= 5; i++)); do
    echo "Count $i"
done

# Over files (very common)
for FILE in *.log; do
    echo "Processing $FILE"
done

# Over a list of servers
for SERVER in web1 web2 db1; do
    echo "Pinging $SERVER"
    ping -c 1 "$SERVER" > /dev/null && echo "  up" || echo "  DOWN"
done
```

### `while`: loop *while* a condition is true
```bash
COUNT=1
while [ $COUNT -le 5 ]; do
    echo "Count: $COUNT"
    COUNT=$((COUNT + 1))     # ⚠️ forget this → infinite loop
done
```

**Read a file line by line:**
```bash
while read -r LINE; do
    echo "Line: $LINE"
done < servers.txt
```

### `until`: loop *until* a condition becomes true (the opposite of `while`)
```bash
# Wait until a service is up
until systemctl is-active --quiet nginx; do
    echo "Waiting for nginx..."
    sleep 2
done
echo "nginx is up!"
```

### Loop control
| Keyword | Effect |
|---------|--------|
| `break` | Exit the loop immediately |
| `continue` | Skip to the next iteration |

```bash
for i in {1..10}; do
    [ $i -eq 3 ] && continue    # skip 3
    [ $i -eq 6 ] && break       # stop at 6
    echo $i                     # prints 1 2 4 5
done
```

> 🛑 Stuck in an infinite loop? Press **`Ctrl + C`**.

---

## 9. Functions

A function is a **named, reusable block of code**. You write it once and call it many times.

### Syntax
```bash
greet() {
    echo "Hello, $1!"
}

greet "Aarham"     # Hello, Aarham!
greet "Riya"       # Hello, Riya!
```

- **Define the function before you call it.** The script runs top to bottom.
- You call it **without parentheses**: `greet`, not `greet()`.
- Inside a function, `$1`, `$2`, `$#`, and `$@` refer to the **function's** arguments, not the script's.

### Local variables
```bash
counter() {
    local COUNT=10     # only exists inside this function
    echo "Inside: $COUNT"
}
counter
echo "Outside: $COUNT"   # empty
```
> Without `local`, variables in functions are **global** and can accidentally overwrite variables elsewhere in the script.

### Returning values
Bash functions don't return values the way Python functions do. There are two patterns:

```bash
# 1) "Return" data by echoing it, then capture it with $(...)
add() {
    echo $(( $1 + $2 ))
}
SUM=$(add 5 7)
echo "Sum: $SUM"       # Sum: 12

# 2) `return` gives only a status code (0–255): 0 = success, anything else = failure
is_root() {
    [ "$(id -u)" -eq 0 ]       # the function's status = this test's result
}
if is_root; then echo "Running as root"; else echo "Not root"; fi
```

### A DevOps-style example
```bash
#!/bin/bash
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

check_service() {
    if systemctl is-active --quiet "$1"; then
        log "$1 is running ✅"
    else
        log "$1 is DOWN ❌"
    fi
}

for SVC in nginx ssh cron; do
    check_service "$SVC"
done
```

---

## 10. Exit Codes

➕ Every command finishes with an **exit code** (also called an exit status):
- **`0` = success**
- **Anything else (1–255) = failure**

> ⚠️ This is the opposite of most programming languages, where 0 means false. In the shell, **0 means "it worked."**

```bash
ls /etc > /dev/null
echo $?        # 0

ls /nope 2> /dev/null
echo $?        # 2 (non-zero = failed)
```

**Using exit codes in scripts:**
```bash
# Stop the script with an error if something's wrong
if [ ! -f config.yaml ]; then
    echo "config.yaml not found!"
    exit 1
fi

# Short-circuit chaining
mkdir app && cd app        # cd runs ONLY if mkdir succeeded
ping -c1 google.com || echo "No internet"   # echo runs ONLY if ping failed
```

**Why DevOps cares:** CI/CD tools like Jenkins and GitHub Actions decide whether a step **passed or failed by its exit code**. If your script hits an error but still exits with 0, the pipeline shows green ✅ even though something broke.

> `/dev/null` is a "black hole" file: anything written to it is thrown away. Use it to silence output you don't need.

---

## 11. Class Exercises – Solved

### Exercise 1: `hello.sh`: create a folder and a file, write to it, display it

```bash
#!/bin/bash
# hello.sh

mkdir -p myfolder                         # -p: no error if it already exists
cd myfolder || exit 1                     # stop if cd fails
touch notes.txt
echo "Hello from my first shell script!" > notes.txt
echo "This line was appended." >> notes.txt
cat notes.txt
```
Run it:
```bash
chmod +x hello.sh
./hello.sh
```
Output:
```
Hello from my first shell script!
This line was appended.
```

### Exercise 2: print the day using a variable

```bash
#!/bin/bash
TODAY=$(date +%A)
FULL_DATE=$(date +"%d-%m-%Y")
echo "Today is $TODAY ($FULL_DATE)"
```
Output: `Today is Monday (05-10-2026)`

**Common `date` format codes:**

| Code | Meaning | Example |
|------|---------|---------|
| `%A` | Day name | Monday |
| `%d` | Day of month | 05 |
| `%m` | Month number | 10 |
| `%B` | Month name | October |
| `%Y` | Year | 2026 |
| `%H:%M:%S` | Time | 14:30:05 |

### Exercise 3: print username and hostname, save the process list to a file

```bash
#!/bin/bash
USERNAME=$(whoami)
HOST=$(hostname)
OUTFILE="processes_$(date +%Y%m%d_%H%M%S).txt"

echo "User:     $USERNAME"
echo "Hostname: $HOST"

ps aux > "$OUTFILE"
echo "Process list saved to $OUTFILE ($(wc -l < "$OUTFILE") lines)"
```

### Exercise 4: system admin report with `df` and `ps`

```bash
#!/bin/bash
# sysreport.sh – a quick system health snapshot

echo "===== System Report: $(date) ====="
echo "Host: $(hostname)   User: $(whoami)"
echo

echo "--- Disk Usage ---"
df -h                      # -h = human-readable (G, M)
echo

echo "--- Memory ---"
free -h
echo

echo "--- Top 5 processes by CPU ---"
ps aux --sort=-%cpu | head -n 6    # header + 5 rows
```

### ➕ Bonus: disk-usage alert (a real-world DevOps script)

```bash
#!/bin/bash
# disk_alert.sh – warn when root disk usage crosses a threshold

THRESHOLD=80
USAGE=$(df / | tail -1 | awk '{print $5}' | tr -d '%')

if [ "$USAGE" -ge "$THRESHOLD" ]; then
    echo "⚠️  ALERT: Disk usage is ${USAGE}% (threshold ${THRESHOLD}%)"
    exit 1
else
    echo "✅ Disk usage OK: ${USAGE}%"
fi
```
How the pipeline line works:
```
df /           → table with a header row and a row for /
| tail -1      → keep only the last line (the / row)
| awk '{print $5}'  → take the 5th column: "42%"
| tr -d '%'    → delete the % sign: "42"
```

> ➕ **Run it automatically with cron:** `crontab -e`, then add
> `0 * * * * /home/ubuntu/disk_alert.sh >> /home/ubuntu/disk.log 2>&1`
> to run it **every hour**. (cron format: `minute hour day month weekday command`)

---

## 12. Writing Safer Scripts & Debugging

### ➕ Strict mode: put this at the top of serious scripts
```bash
#!/bin/bash
set -euo pipefail
```

| Flag | Effect |
|------|--------|
| `-e` | **Exit** as soon as any command fails |
| `-u` | Treat **unset** variables as an error (catches typos like `$NMAE`) |
| `-o pipefail` | A pipeline fails if **any** part fails, not just the last command |

**Why:** without `-e`, the script carries on after an error. Imagine `cd /app/build` fails and the next line is `rm -rf *`: it would delete files in the wrong directory.

### ➕ Debugging
```bash
bash -x script.sh       # print each command before running it (with values filled in)
```
Or turn tracing on and off for part of a script:
```bash
set -x    # start tracing
# ... suspicious code ...
set +x    # stop tracing
```

### ➕ Lint with ShellCheck
`shellcheck script.sh` finds common bugs (missing quotes, bad `[ ]` syntax, and more). You can install it (`sudo apt install shellcheck`) or use [shellcheck.net](https://www.shellcheck.net).

### Common mistakes checklist

| ❌ Mistake | ✅ Fix |
|-----------|-------|
| `NAME = "x"` | `NAME="x"` (no spaces) |
| `[$A -gt 5]` | `[ $A -gt 5 ]` (spaces inside brackets) |
| `if [ $A > 5 ]` | `if [ $A -gt 5 ]` (`>` redirects to a file!) |
| `echo $FILE` with spaces in the name | `echo "$FILE"` (quote variables) |
| `Permission denied` | `chmod +x script.sh` |
| `bad interpreter: /bin/bash^M` | The file has Windows line endings. Fix with `dos2unix script.sh` or `sed -i 's/\r$//' script.sh` |
| `[[: not found` | You ran a bash script with `sh`. Use `bash` or `./` |
| Forgetting `fi` / `done` / `esac` | Every block needs its closing keyword |

---

## 13. Hands-on Practice Lab

Practise on your **EC2 instance** (as suggested in class) or any Linux terminal. Keep a Linux command cheat sheet open while you work.

```bash
mkdir ~/script_lab && cd ~/script_lab
```

**Tasks: write each as its own `.sh` file:**

1. **`info.sh`**: print your name, roll number, and a comment, using three variables.
2. **`day.sh`**: print `Today is <Day>, <date>` using command substitution.
3. **`whoami.sh`**: print the username and hostname, and save `ps aux` to `processes.txt`.
4. **`evenodd.sh`**: read a number and print whether it's even or odd.
5. **`table.sh`**: take a number as `$1` and print its multiplication table (1–10) with a `for` loop.
6. **`filecheck.sh`**: take a path as `$1` and say whether it's a file, a directory, or doesn't exist. Exit with code 1 if it doesn't exist.
7. **`backup.sh`**: copy all `.txt` files in the current directory into `backup_<date>/`.
8. **`menu.sh`**: a `case`-based menu: `1) disk usage  2) memory  3) top processes  4) exit`, inside a `while` loop.
9. **Functions**: refactor `sysreport.sh` (section 11) so each section is a function (`show_disk`, `show_memory`, `show_procs`).

<details>
<summary><b>Solution: Task 5 (table.sh)</b></summary>

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
    echo "Usage: $0 <number>"
    exit 1
fi
for i in {1..10}; do
    echo "$1 x $i = $(( $1 * i ))"
done
```
</details>

<details>
<summary><b>Solution: Task 6 (filecheck.sh)</b></summary>

```bash
#!/bin/bash
P="$1"
if [ -f "$P" ]; then
    echo "$P is a file"
elif [ -d "$P" ]; then
    echo "$P is a directory"
else
    echo "$P does not exist"
    exit 1
fi
```
</details>

<details>
<summary><b>Solution: Task 7 (backup.sh)</b></summary>

```bash
#!/bin/bash
DEST="backup_$(date +%Y%m%d)"
mkdir -p "$DEST"
for F in *.txt; do
    [ -e "$F" ] || continue     # skip if there are no .txt files
    cp "$F" "$DEST/"
    echo "Backed up $F"
done
```
</details>

<details>
<summary><b>Solution: Task 8 (menu.sh)</b></summary>

```bash
#!/bin/bash
while true; do
    echo "1) Disk usage  2) Memory  3) Top processes  4) Exit"
    read -p "Choose: " CH
    case "$CH" in
        1) df -h ;;
        2) free -h ;;
        3) ps aux --sort=-%cpu | head -6 ;;
        4) echo "Bye!"; break ;;
        *) echo "Invalid choice" ;;
    esac
done
```
</details>

---

## 14. Quick Revision Cheat Sheet

```
SHEBANG       #!/bin/bash                   (first line, picks the interpreter)
RUN           chmod +x s.sh && ./s.sh       or  bash s.sh  (no x needed)

VARIABLES     NAME="x"   (NO spaces)        use: "$NAME"  "${NAME}_suffix"
CMD SUBST     NOW=$(date)                   MATH: $((A + B))
QUOTES        "expands $VAR"   'literal $VAR'
INPUT         read -p "Prompt: " VAR
ARGS          $0 name  $1..$9 args  $# count  $@ all  $? last exit code
REDIRECT      > overwrite   >> append   2> errors   &> /dev/null discard

IF            if [ cond ]; then ... elif [ cond ]; then ... else ... fi
NUM OPS       -eq -ne -gt -ge -lt -le
STR OPS       =  !=  -z (empty)  -n (not empty)
FILE OPS      -e exists  -f file  -d dir  -r -w -x  -s non-empty
LOGIC         &&  ||  !
CASE          case "$X" in a) ... ;; *) ... ;; esac

FOR           for i in 1 2 3; do ... done      for i in {1..5}   for f in *.log
WHILE         while [ cond ]; do ... done
UNTIL         until [ cond ]; do ... done
CONTROL       break  continue      (Ctrl+C to escape an infinite loop)

FUNCTION      name() { local X=$1; echo "$X"; }      call: name arg1
RETURN DATA   R=$(name arg)        STATUS: return 0/1

EXIT CODES    0 = success, non-0 = failure     exit 1 to fail the script
CHAINING      a && b (b if a ok)    a || b (b if a fails)

SAFETY        set -euo pipefail     DEBUG: bash -x s.sh     LINT: shellcheck
```

---

## 15. Likely Exam / Interview Questions

<details>
<summary><b>Q1. What is a shebang and why is it needed?</b></summary>

The first line of a script, starting with `#!` (e.g., `#!/bin/bash`). It tells the OS which interpreter to run the script with when it's executed directly as `./script.sh`.
</details>

<details>
<summary><b>Q2. Difference between <code>./script.sh</code> and <code>sh script.sh</code>?</b></summary>

`./script.sh` needs execute permission and uses the interpreter named in the shebang. `sh script.sh` only needs read permission, ignores the shebang, and runs the script with `sh` (which may be `dash`, not `bash`).
</details>

<details>
<summary><b>Q3. Why does <code>NAME = "John"</code> fail?</b></summary>

With spaces, bash reads `NAME` as a command name with `=` and `"John"` as its arguments, so you get "command not found". Assignments must have no spaces: `NAME="John"`.
</details>

<details>
<summary><b>Q4. Single vs. double quotes?</b></summary>

Double quotes expand variables and command substitutions (`"$HOME"` → `/home/user`). Single quotes keep everything literal (`'$HOME'` → `$HOME`).
</details>

<details>
<summary><b>Q5. How do you store a command's output in a variable?</b></summary>

With command substitution: `VAR=$(command)`, e.g., `TODAY=$(date +%A)`.
</details>

<details>
<summary><b>Q6. What do <code>$#</code>, <code>$@</code>, <code>$?</code>, and <code>$0</code> mean?</b></summary>

`$#` = number of arguments, `$@` = all arguments, `$?` = exit code of the last command, `$0` = script name.
</details>

<details>
<summary><b>Q7. What does exit code 0 mean? Why does it matter in CI/CD?</b></summary>

0 = success; non-zero = failure. CI/CD tools decide whether a step passed by its exit code, so scripts must `exit 1` (non-zero) when something fails.
</details>

<details>
<summary><b>Q8. Why use <code>-gt</code> instead of <code>&gt;</code> inside <code>[ ]</code>?</b></summary>

Inside `[ ]`, `>` is treated as output redirection and creates a file. Number comparisons use `-gt`, `-lt`, `-eq`, and so on.
</details>

<details>
<summary><b>Q9. <code>while</code> vs. <code>until</code>?</b></summary>

`while` keeps looping as long as the condition is **true**. `until` keeps looping as long as the condition is **false** (it stops once the condition becomes true).
</details>

<details>
<summary><b>Q10. How does a bash function return a value?</b></summary>

`return` only sets a numeric status (0–255). To return data, `echo` it inside the function and capture it with `RESULT=$(func args)`.
</details>

<details>
<summary><b>Q11. What does <code>set -e</code> do?</b></summary>

It makes the script exit immediately when any command fails, instead of continuing in a broken state.
</details>

<details>
<summary><b>Q12. Write a one-liner to count the lines in all <code>.log</code> files in the current directory.</b></summary>

```bash
for f in *.log; do echo "$f: $(wc -l < "$f")"; done
```
</details>
