---
name: wsl-repos-from-windows
description: Operate safely on git repos stored inside WSL (reached from Windows via \\wsl.localhost UNC paths) when the agent itself runs on Windows, e.g. the Claude Code desktop app. Use whenever the working directory is a \\wsl.localhost or \\wsl$ path; whenever git, node, cargo, or other project tooling must run against a WSL-side repo; when you see "fatal: not a git repository: //wsl.localhost/..." or a path mangled into "C:/Program Files/Git/home/..."; when a .claude/worktrees worktree is broken for WSL git; or when CRLF line endings or Windows artifacts appear in a Linux-side repo — even if the user only mentions committing, branches, worktrees, or line endings. Also use when deciding where a new repo, project, or vault should live in a Windows+WSL setup: this is the preferred layout, and the sibling skill windows-repos-from-wsl covers the exception where a Windows app pins the repo to the Windows filesystem.
---

# WSL-side repos from a Windows agent

The repo lives on the WSL ext4 filesystem (`/home/...`). Windows sees it through the `\\wsl.localhost\<distro>\...` network mount. This is the layout to prefer when placing a new repo; only pin a repo to the Windows filesystem when a Windows app requires native file semantics (file watchers, sync clients — e.g. Obsidian vaults), in which case use the `windows-repos-from-wsl` skill instead. That mount is fine for *reading and writing file contents*, but every *command* — git, node, cargo, package managers, anything that generates files — must run inside WSL. Windows-side git against a WSL repo causes four distinct problems: `core.autocrlf` rewrites (Git for Windows ships `autocrlf=true` in its system config, so checkouts silently become CRLF), wrong or missing author identity (the user's identity usually lives in WSL's `~/.gitconfig`), index stat-cache thrash (each side's `stat()` results differ, so the other side rehashes everything), and network-mount slowness.

## Running commands in WSL

```
wsl.exe -d <distro> -- sh -c 'cd /home/<user>/path/to/repo && <command>'
```

- Always `cd` to the Linux path inside the command. The shell inherits the Windows UNC working directory, which is unresolvable inside Linux.
- From a Git Bash / MSYS shell, prefix with `MSYS_NO_PATHCONV=1`. Otherwise MSYS rewrites Linux path arguments into Windows paths (`/home/u/x` becomes `C:/Program Files/Git/home/u/x`) before wsl.exe ever sees them. From PowerShell or cmd this is not needed.
- Translate paths with `wslpath`: `wslpath -u 'C:\...'` → Linux, `wslpath -w /home/...` → Windows. The manual mapping is `\\wsl.localhost\<distro>\home\u\x` ↔ `/home/u/x`.
- Multi-line arguments (commit messages) survive best as multiple `-m` flags or a single-quoted `sh -c` body; avoid nesting heredocs through wsl.exe.

Reading and editing files through the UNC path with filesystem tools (editors, an agent's Read/Write/Edit tools) is fine — they don't run line-ending filters. Two checks after writing: confirm the tool writes LF (`git ls-files --eol`), and know that **overwriting an executable file via UNC drops its executable bit** (the 9P mount creates files with default modes) — git will show a `100755 => 100644` mode change; `chmod +x` from WSL to restore it. Watch for that mode line whenever a script was edited from the Windows side.

## Repair broken worktree pointers FIRST

Worktrees created by Windows-side tooling (including Claude Code's `.claude/worktrees/`) get **UNC paths baked into their gitdir linkage**, which WSL git cannot resolve. Symptom, from inside WSL:

```
fatal: not a git repository: //wsl.localhost/<distro>/home/<user>/repo/.git/worktrees/<name>
```

Diagnose by reading the two pointer files. Repair both **before any other git operation** in that worktree:

1. `<worktree>/.git` (a one-line file) — make it **relative** so that both WSL git and Windows git resolve it (relative gitdir pointers are the same mechanism submodules use, supported by old and new git alike):
   ```
   gitdir: ../../../.git/worktrees/<name>
   ```
   Count the `../` segments from the worktree back to the main repo root.
2. `<main-repo>/.git/worktrees/<name>/gitdir` — set to the **absolute Linux path** of the worktree's `.git` file:
   ```
   /home/<user>/repo/.claude/worktrees/<name>/.git
   ```

Verify from both sides: `git status` via wsl.exe, and (read-only) `git -C <UNC-path> --no-optional-locks status`. The relative pointer is what keeps Windows-side UIs (like the Claude Code desktop change-viewer) able to read the worktree; an absolute Linux path in file 1 would lock Windows out entirely.

Do not "work around" the broken pointer with `GIT_DIR`/`GIT_WORK_TREE` environment overrides: your commands will work while the user's own `git status` stays broken.

## Harden the repo against line-ending conversion

Client configs differ per machine; only `.gitattributes` travels with the repo. Choose one policy:

- `* -text` — never convert anything, in either direction. Choose this when byte-exactness matters (content-addressed files, captured artifacts, hashes derived from file bytes). It disarms any client's `autocrlf` completely.
- `* text=auto eol=lf` — normalize text files to LF in the repo *and* the working tree. Choose this for ordinary source/docs repos; note it may rewrite files on the next touch, which shows up as diffs.

Audit where conversion settings come from when something is rewriting files:

```
git config --show-origin --get-all core.autocrlf
```

Expect to find `autocrlf=true` in `C:/Program Files/Git/etc/gitconfig` — that is the Git for Windows *system* default and it applies whenever Windows git touches any repo, which is why the repo-level `.gitattributes` matters more than any config advice.

## Recover a CRLF-damaged checkout

If Windows git ever checked files out into the WSL repo, WSL git will show many files modified. Confirm the damage is line-endings-only:

```
git diff --stat --ignore-cr-at-eol   # empty output = CRLF-only damage
git ls-files --eol                   # w/crlf rows show affected files
```

If confirmed, restoring the working copies (`git checkout -- <paths>` or `git stash`) discards only the CRLF rewrite — but it is still a discard operation: show the user the evidence and get their OK first. CRLF working copies also break tools that parse repo files byte-wise (regexes where `.` excludes `\r`, hash checks), so don't leave them in place.

## Checklist for a new WSL-side repo touched from Windows

1. Pointer files of any worktree repaired (relative + absolute-Linux, above).
2. `.gitattributes` policy committed (`* -text` or `* text=auto eol=lf`).
3. All git/build commands routed through `wsl.exe` (with `MSYS_NO_PATHCONV=1` from MSYS shells).
4. `git ls-files --eol` shows no unexpected `crlf`/`mixed` rows in index or working tree.
5. Author identity resolves inside WSL (`wsl.exe -- git config user.email`).
