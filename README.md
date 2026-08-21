# Shell Version Control

A working subset of `git` — staging area, commits, branches, three-way merge — implemented as ten **POSIX shell** scripts. No Python, no `git`, no external tools beyond what a POSIX system already provides.

This was an individual coursework project for **Software Construction** during my postgraduate study at the University of New South Wales. Writing a version control system in shell is a good way to find out how much of `git`'s design is essential and how much is optimisation: everything here is a full file copy, and the model still works.

## Commands

| Command | What it does |
| --- | --- |
| `svc-init` | Create the `.svc` repository |
| `svc-add` | Copy files into the staging area |
| `svc-commit -m msg` | Snapshot the staging area; `-a` first re-stages already-tracked files |
| `svc-log` | List every commit, newest first |
| `svc-show [commit]:file` | Print a file as it was at a given commit, or in the staging area |
| `svc-rm` | Remove from the staging area, the working directory, or both |
| `svc-status` | Compare working directory, staging area and repository |
| `svc-branch [-d] [name]` | Create, delete, or list branches |
| `svc-checkout branch` | Switch branches |
| `svc-merge <branch\|commit> -m msg` | Three-way merge, with fast-forward |

## How the repository is laid out

```
.svc/stag/         the staging area — plain files, one per staged path
.svc/repo/N/       commit N as a full directory snapshot
.svc/repo/N.msg    commit N's message
.svc/repo/N.hist   every commit reachable from N, N included
.svc/branch/NAME   the commit number a branch points at (-1 = no commit yet)
.svc/current       the current branch name
.svc/index         the next commit number
```

Commits are **whole-tree copies**, not deltas. In shell that is the right trade: a delta format would need a diff/patch engine and careful escaping, while `cp` and `diff -r` are already correct.

## Design notes

**Finding the merge base is one line, and it is exact.** Commit numbers increase monotonically, so the latest common ancestor of two commits is simply the largest number present in both ancestry sets:

```sh
ance_head=$(sort -n ".svc/repo/${curr_head}.hist" ".svc/repo/${targ_head}.hist" | uniq -d | tail -n 1)
```

No graph walk, no recursion. That is the whole reason `.hist` stores the full reachable set rather than just a parent pointer — it trades a little disk for turning an ancestry query into a sort.

**Merge decides per file, from three sides.** A file conflicts only when it changed on *both* sides since the ancestor *and* the two results differ. Otherwise, whichever side changed it wins:

```sh
if same_file "target/$f" "ancestor/$f"; then src="current/$f"; else src="target/$f"; fi
```

**A missing file is a first-class state.** `same_file()` treats "absent" as equal only to "absent", which is what makes deletions merge correctly instead of silently resurrecting a file the other branch removed.

**"Nothing to commit" is `diff -r`.** Rather than tracking dirtiness, `svc-commit` compares the staging directory against the head commit's directory. The check cannot drift out of sync with reality because it *is* reality.

**`svc-status` enumerates twelve states, not three.** Every file is classified by comparing working directory, staging area, and repository — so it distinguishes, for example, `file changed, changes staged for commit` from `file changed, different changes staged for commit` from `added to index, file deleted`. Getting all twelve right is most of the work in that script.

**A branch head of `-1` means "no commit yet"**, which lets `master` exist from `svc-init` onward without a special case for the empty repository everywhere else.

## Requirements

`dash` — `/bin/dash` on macOS, `apt install dash` on Debian/Ubuntu. The scripts are POSIX `sh` and use no bashisms.

Put the directory on your `PATH`:

```bash
export PATH="$PWD:$PATH"
```

## Run

```bash
mkdir demo && cd demo
svc-init
echo hello > a.txt && svc-add a.txt && svc-commit -m "first"
echo world >> a.txt && svc-add a.txt && svc-commit -m "second"
svc-log
svc-show 0:a.txt          # a.txt as it was at commit 0
svc-status
```

Branching and merging:

```bash
svc-branch feature
svc-checkout feature
echo change > b.txt && svc-add b.txt && svc-commit -m "on the branch"
svc-checkout master
svc-merge feature -m "merge feature"
```

Two branches editing *different* files merge cleanly; two branches editing the *same* file differently are refused:

```console
$ svc-merge other -m "should conflict"
svc-merge: error: These files can not be merged:
x.txt
$ echo $?
1
```

## Test

The suite works by **differential testing**: `runcase()` in `test-lib.sh` runs each scenario twice in two fresh temporary directories — once against these scripts, once against the course's reference implementation invoked as `2041 svc-...` — and compares the combined stdout and stderr. A case passes only if the two are byte-identical after paths are normalised.

That reference implementation exists only on the university's lab machines. **Anywhere else the suite cannot run**: `2041` is not found, so every case reports as failed on the *reference* side while this implementation produces correct output.

```console
$ dash testall.sh
fail - 'merge conflict error'
< svc-merge: error: These files can not be merged:     # <- ours, correct
< exit:1
---
> ./test9.sh: 107: 2041: not found                     # <- the reference, absent
> exit:127
```

On a lab machine:

```bash
dash testall.sh                      # everything
for t in test?.sh; do dash "$t"; done # one area at a time
```

Off a lab machine, the **Run** section above is the way to exercise it — every command there was verified by hand.

| Script | Area |
| --- | --- |
| `test0` | Initialising a repository |
| `test1` | Adding files to the index |
| `test2` | Committing, with and without `-a` |
| `test3` | Listing commits |
| `test4` | Showing a file at a given commit |
| `test5` | Removing files |
| `test6` | Status across all its states |
| `test7` | Creating, listing and deleting branches |
| `test8` | Switching branches |
| `test9` | Merging, including conflicts |

## Known limitations

- **No conflict markers.** A conflicting merge is refused outright rather than writing `<<<<<<<` into the file and letting you resolve it.
- **No remotes, no tags, no rebase, no history rewriting.** The model is local-only.
- **Every commit stores every file in full.** Disk grows linearly with commits × tree size.
- **Only regular files in the repository root are tracked.** No subdirectories, no modes, no symlinks.
- **`.hist` stores the whole reachable set per commit**, so its size also grows with history length. Fine at this scale, wrong at any real one.

## Project structure

```
svc-init svc-add svc-commit svc-log svc-show svc-rm
svc-status svc-branch svc-checkout svc-merge     # the ten commands
test-lib.sh                                      # shared assertions
testall.sh                                       # runs the suite
test0.sh .. test9.sh                             # one area each
```

## License

[MIT](LICENSE)

## Author

**Tianshu Shen** — this was an individual assignment, written and tested by me.
