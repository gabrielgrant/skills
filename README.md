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
- **Obsidian inside WSL (WSLg)**: for vaults, the community workaround that moves the *app* across the boundary instead of the repo, restoring ext4 speed. In real-world testing it is not usable: WSLg's rendering of Electron apps is badly broken (mangled icons, general visual glitches). Kept here as a warning rather than a recommendation.

## Layout

```
skills/
├── wsl-repos-from-windows/SKILL.md
└── windows-repos-from-wsl/SKILL.md
```

There is deliberately no top-level routing skill. The routing key is **which filesystem the repo lives on**, and every way that fact shows up in context routes to one skill: `\\wsl.localhost\...` paths → `wsl-repos-from-windows`; `/mnt/<drive>/...` *or* Windows-native `C:\...` paths → `windows-repos-from-wsl` (a Windows-based agent sees a Windows-side repo as `C:\...`, so that skill triggers on both spellings). The placement decision for *new* repos is handled by `wsl-repos-from-windows` (the preferred layout), which points to its sibling for the exceptions. Revisit if this collection grows.

## Running the agent itself: which side?

The skills above are about repos; there's a prior human-setup question of where the *agent* runs. This belongs here rather than in a skill — an agent can't relocate its own host process. Options, from most to least integrated:

1. **Claude Code CLI inside WSL** ([official guidance](https://code.claude.com/docs/en/setup)): everything Linux-native, the boundary disappears, and sandboxing works (it doesn't on native Windows). Terminal UI only.
2. **Windows desktop app executing Windows-side, reaching WSL via `wsl.exe`** — the mode these skills were written from. Full desktop UI; every command crosses the boundary, hence the hygiene above. Note the desktop app has no documented WSL execution target (its environments are Local, Remote/cloud, and SSH).
3. **CLI inside WSL + [Remote Control](https://code.claude.com/docs/en/remote-control.md)**: `claude --remote-control` (or `/remote-control` in a running session) makes the WSL session controllable from claude.ai/code in a browser or the mobile app — *not* from the Windows desktop app. Execution is fully Linux-side; UI is a web tab. Operational caveats: the local process must stay alive (tmux/screen is the community answer; nothing official, no auto-restart), a session that loses network for >10 minutes exits, and a failed reconnect requires re-running `/remote-control`. On resume: current docs state Remote Control sessions appear in the normal `--resume` picker and reconnect automatically; field reports of sessions that could only be reattached by session ID likely involve the headless server mode (`claude remote-control`, no dashes) rather than the interactive flows — if resumability matters, start interactively and enable `/remote-control` from within.
4. **Desktop app → SSH → sshd inside WSL** (plausible, unverified for WSL): the desktop app's SSH environment pointed at a loopback sshd running in the distro would give desktop UI with true Linux-side execution — the closest analog of VS Code Remote-WSL. Untested; the docs don't address WSL as an SSH target specifically.

## Open next steps

- Run the skill eval loop (baseline-vs-skill comparison on realistic prompts) and trigger-description optimization; neither has been done yet.

## License

[MIT](LICENSE)
