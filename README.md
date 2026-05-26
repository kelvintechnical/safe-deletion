# Lab: Safe Deletion of Files and Directories — `rm`, `rmdir`, `rm -rf`

**Series:** linux-ops-mastery — RHCSA Essential Tools & File Operations
**Subjects covered:** `rm` for files, `rmdir` for empty directories, `rm -rf` for recursive trees, the no-undo nature of `rm`, `--preserve-root` protection, `--one-file-system` boundary, `-i` and `-I` interactive prompts, `-v` verbose, the quarantine-first habit (`mv → /tmp/trash`), `find -delete` for criteria-based cleanup, the catastrophic `rm -rf $UNSET/` family of incidents, why `rm` does not free data immediately if a process holds it open, and the senior-engineer protective routine `pwd; ls; quote variables; absolute paths`
**Career arcs covered:** RHCSA ("remove the file" exam tasks and cleanup of test fixtures), RHCE (Ansible `file: state=absent`), SRE (incident cleanup of stale lockfiles, log rotation), DevOps (CI cache invalidation), AI/MLOps (cleaning up old checkpoints / scratch space)
**Prerequisite:** Labs 05–10 (navigation, listing, copy, links, move)
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 foundation (`rm FILE`) · 2 `rmdir` vs `rm -r` · 3 `rm -rf` carefully · 4 `-i`/`-I` safety prompts · 5 quarantine + `find -delete` patterns · 6 RHCSA exam-realistic capstone

---

## Objective

Delete files and directories **deliberately and safely**. By the end of this lab you can run `rm -rf` on a directory tree without flinching — because the discipline of `pwd; ls; quote vars; absolute paths` has become a reflex, and you know exactly which rm options exist to catch the mistakes everybody makes.

The capstone is an exam-realistic prompt: *"Clean out `/root/cleanup/` of every file older than 30 days and every empty subdirectory, but leave files newer than 30 days intact. Confirm the count of survivors and report the count of removed items."*

> **Lab safety note:** Every command in this lab targets `/tmp/rm-lab` and `/root/cleanup`. No system files are touched. The infamous `rm -rf /` and `rm -rf "$DIR"/` failure modes are demonstrated **only** in scenarios that cannot reach real paths.

---

## Concept: `rm` Is the Shredder, Not the Recycle Bin

`rm` removes the **directory entry** that names a file. If the entry was the last name pointing at that inode (and no process has the file open), the kernel frees the inode and its data blocks. There is **no recycle bin** at the command line on Linux.

```
   ┌────────────────────────────────────────────────────────────────┐
   │   rm file.txt                                                  │
   ├────────────────────────────────────────────────────────────────┤
   │   1.  unlink("file.txt")            ← remove directory entry   │
   │   2.  If inode link count == 0:                                │
   │         If no process holds the file open:                     │
   │           Free inode + data blocks immediately.                │
   │         Else:                                                  │
   │           Defer free until last FD is closed.                  │
   │       Else:                                                    │
   │         Other names still point at the inode — data lives on.  │
   └────────────────────────────────────────────────────────────────┘
```

> **Why this matters:** The single biggest cause of catastrophic data loss in Linux history is a `rm -rf` command where a variable expanded to empty or to `/`. The kernel will do exactly what you typed — no second chances.

---

## 📜 Why `rm` Is Dangerous by Design — The Story

When Unix shipped in **1971** on a PDP-7 with kilobytes of memory, disks measured in megabytes, and no notion of "trash folders," there was simply no room to keep a hidden duplicate of every deleted file. So `rm` ships with **immediate, irreversible** deletion and zero prompting unless you ask for it.

The decision stuck for 55 years. Modern Linux still ships `rm` as a fast, scriptable, no-frills deletion tool — because **scripts** need exactly this behavior: run, free space, exit zero. Safety features were grafted on one at a time over the decades:

- **`-i`** (1980s) — prompt before every deletion. Annoying enough that nobody uses it interactively.
- **`-I`** (GNU coreutils) — prompt **once** for the whole operation when removing >3 files or recursing.
- **`--preserve-root`** (added 2004 after too many accidents) — refuse to remove `/`. Default since coreutils 6.x.
- **`--one-file-system`** — refuse to cross mount points during recursive delete.
- **`shred -u`** — overwrite-then-delete for sensitive data on spinning disks.

These exist because the consequences of "no undo" became more painful at scale. Today's senior engineers don't avoid `rm` — they build **habits** around it.

> **The point of the story:** `rm` is a knife. The reason it has no safety lock is that scripts need it sharp. The reason senior engineers never cut themselves with it is that they always look at the blade before they swing.

---

## 👪 The Deletion Family — Who Lives There

| Task | Command | Notes |
|---|---|---|
| Remove a single file | `rm FILE` | Irreversible, no prompt by default |
| Remove an empty directory | `rmdir DIR` | Refuses non-empty dirs |
| Remove empty dirs up the chain | `rmdir -p A/B/C` | Removes A/B/C, then A/B, then A if empty |
| Recursive remove | `rm -r DIR` | Removes everything below DIR + DIR itself |
| Recursive + ignore missing + no prompts | `rm -rf DIR` | The "just do it" form |
| Prompt before every deletion | `rm -ri DIR` | Per-file Y/N |
| Prompt once for whole operation | `rm -Ir DIR` | Most useful interactive safety |
| Verbose | `rm -rv DIR` | Audit trail of removals |
| Stop at filesystem boundaries | `rm -rf --one-file-system DIR` | Refuses to cross mounts |
| Default `/` protection | (built-in `--preserve-root`) | Cannot remove `/` even with `-rf` |
| Remove a file whose name starts with `-` | `rm -- -file` or `rm ./-file` | The `--` ends option processing |
| Delete by criteria | `find PATH -PRED -delete` | Used for "older than N days" cleanups |
| Quarantine instead | `mv FILE /tmp/trash/` | Undo-friendly |
| Sensitive data on a spinning disk | `shred -u FILE` | Overwrite + unlink (less effective on SSD) |
| Empty a file held open by a daemon | `: > FILE` or `truncate -s 0 FILE` | Do not unlink — daemon's FD becomes useless |

### `rm` flag reference

| Flag | Long form | Purpose |
|---|---|---|
| `-r` / `-R` | `--recursive` | Recurse into directories |
| `-f` | `--force` | Ignore nonexistent files; do not prompt |
| `-i` | `--interactive=always` | Prompt before every deletion |
| `-I` | `--interactive=once` | Prompt once for the whole operation |
| `-v` | `--verbose` | Print each removal |
| `-d` | `--dir` | Remove an empty directory (like `rmdir`) |
| `--preserve-root` | (default) | Refuse to remove `/` |
| `--no-preserve-root` | (override) | Allow `/` (please never) |
| `--one-file-system` | — | Refuse to cross mount points |

> **The point of the family tree:** Pick the smallest blast radius for the job. `rmdir` for one empty directory. `rm` for one file. `rm -rf` only for trees you absolutely know. `find -delete` for criteria-based cleanup.

---

## 🔬 The Anatomy of `rm -rf` — In One Diagram

```
$ rm -rf /tmp/build-cache

What actually happens:
  1.  fts_open(/tmp/build-cache)
  2.  For each entry encountered (depth-first):
        - regular file       → unlink(2)
        - empty directory    → rmdir(2)
        - non-empty directory → recurse first, then rmdir(2)
        - symlink            → unlink(2)  (target untouched)
  3.  Finally rmdir the top.

`-f` modifies behavior at every step:
   - missing target          → silent (exit 0)
   - permission error        → silent (exit 0)   ← THE DANGER
   - prompt-worthy condition → suppressed

`--preserve-root` (default):
   - If target literally resolves to `/`, refuse with an error.
   - Does NOT protect against `rm -rf "$UNSET"/` expanding to `rm -rf /`
     unless the `/` is literal at the time of parse.
```

> **Reading rule:** `rm -rf` plus an unquoted variable is the single most common production-incident command. Always quote `"$VAR"` and always check `pwd` and `ls` before you commit.

---

## 📚 Deletion Reference Table

| Task | Command | Notes |
|---|---|---|
| Delete file | `rm FILE` | Use `rm -i` for safety |
| Delete file with verbose audit | `rm -v FILE` | Prints `removed 'FILE'` |
| Delete empty dir | `rmdir DIR` | Refuses non-empty |
| Delete empty dir + empty parents | `rmdir -p A/B/C` | Whole empty chain |
| Recursive delete of a tree | `rm -r DIR` | Removes everything under DIR + DIR |
| Recursive + silent + force | `rm -rf DIR` | The "fully automatic" form |
| Recursive + prompt-once | `rm -Ir DIR` | One prompt; safe default for interactive use |
| Recursive + audit trail | `rm -rv DIR` | Verbose recursive delete |
| Block crossing mount points | `rm -rf --one-file-system DIR` | Bind mounts and submounts stay safe |
| File whose name starts with `-` | `rm -- -file` | `--` ends option parsing |
| Wipe sensitive data on HDD | `shred -u FILE` | Overwrite + unlink (HDD-effective; SSD wear-leveled) |
| Empty a held-open file in place | `: > FILE` | Truncate without unlinking (preserves daemon's FD) |
| Delete by criteria | `find PATH -mtime +30 -delete` | Replaces `xargs rm` for many cases |
| Quarantine instead of delete | `mv FILE /tmp/trash/` | Reversible until you `rm` `/tmp/trash` later |

> **Rule one of deletion:** Always `pwd; ls` before `rm -rf`. Always quote `"$VAR"`. Always prefer absolute paths in scripts.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | "Remove the file `/tmp/old.log`" tasks; cleanup at end of every exam task. |
| **RHCE candidate** | Ansible `file: state=absent` is idempotent `rm -rf`. |
| **SRE / Platform** | Cleaning up stale lockfiles, rotated logs, expired sessions; `find -mtime +N -delete` is the universal cron cleanup. |
| **DevOps** | CI cache eviction; `docker prune`-style cleanups. |
| **AI / MLOps** | Removing old checkpoints to free GPU scratch; cleaning failed-run directories. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **pwd → ls → quote → delete → verify** habit.

---

### Task 1 — Remove single files safely

**Purpose:** Build the sandbox, create test files, and remove them one at a time using `rm`, `rm -v`, and `rm -i`.

```bash
mkdir -p /tmp/rm-lab && cd /tmp/rm-lab

touch a.txt b.txt c.txt
ls -l

rm a.txt
ls
rm -v b.txt
ls
rm -i c.txt        # type 'y' to confirm
ls
```

**Human-Readable Breakdown:** Create three empty files, remove them three ways: silent, verbose, interactive. See what each operator looks like.

**Reading it left to right:** Plain `rm` removes silently. `-v` prints the action. `-i` reads from stdin before removing. None of them recover anything.

**The story:** Default `rm` is the muscle memory you build for known-safe paths. `rm -v` is the audit-friendly form for scripts. `rm -i` is for "did I really mean this?" moments.

**Expected output:**

```text
-rw-r--r--. 1 user user 0 May 26 14:30 a.txt
-rw-r--r--. 1 user user 0 May 26 14:30 b.txt
-rw-r--r--. 1 user user 0 May 26 14:30 c.txt
b.txt  c.txt
removed 'b.txt'
c.txt
rm: remove regular empty file 'c.txt'? y
```

**Switches**

| Token | Meaning |
|---|---|
| `rm FILE` | Silent removal |
| `-v` | Verbose |
| `-i` | Prompt before each removal |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `rm: cannot remove 'X': Permission denied` | You need write on the **parent directory**, not the file |
| `rm` did not free disk space | Process still holds it open; use `lsof FILE` and close |
| `rm` removed only the symlink, not the target | Correct behavior — symlinks are independent inodes |

---

### Task 2 — `rmdir` vs `rm -r` for directories

**Purpose:** Show that `rmdir` requires empty directories and `rm -r` is needed for non-empty trees.

```bash
cd /tmp/rm-lab
mkdir -p empty-dir nest/a/b/c
touch nest/a/file1.txt nest/a/b/file2.txt

rmdir empty-dir
ls

rmdir nest
ls
rmdir nest/a/b/c
ls -R nest

rm -r nest
ls
```

**Human-Readable Breakdown:** `rmdir empty-dir` succeeds. `rmdir nest` fails because nest is non-empty. `rmdir nest/a/b/c` succeeds because `c` is empty. Finally `rm -r nest` cleans the whole tree.

**Reading it left to right:** `rmdir` calls `rmdir(2)`, which only works on empty directories. `rm -r DIR` recurses into DIR, unlinking files and rmdir'ing emptied subdirs depth-first.

**The story:** Use `rmdir` when you know the directory is empty — it's a sanity check. Use `rm -r` only when you have audited the contents. Use `rm -rf` only when the contents do not matter at all.

**Expected output:**

```text
nest
rmdir: failed to remove 'nest': Directory not empty
nest
nest/a:
b  file1.txt

nest/a/b:
file2.txt

(empty)
```

**Switches**

| Token | Meaning |
|---|---|
| `rmdir DIR` | Remove empty directory |
| `rmdir -p A/B/C` | Remove A/B/C, then A/B, then A if empty |
| `rm -r DIR` | Recursive — directories and contents |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `rmdir: 'X': Directory not empty` | Use `rm -r` or empty it first |
| `rm: cannot remove 'X': Is a directory` | Add `-r` |
| Directory came back after `rm -r` | Something else is recreating it (cron, service) |

---

### Task 3 — `rm -rf` carefully (with `pwd` discipline)

**Purpose:** Use `rm -rf` to clear a tree, but practice the senior-engineer ritual `pwd; ls; quote variables` first.

```bash
cd /tmp/rm-lab
mkdir -p big-tree/a/b/c big-tree/d/e
for i in 1 2 3 4 5; do touch "big-tree/file$i"; done

# Pre-delete inspection — DO THIS EVERY TIME
pwd
ls big-tree
du -sh big-tree

# Quoted variable — safer than typing the path twice
TARGET="/tmp/rm-lab/big-tree"
echo "About to remove: $TARGET"
test -d "$TARGET" && rm -rfv "$TARGET" | head -n 10
echo "After: "
ls /tmp/rm-lab/

# Counter-example — what NOT to type:
# rm -rf $TARGET/   ← unquoted: if TARGET is unset, becomes `rm -rf /`
# rm -rf "$TARGET"/ ← quoted but with trailing slash; safer but still verify path first
```

**Human-Readable Breakdown:** Build a non-trivial tree, pre-inspect, store target in a quoted variable, guard with `test -d`, run `rm -rfv` to see each removal. The commented counter-example shows the unquoted-variable footgun.

**Reading it left to right:** `pwd` confirms you're where you think you are. `ls` confirms the target exists and is what you expect. `test -d "$TARGET"` guards against missing path. `rm -rfv "$TARGET"` removes verbosely; output piped to `head` for sanity.

**The story:** This is the canonical senior-engineer rm pattern. Three commands of friction prevent the worst incident of your career.

**Expected output:**

```text
/tmp/rm-lab
a  d  file1  file2  file3  file4  file5
20K  big-tree
About to remove: /tmp/rm-lab/big-tree
removed 'big-tree/file1'
removed 'big-tree/file2'
removed 'big-tree/file3'
...
removed directory 'big-tree'
After:
```

**Switches**

| Token | Meaning |
|---|---|
| `rm -rf PATH` | Force recursive — no prompt, ignore missing |
| `rm -rfv PATH` | Same, verbose |
| `test -d "$VAR"` | Guard: only proceed if `$VAR` is a directory |
| `du -sh DIR` | Summarized size (sanity check) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wrong directory removed | `pwd; ls` ritual catches this every time |
| `rm -rf` silently did nothing | `-f` silences missing-path errors — use `-rv` to see |
| `rm: cannot remove '/'` | `--preserve-root` blocks; never use `--no-preserve-root` |

---

### Task 4 — `-I` once-per-operation prompts and `-i` per-file prompts

**Purpose:** Differentiate `-i` (every file) from `-I` (once at the start) and adopt `-I` as the safer interactive default.

```bash
cd /tmp/rm-lab
mkdir -p batch
touch batch/f{1..10}

# -i prompts for EACH file
rm -i batch/f{1..3}     # type y y y

# -I prompts ONCE if >3 files OR recursive
rm -Ir batch            # type y

ls
```

**Human-Readable Breakdown:** Create a batch of files. `rm -i` produces 10 prompts; `rm -Ir` produces one. Use `-I` for any "are you sure?" question and `-i` only when you genuinely want per-file control.

**Reading it left to right:** `-i` is interactive=always. `-I` is interactive=once when the operation removes >3 files or is recursive. Both read stdin.

**The story:** Most engineers never use `-i` because the prompts are unbearable. `-I` gives you the safety with one keystroke of friction.

**Expected output:**

```text
rm: remove regular empty file 'batch/f1'? y
rm: remove regular empty file 'batch/f2'? y
rm: remove regular empty file 'batch/f3'? y
rm: remove 7 arguments recursively? y
(empty)
```

**Switches**

| Token | Meaning |
|---|---|
| `-i` | Prompt before every operation |
| `-I` | Prompt once if >3 args or `-r` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-i` overwhelmed me | Use `-I` next time |
| `-I` did not prompt | The operation was <4 files and not recursive — that's by design |

---

### Task 5 — Quarantine pattern and criteria-based cleanup

**Purpose:** Use `mv → /tmp/trash` to "delete" reversibly, and `find -delete` to remove by criteria (age, size, name).

```bash
mkdir -p /tmp/rm-lab/trash
cd /tmp/rm-lab

touch maybe-delete.txt
ls maybe-delete.txt
mv maybe-delete.txt /tmp/rm-lab/trash/
ls maybe-delete.txt 2>&1 | head -n 1
ls /tmp/rm-lab/trash/

# Empty the trash later
rm -rf /tmp/rm-lab/trash/*
ls /tmp/rm-lab/trash/

# Criteria-based deletion with find
mkdir -p age
touch -d "40 days ago" age/old1 age/old2 age/old3
touch age/new1 age/new2
ls -lt age/

find age -type f -mtime +30
find age -type f -mtime +30 -delete
ls age/
```

**Human-Readable Breakdown:** "Delete" `maybe-delete.txt` by moving it to the trash directory — fully reversible until you empty the trash. Then create five files with mixed ages and use `find -delete` to remove only those older than 30 days.

**Reading it left to right:** `mv FILE /tmp/rm-lab/trash/` relocates instead of unlinks. Later `rm -rf` empties the trash for real. `find -type f -mtime +30 -delete` lists files matching the predicate and unlinks each; without `-delete` it just prints (the safe preview mode).

**The story:** Quarantine before delete is how senior engineers run a `cleanup.sh` script the first time on a new system. Once the script has been audited and the trash directory has been monitored for a week, you can switch to direct `rm` confidently.

**Expected output:**

```text
-rw-r--r--. 1 user user 0 May 26 14:35 maybe-delete.txt
ls: cannot access 'maybe-delete.txt': No such file or directory
maybe-delete.txt
(empty)
-rw-r--r--. 1 user user 0 ... new2
-rw-r--r--. 1 user user 0 ... new1
-rw-r--r--. 1 user user 0 2026-04-16 ... old3
-rw-r--r--. 1 user user 0 2026-04-16 ... old2
-rw-r--r--. 1 user user 0 2026-04-16 ... old1
age/old1
age/old2
age/old3
new1  new2
```

**Switches**

| Token | Meaning |
|---|---|
| `mv FILE /tmp/trash/` | Quarantine instead of unlink |
| `find PATH -mtime +N` | Files older than N*24h |
| `find ... -delete` | Unlink each matching file |
| `find ... -print -delete` | Print before deleting |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `find -delete` removed too much | Run without `-delete` first to preview |
| `find -delete` did not remove dirs | Add `-depth` to delete contents-before-parent |
| Quarantine dir is filling up | Set up a cron `find /tmp/trash -mtime +7 -delete` |

---

### Task 6 — Capstone: RHCSA-realistic age-based cleanup

**Task statement:** *"Clean out `/root/cleanup/` of every file older than 30 days and every empty subdirectory. Leave files newer than 30 days intact. Report the count of files removed and the count of survivors. Save both counts to `/root/cleanup-report.txt`."*

**Purpose:** Execute a complete cleanup with audit trail.

```bash
sudo -i

# Build a controlled fixture
rm -rf /root/cleanup
mkdir -p /root/cleanup/{recent,old,mixed,empty}
touch -d "40 days ago" /root/cleanup/old/{file1,file2,file3}
touch -d "40 days ago" /root/cleanup/mixed/oldA
touch                  /root/cleanup/mixed/newA
touch                  /root/cleanup/recent/{r1,r2,r3}

echo "Before cleanup:"
find /root/cleanup -type f | wc -l

# Stage 1 — remove old files
REMOVED=$(find /root/cleanup -type f -mtime +30 -print -delete | wc -l)

# Stage 2 — remove now-empty subdirectories (depth-first)
find /root/cleanup -mindepth 1 -type d -empty -delete

echo "After cleanup:"
SURVIVORS=$(find /root/cleanup -type f | wc -l)
find /root/cleanup -type f

{
  echo "Cleanup report  $(date -Is)"
  echo "  removed: $REMOVED"
  echo "  survivors: $SURVIVORS"
} | tee /root/cleanup-report.txt

test -s /root/cleanup-report.txt && echo "VERIFY: report exists and is non-empty"
```

**Human-Readable Breakdown:** Become root, build a fixture with three age groups, two-stage cleanup: (1) `find -mtime +30 -delete` removes old files and counts them, (2) `find -empty -delete` removes any subdirectories that the first stage emptied. Tally survivors. Write everything to `/root/cleanup-report.txt`.

**Layer stack you built:**

```text
/root/cleanup/
   ├── recent/ (3 new files — kept)
   ├── old/    (3 old files — removed → dir now empty → removed)
   ├── mixed/  (1 old + 1 new → old removed, new kept)
   └── empty/  (already empty → removed)

After cleanup:
/root/cleanup/recent/r1
/root/cleanup/recent/r2
/root/cleanup/recent/r3
/root/cleanup/mixed/newA
```

**The story:** This is the **canonical scheduled-cleanup pattern.** Memorize the spine: `find PATH -type f -mtime +N -delete` then `find PATH -mindepth 1 -type d -empty -delete`. Two stages, no surprises.

**Expected verification output:**

```text
Before cleanup:
8
After cleanup:
4
/root/cleanup/recent/r3
/root/cleanup/recent/r2
/root/cleanup/recent/r1
/root/cleanup/mixed/newA
Cleanup report  2026-05-26T14:40:00-04:00
  removed: 4
  survivors: 4
VERIFY: report exists and is non-empty
```

**Cleanup**

```bash
rm -rf /tmp/rm-lab /root/cleanup /root/cleanup-report.txt
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Removed too many files | Verify with `find -mtime +30 -print` (no -delete) first |
| Empty directories remain | `-mindepth 1` includes them only if they had no contents at scan time |
| `Permission denied` writing report | Not root — `sudo -i` |
| Counts disagree across runs | The clock moved — re-touch fixtures with `-d` |

---

## 🔍 Deletion Decision Guide

```
Need to delete something?
  │
  ├── "One file"
  │       └── ✅ rm FILE
  │
  ├── "One empty directory"
  │       └── ✅ rmdir DIR
  │
  ├── "Whole directory tree"
  │       └── ✅ rm -r DIR             (safer when interactive)
  │       └── ✅ rm -rf DIR            (force; for scripts)
  │       └── ✅ rm -Ir DIR            (prompt once; safer interactive default)
  │
  ├── "By criteria — age, size, owner"
  │       └── ✅ find PATH -PRED -delete
  │
  ├── "First time on a new system — undo-friendly"
  │       └── ✅ mv FILE /tmp/trash/         (then rm -rf the trash later)
  │
  ├── "Sensitive data on a spinning disk"
  │       └── ✅ shred -u FILE
  │
  ├── "Empty a log file held open by a daemon"
  │       └── ✅ : > /var/log/app.log         (do NOT rm — daemon's FD is fine)
  │
  ├── "Refuse to cross mount boundaries"
  │       └── ✅ rm -rf --one-file-system DIR
  │
  └── "Filename starts with -"
          └── ✅ rm -- -filename            (or rm ./-filename)
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Set up `/tmp/rm-lab` and remove files with `rm`, `rm -v`, `rm -i`
- [ ] 02 Distinguish `rmdir` (empty only) from `rm -r` (everything)
- [ ] 03 Practice the `pwd; ls; quoted variable; test -d` ritual before `rm -rf`
- [ ] 04 Use `-I` for once-per-operation prompts; understand `-i` per-file prompts
- [ ] 05 Quarantine with `mv → /tmp/trash`; criteria-delete with `find -delete`
- [ ] 06 Execute the RHCSA capstone — age-based two-stage cleanup of `/root/cleanup/`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Unquoted variable in `rm -rf` | Expansion to `/` or to other paths | Always quote `"$VAR"` |
| Trailing space in `rm -rf $VAR /something` | Tries to remove two paths | Re-read carefully |
| Confused `rm -r` and `rmdir` | Either "directory not empty" or "is a directory" | Pick the right tool |
| `rm` did not free disk | Open FD pinning the inode | `lsof FILE` and close, or restart daemon |
| Deleted symlink expecting target gone | Only the link removed | Use `readlink -f` first if you want target |
| `find -delete` skipped subdirs | Need `-depth` for inside-out traversal | `find ... -depth -delete` |
| `rm` of file with leading `-` | Treated as flag | `rm -- -file` or `rm ./-file` |
| Removed a file across mount boundary you did not realize | Surprise | Use `--one-file-system` |
| Cleaned `/tmp/trash` too aggressively | Lost quarantine value | Keep trash at least a week |
| Trusted `rm -f` to "make sure" | Suppresses real errors | Read errors at least once before silencing |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Always end a task with cleanup if the prompt asks for it. Always `pwd; ls` before any `rm -rf`.

**RHCE candidate**
- Ansible `file: state=absent  path: /target` is idempotent `rm -rf`. Use `force: yes` to override `_recursive` safety.

**SRE / Platform interview**
- "How would you safely clean up old log files?" → "Two-stage `find`: `find /var/log -type f -mtime +30 -delete` then `find /var/log -mindepth 1 -type d -empty -delete`. Audit `-print` before `-delete`."

**DevOps**
- CI cache eviction: per-job `find $CACHE -atime +7 -delete`.

**AI / MLOps**
- Old checkpoints: keep last N, delete the rest. `ls -t ckpt-*.pt | tail -n +6 | xargs -r rm -v`.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Directory Navigation | The `pwd; ls` discipline starts here |
| Lab 08 — Copying Files | `cp -a` to staging before `rm` is the safe-deploy pattern |
| Lab 09 — Hard and Soft Links | Why `rm` of one of N hard links doesn't free data |
| Lab 14 — File Searching with `find` | `-delete` predicate for criteria-based cleanup |
| Lab — Cron Cleanup of /var/tmp *(later)* | Production schedule of these patterns |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
