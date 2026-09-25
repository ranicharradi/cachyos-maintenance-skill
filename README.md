# cachyos-maintenance

An audit-first CachyOS maintenance skill for AI coding agents. It uses the [Agent Skills](https://agentskills.io) format and includes Codex interface metadata.

The skill treats CachyOS as its own distribution where behavior differs from Arch Linux. It uses CachyOS sources for Shelly, repositories, kernels, graphics, boot tooling, and snapshots. It applies Arch's full-upgrade rules to inherited pacman behavior.

## What it covers

- Read-only audits of updates, news, services, logs, package files, disk space, mirrors, kernels, modules, boot artifacts, and recovery state
- Separate checks for Arch News and CachyOS Announcements
- Full repository upgrades through `shelly upgrade standard` or `sudo pacman -Syu`
- Complete AUR source review before approved Shelly builds
- Deliberate `.pacnew` and `.pacsave` handling
- Approval-gated orphan removal and bounded cache cleanup
- CachyOS-specific checks for `chwd`, Limine, Snapper, NVIDIA, ZFS, and optimized repositories
- Post-upgrade verification and reboot guidance

The agent starts with read-only inspection. It reports the exact command, scope, risk, recovery path, and reboot need before any package, service, boot, configuration, or cleanup change.

## Requirements

- An installed CachyOS system
- `pacman` and `pacman-contrib`
- Shelly when it is present on the system
- Optional tools such as `informant`, `arch-wiki-docs` or `arch-wiki-lite`, `checkrebuild`, Snapper, and boot-manager utilities are detected rather than installed automatically

## Installation

### Codex

```bash
git clone https://github.com/ranicharradi/cachyos-maintenance-skill ~/.codex/skills/cachyos-maintenance
```

### Claude Code

```bash
git clone https://github.com/ranicharradi/cachyos-maintenance-skill ~/.claude/skills/cachyos-maintenance
```

For a project-local installation, clone it into the agent's project skill directory instead.

## Usage

Use ordinary maintenance requests, for example:

- "Is it safe to update CachyOS?"
- "Use Shelly to update my system."
- "Review these AUR updates."
- "Check my CachyOS kernels and Limine entries."
- "Clean the pacman cache without removing recovery options."

The skill stops after its audit and proposed plan unless the user explicitly approves the system changes.

## Files

- `SKILL.md` contains the main workflow and safety boundaries.
- `references/cachyos-differences.md` explains CachyOS-specific routing and sources.
- `evals/evals.json` contains behavioral regression scenarios.
- `agents/openai.yaml` provides Codex interface metadata.

## License

[MIT](LICENSE)
