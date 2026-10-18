# Foundations: Linux, Shell, Networking & Git – Advanced MCQs

> **Practice for the first four lectures: [1 – Linux](01-Linux-Permissions-and-Commands.md), [2 – Shell Scripting](02-Shell-Scripting.md), [3 – Networking](03-Networking.md) and [4 – Git](04-Git-Version-Control.md).** Each answer is shown right below its question. Cover it if you want to test yourself first.

| Part | Lecture | Questions |
|------|---------|-----------|
| [A](#part-a-lecture-1--linux-permissions--commands) | 1 – Linux Permissions & Commands | Q1–Q10 |
| [B](#part-b-lecture-2--shell-scripting) | 2 – Shell Scripting | Q11–Q20 |
| [C](#part-c-lecture-3--networking) | 3 – Networking | Q21–Q30 |
| [D](#part-d-lecture-4--git--version-control) | 4 – Git & Version Control | Q31–Q40 |

---

## Part A: Lecture 1 – Linux Permissions & Commands

### Q1. Read but no execute on a directory
You have `r--` (read only, no `x`) on the directory `reports/`. Which of these works?
- A) `cd reports`
- B) `ls reports` (you see the file names, possibly with "Permission denied" for the details)
- C) `cat reports/q1.txt`
- D) `touch reports/new.txt`

> ✅ **Answer: B.** On a directory, `r` lets you **list names**, but `x` is what lets you **enter** it and **reach** files inside. Without `x`, `cd`, `cat` and creating files all fail. · 📖 [L1 §3](01-Linux-Permissions-and-Commands.md#3-the-three-permissions-r-w-x)

### Q2. Deleting a read-only file
`notes.txt` belongs to **bob** and is `r--r--r--` (444). It sits in `shared/`, which is `rwxrwxrwx` (777, **no** sticky bit). You are **alice**. Can you delete `notes.txt`?
- A) No, because the file is read-only for everyone
- B) No, because only bob owns it
- C) Yes, because deleting depends on **write + execute on the directory**, not on the file's permissions
- D) Only with `sudo`

> ✅ **Answer: C.** Deleting a file changes the **directory's** list of names, so the directory's `w+x` decides. (`rm` may ask "remove write-protected file?", but it will succeed.) This is exactly the hole the sticky bit closes. · 📖 [L1 §3](01-Linux-Permissions-and-Commands.md#3-the-three-permissions-r-w-x)

### Q3. The same folder with the sticky bit
Now `shared/` is `rwxrwxrwt` (1777, like `/tmp`). Alice tries `rm shared/notes.txt` (owned by bob). What happens?
- A) It's deleted, because the directory is world-writable
- B) `Operation not permitted`: with the sticky bit, only the file's owner, the directory's owner or root can delete it
- C) The file becomes owned by alice
- D) The sticky bit only affects executables, so nothing changes

> ✅ **Answer: B.** The sticky bit (`t`, octal 1000) stops users deleting or renaming **other people's** files in a shared writable directory. · 📖 [L1 §7](01-Linux-Permissions-and-Commands.md#7-special-permissions-setuid-setgid-sticky-bit)

### Q4. Letters to numbers
What is `rwxr-x--x` in octal?
- A) 755
- B) 751
- C) 741
- D) 571

> ✅ **Answer: B.** `rwx` = 4+2+1 = **7**, `r-x` = 4+1 = **5**, `--x` = **1**. · 📖 [L1 §4](01-Linux-Permissions-and-Commands.md#4-numeric-octal-permissions)

### Q5. `=` in symbolic mode
A file is `rwxrwxrwx`. You run `chmod o=r file`. What are the permissions now?
- A) `rwxrwxrwx`, because `=` only adds
- B) `rwxrwxr--`
- C) `r--r--r--`
- D) `rwxrwxrw-`

> ✅ **Answer: B.** `=` sets **exactly** what's listed for that class (others = read only) and leaves owner and group alone. `+` would add, `-` would remove. · 📖 [L1 §5](01-Linux-Permissions-and-Commands.md#5-changing-permissions-chmod)

### Q6. Uppercase `S`
`ls -l` shows `-rwSr--r--` for a file. What does the capital `S` mean?
- A) setuid is set and the owner can execute: normal for programs like `passwd`
- B) setuid is set but the owner has **no** execute permission, which is usually a mistake
- C) The sticky bit is set
- D) The file is a symbolic link

> ✅ **Answer: B.** Lowercase `s` = special bit **plus** `x`. Uppercase `S` = special bit **without** `x`, so the setuid has no effect. · 📖 [L1 §7](01-Linux-Permissions-and-Commands.md#7-special-permissions-setuid-setgid-sticky-bit)

### Q7. SSH refuses your key
`ssh -i mykey.pem ubuntu@<ip>` fails with `UNPROTECTED PRIVATE KEY FILE! Permissions 0644 ... are too open.` What's the fix?
- A) `chmod 777 mykey.pem`
- B) `chmod 600 mykey.pem`
- C) `chmod +x mykey.pem`
- D) `chown root mykey.pem`

> ✅ **Answer: B.** SSH refuses private keys that group or others can read. `600` = owner read/write only. · 📖 [L1 §8.5](01-Linux-Permissions-and-Commands.md#85-remote-access--services)

### Q8. Why not `kill -9` first?
A process is misbehaving. Why do admins usually try `kill <PID>` **before** `kill -9 <PID>`?
- A) `kill -9` only works for root
- B) `kill` sends SIGTERM, which lets the program **clean up** (save state, close files). SIGKILL stops it instantly with no clean-up.
- C) `kill` is faster
- D) `kill -9` restarts the process

> ✅ **Answer: B.** SIGTERM (15) is a polite request the program can handle. SIGKILL (9) is enforced by the kernel and can't be caught, so it's a last resort. (Docker's `stop` uses the same idea: SIGTERM, then SIGKILL after a timeout.) · 📖 [L1 §8.4](01-Linux-Permissions-and-Commands.md#84-processes--system-monitoring)

### Q9. Gone after a reboot
You ran `sudo systemctl start nginx` and it worked. After the server reboots, nginx isn't running. Why?
- A) The config was reset on reboot
- B) `start` affects **now** only. You also need `enable` (or `enable --now`) to start it at boot.
- C) nginx needs `restart`, not `start`
- D) systemd doesn't support nginx

> ✅ **Answer: B.** `start` / `stop` = the current session. `enable` / `disable` = future boots. · 📖 [L1 §8.5](01-Linux-Permissions-and-Commands.md#85-remote-access--services)

### Q10. A recursive chmod on a website
You run `chmod -R 755 /var/www/html` to fix a website. What's the side effect?
- A) None, because 755 is correct for everything
- B) Every **file** (HTML, CSS, images) also becomes executable, which isn't needed. Directories want 755 and files usually 644.
- C) Directories lose their `x` bit
- D) Ownership changes to root

> ✅ **Answer: B.** `-R` applies the same mode to files and folders. ➕ The usual pattern is: `find /var/www/html -type d -exec chmod 755 {} +` and `find /var/www/html -type f -exec chmod 644 {} +`. · 📖 [L1 §5](01-Linux-Permissions-and-Commands.md#5-changing-permissions-chmod)

---

## Part B: Lecture 2 – Shell Scripting

### Q11. Spaces around `=`
A script contains `NAME = "Aarham"`. What happens when it runs?
- A) `NAME` is set to `Aarham`
- B) Error `NAME: command not found`, because bash treats `NAME` as a command with arguments `=` and `Aarham`
- C) `NAME` is set to `= Aarham`
- D) Syntax error, and the script won't start

> ✅ **Answer: B.** Assignments must have **no spaces**: `NAME="Aarham"`. · 📖 [L2 §4](02-Shell-Scripting.md#4-variables)

### Q12. Single quotes
What does `echo 'Home is $HOME'` print?
- A) `Home is /home/aarham`
- B) `Home is $HOME`
- C) `Home is`
- D) An error

> ✅ **Answer: B.** Single quotes are **literal**: no variable expansion. Double quotes would print the real path. · 📖 [L2 §4](02-Shell-Scripting.md#4-variables)

### Q13. Permission denied, but `bash` works
`./deploy.sh` gives `Permission denied`, but `bash deploy.sh` runs fine. Why?
- A) `bash` runs it as root
- B) `./deploy.sh` needs the **execute** bit. With `bash deploy.sh`, bash is the program being run and only needs to **read** the script.
- C) The shebang line is wrong
- D) `./` only works for files in `/usr/bin`

> ✅ **Answer: B.** Fix with `chmod +x deploy.sh`. (Same reason the CI workflow in L15 runs `chmod +x build.sh`.) · 📖 [L2 §3](02-Shell-Scripting.md#3-running-a-script-scriptsh-vs-sh-scriptsh)

### Q14. `sh` on Ubuntu
A script starts with `#!/bin/bash` and uses `[[ ]]` and `{1..5}`. It works with `./script.sh` but breaks with `sh script.sh` on Ubuntu. Why?
- A) `sh` needs `sudo`
- B) `sh script.sh` **ignores the shebang**, and on Ubuntu `sh` is **dash**, which doesn't support bash-only features
- C) `[[ ]]` was removed from bash
- D) The file needs a `.bash` extension

> ✅ **Answer: B.** `sh file` picks the interpreter itself. Run bash scripts with `./script.sh` or `bash script.sh`. · 📖 [L2 §3](02-Shell-Scripting.md#3-running-a-script-scriptsh-vs-sh-scriptsh)

### Q15. Missing braces
```bash
FILE="report"
echo "$FILE_2024.txt"
```
What's printed?
- A) `report_2024.txt`
- B) `.txt`
- C) `$FILE_2024.txt`
- D) `report.txt`

> ✅ **Answer: B.** Bash reads the variable name as `FILE_2024` (letters, digits and `_` all count), which is unset, so it becomes empty. Write `"${FILE}_2024.txt"`. · 📖 [L2 §4](02-Shell-Scripting.md#4-variables)

### Q16. `>` inside `[ ]`
A script has `if [ $A > 5 ]; then echo big; fi` with `A=2`. What happens?
- A) Nothing is printed, because 2 is not greater than 5
- B) It prints `big` **and** creates a file named `5`, because `>` is a **redirection** here
- C) Syntax error
- D) It compares the values as strings and prints nothing

> ✅ **Answer: B.** Inside `[ ]`, `> 5` redirects output to a file called `5`. What's left is `[ 2 ]`, which is "true" because the string isn't empty. Use `-gt` for numbers: `[ "$A" -gt 5 ]`. · 📖 [L2 §7](02-Shell-Scripting.md#7-conditionals-if--elif--else)

### Q17. `return` with a big number
```bash
add() { return $(( $1 + $2 )); }
add 200 100
echo $?
```
What's printed?
- A) `300`
- B) `44`
- C) `0`
- D) `255`

> ✅ **Answer: B.** `return` gives an **exit status** (0–255), so 300 wraps around to 300 − 256 = **44**. To return data, `echo` it and capture it with `SUM=$(add 200 100)`. · 📖 [L2 §9](02-Shell-Scripting.md#9-functions)

### Q18. Chaining with `&&` and `||`
Which line runs `cd app` **only if** `mkdir app` succeeded?
- A) `mkdir app; cd app`
- B) `mkdir app && cd app`
- C) `mkdir app || cd app`
- D) `mkdir app | cd app`

> ✅ **Answer: B.** `&&` runs the next command only on exit code 0 (success). `||` runs it only on failure. `;` always runs it. `|` pipes output. · 📖 [L2 §10](02-Shell-Scripting.md#10-exit-codes)

### Q19. Counting arguments
`./greet.sh "Ravi Kumar" 21` is run. Inside the script, what are `$#` and `$1`?
- A) `3` and `Ravi`
- B) `2` and `Ravi Kumar`
- C) `2` and `Ravi`
- D) `1` and `Ravi Kumar 21`

> ✅ **Answer: B.** Quotes keep `Ravi Kumar` together as **one** argument, so there are 2 arguments. · 📖 [L2 §6](02-Shell-Scripting.md#6-input-read-and-script-arguments)

### Q20. The order of redirections
What's the difference between `cmd > all.log 2>&1` and `cmd 2>&1 > all.log`?
- A) None
- B) The first sends output **and** errors to `all.log`. The second sends errors to the **terminal** and only normal output to the file.
- C) The second one overwrites the file twice
- D) The first one only saves errors

> ✅ **Answer: B.** Redirections are processed left to right. `2>&1` means "send errors wherever output goes **right now**". In the second form, output still points at the terminal at that moment. · 📖 [L2 §5](02-Shell-Scripting.md#5-command-substitution-storing-a-commands-output)

---

## Part C: Lecture 3 – Networking

### Q21. Subnet maths
For `192.168.10.75/26`, what are the network address, broadcast address and highest usable host?
- A) `.0`, `.255`, `.254`
- B) `.64`, `.127`, `.126`
- C) `.64`, `.128`, `.127`
- D) `.75`, `.127`, `.126`

> ✅ **Answer: B.** /26 → mask `.192` → block size 256 − 192 = **64**. 75 falls in the 64–127 block. Highest host = broadcast − 1 = **.126**. · 📖 [L3 §4](03-Networking.md#4-subnetting--cidr)

### Q22. Same subnet?
Are `10.0.0.120/25` and `10.0.0.130/25` in the same subnet?
- A) Yes, because both start with `10.0.0`
- B) No: /25 splits the last octet at 128, so `.120` is in `0–127` and `.130` is in `128–255`
- C) Yes, because /25 holds 126 hosts
- D) It can't be told without the gateway

> ✅ **Answer: B.** Block size for /25 = 128. Different blocks means different subnets, so the two need a router to talk. · 📖 [L3 §4](03-Networking.md#4-subnetting--cidr)

### Q23. A `169.254.x.x` address
A laptop shows the IP `169.254.23.7`. What does that most likely mean?
- A) It's on a corporate VPN
- B) It never got an address from the **DHCP** server, so it assigned itself a link-local address
- C) It's a public IP
- D) It's the loopback address

> ✅ **Answer: B.** `169.254.0.0/16` is link-local. You see it when DHCP fails. · 📖 [L3 §2](03-Networking.md#2-ip-addresses), [§5](03-Networking.md#5-dhcp-automatic-ip-assignment)

### Q24. Refused vs. timeout
`telnet 13.234.10.5 5432` returns **`Connection refused`** immediately. What does that tell you?
- A) A firewall is silently dropping the traffic
- B) The host is **reachable**, but nothing is listening on port 5432, so check the app or DB
- C) DNS failed
- D) The server is powered off

> ✅ **Answer: B.** **Refused** = the host replied "no one here". **Timeout** = packets are being **dropped** (a firewall or security group). · 📖 [L3 §11.4](03-Networking.md#114-telnet-test-whether-a-port-is-open)

### Q25. No ping, but the site works
`ping` to your EC2 instance gets 100% packet loss, but `curl https://<its-ip>` works. Most likely reason?
- A) The instance is down
- B) The security group doesn't allow **ICMP**, so ping is blocked while HTTPS (TCP 443) is allowed
- C) DNS is broken
- D) HTTPS uses UDP

> ✅ **Answer: B.** "No ping reply" doesn't mean "server down". AWS security groups block ICMP unless you allow it. · 📖 [L3 §11.2](03-Networking.md#112-ping-is-the-host-reachable)

### Q26. TCP or UDP?
Why do video calls and DNS lookups usually use **UDP** rather than TCP?
- A) UDP is encrypted
- B) UDP has no handshake or retransmission, so it's **faster**. A lost packet is better than a late one for live video, and DNS queries are tiny.
- C) UDP guarantees order
- D) TCP can't carry audio

> ✅ **Answer: B.** TCP = reliable and ordered (HTTP, SSH, databases). UDP = fast with no guarantees (DNS, DHCP, streaming, games). · 📖 [L3 §9](03-Networking.md#9-tcp-vs-udp)

### Q27. `unknown host`
`ping api.myapp.com` replies `ping: unknown host api.myapp.com`. Which part failed?
- A) The network cable
- B) **DNS**: the name couldn't be turned into an IP address
- C) The server's firewall
- D) TCP port 443

> ✅ **Answer: B.** The error comes **before** any packet is sent. Check with `nslookup` / `dig`. · 📖 [L3 §11.2](03-Networking.md#112-ping-is-the-host-reachable), [§7](03-Networking.md#7-dns-the-internets-phonebook)

### Q28. Stars in traceroute
`traceroute google.com` shows `3  * * *` at hop 3, but hops 4–8 reply and the destination is reached. What does it mean?
- A) The route is broken at hop 3
- B) Router 3 just doesn't reply to traceroute probes (often filtered). Traffic still passes through it normally.
- C) Packets are looping
- D) DNS failed at hop 3

> ✅ **Answer: B.** If later hops answer, the path works. Only worry when the `*`s continue to the end. · 📖 [L3 §11.3](03-Networking.md#113-traceroute--tracert-which-path-does-traffic-take)

### Q29. Listening on localhost only
An app on a server listens on `127.0.0.1:8080`. `curl localhost:8080` works **on the server**, but from your laptop `curl <server-ip>:8080` is refused. Why?
- A) Port 8080 is reserved
- B) `127.0.0.1` accepts connections **from the same machine only**. Bind to `0.0.0.0` to listen on all interfaces.
- C) The laptop needs IPv6
- D) DNS isn't set up

> ✅ **Answer: B.** It's the same reason a Flask app in Docker needs `host="0.0.0.0"` (L16). · 📖 [L3 §2](03-Networking.md#2-ip-addresses)

### Q30. OSI layers
Which pairing is correct?
- A) Switches and MAC addresses → Layer 3
- B) TCP/UDP and **ports** → Layer 4, IP addresses and **routers** → Layer 3
- C) HTTP → Layer 4
- D) Routers → Layer 2

> ✅ **Answer: B.** L2 = MAC and switches · L3 = IP and routers · L4 = TCP/UDP and ports · L7 = HTTP, DNS, SSH. · 📖 [L3 §8](03-Networking.md#8-the-osi-model)

---

## Part D: Lecture 4 – Git & Version Control

### Q31. `commit -am` and a new file
You edited `app.py` (already tracked) and created `utils.py` (new). You run `git commit -am "Add utils"`. What's in the commit?
- A) Both files
- B) Only `app.py`. `utils.py` stays untracked, because `-a` only stages files Git already tracks.
- C) Only `utils.py`
- D) Nothing: `-am` needs `git add` first

> ✅ **Answer: B.** Run `git add utils.py` (or `git add .`) for new files. · 📖 [L4 §5](04-Git-Version-Control.md#5-staging--committing)

### Q32. Fast-forward
You created `feature` from `main`, made 3 commits on `feature`, and **nobody committed to `main`**. What does `git checkout main && git merge feature` do?
- A) Creates a merge commit with two parents
- B) **Fast-forward**: the `main` pointer simply moves to `feature`'s latest commit, with no merge commit
- C) Rebases `feature` onto `main`
- D) Fails with a conflict

> ✅ **Answer: B.** A merge commit is only needed when **both** branches have new commits (a 3-way merge). · 📖 [L4 §7](04-Git-Version-Control.md#7-merging--merge-conflicts)

### Q33. Reading conflict markers
During `git merge feature` while on `main`, a file shows:
```
<<<<<<< HEAD
timeout = 30
=======
timeout = 60
>>>>>>> feature
```
Which value came from `main`?
- A) `30`
- B) `60`
- C) Neither: both are new
- D) It depends on which commit is newer

> ✅ **Answer: A.** The part between `<<<<<<< HEAD` and `=======` is your **current branch** (`main`), and the part below is the branch being merged in. Keep what you want, delete the markers, `git add`, then `git commit`. · 📖 [L4 §7](04-Git-Version-Control.md#7-merging--merge-conflicts)

### Q34. `reset --soft`
You run `git reset --soft HEAD~1`. What happens to the last commit's changes?
- A) They're deleted
- B) The commit is undone, and its changes stay **staged** (ready to commit again)
- C) The commit is undone, and the changes are **unstaged**
- D) Nothing: `--soft` only previews

> ✅ **Answer: B.** `--soft` keeps them staged, `--mixed` (the default) keeps them unstaged, and `--hard` deletes them. · 📖 [L4 §8](04-Git-Version-Control.md#8-undoing-changes-reset--restore)

### Q35. Undoing a pushed commit
A bad commit is **already pushed** to the shared `main`, and teammates have pulled it. What's the safe way to undo it?
- A) `git reset --hard HEAD~1` then `git push --force`
- B) `git revert <commit-id>` then `git push`: a new commit that undoes it, without rewriting history
- C) Delete the repo and re-clone
- D) `git restore .`

> ✅ **Answer: B.** `reset` + force-push rewrites history that others already have, which breaks their copies. `revert` is safe for shared branches. · 📖 [L4 §8](04-Git-Version-Control.md#8-undoing-changes-reset--restore)

### Q36. `fetch` vs. `pull`
You want to **see** what teammates pushed to `origin/main` **without changing your working files**. Which command?
- A) `git pull`
- B) `git fetch`, then e.g. `git log main..origin/main`
- C) `git push`
- D) `git merge origin/main`

> ✅ **Answer: B.** `fetch` only downloads commits. `pull` = `fetch` + `merge`, which changes your files and may cause conflicts. · 📖 [L4 §11](04-Git-Version-Control.md#11-remotes-fetch-pull-push)

### Q37. Rebasing a shared branch
Why is `git rebase` on a branch that teammates have already pulled considered dangerous?
- A) Rebase deletes the branch
- B) Rebase **rewrites commit IDs**, so the shared history no longer matches what others have, and their next pull or push gets messy
- C) Rebase only works on `main`
- D) Rebase can't handle conflicts

> ✅ **Answer: B.** The golden rule: rebase only your **own local** commits. Use merge for shared branches. · 📖 [L4 §10](04-Git-Version-Control.md#10-rebasing)

### Q38. `.gitignore` too late
You committed `.env` by mistake, then added `.env` to `.gitignore`. `git status` still shows changes to `.env`. Why?
- A) `.gitignore` needs a restart of Git
- B) `.gitignore` only affects **untracked** files. `.env` is already tracked, so run `git rm --cached .env` and commit. (Also rotate any secrets it held: they're still in the history.)
- C) `.gitignore` must be in `.git/`
- D) Dotfiles can't be ignored

> ✅ **Answer: B.** Ignoring doesn't un-track a file. And removing it doesn't erase it from older commits, so treat leaked secrets as compromised. · 📖 [L4 §12](04-Git-Version-Control.md#12-best-practices--gitignore)

### Q39. Cherry-pick
You run `git cherry-pick 7f3a9c1` on `main` (the commit came from `feature-x`). What ends up on `main`?
- A) The whole `feature-x` branch
- B) A **new** commit with the same changes as `7f3a9c1` but a **different** commit ID
- C) The exact same commit `7f3a9c1`, moved from `feature-x`
- D) Nothing until you merge

> ✅ **Answer: B.** Cherry-pick copies **one** commit's changes as a new commit. The original stays on `feature-x`. · 📖 [L4 §9](04-Git-Version-Control.md#9-cherry-picking)

### Q40. Push rejected
`git push origin main` fails with `rejected ... (non-fast-forward)`. What's the right next step?
- A) `git push --force`
- B) `git pull` (or `git pull --rebase`), fix any conflicts, then push again
- C) Delete your local branch
- D) `git reset --hard`

> ✅ **Answer: B.** The remote has commits you don't have yet. Bring them in first. Force-pushing would **erase** your teammates' work. · 📖 [L4 §11](04-Git-Version-Control.md#11-remotes-fetch-pull-push)
