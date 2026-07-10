# skills

Agent skills for keeping git repos healthy across the Windows ⇄ WSL2 boundary, born from running the Claude Code desktop app (Windows) against repos that live in WSL.

Compatible with [vercel-labs/skills](https://github.com/vercel-labs/skills):

```bash
npx skills add gabrielgrant/skills          # install all
npx skills add gabrielgrant/skills@wsl-repos-from-windows
```

## The problem

A repo and the tools that touch it can each sit on either side of the WSL boundary, and every mismatch has its own failure mode: CRLF rewrites from Git for Windows' system-level `autocrlf=true`, worktree gitdir pointers baked as UNC paths that WSL git can't resolve (`fatal: not a git repository: //wsl.localhost/...`), phantom filemode diffs on `/mnt`, the 260-char path limit, MSYS path mangling (`/home/u/x` → `C:/Program Files/Git/home/u/x`), and 9P slowness that turns `git status` into a 30-second operation.

## The skills

| Skill | Scenario |
|---|---|
| [wsl-repos-from-windows](skills/wsl-repos-from-windows/SKILL.md) | Repo on WSL ext4 (`/home/...`), driven by a Windows-based agent (Claude Code desktop). The preferred layout: fast, Linux-native tooling; Windows only reads via `\\wsl.localhost`. Covers wsl.exe invocation, worktree pointer repair, `.gitattributes` hardening, CRLF recovery. |
| [windows-repos-from-wsl](skills/windows-repos-from-wsl/SKILL.md) | Repo pinned to the Windows filesystem (`C:\...`, `/mnt/c/...` from WSL) because a Windows app needs it there — Obsidian vaults being the canonical case (its file watcher fails with `EISDIR` over `\\wsl.localhost`). Covers dual-git hygiene, the performance inversion, long paths, case sensitivity, credential sharing. |

**Which side should a repo live on?** WSL ext4, unless a Windows app needs native filesystem semantics (file watching, sync clients). [Microsoft's guidance](https://learn.microsoft.com/en-us/windows/wsl/filesystems) is the same: don't work across filesystems.

## How this compares to other approaches

These skills teach *repo hygiene and command routing* — configuration that travels with the repo plus rules for which side runs what. Other tools attack adjacent parts of the problem and mostly compose with, rather than replace, this approach:

- **[Claude Code inside WSL](https://code.claude.com/docs/en/setup)** (Anthropic's supported option): install and launch `claude` in the WSL terminal; git, files, and shell are then all Linux-native and the boundary disappears — this also enables sandboxing, which native Windows doesn't support. The skills here exist for when you're on the Windows *desktop app* instead, whose UI runs Windows-side while your repos live in WSL.
- **[gandr](https://mcpmarket.com/server/gandr)** (MCP server): routes Claude Desktop's tool calls into Claude Code running inside WSL — solving *where commands execute*, the same goal as the wsl.exe patterns in these skills, via a persistent bridge. Orthogonal to repo hygiene: CRLF, filemode, and worktree-pointer issues still need the fixes documented here. (Description based on secondary sources; the site rate-limited direct verification.)
- **[wslgit](https://github.com/andy-5/wslgit)**: a Windows shim binary that forwards git invocations to WSL's git with path translation, mainly so VS Code's Windows-side git integration drives Linux git. Same routing idea, packaged as a fake `git.exe`; limited path translation, appears low-maintenance.
- **[VS Code Remote-WSL](https://code.visualstudio.com/docs/remote/wsl)**: the architectural fix — UI on Windows, a server component inside WSL runs all tooling natively. The gold standard when your editor supports it; even so, its docs still tell you to sort out line endings and credentials, i.e. the hygiene layer remains.
- **[git-worktree-relative](https://github.com/Kristian-Tan/git-worktree-relative)**: community tool automating exactly the relative worktree-pointer rewrite that `wsl-repos-from-windows` documents manually — prior art showing relative pointers are the portable answer (`git worktree repair` is the built-in for reattaching after moves).
- **Obsidian inside WSL (WSLg)**: for vaults, the community workaround that moves the *app* across the boundary instead of the repo, restoring ext4 speed. Trade-off: Windows-side sync/integrations.

## Layout

```
skills/
├── wsl-repos-from-windows/SKILL.md
└── windows-repos-from-wsl/SKILL.md
```
