# Version Control with Git — Skills Lab

**Topic:** Git fundamentals (local version control)  
**Date:** 2026-10-15  
**Author:** Allan Collins  
**Environment:** throwaway sandbox folder on my own machine

> Why this matters for a security/IT career: every team tracks its code,
> scripts, and configs in git. Version control is a baseline professional
> skill — this lab shows I can create a repository, record changes, and
> read a project's history.

## What I practiced

**Turn this folder into a git repository.**

```console
$ git init -b main
Initialized empty Git repository in ~/security-labs/git-sandbox/.git/
```

**Proof: the hidden .git history folder now exists.**

```console
$ ls -a | tr '\n' ' '; echo
. .. .git 
```

**Set an identity for this practice repo only (not system-wide).**

```console
$ git config --local user.name 'Allan Collins'; git config --local user.email 'denlan711@users.noreply.github.com'; echo 'identity set for this folder'
identity set for this folder
```

**Create a first file in the project.**

```console
$ printf 'My Security+ study log\n- Day 1: set up a git repo\n' > study-log.md; echo 'created study-log.md'
created study-log.md
```

**Ask git what it notices — the file is 'untracked' (git sees it, isn't saving it yet).**

```console
$ git status --short; echo '(?? = git sees it but is not tracking it yet)'
?? study-log.md
(?? = git sees it but is not tracking it yet)
```

**Stage the file (mark it to be included in the next snapshot).**

```console
$ git add study-log.md

```

**Now it's 'staged' — ready to be saved (A = added/staged).**

```console
$ git status --short
A  study-log.md
```

**Commit: save the snapshot with a message describing what you did.**

```console
$ git commit -m 'Start my Security+ study log'
[main (root-commit) f6dffed] Start my Security+ study log
 1 file changed, 2 insertions(+)
 create mode 100644 study-log.md
```

**Add a new line to the file.**

```console
$ printf -- '- Day 2: learned add, commit, diff, log\n' >> study-log.md; echo 'updated the file'
updated the file
```

**git diff shows exactly what changed ( + = added line ).**

```console
$ git diff
diff --git i/study-log.md w/study-log.md
index fb42a33..8b0f74d 100644
--- i/study-log.md
+++ w/study-log.md
@@ -1,2 +1,3 @@
 My Security+ study log
 - Day 1: set up a git repo
+- Day 2: learned add, commit, diff, log
```

**Stage and commit the change in one view.**

```console
$ git add study-log.md && git commit -m 'Log day 2 progress'
[main c15ede7] Log day 2 progress
 1 file changed, 1 insertion(+)
```

**The full history of this repo (two commits).**

```console
$ git log --oneline
c15ede7 Log day 2 progress
f6dffed Start my Security+ study log
```

## Commands I can now use

| Command | What it does |
|---|---|
| `git init` | turn a folder into a repository |
| `git status` | see what has changed |
| `git add <file>` | stage changes for the next commit |
| `git commit -m "..."` | save a snapshot with a message |
| `git diff` | see exactly what changed |
| `git log --oneline` | read the project history |
| `git push` / `git pull` | upload to / download from a remote (GitHub) |

---
_Practiced hands-on in a sandbox repository._
