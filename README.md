# Lab 11: Safe Deletion of Files and Directories — `rm`, `rmdir`, `rm -rf`

**Series:** File Operations & Shell Fundamentals · **Lab 11 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (Tasks 8, 11, 16, 18, 20), RHCE EX294 (Ansible `file: state=absent`), CKA (`kubectl delete`, manifest removal), RHCA building blocks (RH342, RH358)  
**Prerequisite:** Labs 05–10  
**Time Estimate:** 40–55 minutes  
**Difficulty arc:** Tasks 1–6 foundation · 7–13 practical · 14–18 advanced · 19–20 exam-realistic

---

## 🎯 Objective

Delete files and directories **safely and intentionally**. Master `rm`, `rmdir`, and the infamous `rm -rf` — and learn the verification, dry-run, and safer-alternative habits that prevent the career-ending mistake of `rm -rf /` or `rm -rf $UNDEFINED/`.

---

## 🧠 Concept: There Is No Undelete

Linux has **no recycle bin** at the command line. When `rm` returns success, the directory entry is gone, and (after the last hard link is removed and no process holds the file open) the inode and data blocks are released. Forensic recovery is sometimes possible, often not. **There is no Ctrl-Z.**

```
rm file.txt
   │
   └── directory entry "file.txt" removed
        └── if link count hits 0 AND no FD open → inode + blocks freed → data gone
```

> The single biggest cause of catastrophic data loss in Linux history: a `rm -rf` command where a variable expanded to empty or to `/`.

### Command mental model

| Command | Removes files? | Removes directories? | Recursive? | Prompts? |
|---|---|---|---|---|
| `rm` | ✅ | ❌ (unless `-d` + empty) | ❌ | Sometimes (alias) |
| `rm -d` | ✅ | ✅ (only empty) | ❌ | Sometimes |
| `rm -r` | ✅ | ✅ (any state) | ✅ | Sometimes |
| `rm -i` | ✅ | with `-r` | with `-r` | **Always** |
| `rm -I` | ✅ | with `-r` | with `-r` | **Once** for >3 files or recursive |
| `rm -rf` | ✅ | ✅ (any) | ✅ | **Never** |
| `rmdir` | ❌ | ✅ (only empty) | ❌ (unless `-p`) | Never |

---

## 📚 `rm` and `rmdir` Reference

### `rm`

| Flag | Long form | Purpose |
|---|---|---|
| `-r` / `-R` | `--recursive` | Delete directories and their contents |
| `-f` | `--force` | Ignore nonexistent files; do not prompt |
| `-i` | `--interactive=always` | Prompt before **every** deletion |
| `-I` | `--interactive=once` | Prompt once if >3 files or recursive |
| `-d` | `--dir` | Allow removing **empty** directories without `-r` |
| `-v` | `--verbose` | Print each file removed |
| `--one-file-system` | — | When recursing, don't cross filesystem boundaries |
| `--preserve-root` | — | (Default on GNU rm) refuse to operate recursively on `/` |
| `--no-preserve-root` | — | **DANGER** — disables the `/` safeguard |

### `rmdir`

| Flag | Long form | Purpose |
|---|---|---|
| (none) | — | Remove an empty directory |
| `-p` | `--parents` | Remove the directory **and** any empty parents up the chain |
| `-v` | `--verbose` | Print each directory removed |
| `--ignore-fail-on-non-empty` | — | Don't fail on non-empty (just skip) |

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **RHCSA EX200** | Tasks 8, 11, 16, 18, 20 require careful deletion |
| **RHCE EX294** | `ansible.builtin.file: state=absent` mirrors `rm -rf`; idempotent |
| **CKA** | `kubectl delete -f`; manual cleanup of `/etc/kubernetes/manifests/*.yaml.bak` |
| **RHCA — RH342** | Quarantine before deletion; recovery via inode forensics |
| **RHCA — RH358** | Service cleanup: stop, then `rm` of runtime state |

---

## 🔧 The 20 Tasks

---

### Task 1 — Set up the lab workspace

**Purpose:** Build a realistic mix of empty and populated directories.

```bash
mkdir -p ~/rm-lab/{logs,configs,empty,empty/inner,tree/a/b/c}
cd ~/rm-lab
touch logs/app_{1..3}.log configs/{httpd,nginx,sshd}.conf tree/a/b/c/deep.txt
ls -lR
```

**Expected output (excerpt):**

```
.:
total 0
drwxr-xr-x. 2 ec2-user ec2-user 30 Sep 12 15:00 configs
drwxr-xr-x. 3 ec2-user ec2-user 18 Sep 12 15:00 empty
drwxr-xr-x. 2 ec2-user ec2-user 53 Sep 12 15:00 logs
drwxr-xr-x. 3 ec2-user ec2-user 13 Sep 12 15:00 tree
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p` | Create with missing parents |
| `{logs,configs,empty,...}` | Brace expansion → multiple dirs in one call |
| `app_{1..3}.log` | Three log files |

**Output decoded**

| Entry | Meaning |
|---|---|
| Four top-level dirs | Workspace ready |
| Each is populated except `empty/` and its child | We'll test both states |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mkdir: cannot create` | Path inside your home |

---

### Task 2 — Remove a single file with `rm`

**Purpose:** Most basic case — plain `rm` on one file.

```bash
ls logs/
rm logs/app_1.log
ls logs/
```

**Expected output:**

```
app_1.log  app_2.log  app_3.log
app_2.log  app_3.log
```

**Switches**

| Token | Meaning |
|---|---|
| `rm FILE` | Remove one (or more) regular files |

**Output decoded**

| Phase | What happened |
|---|---|
| Before | 3 logs |
| After | 2 logs — `app_1.log` removed |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `rm: cannot remove ... : Operation not permitted` | Immutable flag set — `sudo chattr -i FILE` first |

---

### Task 3 — `rm` refuses directories without `-r`

**Purpose:** First safety guardrail: `rm` won't delete directories unless you opt in.

```bash
rm logs/ 2>&1
```

**Expected output:**

```
rm: cannot remove 'logs/': Is a directory
```

**Switches**

| Token | Meaning |
|---|---|
| `rm DIR` | Refused unless `-d` (empty) or `-r` (recursive) is set |

**Output decoded**

| Line | Meaning |
|---|---|
| `Is a directory` | `rm` is telling you it won't proceed |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wanted to remove an empty dir | Use `rmdir` or `rm -d` |
| Wanted to remove a populated dir | Use `rm -r` |

---

### Task 4 — Remove an empty directory with `rmdir`

**Purpose:** Safest directory removal — refuses if not empty.

```bash
rmdir empty/inner
ls empty/
```

**Expected output:**

```
(empty parent directory listing)
```

**Switches**

| Token | Meaning |
|---|---|
| `rmdir DIR` | Remove empty directory; refuse non-empty |

**Output decoded**

| Phase | Effect |
|---|---|
| Before | `empty/inner` existed |
| After | Gone; parent `empty/` is now empty |

**Why a sysadmin needs this:** `rmdir` forces you to confirm there's no data inside — it refuses otherwise.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `rmdir: failed: Directory not empty` | Use `rm -r` (with care) or empty it first |

---

### Task 5 — Remove an empty parent chain with `rmdir -p`

**Purpose:** Symmetric cleanup after `mkdir -p`.

```bash
mkdir -p ~/rm-lab/empty/a/b/c
rmdir -p ~/rm-lab/empty/a/b/c
ls ~/rm-lab
```

**Expected output:**

```
configs  empty  logs  tree
```

**Switches**

| Flag | Meaning |
|---|---|
| `-p` | Remove the directory **and** any empty parents up the chain |

**Output decoded**

| Token | Meaning |
|---|---|
| `empty/` survives | `rmdir -p` stops at the first non-empty parent (or at the start of an absolute path) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `rmdir -p` failed midway | Hit a non-empty parent — that's by design |

---

### Task 6 — Remove an empty directory with `rm -d`

**Purpose:** A one-shot way to delete an empty dir without switching to `rmdir`.

```bash
mkdir ~/rm-lab/just_empty
rm -d ~/rm-lab/just_empty
ls ~/rm-lab
```

**Expected output:**

```
configs  empty  logs  tree
```

**Switches**

| Flag | Meaning |
|---|---|
| `-d` | Allow `rm` to remove an empty directory (no `-r`, no recursion) |

**Output decoded**

| Token | Meaning |
|---|---|
| Directory gone | Worked like `rmdir` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Directory not empty` | `-d` refuses non-empty — use `-r` if you really mean it |

---

### Task 7 — Recursive removal with `-r`

**Purpose:** Descend into a directory tree and remove everything.

```bash
ls -R tree/
rm -r tree/
ls ~/rm-lab
```

**Expected output:**

```
tree/:
a

tree/a:
b

tree/a/b:
c

tree/a/b/c:
deep.txt

configs  empty  logs
```

**Switches**

| Flag | Meaning |
|---|---|
| `-r` / `-R` | Recursive |

**Output decoded**

| Phase | What happened |
|---|---|
| Before | `tree/a/b/c/deep.txt` existed |
| After | Entire `tree/` chain is gone |

> ⚠️ **No prompt by default** on a fresh user. Many systems set `alias rm='rm -i'`. Type `\rm` to bypass the alias.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Asked for confirmation per file | An `-i` alias is active — type `\rm -r` or use `-I` |

---

### Task 8 — Prompt-per-file with `-i`

**Purpose:** Maximum safety — every deletion confirmed.

```bash
mkdir -p ~/rm-lab/promptdir
touch ~/rm-lab/promptdir/{a,b,c}.txt
rm -ri ~/rm-lab/promptdir
```

**Expected interaction:**

```
rm: descend into directory '/home/ec2-user/rm-lab/promptdir'? y
rm: remove regular empty file '/home/ec2-user/rm-lab/promptdir/a.txt'? y
rm: remove regular empty file '/home/ec2-user/rm-lab/promptdir/b.txt'? y
rm: remove regular empty file '/home/ec2-user/rm-lab/promptdir/c.txt'? y
rm: remove directory '/home/ec2-user/rm-lab/promptdir'? y
```

**Switches**

| Flag | Meaning |
|---|---|
| `-r` | Recursive |
| `-i` | Prompt for **every** deletion |

**Output decoded**

| Prompt | Action |
|---|---|
| Each `?` | Type `y` (yes) or `n` (no) |

**Why a sysadmin needs this:** When unsure of the contents, `-i` makes you read every line.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Too many prompts | Use `-I` (Task 9) for a single prompt |

---

### Task 9 — Single prompt with `-I`

**Purpose:** Middle ground — one prompt for the entire operation when >3 files or recursive.

```bash
mkdir -p ~/rm-lab/manyfiles
touch ~/rm-lab/manyfiles/file_{01..10}.tmp
rm -Ir ~/rm-lab/manyfiles
```

**Expected interaction:**

```
rm: remove 1 argument recursively? y
```

**Switches**

| Flag | Meaning |
|---|---|
| `-I` | Prompt **once** before recursive deletion or removing >3 files |

**Output decoded**

| Prompt | Meaning |
|---|---|
| `remove 1 argument recursively?` | One yes/no covers the whole tree |

**Why a sysadmin needs this:** Many `~/.bashrc` set `alias rm='rm -I'` as a sane default.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Not getting the single prompt | `-I` only triggers for >3 files or recursive — for one file, it's silent |

---

### Task 10 — Verbose deletion with `-v`

**Purpose:** See what `rm` actually removed — your audit log.

```bash
mkdir -p ~/rm-lab/loud
touch ~/rm-lab/loud/file_{01..05}.tmp
rm -rv ~/rm-lab/loud
```

**Expected output:**

```
removed '/home/ec2-user/rm-lab/loud/file_01.tmp'
removed '/home/ec2-user/rm-lab/loud/file_02.tmp'
removed '/home/ec2-user/rm-lab/loud/file_03.tmp'
removed '/home/ec2-user/rm-lab/loud/file_04.tmp'
removed '/home/ec2-user/rm-lab/loud/file_05.tmp'
removed directory '/home/ec2-user/rm-lab/loud'
```

**Switches**

| Flag | Meaning |
|---|---|
| `-r` | Recursive |
| `-v` | Verbose — print each removal |

**Output decoded**

| Line | Meaning |
|---|---|
| `removed '...'` | One file removed |
| `removed directory '...'` | A directory removed after its children |

**Why a sysadmin needs this:** Always pair `-r` with `-v` when destroying anything important.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Output overwhelming | Pipe to a log: `rm -rv DIR > /tmp/deleted.log 2>&1` |

---

### Task 11 — Idempotent force with `-f`

**Purpose:** Don't fail when files are missing; never prompt.

```bash
rm -f ~/rm-lab/this_does_not_exist.txt
echo "exit code = $?"
```

**Expected output:**

```
exit code = 0
```

**Switches**

| Flag | Meaning |
|---|---|
| `-f` | Force — silently ignore nonexistent files, never prompt |

**Output decoded**

| Token | Meaning |
|---|---|
| `exit code = 0` | Success even though the file didn't exist |

**Why a sysadmin needs this:** Scripts that should be idempotent ("delete this if it exists, otherwise do nothing").

> `-f` does **not** bypass directory permissions. It can remove read-only files only because `rm` deletes the **directory entry**, which needs write+execute on the **parent directory**.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Still `Permission denied` | You lack write on the parent — `sudo` or fix the parent's mode |

---

### Task 12 — The infamous `rm -rf`

**Purpose:** Most dangerous combination. Understand it deeply before ever typing it on production.

```bash
ls ~/rm-lab/configs
rm -rf ~/rm-lab/configs
ls ~/rm-lab
```

**Expected output:**

```
httpd.conf  nginx.conf  sshd.conf
empty  logs
```

**Switches**

| Flag combo | Meaning |
|---|---|
| `-r` | Recursive |
| `-f` | Force — no prompts, ignore missing |
| Together | Silently delete everything, even read-only files, no second chances |

**Output decoded**

| Phase | What happened |
|---|---|
| Before | `configs/` with three files |
| After | All gone, no confirmation |

### `rm -rf` Safety Checklist (memorize before pressing Enter)

1. **`pwd` first** — confirm where you are.
2. **`ls` the target** — confirm what you're about to delete.
3. **Use absolute paths** — never `rm -rf $VAR/something` when `$VAR` might be empty.
4. **Quote and brace variables** — `rm -rf "${VAR:?error}"/path` exits if `VAR` is unset/empty.
5. **Prefer `mv` first** — `mv target /tmp/target.delete-me` then `rm -rf` later.
6. **`echo` first** — `echo rm -rf $TARGET` shows exactly what will run.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| You realized too late | Check backups / snapshots immediately |

---

### Task 13 — Variable-safety: `${VAR:?}` saves careers

**Purpose:** Make `rm -rf` impossible to run with an unset variable.

```bash
TARGET=""
rm -rf "${TARGET:?TARGET must be set}/some/path" 2>&1 || echo "Refused"

TARGET="$HOME/rm-lab/safe-demo"
mkdir -p "$TARGET"
rm -rf "${TARGET:?TARGET must be set}"
echo "exit code = $?"
```

**Expected output:**

```
bash: TARGET: TARGET must be set
Refused
exit code = 0
```

**Switches / shell tokens**

| Token | Meaning |
|---|---|
| `${VAR:?msg}` | Bash exits the command with `msg` if VAR is unset/empty |
| `${VAR:-default}` | Substitute default if VAR is unset/empty |
| `--` (in `rm -rf -- "$VAR"`) | Treat everything after `--` as filenames, not flags |

**Output decoded**

| Phase | Meaning |
|---|---|
| First call | `TARGET` empty → bash refused before `rm` ran |
| Second call | `TARGET` set → directory removed successfully |

**Why a sysadmin needs this:** A misspelled `$TARGET` becoming empty is exactly how `rm -rf /` happens.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `TARGET` mistakenly set to `/` | Add `[[ "$TARGET" == "/" ]] && exit 1` as a guard |

---

### Task 14 — `--one-file-system` for paranoid recursive deletes

**Purpose:** Prevent `rm -rf` from crossing mount boundaries by accident.

```bash
mkdir -p ~/rm-lab/safety/inside
touch ~/rm-lab/safety/inside/file_{1..3}.tmp
rm -rfv --one-file-system ~/rm-lab/safety
```

**Expected output:**

```
removed '/home/ec2-user/rm-lab/safety/inside/file_1.tmp'
removed '/home/ec2-user/rm-lab/safety/inside/file_2.tmp'
removed '/home/ec2-user/rm-lab/safety/inside/file_3.tmp'
removed directory '/home/ec2-user/rm-lab/safety/inside'
removed directory '/home/ec2-user/rm-lab/safety'
```

**Switches**

| Flag | Meaning |
|---|---|
| `--one-file-system` | Refuse to cross mount points during recursion |

**Output decoded**

| Line | Meaning |
|---|---|
| Each `removed` line | Stayed within the home filesystem |

**Why a sysadmin needs this:** If `/some/tree` contains a mountpoint like `/mnt/data`, `--one-file-system` stops `rm` from wiping `/mnt/data`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Skipped a subtree unexpectedly | That subtree was on another mount — fine, that's the safety working |

---

### Task 15 — `--preserve-root` is the default; never disable it

**Purpose:** Demonstrate the GNU `rm` safety net for `/`.

```bash
sudo rm -rf / 2>&1 | head -2
```

**Expected output:**

```
rm: it is dangerous to operate recursively on '/'
rm: use --no-preserve-root to override this failsafe
```

**Switches**

| Flag | Meaning |
|---|---|
| `--preserve-root` (default) | Refuses `rm -rf /` |
| `--no-preserve-root` | **Removes the safety.** Don't. Ever. |

**Output decoded**

| Line | Meaning |
|---|---|
| `dangerous to operate recursively on '/'` | Saved your career |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| You see `--no-preserve-root` in a script | Reject the change. There's no legitimate use case in normal sysadmin work. |

---

### Task 16 — Awkward filenames: dash-prefix, spaces, special chars

**Purpose:** Some filenames need defensive quoting or the `--` end-of-options.

```bash
cd ~/rm-lab
touch -- "-file_with_dash.txt"
touch "   spaces.txt"
touch 'special_$char.txt'

rm -- -file_with_dash.txt
rm "   spaces.txt"
rm 'special_$char.txt'
ls
```

**Expected output:**

```
empty  logs
```

**Switches / patterns**

| Pattern | Use case |
|---|---|
| `rm -- -file_with_dash.txt` | `--` tells `rm` "no more flags" — everything after is a name |
| `rm ./-file_with_dash.txt` | Alternative — explicit relative path |
| `rm "   spaces.txt"` | Double quotes preserve spaces |
| `rm 'special_$char.txt'` | Single quotes prevent `$` expansion |

**Output decoded**

| Phase | Meaning |
|---|---|
| Each `rm` succeeded silently | Quoting solved the naming traps |

**Universal escape hatch** — when you can't type the name, delete by inode:

```bash
ls -li
find . -inum INODE -exec rm -i {} \;
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `rm: invalid option -- 'f'` for `rm -file.txt` | Use `rm -- -file.txt` or `rm ./-file.txt` |

---

### Task 17 — Bulk-by-criteria with `find -delete`

**Purpose:** Delete files matching age/size/name — the right tool for log rotation.

```bash
mkdir -p ~/rm-lab/expiry
touch -d "60 days ago" ~/rm-lab/expiry/old_{1..3}.log
touch              ~/rm-lab/expiry/new_{1..3}.log

find ~/rm-lab/expiry -type f -mtime +30 -print
find ~/rm-lab/expiry -type f -mtime +30 -delete
ls ~/rm-lab/expiry/
```

**Expected output:**

```
/home/ec2-user/rm-lab/expiry/old_1.log
/home/ec2-user/rm-lab/expiry/old_2.log
/home/ec2-user/rm-lab/expiry/old_3.log
new_1.log  new_2.log  new_3.log
```

**Switches**

| Token | Meaning |
|---|---|
| `find PATH` | Start at PATH |
| `-type f` | Regular files only |
| `-mtime +30` | mtime > 30 days ago |
| `-print` | Default action — list matches |
| `-delete` | Built-in deletion |

**Output decoded**

| Phase | What happened |
|---|---|
| `-print` (dry run) | Lists files that **would** be deleted |
| `-delete` (production run) | Same files actually deleted |
| `ls` after | Only the recent files remain |

> **Rule:** `-print` first, **then** swap to `-delete`. Never skip the dry run.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `find: -delete: cannot delete` on non-empty dirs | Add `-depth` so `find` deletes contents before parent |

---

### Task 18 — Safer alternative: quarantine with `mv`

**Purpose:** When unsure, move to a trash dir instead of deleting.

```bash
mkdir -p /tmp/trash-$(date +%F)
echo "draft" > ~/rm-lab/maybe.txt
mv -v ~/rm-lab/maybe.txt /tmp/trash-$(date +%F)/
ls /tmp/trash-*/
```

**Expected output:**

```
renamed '/home/ec2-user/rm-lab/maybe.txt' -> '/tmp/trash-2026-09-12/maybe.txt'
maybe.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `mv -v` | Move with verbose audit (Lab 10) |
| `$(date +%F)` | Today's date in `YYYY-MM-DD` |

**Output decoded**

| Token | Meaning |
|---|---|
| File now in `/tmp/trash-2026-09-12/` | Recoverable until the next reboot (or scheduled cleanup) |

**Alternatives table**

| Tool | Behavior | When to prefer |
|---|---|---|
| `mv target /tmp/trash-YYYYMMDD/` | Quarantine; delete later | When unsure |
| `trash-put` (trash-cli) | Real recycle bin | Daily desktop use; not always on exam VMs |
| `find ... -delete` | Predicate-driven | Targeted bulk |
| `shred -u` | Overwrite then delete | Sensitive data on spinning disks (not SSD/COW) |
| `: > file` | Truncate (don't delete) | Logs held open by a daemon |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Trash directory fills disk | Add a cron: `find /tmp/trash-* -mtime +7 -delete` |

---

### Task 19 — RHCSA-style scenario

**Task statement:** *"Remove the empty directory `/srv/old_empty`, then recursively delete `/srv/cache` if it exists, ignoring any 'does not exist' errors."*

```bash
sudo mkdir -p /srv/old_empty /srv/cache/{a,b,c}
sudo touch /srv/cache/file.bin

ls /srv

sudo rmdir /srv/old_empty 2>/dev/null || sudo rm -d /srv/old_empty 2>/dev/null
sudo rm -rfv /srv/cache

ls /srv 2>/dev/null
```

**Expected output:**

```
cache  old_empty
removed '/srv/cache/file.bin'
removed directory '/srv/cache/a'
removed directory '/srv/cache/b'
removed directory '/srv/cache/c'
removed directory '/srv/cache'
(empty /srv)
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `rmdir` first | Confirms `old_empty` is truly empty |
| `\|\| rm -d` fallback | One-liner safety |
| `rm -rfv /srv/cache` | Recursive, force (idempotent), verbose (audit) |
| Final `ls` | Verification |

**Output decoded**

| Line | Meaning |
|---|---|
| Each `removed` line | Audit trail |
| Empty `/srv` | Success |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `rmdir: failed: Directory not empty` | Forget `rmdir` — switch to `rm -rf` if you confirm contents are disposable |

---

### Task 20 — CKA/Ansible-style scenario: idempotent removal

**Task statement (CKA-flavored):** *"Ensure `/etc/kubernetes/manifests/kube-scheduler.yaml.bak` is removed if it exists. Operation must not fail on a fresh node."*

```bash
sudo mkdir -p /etc/kubernetes/manifests
sudo touch /etc/kubernetes/manifests/kube-scheduler.yaml.bak

ls /etc/kubernetes/manifests/

sudo rm -fv /etc/kubernetes/manifests/kube-scheduler.yaml.bak
sudo rm -fv /etc/kubernetes/manifests/kube-scheduler.yaml.bak   # second run — must not fail

echo "exit code on missing = $?"
```

**Expected output:**

```
kube-scheduler.yaml.bak
removed '/etc/kubernetes/manifests/kube-scheduler.yaml.bak'
exit code on missing = 0
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `rm -fv` | Force = ignore missing; verbose = audit |
| Second `rm -fv` (file already gone) | Returns exit code 0 — proves idempotency |

**Equivalent Ansible (RHCE EX294)**

```yaml
- name: Remove old scheduler backup
  ansible.builtin.file:
    path: /etc/kubernetes/manifests/kube-scheduler.yaml.bak
    state: absent
```

| Key | Mirrors `rm` |
|---|---|
| `state: absent` | `rm -rf` if a dir, `rm -f` if a file |
| (no `force:` needed) | Idempotent by design |

**Output decoded**

| Line | Meaning |
|---|---|
| First `removed` | File deleted |
| Empty (no second `removed`) | Already gone — `-f` quieted the failure |
| `exit code = 0` | Script safe to re-run |

**Final lab cleanup**

```bash
cd ~
pwd
ls -la rm-lab
rm -rfv rm-lab
ls -la rm-lab 2>/dev/null
sudo rm -rf /srv/old_empty /srv/cache /etc/kubernetes/manifests/kube-scheduler.yaml.bak
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `rm: refusing to remove` on `/` ancestors | You typed an ancestor by mistake — abort and review |

---

## 🔍 Deletion Decision Guide

```
Empty directory?
  ├── Yes → rmdir <dir>          (refuses non-empty)
  ├── Yes, plus empty parents → rmdir -p <dir>
  └── In a one-shot script → rm -d <dir>

Single file?
  ├── Confident → rm <file>
  └── Unsure    → rm -i <file>

Directory + contents?
  ├── Confident → rm -rv <dir>
  ├── Unsure    → rm -ri <dir>   (every prompt) OR rm -Ir <dir>  (one prompt)
  └── Idempotent (no fail if missing) → rm -rf <dir>

Bulk by criteria (age/size/name)?
  ├── find <path> ... -print           (dry run first)
  └── find <path> ... -delete          (only after dry run is correct)

Cross-mount safety needed?
  └── rm -rf --one-file-system <dir>

Filename starts with - or has odd chars?
  └── rm -- <name>   or   rm ./<name>

Want a safety net?
  └── mv target /tmp/trash-$(date +%F)/   (then real rm later)
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Set up `~/rm-lab` with mixed empty/populated dirs
- [ ] 02 Delete a single file with `rm`
- [ ] 03 Confirm `rm` refuses directories without `-r` or `-d`
- [ ] 04 Delete an empty directory with `rmdir`
- [ ] 05 Delete an empty parent chain with `rmdir -p`
- [ ] 06 Delete an empty directory with `rm -d`
- [ ] 07 Delete a populated tree with `rm -r`
- [ ] 08 Prompt-per-file with `rm -ri`
- [ ] 09 Single prompt with `rm -Ir`
- [ ] 10 Verbose audit with `rm -rv`
- [ ] 11 Idempotent `rm -f` (no fail on missing)
- [ ] 12 Use `rm -rf` (in your own lab dir only)
- [ ] 13 Harden scripts with `${VAR:?must be set}`
- [ ] 14 `--one-file-system` for cross-mount safety
- [ ] 15 Confirm `--preserve-root` is the default
- [ ] 16 Delete files with awkward names (`-name`, spaces, special chars)
- [ ] 17 Bulk delete with `find -delete` (dry run first)
- [ ] 18 Safer alternative: quarantine with `mv`
- [ ] 19 Exam: `rmdir` + `rm -rf` combo
- [ ] 20 CKA/Ansible: idempotent `rm -f` and `state: absent`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `rm -rf $VAR/*` with `$VAR` empty | Wipes current dir or root | Use `${VAR:?must be set}` and absolute paths |
| `rm -rf .*` to remove dotfiles | Glob matches `..` → tries to ascend | Use `find . -maxdepth 1 -name '.*' -not -name '.' -not -name '..' -exec rm -rf {} +` |
| Forgetting `rm` deletes symlinks, not targets | "I lost the file" — actually just the link | `ls -l link` before removing |
| `rm -rf` on production after a typo | Catastrophic loss | Quarantine pattern: `mv` to `/tmp/trash/` first |
| Relying on `alias rm='rm -i'` | Alias missing on fresh shell | Always type `-i` / `-I` explicitly |
| Filename starting with `-` | `rm` interprets as flag | `rm -- -file` or `rm ./-file` |
| `shred` on SSD/COW filesystem | Doesn't actually overwrite all copies | Use FDE (`cryptsetup`); `shred` only on spinning disks |
| Forgetting graders' working directory | Deleted wrong thing | Always `pwd && ls` right before destructive commands |

---

## 📌 Exam Strategy

**RHCSA EX200**
- "Remove file X" → `rm` (plain).
- "Remove directory X":
  - Empty → `rmdir X`
  - Not empty → `rm -r X` (add `-f` if task says "force" or "ignore missing")
- Always `ls -la <target>` immediately before any `rm -rf` — cheapest insurance.

**RHCE EX294 (Ansible)**
- `ansible.builtin.file: state: absent` = idempotent `rm -rf`.
- Works on files and directories alike (no separate flag).

**CKA**
- `kubectl delete -f manifest.yaml` is the K8s-aware equivalent (handles finalizers).
- On nodes: `rm` of `/etc/kubernetes/manifests/<file>.yaml` stops a static pod; reverse with `mv` from backup.
- Avoid `rm -rf` on `/var/lib/etcd`, `/var/lib/containerd`, `/etc/kubernetes/pki` without snapshots.

**RHCA**
- RH342: quarantine pattern, `find -delete` for log forensics.
- RH358: stop service before `rm` of runtime state directories.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Navigation | `pwd` before `rm -rf` is non-negotiable |
| Lab 06 — `ls -l` | Verify what's there before deletion |
| Lab 08 — `cp --backup` | Safety net `rm` lacks |
| Lab 09 — Links | `rm` removes a symlink; target is untouched |
| Lab 10 — `mv` | Quarantine pattern (`mv` to `/tmp/trash/` then `rm`) |
| Task 8 — Create dirs | Symmetrical cleanup with `rmdir`/`rm -r` |
| Task 13/14 — cron + find | `find -delete` for scheduled cleanups |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
