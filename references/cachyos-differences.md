# CachyOS differences from generic Arch maintenance

Load this reference when a task touches CachyOS-owned integration. Re-check the linked official sources before mutation because commands, packages, and repository layout can change.

## Contents

- What carries over unchanged
- What to remove or replace from an Arch-only skill
- Shelly routing
- News gates
- Repositories and mirrors
- Kernels, graphics, and companion modules
- Boot manager routing
- Btrfs and snapshots
- `cachy-update`
- Recovery
- Primary sources

## What carries over unchanged

- Pacman supports full system upgrades; partial upgrades are unsupported.
- Routine updates use `sudo pacman -Syu`.
- AUR packages remain unofficial and require PKGBUILD/diff review.
- `.pacnew`/`.pacsave`, orphan, cache, service, journal, disk-space, and rebuild checks remain relevant.
- Upgrade, cleanup, configuration replacement, boot changes, and reboot remain separate approval gates.

The CachyOS FAQ explicitly sends regular maintenance guidance to the Arch Wiki. It also shows `pacman -Syu` as the normal update path.

## What to remove or replace from an Arch-only skill

| Arch-only assumption | CachyOS treatment |
| --- | --- |
| `informant` is mandatory | Optional if installed. Independently check Arch News and CachyOS Announcements. |
| A standalone AUR helper is assumed | CachyOS has used Shelly as the default frontend since 26.04 and removed the standalone AUR helper in 26.06. Use Shelly's built-in AUR support and do not add another frontend. |
| Offline Arch Wiki is the top source for every fact | Use current CachyOS Wiki/source for CachyOS-owned tooling; use Arch docs for inherited behavior. |
| Only `/etc/pacman.d/mirrorlist` matters | Inspect Arch and all installed `cachyos*-mirrorlist` files. Use `cachyos-rate-mirrors` for approved repair/ranking. |
| Kernel checks mean `linux`/`linux-lts` plus generic DKMS | Enumerate installed `linux-cachyos*` kernels and match precompiled NVIDIA/ZFS companions or DKMS builds to each kernel. |
| Regenerate GRUB after kernel work | Detect the manager. Limine entries are normally hook-managed; systemd-boot uses `sdboot-manage`; GRUB uses its own generator; rEFInd has different configuration. |
| Arch Linux Archive is a general rollback source | It does not cover arbitrary CachyOS builds. Prefer snapshots and local cache; confirm package origin before using the Arch archive. |
| Cleanup may remove old snapshots | Package cleanup never implies snapshot cleanup. Snapshots are separately governed recovery state. |
| An updater frontend completed every maintenance phase | `cachy-update` does not replace independent news matching, `.pacnew` review, or post-upgrade kernel/boot checks. |

## Shelly routing

The [CachyOS GUI Installer changelog](https://wiki.cachyos.org/cachyos_basic/changelogs/gui_installer/) records that release 26.04 replaced Octopi with Shelly. Release 26.06 removed the previously bundled standalone AUR helper from new installations. This workflow supports Shelly only; do not detect, install, or invoke another AUR frontend.

Detect the installed/intended frontend:

```bash
command -v shelly
command -v shelly >/dev/null && shelly --version
```

Shelly is broader than a traditional AUR helper: it manages ALPM repository packages, AUR packages, and optionally Flatpak/AppImage applications. Verify current syntax using installed help and the upstream [Shelly CLI Reference](https://www.seafoam-labs.org/shelly-alpm/docs/cli-reference/). Prefer explicit commands:

```bash
shelly list-updates all                        # query all enabled backends
shelly upgrade standard                        # full repository upgrade
shelly search aur <package> --pkgbuild          # display an exact AUR PKGBUILD
shelly install aur <package> --check            # approved AUR install with check()
shelly upgrade aur --check                      # upgrade every reviewed AUR package
```

Apply Arch's package-state boundary through Shelly:

- Do not run `shelly sync` alone during routine maintenance; it refreshes ALPM databases without upgrading installed packages.
- Do not use `shelly update standard <packages>`; upstream explicitly labels this a partial upgrade.
- Use `shelly install standard <package> --upgrade` when adding a repository package so the system is fully upgraded first.
- Do not use `shelly upgrade all` in this workflow. Run the approved backends separately so a repository failure stops AUR, Flatpak, and AppImage work.
- Before a separate `shelly install aur` or `shelly upgrade aur`, complete or confirm the repository full upgrade and review the full AUR Git tree for every target. `shelly search aur <package> --pkgbuild` shows only the PKGBUILD and is not a complete review.
- Do not use a bare `shelly` invocation as a scripted upgrade interface. CLI behavior has changed; use explicit subcommands supported by the installed version.
- `shelly news` tracks Arch News viewed state but does not replace the CachyOS announcement check.

Shelly-native optional equivalents:

```bash
shelly utility --pacfiles --output                     # enumerate pacnew/pacsave files
shelly purify standard --dry-run --orphans --cache 3  # preview cleanup only
```

Interactive pacfile merging and a cleanup without `--dry-run` mutate state and need separate approval. If Shelly is absent, continue repository work with pacman but report AUR status as uninspected; do not install another frontend during maintenance.

## News gates

Before an upgrade verdict, inspect:

- [CachyOS Announcements](https://discuss.cachyos.org/c/announcements/5) for distribution-specific migrations, regressions, and package changes.
- [Arch News](https://archlinux.org/news/) for upstream manual interventions inherited through Arch repositories.

Match notices against installed versions and configuration; the existence of a recent notice alone does not make every machine unsafe. `cachy-update` and `informant` may surface Arch News but are not substitutes for the CachyOS announcement check.

## Repositories and mirrors

Read [Optimized Repositories](https://wiki.cachyos.org/features/optimized_repos/) before modifying repository tiers. Inspect rather than infer:

```bash
pacman-conf --repo-list
command -v kerver >/dev/null && kerver
rg -n '^(\[|Include|Server|Architecture|IgnorePkg)' /etc/pacman.conf /etc/pacman.d/*.conf 2>/dev/null
stat -c '%y %n' /etc/pacman.d/mirrorlist /etc/pacman.d/cachyos*-mirrorlist 2>/dev/null
```

Common CachyOS lists include `cachyos-mirrorlist`, `cachyos-v3-mirrorlist`, and `cachyos-v4-mirrorlist`, but trust the live config. Do not switch v3/v4/znver4 repository families merely because `kerver` reports CPU capability; repository migration is a separate, current-document procedure.

For an approved mirror repair, CachyOS provides:

```bash
sudo cachyos-rate-mirrors
```

This changes mirrorlists, so it needs approval. Do not substitute Arch-only mirror ranking for CachyOS repositories.

## Kernels, graphics, and companion modules

Do not hardcode a single kernel flavor. Inventory module trees and owners:

```bash
uname -r
find /usr/lib/modules -mindepth 2 -maxdepth 2 -name pkgbase -print -exec cat {} \;
pacman -Qq | rg '^(linux|linux-lts|linux-zen|linux-hardened|linux-cachyos.*|nvidia.*|zfs.*)$'
command -v dkms >/dev/null && dkms status
command -v chwd >/dev/null && sudo chwd --list-installed
```

For each installed kernel, verify the initramfs and any required precompiled NVIDIA/ZFS companion package or DKMS build. Keep a known-good fallback kernel when possible. A running old kernel immediately after an upgrade is normal until reboot; a missing module tree or boot artifact is not.

Use current [`chwd` documentation](https://wiki.cachyos.org/features/chwd/chwd/) for driver-profile changes. Do not manually replace GPU packages during routine maintenance when the profile manager owns the migration.

## Boot manager routing

Read [Boot Manager Configuration](https://wiki.cachyos.org/configuration/boot_manager_configuration/) before changing boot configuration.

- **Limine:** kernel entries are normally updated by `limine-mkinitcpio-hook`. After an approved configuration change or repair, the documented manual regeneration command is `sudo limine-mkinitcpio`.
- **systemd-boot:** CachyOS manages generation with `sudo sdboot-manage gen`.
- **GRUB:** use `sudo grub-mkconfig -o /boot/grub/grub.cfg` only when GRUB is the detected manager.
- **rEFInd:** its menu and kernel-option files take effect without running a GRUB/Limine generator; consult the current page for the installed layout.

Normal package hooks should handle kernel upgrades. Do not regenerate a boot configuration just because an upgrade occurred; inspect hook results and artifacts first. Secure Boot adds a separate signing check (`sbctl status` when installed).

Do not invoke `limine-snapper-sync` to discover its interface. The installed executable ignores `--help` and performs its normal sync, which rewrites `/boot/limine.conf`. Treat every invocation as a boot configuration change. Inspect the packaged executable or current upstream source instead.

## Btrfs and snapshots

Read [Btrfs Snapshots](https://wiki.cachyos.org/configuration/btrfs_snapshots/) for the installed boot manager and layout.

```bash
findmnt --target /
pacman -Qq | rg '^(snapper|snap-pac|limine-snapper-sync)$'
command -v snapper >/dev/null && snapper list-configs
command -v snapper >/dev/null && sudo snapper -c root list
systemctl status snapper-timeline.timer snapper-cleanup.timer --no-pager
```

If `snap-pac` is installed, confirm the expected pre/post transaction snapshots appeared. A snapshot does not replace an external backup, may not include every subvolume, and should not be deleted as incidental package cleanup.

## `cachy-update`

The official [`cachy-update` project](https://github.com/CachyOS/cachy-update) can list and apply repository updates and expose Arch News, orphan, cache, service, and reboot checks. Treat it as an optional frontend, not a hard dependency. Do not use its non-Shelly AUR integration in this workflow; audit and update AUR packages through Shelly separately.

- `cachy-update --list` lists updates when available.
- `cachy-update --news` surfaces Arch News.
- `cachy-update --services` checks failed services.
- An unqualified `cachy-update` is an interactive, mutating full-upgrade workflow and needs approval.

Do not run `cachy-update` and `pacman -Syu` for the same transaction. Independently run `pacdiff -o` after the upgrade; do not assume the frontend processed `.pacnew` files.

## Recovery

Prefer a coherent rollback rather than mixing repository generations:

1. Diagnose and complete an interrupted full upgrade.
2. Use a verified pre-transaction snapshot when its scope and boot path are understood.
3. Use exact package versions from `/var/cache/pacman/pkg` for a coherent package-set rollback.
4. Use the Arch Linux Archive only after proving the package came from an Arch repository and current Arch instructions apply.

Temporary `IgnorePkg` entries require a stated reason, a coherent set of kernel/module packages when relevant, and a removal condition. Do not imitate internal/offline-updater exclusions as a manual maintenance strategy.

## Primary sources

- [CachyOS FAQ and regular maintenance](https://wiki.cachyos.org/cachyos_basic/faq/)
- [CachyOS GUI Installer changelog](https://wiki.cachyos.org/cachyos_basic/changelogs/gui_installer/)
- [CachyOS post-install setup](https://wiki.cachyos.org/configuration/post_install_setup/)
- [CachyOS optimized repositories](https://wiki.cachyos.org/features/optimized_repos/)
- [CachyOS boot manager configuration](https://wiki.cachyos.org/configuration/boot_manager_configuration/)
- [CachyOS Btrfs snapshots](https://wiki.cachyos.org/configuration/btrfs_snapshots/)
- [CachyOS chroot recovery](https://wiki.cachyos.org/features/cachy_chroot/)
- [CachyOS announcements](https://discuss.cachyos.org/c/announcements/5)
- [Arch system maintenance](https://wiki.archlinux.org/title/System_maintenance)
- [Arch partial upgrades](https://wiki.archlinux.org/title/System_maintenance#Partial_upgrades_are_unsupported)
- [Shelly CLI reference](https://www.seafoam-labs.org/shelly-alpm/docs/cli-reference/)
