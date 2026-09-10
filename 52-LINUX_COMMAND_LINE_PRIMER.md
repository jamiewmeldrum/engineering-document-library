# Linux & the Command Line — A Primer №52

*The operating system almost everything you write runs on, and the interface you'll use to understand it when something goes wrong. Underpins containers (№50 — a container **is** a Linux process with restricted views), servers, CI runners, and every production incident. Written for someone who can find their way around a terminal but has never been taught the model underneath.*

The organising idea, and the one that makes Linux coherent rather than a pile of commands: **almost everything is a file.** Regular files, directories, devices, network sockets, running processes, kernel parameters — all exposed through the same filesystem interface, all readable and writable with the same tools. `/proc/1234/status` is a "file" that reports on process 1234. `/dev/null` is a "file" that discards anything written to it. That uniformity is why a small set of tools composes into an enormous range of capability.

The second idea, which explains the *shape* of the tooling: **each program does one thing, and programs compose through pipes.** There is no giant Linux "system tool" — there's `grep`, and `sort`, and `cut`, and the ability to chain them. Learning Linux is mostly learning a modest vocabulary of small tools plus the grammar that joins them.

Contents:

- **Part 1** — the model: kernel, userspace, everything-is-a-file
- **Part 2** — the filesystem
- **Part 3** — processes and signals
- **Part 4** — users, permissions and ownership
- **Part 5** — the shell: pipes, redirection, expansion, quoting
- **Part 6** — the essential toolkit
- **Part 7** — text processing: grep, sed, awk
- **Part 8** — inspecting a running system
- **Part 9** — services, packages, and the boot picture
- **Part 10** — shell scripting
- **Part 11** — troubleshooting playbook
- **Part 12** — when to use what

## Task index

| I need to… | Reach for | §|
|---|---|---|
| Find files by name/size/age | `find` | §6.2 |
| Search text inside files | `grep -r` | §7.1 |
| See what's using the disk | `du -sh *` / `df -h` | §8.4 |
| See what's using CPU/memory | `top` / `htop` / `ps aux` | §8.1 |
| Find what's listening on a port | `ss -tlnp` | §8.5 |
| Find which process holds a file | `lsof` | §8.5 |
| Kill something | `kill` / `pkill` | §3.4 |
| Follow a log as it grows | `tail -f` | §6.3 |
| Extract a column from output | `awk '{print $2}'` / `cut` | §7.3 |
| Replace text across files | `sed -i` | §7.2 |
| Run something after I log out | `nohup` / `systemd` / `tmux` | §3.5 |
| Fix "permission denied" | `chmod` / `chown` / check ownership | §4 |
| See why a service won't start | `systemctl status` / `journalctl -u` | §9.1 |
| Understand a container's process | `ps` inside it; it's just a process | №50 §2 |

---

# Part 1 — The model

## 1.1 Kernel and userspace

The **kernel** manages hardware, memory, processes and filesystems, and enforces isolation. Your programs run in **userspace** and cannot touch hardware directly — they ask the kernel via **system calls** (`open`, `read`, `write`, `fork`, `execve`). That boundary is the fundamental protection in the system: a userspace crash takes down a process, not the machine.

This matters for you concretely: **containers share the host kernel** (№50 §1.2). A container isn't a machine — it's userspace processes with restricted views, enforced by kernel features. So Linux knowledge transfers directly to debugging containers, and a kernel-level vulnerability crosses container boundaries in a way it wouldn't cross VM boundaries.

## 1.2 Everything is a file

The unifying abstraction. A **file descriptor** is a small integer identifying an open resource within a process, and the kernel gives every process three by default:

| FD | Name | Default |
|---|---|---|
| **0** | stdin | keyboard |
| **1** | stdout | terminal |
| **2** | stderr | terminal |

That numbering is why `2>&1` (§5.2) means "send stderr wherever stdout is going." Regular files, pipes, sockets, devices and terminals are all file descriptors, all read and written the same way — which is the mechanism that makes redirection and piping work universally.

## 1.3 The distributions

Same kernel, different packaging. **Debian/Ubuntu** (`apt`, `.deb`) — the common default, and what your containers likely run. **RHEL/Fedora/Amazon Linux** (`dnf`/`yum`, `.rpm`) — enterprise and AWS. **Alpine** (`apk`, musl libc) — tiny, popular for containers, but its **musl** C library instead of glibc causes subtle breakage with the JVM and pre-compiled native binaries (№50 §7.6). Differences are mostly package manager, init details and paths; the model above is universal.

---

# Part 2 — The filesystem

## 2.1 One tree

There are no drive letters. Everything hangs off `/`, and additional disks are **mounted** at a directory within that tree.

| Path | Holds |
|---|---|
| `/` | the root of everything |
| `/home/jamie` | user home directories (`~`) |
| `/etc` | system-wide **configuration** (text files) |
| `/var` | variable data — **`/var/log`** especially |
| `/tmp` | temporary files, cleared on reboot |
| `/usr/bin`, `/usr/local/bin` | executables |
| `/opt` | optional/third-party software |
| `/proc` | **virtual** — live kernel and process state |
| `/sys` | **virtual** — devices and kernel parameters |
| `/dev` | device files (`/dev/null`, `/dev/random`) |
| `/mnt`, `/media` | mount points |

**`/proc` is worth knowing about** because it's the everything-is-a-file idea at its most useful: `/proc/cpuinfo`, `/proc/meminfo`, `/proc/<pid>/status`, `/proc/<pid>/environ`, `/proc/<pid>/fd/`. It isn't on disk — the kernel synthesises it on read. Most tools in Part 8 are readable front-ends to `/proc`.

## 2.2 Paths and navigation

```bash
pwd                     # where am I
cd /var/log             # absolute path (starts at /)
cd ../config            # relative
cd ~                    # home
cd -                    # previous directory
ls -lah                 # long, all (incl. hidden), human-readable sizes
tree -L 2               # directory structure, 2 levels
```

Files beginning with `.` are hidden by convention (`.gitignore`, `.env`) — only convention, but `ls` respects it.

## 2.3 File operations

```bash
cp -r src/ dest/                 # recursive copy
mv old new                       # move OR rename (same operation)
rm -rf dir/                      # recursive force delete — no undo, no bin
mkdir -p a/b/c                   # create parents as needed
ln -s /path/to/target linkname   # symbolic link
touch file                       # create empty / update timestamp
```

**`rm -rf` is genuinely irreversible.** There's no trash. Habits worth having: type the path before the flags, use `ls` on the glob first to see what it matches, and never build the path by variable substitution in a script without checking the variable isn't empty (`rm -rf "$DIR/"` with an unset `DIR` is the classic catastrophe).

**Hard link vs symbolic link:** a hard link is another name for the *same inode* (the same data — deleting one name leaves the data intact while any name remains); a symlink is a small file *pointing at a path* (delete the target and the link dangles). Symlinks can cross filesystems and point at directories; hard links can't.

---

# Part 3 — Processes and signals

## 3.1 What a process is

A running program with its own memory space, file descriptors, environment and a **PID**. Every process (except PID 1) has a parent, forming a tree — `pstree` shows it. When a process starts another, it typically `fork`s (clones itself) then `execve`s (replaces its image with the new program).

**PID 1** is the init process (usually systemd). It adopts orphaned processes and reaps zombies. **In a container, your application is PID 1** (№50 §2.2), which has two consequences people hit: it receives signals directly (so it must handle SIGTERM to shut down gracefully), and it inherits the duty of reaping child processes (which a JVM or web server doesn't do, hence `--init` or `tini` when your container spawns children).

## 3.2 Process state

```bash
ps aux                          # all processes, detailed
ps -ef --forest                 # as a tree
pgrep -a java                   # find by name
top                             # live view
htop                            # nicer live view (usually needs installing)
```

Reading `ps aux` columns: `USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND`. **RSS** (resident set size) is the physical memory actually in use — the number you usually care about; **VSZ** is virtual address space and is routinely huge and misleading for JVMs. **STAT** shows state: `R` running, `S` sleeping (normal), `D` uninterruptible sleep (usually blocked on I/O — a pile of these means a storage problem), `Z` zombie, `T` stopped.

## 3.3 Foreground, background, jobs

```bash
./long-task &            # start in background
jobs                     # list this shell's jobs
fg %1                    # bring job 1 to foreground
Ctrl+Z                   # suspend the foreground job
bg %1                    # resume it in the background
Ctrl+C                   # send SIGINT (usually terminates)
```

## 3.4 Signals

Signals are how you ask a process to do something:

| Signal | Number | Means | Catchable |
|---|---|---|---|
| **SIGTERM** | 15 | please terminate (**the polite default**) | yes |
| **SIGKILL** | 9 | die now, kernel-enforced | **no** |
| **SIGINT** | 2 | interrupt (Ctrl+C) | yes |
| **SIGHUP** | 1 | terminal closed; by convention "reload config" | yes |
| **SIGSTOP/SIGCONT** | 19/18 | pause / resume | no / yes |
| **SIGQUIT** | 3 | quit with core dump; **a JVM prints a thread dump** | yes |

```bash
kill 1234                # SIGTERM — always try this first
kill -9 1234             # SIGKILL — last resort
pkill -f "java.*practiq" # by command pattern
kill -3 <java-pid>       # thread dump to the JVM's stdout (№12 §8)
```

**Reach for SIGTERM, not SIGKILL.** SIGTERM lets the process flush buffers, close connections, finish in-flight requests and shut down cleanly; SIGKILL gives it no chance and can leave corrupt state or half-written files. `docker stop` sends SIGTERM, waits (10s by default), then SIGKILL — which is exactly why your container's main process must run in the foreground and handle SIGTERM (№50 §7.1).

`kill -3` on a JVM is a genuinely useful trick: it dumps every thread's stack, which is how you diagnose a hang or deadlock in production (№12 §8).

## 3.5 Surviving logout

A process started from your shell dies when the shell exits (it gets SIGHUP). Options: `nohup cmd &` (ignores SIGHUP, output to `nohup.out`); **`tmux`/`screen`** (a persistent terminal session you detach from and reattach later — the right tool for long interactive work over SSH); or, properly, **a systemd service** (§9.1) for anything that should genuinely run as a service.

---

# Part 4 — Users, permissions and ownership

## 4.1 The model

Every file has an **owner** (a user), a **group**, and three permission triples: for the owner, the group, and everyone else.

```
-rw-r--r--  1 jamie developers  4096 Jul 23 10:00 questions.md
│└┬┘└┬┘└┬┘     │       │
│ │  │  └── other: r--     (read)
│ │  └───── group: r--     (read)
│ └──────── owner: rw-     (read, write)
└────────── type: - file, d directory, l symlink
```

**Permissions mean something different on directories**, which trips everyone up:

| Bit | On a file | On a directory |
|---|---|---|
| **r** (4) | read contents | **list** its entries |
| **w** (2) | modify contents | **create/delete** entries within it |
| **x** (1) | execute it | **enter/traverse** it |

So you can have `r` on a directory (see the names) without `x` (can't actually access anything inside), and `x` without `r` (can access a file if you know its exact name, but can't list). And note **deleting a file requires write permission on its directory**, not on the file — which surprises people.

## 4.2 chmod and chown

```bash
chmod 644 file           # rw- r-- r--   (typical file)
chmod 755 script.sh      # rwx r-x r-x   (typical executable/directory)
chmod 600 ~/.ssh/id_rsa  # rw- --- ---   (private key — anything looser is REFUSED by ssh)
chmod +x script.sh       # symbolic form: add execute for all
chmod -R u+w dir/        # recursive

chown jamie:developers file
chown -R jamie: dir/
```

The octal digits are just the r=4, w=2, x=1 bits summed per triple: `7`=rwx, `6`=rw-, `5`=r-x, `4`=r--.

**Avoid `chmod 777`.** It's the reflex fix for a permission problem and it makes the file world-writable, which is a security hole and usually masks the real issue (wrong owner, or a missing `x` on a parent directory).

## 4.3 root, sudo, and least privilege

**root** (UID 0) bypasses permission checks entirely. You don't log in as root; you use `sudo` for individual commands, which is auditable and limits blast radius. Configuration lives in `/etc/sudoers` (edit only via `visudo`, which validates before saving — a syntax error there can lock you out).

The container connection: by default a container's processes run as root *inside* the container, which — without user namespaces — maps to real root capabilities on shared resources. Hence `USER appuser` in a Dockerfile (№50 §7.6) and rootless runtimes like Podman (№50 §6.1).

Beyond the basic bits: **setuid/setgid** (run as the file's owner/group — the mechanism behind `passwd`, and a classic privilege-escalation vector), the **sticky bit** on `/tmp` (anyone can create, only the owner can delete their own), **umask** (default permissions for newly created files), and **ACLs** (`getfacl`/`setfacl`) for finer-grained rules.

---

# Part 5 — The shell

Bash is the common default; zsh is similar for everyday use. The shell is both an interactive interface and a programming language.

## 5.1 Pipes — the core idea

```bash
cat access.log | grep ERROR | wc -l
```

The pipe `|` connects one program's **stdout** to the next's **stdin**. Each tool does one job; composition does the rest. This is the single most important thing to be fluent in.

```bash
# how many errors per hour, top 5
grep ERROR app.log | awk '{print $2}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -5
```

That pipeline — filter, extract, cut, sort, count, sort, limit — is the archetypal shape, and it's `GROUP BY` and `ORDER BY` implemented by composition.

## 5.2 Redirection

```bash
cmd > file           # stdout to file (overwrite)
cmd >> file          # stdout to file (append)
cmd 2> errors.log    # stderr to file
cmd > out 2>&1       # both to the same file  ← the classic
cmd &> out           # bash shorthand for the same
cmd < input.txt      # file as stdin
cmd 2>/dev/null      # discard errors
cmd | tee file       # to stdout AND a file (tee is invaluable in pipelines)
```

`2>&1` must come *after* the stdout redirection — `> out 2>&1` sends both to `out`; `2>&1 > out` sends stderr to the terminal and stdout to the file, which is almost never what you want.

## 5.3 Expansion and globbing

```bash
*.log                # any characters
question?.md         # exactly one character
file[1-3].txt        # a character range
{dev,test,prod}.env  # brace expansion → dev.env test.env prod.env
~                    # home directory
$HOME  ${VAR}        # variable expansion
$(date +%F)          # command substitution — prefer over backticks
$((2 + 3))           # arithmetic
```

Note the shell expands globs **before** the command sees them — `rm *.log` hands `rm` a list of filenames, so `rm` never sees the `*`. That's why an empty match can behave surprisingly, and why quoting matters.

## 5.4 Quoting — the thing that causes most scripting bugs

```bash
echo "$HOME"         # double quotes: expansion happens → /home/jamie
echo '$HOME'         # single quotes: literal        → $HOME
echo "It's $USER"    # double quotes handle apostrophes
```

**Always quote your variables**: `"$file"`, not `$file`. Unquoted, a value containing spaces splits into multiple arguments, and a value containing `*` gets glob-expanded. This single habit prevents a large share of shell bugs.

## 5.5 Chaining and history

```bash
cmd1 && cmd2         # run cmd2 only if cmd1 SUCCEEDED (exit 0)
cmd1 || cmd2         # run cmd2 only if cmd1 FAILED
cmd1 ; cmd2          # run both regardless
cmd1 & cmd2          # cmd1 in background, cmd2 immediately
```

Every command returns an **exit status**: `0` means success, anything else is failure (check with `echo $?`). That convention is what `&&` relies on, and what CI systems use to decide whether a step passed.

Interactive essentials worth knowing: `Ctrl+R` (reverse-search history — the biggest single productivity win in the shell), `!!` (last command, so `sudo !!` reruns with sudo), `Ctrl+A`/`Ctrl+E` (line start/end), `Ctrl+W` (delete previous word), `Ctrl+L` (clear).

---

# Part 6 — The essential toolkit

## 6.1 Viewing files

```bash
cat file                 # dump it (fine for small files)
less file                # page through it: /search, n next, G end, g start, q quit
head -20 file            # first 20 lines
tail -20 file            # last 20
tail -f app.log          # FOLLOW as it grows  ← the log-watching command
tail -f app.log | grep ERROR    # follow, filtered
wc -l file               # count lines
```

## 6.2 find — locate files

```bash
find . -name "*.java"                          # by name
find . -type f -size +100M                     # files over 100MB
find . -type f -mtime -7                       # modified in the last 7 days
find . -name "*.tmp" -delete                   # find and delete
find . -name "*.java" -exec grep -l "TODO" {} +  # run a command on each result
find /var/log -type f -mtime +30 -delete       # log cleanup
```

`find` is the tool for "which files match these criteria," and it composes with `-exec` or `xargs`.

## 6.3 The rest of the daily vocabulary

```bash
sort file                # sort lines ( -n numeric, -r reverse, -u unique, -k2 by field 2 )
uniq -c                  # count adjacent duplicates (ALWAYS sort first)
cut -d, -f2,3 data.csv   # extract fields by delimiter
tr 'a-z' 'A-Z'           # translate/delete characters
diff a.txt b.txt         # compare files ( -u for unified diff format )
xargs                    # turn stdin into arguments
tee                      # write to a file and pass through

# xargs example: git-add every file containing a TODO
grep -rl "TODO" . | xargs git add
```

`sort | uniq -c | sort -rn` is the "count occurrences, most frequent first" idiom — you'll use it constantly for log analysis.

## 6.4 Archives, transfer, and downloading

```bash
tar -czf archive.tar.gz dir/     # create gzipped tarball
tar -xzf archive.tar.gz          # extract
zip -r out.zip dir/ ; unzip out.zip

scp file user@host:/path/        # copy over SSH
rsync -avz --progress src/ user@host:/dest/    # sync — resumable, only transfers differences

curl -sSL https://api.example.com/health       # fetch a URL (№51 §11)
curl -X POST -H "Content-Type: application/json" -d '{"a":1}' https://api/x
wget https://example.com/file.tar.gz
```

`rsync` over `scp` for anything large or repeated — it transfers only what changed and resumes.

---

# Part 7 — Text processing

The three tools that do most of the work. Rough division: **grep finds lines, sed edits lines, awk processes fields.**

## 7.1 grep — find lines

```bash
grep "ERROR" app.log
grep -i "error" app.log            # case-insensitive
grep -r "findApproved" src/        # recursive through a directory
grep -n "TODO" file                # show line numbers
grep -v "DEBUG" app.log            # INVERT — lines that don't match
grep -c "ERROR" app.log            # count matches
grep -A3 -B3 "Exception" app.log   # 3 lines After and Before each match  ← for stack traces
grep -E "ERROR|WARN" app.log       # extended regex (alternation)
grep -l "TODO" *.java              # just list matching filenames
```

`-A`/`-B`/`-C` (context) is the one to remember for logs — an exception line alone is rarely enough; you want the stack trace around it. (`ripgrep`/`rg` is a much faster modern alternative worth installing.)

## 7.2 sed — stream editing

```bash
sed 's/old/new/' file              # replace FIRST occurrence per line
sed 's/old/new/g' file             # replace all (g = global)
sed -i 's/old/new/g' file          # edit the file IN PLACE
sed -i.bak 's/old/new/g' file      # in place, keeping a .bak backup
sed -n '10,20p' file               # print lines 10–20 only
sed '/^#/d' config                 # delete comment lines
```

`sed -i` across a directory is the standard bulk find-and-replace — but take the backup (`-i.bak`) or make sure it's committed to git first.

## 7.3 awk — field processing

`awk` splits each line into fields (`$1`, `$2`, … `$NF` for the last) and runs a program against them.

```bash
awk '{print $1}' access.log                    # first field of every line
awk -F: '{print $1}' /etc/passwd               # custom delimiter
awk '$3 > 100 {print $1, $3}' data.txt         # conditional
awk '{sum += $2} END {print sum}' data.txt     # accumulate and report
ps aux | awk '$3 > 50 {print $2, $11}'         # PIDs using >50% CPU
df -h | awk '$5+0 > 80 {print $6, $5}'         # filesystems over 80% full
```

You don't need to learn awk deeply — `{print $N}`, a condition, and the `sum/END` idiom cover the vast majority of real use.

## 7.4 A worked pipeline

Top 10 IPs hitting your API with errors, from an access log:

```bash
grep " 5[0-9][0-9] " access.log \
  | awk '{print $1}' \
  | sort \
  | uniq -c \
  | sort -rn \
  | head -10
```

Filter to 5xx responses → extract the IP field → sort so duplicates are adjacent → count them → sort by count descending → take the top ten. Six small tools, one question answered, no script written.

---

# Part 8 — Inspecting a running system

## 8.1 CPU and memory

```bash
top                     # live; press M (memory), P (cpu), 1 (per-core), k (kill)
htop                    # friendlier
uptime                  # load averages: 1, 5, 15 minutes
free -h                 # memory usage
vmstat 1                # system stats every second
```

**Load average** is the most misread number in Linux. It's the average number of processes *runnable or waiting on uninterruptible I/O* — so it counts disk waits too, not just CPU. Interpret it relative to core count: a load of 4.0 on a 4-core machine is fully utilised; on a 16-core machine it's quiet.

**Memory:** the `free` output's "free" column being near zero is **normal and good** — Linux uses spare RAM as disk cache and releases it on demand. Look at **available**, not free. And if `swap` usage is climbing and `si/so` in `vmstat` are non-zero, the machine is actively swapping to disk and performance has fallen off a cliff (№00 §1.2).

## 8.2 Processes

Covered in §3.2. For a specific process: `ps -p <pid> -o pid,ppid,rss,pcpu,etime,cmd`, or read `/proc/<pid>/` directly.

## 8.3 Disk

```bash
df -h                          # free space per FILESYSTEM
du -sh *                       # size of each item in the current directory
du -sh /var/log/* | sort -rh | head    # biggest log directories
ncdu /var                      # interactive disk usage (worth installing)
```

`df` vs `du`: **`df` reports what the filesystem says; `du` sums what files claim.** They disagree when a deleted file is still held open by a process — the space isn't freed until the file descriptor closes. That's the classic "I deleted the logs and got no space back" incident; find the culprit with `lsof | grep deleted` and restart the process.

## 8.4 "Disk full" and inodes

A filesystem can be "full" with space remaining if it's out of **inodes** (one per file) — millions of tiny files exhaust them. `df -i` shows inode usage. Rare but baffling if you don't know it exists.

## 8.5 Network and open files

```bash
ss -tlnp                # TCP, listening, numeric, with process  ← "what's on this port"
ss -tan                 # all TCP connections and states (№51 §4.2)
lsof -i :8080           # what's using port 8080
lsof -p <pid>           # all files a process has open
lsof | grep deleted     # deleted-but-held files (the disk-space mystery)
```

`ss` has replaced the older `netstat`. For diagnosing connectivity itself — DNS, routes, TLS, reachability — the ladder is in №51 §11.

## 8.6 Logs

```bash
journalctl -u practiq-api -f          # follow a systemd service's log
journalctl -u practiq-api --since "1 hour ago"
journalctl -p err -b                  # errors since last boot
tail -f /var/log/syslog               # traditional log files
dmesg -T | tail                       # kernel messages — where OOM kills appear
```

**`dmesg` is where you find OOM kills.** If a process "just disappeared," look for `Out of memory: Killed process` — the kernel's OOM killer terminated it, and nothing in your application logs will explain the disappearance. This is a common container failure (memory limit exceeded, №50 §2.3).

---

# Part 9 — Services, packages, boot

## 9.1 systemd

The init system on most modern distributions: PID 1, responsible for starting services, ordering dependencies, restarting failures and collecting logs.

```bash
systemctl status practiq-api          # state, recent log lines, PID  ← start here
systemctl start|stop|restart practiq-api
systemctl reload practiq-api          # re-read config without restarting
systemctl enable practiq-api          # start at boot
systemctl list-units --failed         # what's broken
journalctl -u practiq-api -f          # its logs
```

A unit file (`/etc/systemd/system/practiq-api.service`) declares how to run it:

```ini
[Unit]
Description=Practiq API
After=network.target

[Service]
Type=simple
User=practiq
WorkingDirectory=/opt/practiq
ExecStart=/usr/bin/java -jar /opt/practiq/app.jar
Restart=on-failure
RestartSec=5
Environment=JAVA_OPTS=-XX:MaxRAMPercentage=75

[Install]
WantedBy=multi-user.target
```

After editing: `systemctl daemon-reload`, then `restart`. Note `Restart=on-failure` gives you supervision for free — the same job a container orchestrator does at a different level.

## 9.2 Packages

```bash
# Debian/Ubuntu
apt update && apt upgrade
apt install jq
apt search postgres ; apt show jq ; apt remove jq

# RHEL/Amazon Linux
dnf install jq ; dnf search jq

# Alpine (containers)
apk add --no-cache jq
```

`--no-cache` on Alpine and `rm -rf /var/lib/apt/lists/*` after `apt install` are the container idioms that stop package metadata bloating your image layers (№50 §7.6).

## 9.3 Boot, briefly

BIOS/UEFI → bootloader (GRUB) → kernel → initramfs → systemd (PID 1) → targets and services. You rarely touch this, but knowing the chain helps when a machine won't come up. **Environment and shell startup** you *will* touch: login shells read `~/.bash_profile`/`~/.profile`, interactive non-login shells read `~/.bashrc` — which is why "my PATH works in the terminal but not in cron/systemd" is such a common confusion. Cron and systemd have minimal environments; set what you need explicitly.

## 9.4 Cron

```bash
crontab -e            # edit your user's scheduled jobs
crontab -l            # list

# minute hour day-of-month month day-of-week  command
0 2 * * *   /opt/practiq/backup.sh >> /var/log/backup.log 2>&1
*/15 * * * * /usr/bin/curl -sS localhost:8080/health
```

The two classic cron failures: **a minimal environment** (use absolute paths for everything, and set `PATH` explicitly) and **silently discarded output** (always redirect to a log, including stderr). Modern alternative: systemd timers, which give you journald logging and dependency handling.

---

# Part 10 — Shell scripting

## 10.1 The skeleton

```bash
#!/usr/bin/env bash
set -euo pipefail          # ← put this in every script

readonly BACKUP_DIR="/var/backups/practiq"
readonly DB_NAME="${1:?usage: backup.sh <dbname>}"

main() {
    mkdir -p "$BACKUP_DIR"
    local stamp
    stamp="$(date +%Y%m%d-%H%M%S)"
    pg_dump "$DB_NAME" | gzip > "$BACKUP_DIR/${DB_NAME}-${stamp}.sql.gz"
    find "$BACKUP_DIR" -name '*.sql.gz' -mtime +30 -delete
    echo "backup complete: ${DB_NAME}-${stamp}.sql.gz"
}

main "$@"
```

**`set -euo pipefail` is the most important line in shell scripting:**

- `-e` — exit immediately if any command fails (otherwise the script ploughs on after an error, which is how backup scripts silently produce empty files).
- `-u` — error on undefined variables (this is what stops `rm -rf "$DIR/"` with an unset `DIR`).
- `-o pipefail` — a pipeline fails if *any* stage fails, not just the last.

## 10.2 The constructs

```bash
if [[ -f "$file" ]]; then ... elif ...; else ...; fi     # -f file, -d dir, -z empty string, -n non-empty
for f in *.log; do echo "$f"; done
for i in {1..5}; do ...; done
while read -r line; do echo "$line"; done < input.txt
case "$1" in start) ...;; stop) ...;; *) echo "unknown";; esac

count=$(grep -c ERROR app.log)
[[ "$count" -gt 10 ]] && echo "too many errors"
```

Use `[[ ]]` rather than `[ ]` in bash — it's safer with unquoted variables and supports pattern matching.

## 10.3 When to stop using bash

Shell is excellent for **gluing commands together**. It's poor at data structures, arithmetic, error handling and anything above about 100 lines. **When you find yourself writing functions that return values, parsing JSON by hand, or managing state — switch to Python.** `jq` is the right tool for JSON in a pipeline, and `shellcheck` will catch most of the quoting and portability bugs before they bite.

---

# Part 11 — Troubleshooting playbook

A rough order of operations for "the server is unhealthy":

1. **What changed?** Deployments, config, data volume. Most incidents are recent changes (№31 §11).
2. **Is it up?** `systemctl status <service>`, `ps aux | grep <proc>`, `docker ps`.
3. **Resources.** `top`/`htop` (CPU), `free -h` (memory, and swap activity), `df -h` (disk), `df -i` (inodes).
4. **Logs.** `journalctl -u <service> -n 200`, application logs, and `dmesg -T | tail` for **OOM kills**.
5. **Network.** `ss -tlnp` (is it listening?), then the layered ladder in №51 §11 (resolve → reach → connect → speak).
6. **Disk mystery?** `du -sh /* | sort -rh`, and `lsof | grep deleted` if `df` and `du` disagree.
7. **Hung process?** `ps` for `D` state (I/O block); for a JVM, `kill -3 <pid>` for a thread dump (№12 §8).
8. **Still stuck?** `strace -p <pid>` shows the system calls a process is making — heavy, but it tells you exactly where a process is stuck.

The container version of this is the same list, run inside the container (`docker exec -it <id> sh`) or against the host — because a container is just a process (§1.1, №50 §2).

---

# Part 12 — When to use what

**A. grep, sed, or awk?** Tell → grep: find matching lines. Tell → sed: substitute or delete text. Tell → awk: work with fields, or compute. Default: **grep to find, sed to change, awk when columns matter.**

**B. `find` or `grep -r`?** Tell → find: select files by *name, size, age, type*. Tell → grep -r: select by *content*. Default: **find for files, grep for text; combine with `-exec` or `xargs`.**

**C. SIGTERM or SIGKILL?** Tell → SIGTERM: always first — it allows clean shutdown. Tell → SIGKILL: only after SIGTERM has been ignored. Default: **`kill`, then `kill -9` only if it won't die.**

**D. nohup, tmux, or systemd?** Tell → nohup: a one-off long command. Tell → tmux: interactive work over SSH you want to reattach to. Tell → systemd: anything that should run as a managed service. Default: **tmux for sessions, systemd for services.**

**E. Bash or Python?** Tell → bash: chaining commands, under ~100 lines, no real data structures. Tell → Python: parsing, logic, error handling, anything you'll maintain. Default: **bash for glue, Python the moment it grows a data structure.**

**F. `df` or `du`?** Tell → df: how full is the filesystem. Tell → du: what's taking the space. Default: **df to detect, du to locate — and if they disagree, look for deleted-but-open files.**

**G. Which base image / distro?** Tell → Debian-slim: sane default, glibc. Tell → Alpine: minimal size, **but musl** — avoid for the JVM and native binaries. Tell → distroless: smallest attack surface, hardest to debug. Default: **slim for JVM workloads** (№50 §7.6).

---

# How to expand this

- *Related:* №50 Docker (containers as Linux processes — namespaces and cgroups are the kernel features under all of it); №51 Networking (the network troubleshooting ladder); №12 §8 (thread dumps for hangs); №10 §13.4 (JVM memory in a container — `MaxRAMPercentage` and the OOM killer); №57 Observability (planned).
- *Candidates for deeper treatment:* **shell scripting properly** (functions, traps, arrays, robust argument parsing, testing with bats); **performance analysis on Linux** (`strace`, `perf`, flame graphs, the USE method); **SSH in depth** (keys, agent forwarding, config files, tunnels, bastions); **systemd beyond services** (timers, sockets, resource control).

*Stable material, written from knowledge — the process model, permissions, signals, and the core tools don't drift. Distribution specifics (package managers, default paths, whether `ss` or `netstat` is installed) vary; check `man <command>` on the box you're actually on.*
