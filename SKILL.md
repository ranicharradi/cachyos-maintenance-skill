---
name: cachyos-maintenance
description: Audit-first, approval-gated maintenance for installed CachyOS systems using pacman and Shelly. Use when the user wants to inspect, update, upgrade, repair, or clean CachyOS; list repository or AUR updates through Shelly; review PKGBUILDs; handle .pacnew/.pacsave files; check CachyOS repositories, mirrors, kernels, NVIDIA/ZFS companions, boot entries, snapshots, or reboot state; remove orphans; trim package caches; or verify a system after an upgrade. Trigger for requests such as "is it safe to update CachyOS?", "use Shelly", "run -Syu", "check my Cachy box", or "clean pacman". Apply Arch's no-partial-upgrade rules, but verify CachyOS-specific behavior and current Shelly syntax against official sources rather than assuming Arch defaults.
---

# CachyOS maintenance

Maintain an installed CachyOS system conservatively. Gather facts and report first; make system changes only after showing the exact plan and receiving explicit approval.

## Scope and safety contract

1. Confirm `/etc/os-release` identifies CachyOS. If it does not, stop using CachyOS-specific commands and route to the appropriate distribution workflow.
2. Run relevant read-only checks without asking permission for each command. Never turn an audit-only request into an upgrade or cleanup.
3. Label proposed commands **read-only** or **changes system: yes**. Before mutation, summarize purpose, risk, rollback path, downloads, removals, and likely reboot need; then wait for approval.
4. Re-check current official documentation when package-manager recovery, repository layout, boot tooling, or CachyOS utilities may have changed. Cite the pages consulted.
5. Preserve recovery state. Do not delete snapshots, user data, dotfiles, configuration, every cached package, or the known-good kernel.

## Pacman rules inherited from Arch

These rules apply to CachyOS because it uses Arch repositories and pacman.

- Perform full upgrades. Ordinary repository flows are `sudo pacman -Syu` and, on current Shelly, `shelly upgrade standard`.
- During normal maintenance, never run `pacman -Sy <package>`, `pacman -Syuw`, or any other database refresh without completing the full upgrade. Use `checkupdates -d` to pre-download updates without changing the live sync database.
- The same boundary applies through Shelly: do not run standalone `shelly sync` for routine maintenance, and do not use `shelly update standard <packages>`, which upstream identifies as a partial upgrade. To install a repository package safely, use `shelly install standard <package> --upgrade`.
- Prefer the explicit, locally supported `shelly upgrade standard` command for the repository transaction. Run other approved backends only after it succeeds. Do not use `shelly upgrade all` in this workflow because independent backends may continue after one backend fails. Shelly's CLI has changed; inspect `shelly --version` and `shelly --help` instead of treating a bare `shelly` command from older documentation as a stable automation interface.
- Treat isolated `-Sy` commands in official CachyOS recovery or repository-migration documents as narrow, transactional exceptions. Follow the complete current procedure; never extract that command for routine use.
- If `-Syu` fails after databases synchronize, resolve the error and complete the transaction before installing or removing other packages.
- Never improvise with `--overwrite`, `-Rdd`, blanket `--noconfirm`, or signature-check bypasses. Use an exceptional option only when a current official notice applies exactly, and explain its scope.
- Use `pacman` or an approved CachyOS frontend for system packages. Do not use Pamac or PackageKit frontends such as Discover/GNOME Software for a rolling system upgrade; their Flatpak support is a separate concern.
- Treat AUR packages as unofficial. Review the PKGBUILD, sources, install file, and diff; never use `--skipreview`.
- Inspect `.pacnew` and `.pacsave` files and merge deliberately. Never replace live configuration blindly.

## Source routing

Use the source that owns the behavior:

Read [CachyOS differences from generic Arch maintenance](references/cachyos-differences.md) when work touches Shelly, CachyOS repositories or mirrors, kernels and companion modules, `chwd`, boot managers, snapshots, `cachy-update`, or recovery.

1. Inspect the live machine for installed packages, configuration, filesystem, boot manager, and logs.
2. Use the current [CachyOS Wiki](https://wiki.cachyos.org/) and official CachyOS source repositories for CachyOS tools, optimized repositories, mirrors, kernels, `chwd`, snapshots, boot managers, and which package frontend new installations provide.
3. Use the current upstream [Shelly CLI Reference](https://www.seafoam-labs.org/shelly-alpm/docs/cli-reference/) plus the installed `shelly --help` for Shelly syntax.
4. Use local man pages and the offline or live Arch Wiki for inherited pacman, makepkg, AUR, systemd, and general Arch behavior. Report the version of `arch-wiki-docs` if relying on its snapshot.
5. Before an upgrade verdict, check both current [Arch News](https://archlinux.org/news/) and [CachyOS Announcements](https://discuss.cachyos.org/c/announcements/5). Match manual-intervention notices to the installed package versions and configuration.

Do not treat `informant` as required. If installed, `informant list --unread` is a useful extra Arch News check; do not mark items read without reviewing them. `shelly news` also covers Arch News and tracks viewed entries, but neither covers CachyOS announcements. Do not assume `cachy-update` covers CachyOS announcements, Shelly-managed AUR packages, or `.pacnew` review.

## Workflow

Keep audit, plan, execution, configuration merge, cleanup, and verification as separate phases. Approval for an upgrade does not imply approval for orphan removal, cache trimming, configuration replacement, snapshot deletion, or bootloader changes.

### 1. Audit

Select commands relevant to the request; do not dump unrelated output. Missing optional tools are findings, not reasons to install them during the audit.

Identity and package state:

```bash
cat /etc/os-release
hostnamectl
uname -a
uptime
pacman -Q arch-wiki-docs 2>/dev/null || true
pacman-conf --repo-list
pacman -Qm
pacman -Qtdq
```

Updates and news:

```bash
checkupdates
checkupdates_status=$?
if (( checkupdates_status == 2 )); then
    echo "No repository updates available."
elif (( checkupdates_status != 0 )); then
    echo "checkupdates failed with status $checkupdates_status." >&2
fi
if command -v shelly >/dev/null; then
    shelly --version
    shelly_updates_status=0
    command shelly list-updates all || shelly_updates_status=$?
    if (( shelly_updates_status != 0 )); then
        echo "Shelly update audit incomplete (status $shelly_updates_status); identify failed backends in its diagnostics." >&2
    fi
fi
command -v informant >/dev/null && informant list --unread
```

New CachyOS GUI installations normally use Shelly since release 26.04, when it replaced Octopi. Release 26.06 also removed `paru`. Do not detect or install a separate AUR helper. If Shelly is absent, continue the repository audit with `checkupdates` and report that Shelly/AUR state was not inspected. `checkupdates` exit status `2` means no updates. Any other nonzero status is an audit failure, not evidence that no updates are available. Fetch current Arch News and CachyOS Announcements separately before declaring an upgrade safe.

`shelly list-updates all` can print successful results while another backend fails. Capture its status immediately, preserve diagnostics, and mark each failed backend as **uninspected**, not up to date. If diagnostics do not identify the failure, query the relevant backends separately with `shelly list-updates <backend>` and check each status. A skipped or unavailable backend is also uninspected. Report successful checks separately; do not give a complete upgrade verdict while an in-scope backend remains uninspected.

For each reported AUR update and its required AUR dependencies, complete the read-only clone and inspection in [AUR-only requests](#aur-only-requests) before proposing the upgrade.

Health, configuration, disk, and cache:

```bash
systemctl --failed
journalctl -p 3 -xb --no-pager
command -v checkrebuild >/dev/null && checkrebuild
pacdiff -o
command -v shelly >/dev/null && shelly utility --pacfiles --output
find /etc -type f \( -name '*.pacnew' -o -name '*.pacsave' \) -print 2>/dev/null
df -hT
du -sh /var/cache/pacman/pkg 2>/dev/null
```

For user-level package cache sizes, inspect only Shelly's documented cache locations. Do not scan or clean unrelated home-directory content.

CachyOS integration and recovery readiness:

```bash
command -v kerver >/dev/null && kerver
find /usr/lib/modules -mindepth 2 -maxdepth 2 -name pkgbase -print -exec cat {} \;
command -v dkms >/dev/null && dkms status
command -v chwd >/dev/null && sudo chwd --list-installed
command -v sbctl >/dev/null && sbctl status
pacman -Qq | rg '^(linux|linux-lts|linux-zen|linux-hardened|linux-cachyos.*|nvidia.*|zfs.*|limine.*|grub.*|refind|systemd-boot-manager|snapper|snap-pac|limine-snapper-sync)$'
findmnt --target /
findmnt --target /boot 2>/dev/null || true
findmnt --target /boot/efi 2>/dev/null || true
```

Inspect `/etc/pacman.conf` includes and the timestamps of `/etc/pacman.d/mirrorlist` plus every installed `cachyos*-mirrorlist`; do not assume the Arch mirrorlist is the only one. Detect the boot manager and filesystem from evidence rather than asking the user to choose from generic Arch defaults.

If root is Btrfs and Snapper is installed, inspect configurations, the latest snapshots, `snap-pac`, and relevant timers. A snapshot is rollback convenience, not a backup.

Treat every `limine-snapper-sync` invocation, including `limine-snapper-sync --help`, as **changes system: yes**. The installed command ignores that argument and performs a sync that rewrites `/boot/limine.conf`. Inspect the packaged executable or current upstream source instead of probing it for usage.

### 2. Classify and plan

Report findings in this order:

- **Critical before upgrade:** manual intervention, a partial/interrupted upgrade, broken package database/keyring, insufficient disk, mismatched kernel companions, missing boot artifacts, or current filesystem/service failure that makes upgrading unsafe.
- **Review before upgrade:** high-risk packages, AUR diffs, `.pacnew`, stale/unhealthy mirrors, unexpected foreign packages, or snapshot/recovery gaps.
- **Optional maintenance:** reviewed orphans, conservative cache trim, or nonurgent service cleanup.
- **No action:** checks that passed.

Flag updates involving these categories for explicit review:

- `pacman`, keyrings, mirrorlist/repository packages, `glibc`, OpenSSL, systemd, firmware, Mesa, desktop stack;
- every installed `linux-cachyos*` kernel and matching precompiled NVIDIA/ZFS companion or DKMS module;
- `chwd`, graphics drivers, `cachyos-settings`, `shelly`, `cachy-update`, Limine, `systemd-boot-manager`, GRUB, rEFInd, `snapper`, `snap-pac`, and `limine-snapper-sync`.

Confirm adequate `/boot`, root, and package-cache space. Verify that at least one known-good kernel/recovery path remains. Do not change optimized repository architecture as part of an ordinary upgrade.

Present the proposed commands and wait. Example:

```text
changes system: yes: sudo pacman -Syu
Purpose: complete repository upgrade
Risk: kernel and graphics stack change; reboot expected
Recovery: retained fallback kernel + verified snapshot/local package cache
```

### 3. Execute an approved full upgrade

Choose one explicit route:

- Official repositories: `sudo pacman -Syu`.
- Shelly repositories: `shelly upgrade standard`.
- CachyOS frontend: `cachy-update` only when installed and the user approves its complete interactive scope. Do not assume it updates AUR packages managed through Shelly.
- AUR after a successful repository upgrade: use `shelly upgrade aur --check` only when every available AUR update and required AUR dependency was reviewed and approved. If the user approved only a subset, use `shelly install aur <exact-packages> --check`. Apply the dependency and revision checks in [AUR-only requests](#aur-only-requests) during either flow. Shelly performs its own review. The `--check` option also runs each PKGBUILD's `check()` function; it is not the security-review switch. Do not install or invoke a separate AUR helper.
- Flatpak or AppImage: run each approved backend separately and only after the repository upgrade succeeds.

Do not run multiple frontends redundantly. Capture warnings, optional-dependency changes, hook failures, initramfs/boot-entry failures, package replacements, and files saved as `.pacnew`/`.pacsave`. A successful pacman exit does not prove boot artifacts or out-of-tree modules are healthy.

If the approved transaction fails, stop unrelated maintenance, diagnose the exact failure, consult current official instructions, and finish the full upgrade. Do not retry by weakening safety checks.

Some official troubleshooting pages contain destructive recovery snippets. Confirm the diagnosed condition and preserve recoverable state before proposing cache, keyring, sync-database, or lock removal; never apply a generic snippet merely because an error string looks similar.

### 4. Review configuration files

Use `pacdiff -o` or `shelly utility --pacfiles --output` to enumerate. Use `pacdiff`, explicit diffs, or an approved interactive `shelly utility --pacfiles --backup --threeway` workflow to review and merge. Explain meaningful changes and back up the live file before an approved merge. Re-run the non-mutating enumeration afterward.

`cachy-update` is not evidence this phase is complete; check independently.

### 5. Perform separately approved cleanup

Show exact sizes and package names first.

- `paccache -r` keeps three cached versions by default.
- `paccache -ruk0` removes cached versions of uninstalled packages.
- With current Shelly, `shelly purify standard --dry-run --orphans --cache 3` can preview its cleanup plan. Remove `--dry-run` only after approval of the exact targets.
- Review `pacman -Qtdq` output before proposing `sudo pacman -Rns <exact-list>`. An orphan may still be intentionally useful.

Avoid `pacman -Scc`; it destroys routine package-cache rollback. Never delete snapshots as generic package cleanup.

### 6. Verify and report

After an upgrade, re-check:

```bash
systemctl --failed
journalctl -p 3 -b --no-pager
pacdiff -o
command -v checkrebuild >/dev/null && checkrebuild
command -v dkms >/dev/null && dkms status
find /usr/lib/modules -mindepth 2 -maxdepth 2 -name pkgbase -print -exec cat {} \;
```

Also verify the detected boot manager's artifacts, the upgraded kernel/module pair, snapshot creation if expected, and free space. Recommend reboot when the kernel, microcode, firmware, systemd, graphics stack, or boot components changed. Never reboot without approval.

End with:

```text
System: CachyOS version, running kernel, boot manager, root filesystem
Sources checked: CachyOS Wiki/announcements, Arch News/Wiki/man pages
Audit: commands run and important results
Changes: exact approved commands and outcomes, or "none"
Config: .pacnew/.pacsave state
Recovery: fallback kernel, snapshot/cache status
Reboot: required / recommended / not needed, with reason
Remaining: Critical / Review / Optional / No action
```

## AUR-only requests

An AUR review does not authorize installation. Check for an official CachyOS/Arch repository equivalent. `shelly search aur <package> --pkgbuild` displays only the PKGBUILD, so it is not sufficient for a full review.

Resolve each package's `PackageBase` through AUR metadata, such as `shelly search aur <package> --detail --json`, after checking installed help. A split package's name may differ from its Git repository name. Use the reported package base for the clone URL and directory; keep the selected package names for installation. Review each unique package base once and inspect every tracked file without sourcing the PKGBUILD, building, or installing:

```bash
review_root=$(mktemp -d)
command git clone -- "https://aur.archlinux.org/<package-base>.git" "$review_root/<package-base>"
command git -C "$review_root/<package-base>" ls-files
command git -C "$review_root/<package-base>" log --oneline -n 10
command git -C "$review_root/<package-base>" rev-parse HEAD
```

Inspect source URLs and checksums, `prepare()`, `build()`, `check()`, `package()`, install scripts, patches, service units, privilege escalation, destructive or obfuscated commands, unexpected network access, binary blobs, and upstream identity. If the previously reviewed commit is known, show its diff against the current tree. Otherwise, state that the prior diff is unavailable and review the complete current tree plus recent history. Shelly performs a separate review during its install flow. The `--check` option runs the PKGBUILD's `check()` function; it is not the security-review switch. An approved install is `shelly install aur <package> --check`; do not install or invoke another AUR frontend. Propose installation only after review, and preserve the full-upgrade rule.

Read `.SRCINFO` and the reviewed PKGBUILD to resolve runtime (`depends`), build (`makedepends`), and enabled test (`checkdepends`) dependencies, including architecture-specific and split-package requirements. Recursively review every AUR dependency that will be built or installed, including newly introduced ones absent from the update list. Include repository dependencies and any selected optional dependencies in the transaction plan. If static inspection cannot resolve a dependency, state the uncertainty; do not execute an unreviewed PKGBUILD to discover it.

Record selected package names, package bases, reviewed Git commit IDs, and dependency scope in the approval plan. At Shelly's review and confirmation prompts, compare its actual build files, revisions, and transaction targets with that plan before allowing builds or installation. If a revision changed, files differ, or an unapproved dependency or target appears, pause the affected AUR work, inspect the new files and diff, and obtain approval for the changed scope. If the checkout or revision cannot be verified, stop that AUR transaction. Do not treat Shelly's own review as permission to expand the user's approval. A recorded AUR commit identifies the packaging recipe; it does not pin moving VCS source revisions.

Sources: [Arch AUR review and dependency guidance](https://wiki.archlinux.org/title/Arch_User_Repository#Installing_and_upgrading_packages), [Shelly CLI reference](https://www.seafoam-labs.org/shelly-alpm/docs/cli-reference/).

## Recovery boundaries

Prefer, in order: diagnosis, completion of the transaction, a verified filesystem snapshot, and exact packages in the local cache. Do not assume the Arch Linux Archive contains CachyOS-built packages. Use the Arch archive only for a confirmed Arch-origin package and a procedure verified against current Arch guidance. Never pin a kernel while upgrading an incompatible companion module; pin coherent package sets temporarily and document their removal.
