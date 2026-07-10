---
name: windows-repos-from-wsl
description: Keep a git repo that lives on the Windows filesystem (a C:\ path from Windows, /mnt/c/... from WSL) healthy when WSL is part of the environment at all. Use whenever the repo's path is under /mnt/<drive> or is a Windows-native path like C:\Users\...\Documents — including when a Windows-based agent works on it directly — e.g. an Obsidian vault, a OneDrive/Documents folder, or any repo kept Windows-side because an app cannot open \\wsl.localhost paths; and whenever such a repo shows phantom modified files, executable-bit (100755/100644) diffs, line-ending churn, files that will not stat with very long names, or git status taking tens of seconds.
---

# Windows-side repos used from WSL

Some apps force the repo onto the Windows filesystem: Windows Obsidian cannot open a vault over `\\wsl.localhost` (Electron's file watcher fails with `EISDIR` over the 9P mount), and sync clients (OneDrive) only watch Windows paths. Everything else being equal, prefer the WSL-side layout (see the `wsl-repos-from-windows` skill) — Microsoft's own guidance is to avoid working across filesystems. When a Windows app pins the repo to `C:\`, the goal becomes: make the repo safe to touch from either side, and route heavy operations to the fast side.

**Default to WSL compatibility even here.** The hygiene below costs a Windows-only user essentially nothing — eol attributes, long-path support, and junk ignores are good practice on pure-Windows repos too — while making the repo permanently safe for Linux tooling. The only genuinely WSL-motivated choice is `eol=lf`; a repo owned by a Windows toolchain that insists on CRLF working trees can flip that one attribute line and keep everything else. So there is no separate "Windows-primary" variant of this skill: apply it as written, and treat the eol line as the single knob.

## The performance inversion

For WSL-side repos, WSL git is fast and Windows git is the hazard. Here it inverts: **Windows git is native and fast; WSL git works correctly but crawls through the 9P bridge** (a `git status` that is instant natively can take 30+ seconds on a small repo from WSL — mostly filesystem waiting, not CPU). So:

- Route bulk operations (status/log/diff over many files, clones, checkouts) to whichever git is native to the repo's side — here, Windows git — once the hygiene below makes that safe.
- WSL git remains fine for scripted or occasional operations; just expect latency proportional to file count.
- Avoid *alternating* sides rapidly: each git sees different `stat()` identities, so every switch invalidates the index stat-cache and forces a full re-hash — worst on the slow side.

From the Windows side there are exactly two viable routes into such a repo, and one dead end. These apply equally to a Windows-based agent and to the human at a Windows shell:

1. **Native Windows git on the `C:\` path** — fast; safe once the hygiene below is committed. This is Git for Windows (`git.exe`): the same binary whether invoked from PowerShell, cmd, or Git Bash, and the same one a Windows-based agent uses — so human and agent behavior match with no extra installation. Two checks before trusting it: identity (`git config user.email` — Windows git configs are separate from WSL's and often lack it) and age (`git --version`; Git for Windows self-updates only on request via `git update-git-for-windows`, so years-old versions are common).
2. **`wsl.exe -- git -C /mnt/<drive>/<path> ...`** (or plain `git` from a WSL shell) — runs the Linux toolchain and WSL identity against the Windows repo; correct but slow, for humans just as much as for agents. Prefer route 1 for interactive status/log/diff and use this when the WSL toolchain or identity specifically matters.
3. ~~`\\wsl.localhost\<distro>\mnt\<drive>\...`~~ — looks like it should loop Windows → WSL → back to the Windows drive, and the mount points do appear in the UNC namespace, but WSL's 9P server does not proxy drvfs mounts: directory entries enumerate as nameless ghosts and file access fails with "Access is denied". Do not attempt this route.

## Hygiene: make both gits agree

All of this is one-time setup; commit what can be committed so it protects every clone.

**1. Line endings — commit a `.gitattributes`:**

```
* text=auto eol=lf
*.bat text eol=crlf
*.cmd text eol=crlf
```

For a notes vault, explicit types (`*.md text eol=lf`, `*.json text eol=lf`) work equally well. The point is that attributes travel with the repo and override every client's `core.autocrlf` — including the `autocrlf=true` that Git for Windows ships in its system config. Additionally set the user-level default on the Windows side: `git config --global core.autocrlf false`. If different defaults per filesystem are wanted, git's conditional includes scope config by repo location:

```
[includeIf "gitdir:/mnt/c/"]
    path = ~/.gitconfig-windows-repos
```

**2. Permission bits:** NTFS has no Unix mode bits; through `/mnt` every file looks 0777 or similar. Ensure `core.fileMode=false` is set in the repo config (git usually auto-detects this at init/clone; verify with `git config core.filemode`). Otherwise WSL git reports phantom `old mode 100755 / new mode 100644` diffs on everything.

**3. Long paths:** Windows git fails to stat paths beyond ~260 chars and reports those files as perpetually modified — while WSL git sees a clean tree. Auto-generated filenames (note titles, exports) hit this constantly. Fix on the Windows side: `git config core.longpaths true`. Diagnose the discrepancy by running status from both sides.

**4. Case sensitivity:** NTFS is case-insensitive, so `core.ignorecase=true` will be set here. Never do case-only renames (`Readme.md` → `README.md`) in these repos, and avoid filenames differing only by case — WSL git will happily create both, and Windows can then only see one.

**5. No symlinks** in the repo: `core.symlinks` support differs per side and per Windows privilege level; symlinks checked out by one side appear as plain files or break on the other.

**6. Ignore Windows and app litter:**

```
Thumbs.db
desktop.ini
~$*
```

For Obsidian vaults, also ignore per-device churn: `.obsidian/workspace.json`, `.obsidian/workspace-mobile.json` (and consider whether plugin state under `.obsidian/` should be shared or per-machine at all).

**7. Credentials:** WSL git and Windows git keep separate credential stores. To share the Windows Git Credential Manager from WSL:

```
git config --global credential.helper "/mnt/c/Program\\ Files/Git/mingw64/bin/git-credential-manager.exe"
```

(SSH remotes with a key in WSL's `~/.ssh` sidestep this entirely.)

## Troubleshooting phantom changes

| Symptom | Likely cause | Check | Fix |
|---|---|---|---|
| Everything modified after switching sides | eol conversion or stat-cache flush | `git diff --stat --ignore-cr-at-eol` empty? | `.gitattributes` above; stick to one side |
| Mode 100755/100644 diffs | fileMode on `/mnt` | `git config core.filemode` | set `false` in repo config |
| One long-named file always modified (Windows only) | 260-char path limit | run status from WSL — clean? | `core.longpaths true` on Windows |
| File visible in git, missing on disk (or vice versa) | case-only duplicate | `git ls-files \| sort -f \| uniq -di` | rename apart; avoid case-only names |

## The Obsidian question

Running Obsidian *inside* WSL (via WSLg) is sometimes suggested as a way to keep the vault on ext4, but in practice it is not viable: WSLg renders Electron apps poorly (broken icon rendering and other visual glitches, non-native look) to the point of being unusable for daily writing. Treat vault-on-Windows-filesystem as the realistic setup and apply everything above — a vault is exactly the kind of repo that accumulates long auto-generated filenames and per-device churn.
