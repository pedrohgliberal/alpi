# ALPI — Ansible Linux Post-Install

Architecture proposal (2026-10-05). Revised after an adversarial review. Profiles replaced by the
package picker on 2026-10-08 (see "Decisions (2026-10-08)").

## Goal

A single project that detects the running distro and applies the right post-install. Today it
would be the union of **AFPI** (Fedora, personal machine `noir`) and **AAPI** (AlmaLinux 10, work
machine). Later it should take other distros (Debian, Ubuntu…).

## Starting point

AFPI and AAPI have the same structure and share 6 roles (`update`, `hardware`, `common`, `apps`,
`desktop`, `ai_tools`). AFPI has two more: `nvidia` and `akmods_mok`. The two are already drifting
apart. For example, AAPI did not get the icon-ownership fix (`af80a16`) nor several
`check_mode: false`.

| Difference | Example | Nature |
|---|---|---|
| Package name | `vim` → `vim-enhanced`, `shellcheck` → `ShellCheck`, `p7zip` → `7zip` | data per distro |
| Package missing in the distro | Telegram/Drawy via Flatpak on EL10, Roboto from upstream, argyllcms/chkrootkit/unhide | data per distro |
| Repositories | RPM Fusion + COPR (Fedora); CRB + EPEL + RPM Fusion EL + Oracle (Alma) | tasks per family |
| Behavior | dnf5 × dnf4 (`repo list --json` × `repolist`), VirtualBox akmod + kmodgenca × Oracle + own key, `@groups`, `.i686`, `allowerasing` | tasks per distro |
| Tooling | `ansible` × `ansible-core`, `community.general <12` on EL10 (ansible-core 2.16) | bootstrap |
| Hardware | NVIDIA, ASUS, AMD freeworld | detected |
| Machine/user | hostname, LUKS volume aliases (`noir`'s `open/close-thevoid`: repository aliases since 2026-10-25) | per machine |

## Chosen approach: unification (B)

The discarded alternative (A) would be a thin dispatcher that calls AFPI or AAPI as submodules. It
keeps the duplication, and every new distro would become a whole repository.

### Three independent axes

1. **Distro**: detected from `ansible_facts`.
2. **Hardware**: detected (NVIDIA, ASUS, AMD, Secure Boot). The user can **only veto** it
   (`nvidia: auto | false`).
3. **Machine/user**: lives in `host_vars`, outside git: the hostname and git identity
   (`bootstrap.yml`), the package and feature choice (`selection.yml`, written by the picker) and
   hand-written settings (`custom.yml`). The thevoid UUID leaves the repository and comes here
   (2026-10-08: `luks_volumes: {thevoid: <UUID>}` in `custom.yml` generates `open-thevoid` /
   `close-thevoid`, addressing the partition by UUID instead of `/dev/nvme1n1p3`; back in the
   repository as two aliases on 2026-10-25, decision 6 of that day).

There are no profiles (removed on 2026-10-08). What a profile used to decide is a choice made on
the machine: the hostname is managed only when one is given, and packages and features are picked
from what the running distro offers (see "Package picker").

### Structure

```
alpi/
  bootstrap.sh              # reads /etc/os-release → installs ansible via dnf/apt; pins collections per version
  site.yml
  catalog.yml               # logical package ids → name per distro (see "Customization")
  tasks/env_setup.yml       # support matrix (assert) → detection → group_by → package resolution
  group_vars/
    all/                    # shared: user, git, flatpaks, zsh, ai_tools; secrets.yml (vault)
    os_Fedora/  os_AlmaLinux_10/  os_Debian_13/ …
  pick.py                   # package picker (run by bootstrap, or on its own)
  host_vars/127.0.0.1/      # git-ignored
    bootstrap.yml           # written by bootstrap (hostname, git identity)
    selection.yml           # written by pick.py (differences from the defaults)
    custom.yml              # written by the user (see custom.yml.example)
  roles/
    repos/                  # NEW, runs first: rpmfusion/epel/crb/copr | apt sources/extrepo/PPA
    update/ common/ apps/ desktop/ hardware/ nvidia/ ai_tools/
    mok/                    # separate signers: kmodgenca (/etc/pki/akmods), Oracle vboxdrv
                            # (/var/lib/shim-signed/mok), DKMS (/var/lib/dkms/mok.*). Keeps the
                            # current paths so existing machines need no re-enrollment.
```

Inside each role, the specific tasks live in `tasks/{Fedora,RedHat,Debian}.yml`.
`ansible.builtin.package` is used only for plain package lists. dnf and apt stay explicit where the
behavior differs.

### Design rules

- **Precedence:** do not use `include_vars` for distro data, because it has higher priority than
  `host_vars` and would silently override the user. Instead, `group_by` creates dynamic groups
  (`os_<distro>`, `os_<distro>_<major>`) and the data lives in `group_vars/`. That keeps `host_vars` and `-e` on top. Layers never share a key (each has its own
  `packages_*`/`features_*` names), so the order between groups does not matter;
  `ansible_group_priority` would not help anyway, since it only works in the inventory source.
- **Explicit support:** an allowlist of (distro, version) with an assert before loading any
  variable. No silent fallback by `os_family`: Fedora, Rocky and CentOS Stream have
  `os_family = RedHat`, and Ubuntu falls into `Debian`.
- **Facts:** `inject_facts_as_vars = False` still holds, so always `ansible_facts[...]`.
- **Ansible:** use the distro's own ansible-core, with the code written for the lowest supported
  version (today 2.16, EL10's). Re-evaluate a pinned venv if some distro ships an older version.

## Package customization

### Two levels

- **Features**: items that have configuration tasks (virtualbox, steam, vscode, brave, gh,
  antigravity, clamav, nvidia, asus…). Switched on and off by a flag. A feature is **never** turned
  off by deleting a package name.
- **Packages**: plain items, with no configuration, resolved through the catalog.

### Catalog (in the repository)

```yaml
# catalog.yml: logical id -> package in each distro
vim:      {Fedora: vim, RedHat: vim-enhanced, Debian: vim}
7zip:     {Fedora: [p7zip, p7zip-plugins], RedHat: [7zip, 7zip-standalone]}
telegram: {Fedora: telegram-desktop, RedHat: {flatpak: org.telegram.desktop}}
steam:    {Fedora: steam, RedHat: ~}     # ~ = unavailable: skip and warn
aliases:  {p7zip: 7zip}                  # renamed ids still resolve
```

### Repositories and feature packages (added 2026-10-05, section 2)

- A catalog entry names the repository its native packages need: `brave: {all: [brave-browser,
  brave-origin], repo: brave-browser}`. Repositories are defined by id (`alpi_repos_family` in
  `group_vars/family_<os_family>/repos.yml`, `alpi_repos_os` in `group_vars/os_*`) with a type:
  `rpm` (release package by URL), `package`, `dnf_config` (CRB), `yum` (repo file), `copr`, or
  `deb822` (apt, Debian family).
- `roles/repos` enables the distro's `alpi_repos_always` first, in order (RPM Fusion on Fedora;
  CRB, EPEL, RPM Fusion EL on AlmaLinux), then only the repositories the selected packages need.
  Skipping `brave` means the Brave repository is never added. A repository id with no definition
  for the distro is a catalog bug and stops the run in `env_setup`.
- Features add package sets while they are on (`feature_packages: {asus: [asusctl, …]}`), so
  `features: {asus: false}` drops the packages and, through them, the ASUS COPR.
- (2026-10-06) `swap: true` marks a catalog entry that replaces a conflicting package
  (`ffmpeg-free` → `ffmpeg`, `mesa-va-drivers-freeworld`). `roles/packages` installs those first
  with `allowerasing`, so AFPI's separate "swap" tasks become catalog data and get the same name
  checks. Intel and AMD video acceleration are hardware features (`intel`, `amd`) with
  `feature_packages`, like ASUS. A package that exists on EL but is deliberately left out there
  (RPM Fusion's `@multimedia`, which breaks against EPEL) goes in `packages_os`, not `~`.
- (2026-10-06) NVIDIA: the driver, build, Vulkan and VA-API packages are catalog ids in
  `feature_packages.nvidia` (Fedora only). `roles/akmods_mok` runs before `roles/packages` when
  Secure Boot is on and any `akmod-*` package was resolved, which replaces AFPI's string match and
  will cover `akmod-VirtualBox` without changes. `roles/nvidia` rebuilds with akmods only when no
  module exists for the running kernel (AFPI rebuilt on every run until the reboot).
- (2026-10-06) VirtualBox: a plain feature (`virtualbox`, on by default). The catalog names
  `akmod-VirtualBox` on Fedora, so `roles/akmods_mok` picks it up; on EL a per-distro value
  `{native: […], repo: virtualbox}` adds Oracle's repository only there. `roles/virtualbox`
  (before `roles/packages`) creates the Oracle signing key with Secure Boot and the groups.

- (2026-10-09) Python apps installed with pipx are a third kind of catalog value,
  `{pipx: "<PyPI spec>"}` (extras included, "markitdown[all]"), next to native names and
  `{flatpak: …}`; `pipx:<spec>` is the raw prefix. They are installed for the user by
  `roles/pipx`, checked on PyPI with the other name checks, and picked like any package.
- (2026-10-10) `{upstream: <source id>}` is a fourth kind of catalog value, for software the
  distro does not package (Roboto on EL, from its GitHub release). Sources are data
  (`upstream_sources`, with a type per install method) and `roles/upstream` installs them by
  root. The id stays an ordinary catalog id: selected, skipped, picked and gated on like any other,
  and `packages_absent` removes it. AAPI's warn-and-keep when GitHub cannot be reached stays.
- (2026-10-10) AI tools are features without catalog packages: `claude_code`, `antigravity_cli`
  and `antigravity_ide` (on by default, as AFPI/AAPI install all three). Their vendors ship
  installers and self-updating binaries, not packages, so `roles/ai_tools` runs each installer
  once (only when the binary is missing) and lets the tool update itself; the IDE's AppImage comes
  from the manifest the app polls, checked against its sha512. AFPI's `antigravity_ide_install`
  flag is replaced by the feature. A feature without packages is the one case where a role gates
  on the flag itself; the picker labels it from `picker_descriptions`.

### User file (outside git)

```yaml
# host_vars/127.0.0.1/custom.yml
features:      {virtualbox: false, clamav: false}
packages_add:  [vim, "pkg:sqlitebrowser", "flatpak:org.gimp.GIMP"]   # prefix = raw name, bypasses the catalog
packages_skip: [discord]
packages_absent: [akregator]   # explicit uninstall (see Removal)
```

### Rules

- **Merging:** each layer uses its own keys (`packages_base`, `packages_selection_add`,
  `packages_selection_skip`, `packages_add`, `packages_skip`). Files in the same `host_vars`
  directory do not merge a shared key (the last one loaded wins), so `selection.yml` and
  `custom.yml` never share one. A single `set_fact` builds the final list:
  `(base + selection_add + add) | unique | difference(selection_skip + skip)`. The user layer is applied last. Never use
  `hash_behaviour`.
- **Validation** (in `env_setup`, before anything changes on the system):
  - an unknown id stops the run and lists all unknown ids;
  - an id that is `~` on the current distro is skipped with a warning (or fails, with
    `strict: true`);
  - every native name (catalog and `pkg:`) and Flatpak id is checked to exist at the start of the
    `packages` role, before anything from the lists is installed: a `dnf install --assumeno` dry
    run (resolves provides, same message on dnf4/dnf5) and `flatpak remote-info flathub`. Not in
    `env_setup`, because EPEL/RPM Fusion/COPR names only resolve after the `repos` role ran.
- **Isolation:** the user's extras are installed in a separate task, after the base, so a typo does
  not break the whole install.
- **Removal:** `skip` means "do not install", **never** "uninstall". Dropping a package from the
  defaults does not uninstall it either. There is no record of what ALPI installed. Uninstalling
  happens only through `packages_absent` (catalog ids or prefixed raw names, same validation),
  with `autoremove: false` so dependencies are not cascaded away. An id in both an install list
  and `packages_absent` is an error.
- **Separate files:** bootstrap and the picker rewrite their own files (`bootstrap.yml`,
  `selection.yml`) with PyYAML, which would wipe hand-written comments. That is why `custom.yml` is
  a separate file. Since all three stay outside git, `git pull` never conflicts.

### Package picker (2026-10-08)

`pick.py` (Python, run by `bootstrap.sh` or on its own) replaces the profiles. The 2026-10-05
objection to a picker was that it would re-implement the catalog in bash; in Python it reuses the
catalog files and `filter_plugins/alpi.py` (`alpi_resolve`), so it lists exactly what the playbook
would resolve.

- **Only what this system offers:** ids that are `~` on the running distro are hidden, so are the
  packages of a desktop that is not running (same `XDG_CURRENT_DESKTOP` rule as `env_setup`) and
  features whose packages are all unavailable (Steam on EL). Packages a feature owns
  (`feature_packages`) are switched through the feature, not listed on their own.
- **Hardware is not listed:** it follows detection; vetoes stay in `custom.yml`. The picker would
  have to repeat the detection (`lspci`) to show it.
- **Defaults pre-checked:** `packages_base` + `packages_os` + the running desktop's list, and
  `features_default`.
- **Stored as differences** in `selection.yml`: `packages_selection_add`, `packages_selection_skip`,
  `features_selection`. Defaults added to the repository later still arrive with `git pull`.
  Saved entries for ids the picker does not list this time (another desktop's packages) are kept;
  ids no longer in the catalog are dropped with a warning (the playbook fails on them until then).
- **Interface:** `whiptail` checklists (bootstrap installs `newt`), plain prompts otherwise or with
  `ALPI_PICKER=text`. Cancel leaves `selection.yml` as it was.
- **Groups by purpose (2026-10-21):** a single list of ~60 ids grouped by install method (the
  catalog's sections) was confusing, with features on a screen of their own. Now a menu of groups
  (`picker_groups` in `group_vars/all/picker.yml`: System tools, Security, Internet & chat, …), each
  opened as a checklist of packages and features together, labelled from `picker_descriptions`
  with the source when it is not the distro's packages (Flathub, pipx, upstream release), and a
  review of the changes from the defaults before saving. The text mode lists every group in one
  numbered list. Only the picker reads `picker.yml`; ids it does not name fall under "Other". No terminal, no picker: the
  playbook uses the defaults (and Molecule never runs it).
- **Rows and aliases (2026-10-25):** see "Decisions (2026-10-25)".

### Flatpak permissions (already in AFPI/AAPI)

On 2026-10-05 AFPI and AAPI gained `flatpak_filesystem_overrides` (`group_vars/all/all.yml`) and the
task `Software | Grant extra folders to flatpak applications` (`roles/apps`). It is an
app → folders map, applied with `flatpak override --user --filesystem=…`:

```yaml
flatpak_filesystem_overrides:
  com.rtosta.zapzap:
    - "{{ user_home }}/wks:ro"   # drag files from ~/wks into ZapZap
```

The reason: the Flatpak sandbox only sees granted folders. A file dragged from anywhere else
reaches the app as a path it cannot open (ZapZap says the file "has no content"). The file picker
goes through the portal and needs no grant.

In ALPI:
- The folders are the user's (`~/wks` is personal), so the map goes into `custom.yml`. The
  repository ships the map empty, with ZapZap in `custom.yml.example`.
  (2026-10-15, changed) ALPI creates `~/wks` itself (`workspace_dir`), so ZapZap's read-only grant
  to it is the repository's again, as in AAPI: `flatpak_overrides_common` in
  `group_vars/all/flatpak.yml`, the first layer, for every desktop. `custom.yml`'s
  `flatpak_filesystem_overrides` stays empty for the user's own folders.
- The fixed Bitwarden and Zoom overrides (Wayland, cedilla) can become the same mechanism,
  generalized to `flatpak_overrides: {app: [flags]}` and applied only when the app is in the
  resolved set.
- (2026-10-08, done) Layers, each with its own key: `flatpak_overrides_kde` in
  `group_vars/all/flatpak.yml` (Bitwarden, Zoom; KDE sessions only, as in AFPI), then the user's
  `flatpak_overrides` (flags) and `flatpak_filesystem_overrides` (folders, AFPI's key) in
  `custom.yml`. Flags add up per app; a flag not starting with `--` stops the run in `resolve`.
  Only apps in the resolved Flatpak sets get overrides, so skipping an app skips them. The task
  reports a change only when the override file differs (AFPI always reported none). Removing an
  entry does not undo it: `flatpak override --user --reset <app>`.

### Traps in the current code to fix during migration

- The VirtualBox gate is a string match: `'VirtualBox' in dnf_packages_common`
  (`afpi/roles/apps/tasks/main.yml:25`).
- `clamav-freshclam` is enabled unconditionally: if ClamAV is skipped, the run breaks.
- The Bitwarden and Zoom Flatpak overrides run even when those apps are not installed.

Each of these becomes a feature or is guarded by the resolved package set.

## Tests

- **Golden test:** a play that only prints the resolved package, flatpak and feature sets for each
  (distro, DE, selection). The output is compared with what AFPI and AAPI install today. This is what
  proves parity.
- **Podman containers** (`fedora:44`, `almalinux:10`, …) per role, run twice to check idempotency.
  The full `site.yml` does not run in a container (no systemd/grub).
- **VM** for Secure Boot/MOK and the full desktop.
- Never run on the host, not even with `--check`.

## Migration

1. **New repository `phguima/alpi`** (decided on 2026-10-05, instead of renaming AFPI). Code is
   ported from AFPI and AAPI step by step. The paths `host_vars/127.0.0.1.yml` (migrated to the
   directory) and `group_vars/all/secrets.yml` stay compatible.
2. **Restructure** around the three axes, the catalog, the `repos` role and `group_by`. Personal
   aliases go to `host_vars`.
3. **Fold in the AAPI deltas** as `os_AlmaLinux_10` (the work machine's choices go in its
   `selection.yml`). Carry both legacy
   `.zshrc` block cleanups (`NVIDIA AND API CONFIGURATION` and `API CONFIGURATION`).
4. **Prove parity** with the golden test + containers + VM.
5. **Freeze AAPI** with a README pointing to ALPI. The in-progress AFPI work (`noir` reinstall,
   step 8) finishes first, or is ported.
6. **Debian 13 / Ubuntu 26.04 LTS**: only once a test VM exists. Each one is a port of its own:
   `/var/run/reboot-required`, `update-grub`, Flatpak missing on Ubuntu, Firefox as a snap,
   extrepo/PPA, DKMS.

## Decisions (2026-10-05)

1. **Repository:** new empty `phguima/alpi`, public, GPL-3.0 (not a rename of AFPI). AFPI and AAPI
   stay active until parity.
2. **Uninstall:** yes, only through an explicit `packages_absent` list, without autoremove.
3. **Unknown catalog id:** fail early, in `env_setup`, listing every unknown id.
4. **Profile:** policy + package/feature sets; hardware stays detected and can only be vetoed.
   *Superseded on 2026-10-08: no profiles, see below.*
5. **VirtualBox:** fixed source per distro: RPM Fusion akmod + kmodgenca on Fedora, Oracle repo on
   EL. No user choice.
6. **Debian/Ubuntu:** after Fedora + AlmaLinux parity, and only with a test VM. Reserve the slots
   (support matrix, catalog columns) now; write no tasks for them yet.
7. **Desktops:** GNOME and KDE from the start (parity requires both).

## Decisions (2026-10-08)

1. **Profiles removed.** Deciding a machine's packages by a name chosen in advance (`personal`,
   `work`) does not fit: the choice belongs to the machine. They carried little (`work`: no
   hostname, no Steam). Replaced by:
   - the hostname: managed only when `bootstrap.sh` is given one (empty leaves it alone);
   - packages and features: `pick.py`, listing only what the running distro offers, saved as
     differences from the defaults in `host_vars/127.0.0.1/selection.yml`.
2. **Hardware vetoes stay in `custom.yml`**, not in the picker (no second detection).
3. A `bootstrap.yml` written before this keeps working: `alpi_profile` is ignored by the playbook
   and dropped by the next `bootstrap.sh`.

## Decisions (2026-10-14)

1. **Swap on zram in its own role, `roles/memory`.** Neither AFPI nor AAPI handled it: Fedora
   gets zram0 from `zram-generator-defaults` (min(RAM, 8 GiB)), AlmaLinux 10 does not install
   it, and the work machine ran with no swap at all. zram is a block device in RAM, independent
   of the filesystem (btrfs only matters for a disk swap file). The role installs
   `zram-generator-defaults` on both, so both behave like Fedora; `zram_size` (a zram-generator
   expression) goes into a drop-in only when set.
2. **Read the machine first, and leave other setups alone.** `zram: auto` does nothing when zram
   comes from another tool, when `/etc/systemd/zram-generator.conf` is the user's own, or when
   zswap is on (zram plus zswap compresses every page twice); it says why. `true`/`false` stop
   the run there instead of silently not doing what they say.
3. **Disk swap is reported, never created or removed.** With disk swap present, zram (priority
   100) is used first. A swap file is only needed to hibernate, and on btrfs it needs its own
   subvolume; so `hibernate: true` gives a reminder when disk swap is smaller than RAM.
4. **Off without uninstalling** (uninstalls stay explicit, `packages_absent`): an empty
   `/etc/systemd/zram-generator.conf`, zram-generator's own documented switch, with ALPI's marker
   so switching back on removes it.
5. `vm.swappiness=180` and `vm.page-cluster=0` while zram is on (Fedora's and Pop!_OS's values for
   swap on zram); the kernel defaults come back when it goes off.

## Decisions (2026-10-13)

1. **Cedilla on XWayland, gated on the KWin version, not the distro** (ported from AAPI's
   `fix(desktop): cedilla on KDE via XWayland for Chromium/Electron apps`). Under KWin 6.6,
   Chromium/Electron apps on native Wayland ignore `~/.XCompose` and type "ć". Checked that day:
   EL10's EPEL ships KWin 6.6.4, the very version Fedora 44 was released with (its updates have
   6.7.5, where the bug is gone), so the distro does not tell. `cedilla_xwayland: auto` applies
   the workaround on KDE when KWin is older than `cedilla_xwayland_kwin_below` (6.7); the cutoff
   rests on those two data points, so `custom.yml` can force it (`true`/`false`).
2. **The workaround undoes itself.** A gate only helps if a KWin upgrade removes what it set:
   ALPI's menu-entry copies carry a marker (AAPI's are recognized as the system entry plus the
   flag) and are removed; the flag leaves a user's own copy, the Antigravity entry and the
   aliases; the cedilla apps' Flatpak override files that still grant `~/.XCompose` are reset and
   the remaining layers applied again (an exception to "removing an entry does not undo it").
3. **Decided again after the upgrade, without a reboot.** KWin is read in `resolve.yml` and
   again at the end of `roles/update` (`tasks/cedilla.yml`), so a run that upgrades KWin
   configures for the KWin the next login runs, instead of waiting for the next run. Reading
   before the upgrade and predicting it (`dnf repoquery --upgrades`) was rejected: a second
   query that must agree with what the upgrade really does. A new KWin needs a new login, not a
   reboot, so the reboot gate is unchanged; the run asks for a new login when the decision
   flips. `--tags resolve` shows the decision before the upgrade.

## Decisions (2026-10-24)

Debian 13 and Ubuntu 26.04 LTS (`TODO.md` section 5; drafted in `PLAN-DEBIAN-UBUNTU.md`, reviewed
by an adversarial agent, removed once these were recorded). Decision 6 of 2026-10-05 still holds:
a distro is supported only after it passes on a test VM. Targets: Debian 13 "trixie" (kernel
6.12, GNOME 48 with gnome-terminal / Plasma 6.3, ansible-core 2.19) and Ubuntu 26.04 "Resolute
Raccoon" (kernel 7.0, GNOME with Ptyxis / Kubuntu with Plasma 6.6, ansible-core 2.20). Order: the
shared prerequisites and the apt layer, then Debian 13 (the plainer base), then Ubuntu as a delta.
Checked in `debian:13` and `ubuntu:26.04` containers on 2026-10-24: NVIDIA, VirtualBox, codec and
`uv` packages below; Ubuntu's `kwin-wayland` is `4:6.6.6-0ubuntu0.1` (an epoch); Debian ships
`libheif` 1.23.6 with the `libde265` (HEVC) plugin and VLC 3.0.24 in main.

1. **Family groups.** `env_setup.yml` adds `family_<os_family>` (`family_RedHat`,
   `family_Debian`) to the `group_by` loop, and `pick.py` loads it, for data both apt distros
   share (`os_Debian` is Debian's own group).
2. **Catalog keys `rpm` and `deb`.** Many `all:` entries are RPM names (`ImageMagick`,
   `lm_sensors`, `gstreamer1-*`…); `all` keeps only names that match everywhere. Key order:
   Fedora `fedora, rpm, all`; AlmaLinux 10 `el10, el, rpm, all`; Debian 13
   `debian13, debian, deb, all`; Ubuntu 26.04 `ubuntu2604, ubuntu, deb, all`. The example's `rpm:`
   raw-name prefix in "User file" was a slip for `pkg:`, the prefix the code reads (fixed).
3. **VirtualBox from Oracle's apt repository on both** (`download.virtualbox.org/virtualbox/debian`,
   suites `trixie` and `resolute`, `virtualbox-7.2`), signed by `vboxdrv.sh` with the key in
   `/var/lib/shim-signed/mok`: the code path EL already uses. Debian 13 has no VirtualBox package;
   Ubuntu's multiverse has 7.2.6 against Oracle's 7.2.22, and would be a second signing path.
4. **NVIDIA.** Debian: `nvidia-driver` (550, non-free) built by DKMS and signed with DKMS's key,
   enrolled through `roles/akmods_mok/tasks/enroll.yml`. Ubuntu: a **pinned branch with open
   modules**, `nvidia-driver-NNN-open` with Canonical's signed `linux-modules-nvidia-NNN-open-*`,
   NNN set in `group_vars/os_Ubuntu` and bumped on purpose; the `catalog` scenario checks that
   the signed modules exist. Not `ubuntu-drivers install`, which bypasses the catalog (name
   check, `verify_installed`, `packages_absent`) and cannot be tested in a container. Open modules
   need a Turing or newer GPU; the newest branch on 2026-10-24 (615) already had signed modules
   only in its `-open` flavour (535–610 had both). Both distros: i386 multiarch when
   `nvidia-driver` or `steam` is selected.
5. **Snaps (Ubuntu) left alone:** snapd and its snaps (Firefox) stay; no catalog id maps to a
   snap transitional package (`chromium` is `1snap1`); Flatpak instead.
6. **Ubuntu codecs without the Microsoft fonts:** `ubuntu-restricted-addons` + `libavcodec-extra`
   (`unrar` is already in the catalog), not `ubuntu-restricted-extras`, which recommends
   `ttf-mscorefonts-installer`, whose EULA fails a non-interactive run; nothing is accepted on the
   user's behalf.
7. **`uv` through pipx on deb** (`{pipx: uv}`): packaged on neither distro; the pipx role installs
   and upgrades it, and its PyPI name check already exists.
8. **MOK passwords** `debian-alpi` and `ubuntu-alpi`, like `fedora-alpi` and `alma-alpi`.

## Decisions (2026-10-25)

1. **Steam's own launcher.** Steam's `steam.desktop` sets `PrefersNonDefaultGPU` and
   `X-KDE-RunOnDiscreteGpu`, and the desktop applies switcheroo-control's offload variables:
   checked on noir (Plasma 6.7.5, Wayland), every Steam process had them and `nvidia-smi` listed
   Steam. ALPI's user copy with the PRIME variables (from AFPI) is dropped and removed where it
   exists; `nvidia_prime_env` remains for the `nvidia-run` alias.
2. **Picker rows.** whiptail sizes a checklist to its longest row, so short descriptions left
   most of the box empty. Each row now shows the description and then what it installs (package
   names, `flatpak <app id>`, `pipx <spec>`, `upstream <source>`; the old "(Flathub)" suffix is
   gone), padded to the box, which may be up to 160 columns wide.
3. **zsh aliases are a choice.** Not everyone wants every alias, and some run programs that may
   not be installed (`agycli` without the Antigravity CLI). Every alias in the repository is a
   named definition (`zsh_aliases`, the family's `zsh_aliases_family`, the distro's `zsh_aliases_os`) with a description, `root: true`
   for root's `.zshrc` too, and `requires:` (a catalog id or feature). The picker lists them as
   its last group, showing the command, with a "needs …" note. An alias is hidden while what it
   needs (zsh, and its `requires`) is listed and unchecked, its own check kept for when it comes
   back (first listed regardless, which left `claudecli` checked with Claude Code unchecked:
   confusing in a real test); an id the picker does not list (`nvidia-driver`, from detection)
   hides nothing. The playbook leaves out an alias whose requirement is not installed either way. Choices: `zsh_aliases_selection`
   (picker) then `zsh_aliases_enabled` (`custom.yml`), `{name: bool}`, unknown names fail like
   unknown features. Not choices: the cedilla's `code` alias (part of that workaround), and the
   machine's own `luks_volumes`/`zsh_aliases_custom`. Aliases are single-quoted now, so the
   command is expanded when the alias runs.
4. **Ubuntu 26.04 details** (decisions 3–6 of 2026-10-24 applied, checked in `ubuntu:26.04`):
   the NVIDIA branch is pinned in the catalog's `ubuntu2604` value rather than a variable in
   `group_vars/os_Ubuntu` (`pick.py` reads the catalog untemplated): `nvidia-driver-615-open`
   (615 the newest real branch, 610 transitional) with `linux-modules-nvidia-615-open-generic`,
   whose Canonical-signed modules provide `nvidia-dkms-615-open`, so nothing is built. Secure Boot
   key: Ubuntu's own `update-secureboot-policy --new-key` creates `/var/lib/shim-signed/mok/MOK.*`
   before anything that signs (DKMS, Oracle's `vboxdrv.sh`), one key and one enrollment.
   Steam's 32-bit NVIDIA libraries (`libnvidia-gl-NNN:i386`; Debian's `nvidia-driver-libs:i386`)
   are in the catalog, as Fedora's `.i686`. The Antigravity IDE gets an AppArmor profile granting
   `userns` *(check on the VM: it attaches to the binary in the AppImage's mount)*.
5. **Steam's launcher on Debian and Ubuntu.** steam-installer's `steam.desktop` has no
   `PrefersNonDefaultGPU` (checked on both, 2026-10-25), so Steam would start on the integrated
   GPU of a hybrid laptop. With the NVIDIA driver and Steam selected, ALPI writes a user copy of
   it with `PrefersNonDefaultGPU` and `X-KDE-RunOnDiscreteGpu` and no variables, the mechanism
   decision 1 checked on noir, rebuilt on every run and removed when NVIDIA or Steam goes. Not
   checked on real NVIDIA hardware with Debian's or Ubuntu's driver yet (TODO, Phase 5).
6. **open/close-thevoid are repository aliases.** A fresh install on noir had no `custom.yml`, so
   it lost AFPI's `open-thevoid` / `close-thevoid` (decision of 2026-10-08 moved the UUID there).
   noir's UUID was already public in the repository, and the aliases do nothing harmful where no
   partition has it (udisksctl: no such device), so they return as two named definitions in
   `zsh_aliases` (`thevoid_uuid` in `group_vars/all/zsh.yml`): on by default, listed by the
   picker with "(noir's LUKS disk)" in their description, and switched off by name like any
   alias. `custom.yml`'s `luks_volumes` stays for a machine's other volumes. Same day: an empty
   layer of the user's aliases block (the cedilla template when off, which ansible-core 2.19+
   evaluates to None, or an empty `custom.yml` key) was written as a `None` line; null layers
   are now dropped.

## Decisions (2026-10-28)

1. **Gitea and Forgejo CLIs from upstream everywhere.** `tea` (Gitea's CLI, works with Forgejo
   too) and `fj` (`forgejo-cli`) are defaults next to `gh`. Fedora 44 and AlmaLinux 10 (EPEL
   included) package neither; Debian 13 and Ubuntu 26.04 package `tea-cli` (0.9.2 and 0.11.1,
   installed as `/usr/bin/tea-cli`; Debian's `tea` is a text editor) and Ubuntu `forgejo-cli`
   (0.3.0), against 0.16.0 and 0.7.0 upstream (checked 2026-10-28). Old enough to miss commands
   and the same command name nowhere, so both come from their own releases on every distro,
   bending "upstream only where the distro has no package" on purpose.
2. **`release_binary` source type.** One program from the latest release on GitHub, Gitea or
   Forgejo, whose release APIs share GitHub's shape (`alpi_github_release` parses all three).
   Gitea and Forgejo publish no digests: `checksum_asset` names the release's sha256 file, which
   get_url reads (`sha256:<url>`; tea publishes one per `.xz`); fj publishes none, so HTTPS is its
   only check. Installed by root in `/usr/local/bin` (fj's binary needs glibc 2.39, EL10's
   exactly), the tag in `/var/lib/alpi/upstream/<id>.version`; replaced when the tag differs or
   the program is gone; forge unreachable keeps what is installed, as for the fonts. x86_64
   assets only, like every target machine.

## Decisions (2026-11-07)

1. **OpenLogi, a default.** An open-source alternative to Logi Options+ (buttons, DPI,
   SmartShift over HID++; `AprilNEA/OpenLogi`), selected by default on every distro and listed
   under "System tools" in the picker. No distro, COPR or PPA packages it (checked 2026-11-07,
   0.9.0); the vendor publishes signed (minisign) `.rpm` and `.deb` files on each GitHub release,
   with GitHub digests. Preferring the distro's package format over the vendor's `install.sh`,
   Flatpak bundle or a source build keeps it under dnf/apt: removal, file ownership and the
   package's scripts (udev reload) work as for any package.
2. **`release_package` source type.** The asset for the family's `upstream_package_format`
   (`rpm` or `deb`, templated into the asset regex), checked against GitHub's digest, installed
   with the dnf module (`disable_gpg_check`: the RPM is unsigned; checked with EL10's
   ansible-core 2.16) or apt's `deb:`. No version file: the installed package's version
   (`upstream_package_query`) is compared with the tag (`alpi_upstream_package_current`), so a
   package removed by hand comes back and nothing is downloaded while it is current. The
   minisign signatures are not checked (that would need minisign and the vendor's key on every
   machine); HTTPS and the digest are the check, as for the other sources. The RPM declares no dependencies; the `.deb`
   needs `libxkbcommon0`, `libxkbcommon-x11-0` and `libxcb1`, which any desktop has.
3. **The agent is left to the user.** The packages ship `openlogi-agent.service` as a systemd
   user unit, not enabled; the app's "Launch at login" switch enables the same unit. ALPI does
   not enable it (it would undo the user's switch on every run): `tasks/reminders.yml` reminds
   when the user's `graphical-session.target.wants` link is missing.

## Decisions (2026-11-16)

1. **NVIDIA on AlmaLinux 10 from AlmaLinux's own repository.** `almalinux-nvidia` ("AlmaLinux
   10 - NVIDIA Driver & CUDA"), enabled by `almalinux-release-nvidia-driver` from extras (a
   `package` repository in `roles/repos`), ships the driver (620.70.06 on 2026-11-16) with
   prebuilt open modules signed by AlmaLinux: `nvidia-open-kmod` (→ `kmod-nvidia-open`),
   `nvidia-driver`, `nvidia-driver-cuda` (`nvidia-smi`). Nothing is built (`nvidia-build` is
   `[]` on EL, `kmod_signer` stays `none`); the Vulkan ICD is in `nvidia-driver-libs`, which the
   driver pulls in (`nvidia-vulkan` `[]`); VA-API is EPEL's `libva-nvidia-driver`. The
   repository is x86_64 only: no 32-bit libraries for Steam. RPM Fusion's EL repositories have
   no NVIDIA packages, so nothing conflicts.
2. **ALPI's options in their own file on EL.** `nvidia-kmod-common` ships
   `/usr/lib/modprobe.d/nvidia.conf` (the nouveau blacklist and its power options); Fedora's
   `/etc/modprobe.d/nvidia.conf` (AFPI's name) would mask it, so EL's `nvidia_modprobe_conf` is
   `/etc/modprobe.d/alpi-nvidia.conf`, as on apt. The package's `nvidia-boot-update` edits
   `GRUB_CMDLINE_LINUX` (`rd.driver.blacklist=nouveau`), which `roles/boot` does not manage.
3. **No build step for prebuilt modules.** `roles/nvidia` reports a missing module for the
   running kernel where `kmod_signer` is `none` (a kernel newer than AlmaLinux's
   `kmod-nvidia-open`, which reaches same-ABI kernels through weak-updates) instead of building.
4. **GPUs older than Turing are skipped where only open modules exist.** Open kernel modules
   drive Turing and newer; the NVIDIA device id from `lspci -nn` tells the generation (everything
   below `0x1e00` is Maxwell, Pascal or Volta: `alpi_nvidia_pre_turing`, true only when every
   NVIDIA GPU is older). Where `nvidia_open_modules_only` is set (AlmaLinux 10), such a GPU counts
   as not detected: the nvidia feature is off with a message and nouveau stays. Fedora and Debian
   (proprietary modules) are unaffected; Ubuntu's pinned open branch has the same limit and can
   set the flag once its fallback is decided (TODO section 9). `is_nvidia_pre_turing` is pinned
   in Molecule with the other hardware flags.

## Decisions (2026-11-17)

AlmaLinux's NVIDIA driver on the reporting notebook (Intel iGPU + MX450, KDE Plasma 6 on
Wayland): SDDM worked, then Plasma went black right after login. Fix found and checked there by
hand, then brought into ALPI. Everything below is set in `group_vars/os_AlmaLinux_10`
(defaults off in `group_vars/all/nvidia.yml`): Fedora, Debian and Ubuntu are untouched.

1. **KWin on the iGPU of a hybrid laptop.** With `nvidia-drm` loaded, KWin took the NVIDIA card
   as its primary DRM device; on an Optimus laptop the built-in panel hangs off the iGPU, so
   that card has no screen. A udev rule (`/etc/udev/rules.d/61-igpu-card.rules`, `DRIVERS=="i915|xe"`)
   names the iGPU's card `/dev/dri/igpu-card` and `/etc/environment` sets
   `KWIN_DRM_DEVICES=/dev/dri/igpu-card` (other `KWIN_DRM_DEVICES` lines removed). Not `cardN`
   (the numbering changes between boots) and not `/dev/dri/by-path` (its names hold `:`, the
   variable's separator). When the link does not appear after `udevadm trigger`, the run stops
   and the variable is removed: a missing device would leave KWin with no GPU. "Hybrid" is
   `is_intel_panel` (env_setup): an i915/xe card with an eDP connector, so a desktop whose screens
   hang off the NVIDIA card while the iGPU is enabled is not mistaken for a laptop. Applied
   whenever such a GPU pair is there, driver selected or not (a vetoed driver leaves its packages
   installed); removed when it is not. A desktop is left alone: its monitor usually hangs off the
   NVIDIA card, and KWin on the iGPU would leave Plasma on a GPU with no screen; one whose monitor
   is on the motherboard's port sets `nvidia_kwin_igpu_force` (custom.yml) instead of a detection
   that would read the cabling at run time. External ports wired to the NVIDIA GPU stay dark with
   the iGPU alone: `nvidia_kwin_dgpu` (off until tested on hardware) adds a second rule
   (`61-nvidia-card.rules`, `DRIVERS=="nvidia"`, `/dev/dri/nvidia-card`) and lists that card
   second (`KWIN_DRM_DEVICES=/dev/dri/igpu-card:/dev/dri/nvidia-card`; the first stays the
   primary), only once its link exists (nvidia-drm loaded), so the run after the reboot adds it.
2. **`prime-run`** (`/usr/local/bin`, root 0755; Arch's `nvidia-prime`, not packaged on EL) next
   to the `nvidia-run` alias, while the driver is selected. Its EGL vendor file is the one
   AlmaLinux's `nvidia-driver-libs` installs (`el10-work` checks the path); that EGL apps then
   render on NVIDIA is not validated yet: if an app breaks under `prime-run`, removing the two
   `egl_json` lines is the first thing to try. **`alpi-nvidia-check`** (`/usr/local/bin`; catalog
   `nvidia-check-tools`: glx-utils, egl-utils, vulkan-tools) runs the acceptance checks after the
   reboot, as the user inside Plasma, one PASS/FAIL line each: `nvidia-smi`, no nouveau, the iGPU
   link and its driver, `KWIN_DRM_DEVICES` in `kwin_wayland`'s environment, and `prime-run` with
   glxinfo, eglinfo (Wayland platform) and vulkaninfo (NVIDIA devices only). **switcheroo-control** (catalog `nvidia-switcheroo`, EL only: Fedora's desktops pull
   it in) is enabled for Plasma's "Run using dedicated graphics card".
3. **Kernels and prebuilt modules.** AlmaLinux's modules may come after a new kernel.
   `/usr/local/bin/nvidia-kmod-check` says whether the newest installed `kernel-core` (the
   package `roles/update`'s gate reads) has a module; `roles/nvidia` runs it, the reboot gate
   adds its warning, and EL's `full-update` alias runs it before the reboot advice. `roles/boot`
   keeps older kernels while the driver is selected (`kernel_keep_old`); dnf's
   `installonly_limit`, never lowered by ALPI, prunes them. Before `roles/update`'s upgrade, the
   newest kernel in the repositories is compared with the newest `kmod-nvidia-open`: when the
   kernel lacks `kernel(...)` symbols the module requires (what `weak-modules` checks; the
   package is kABI-tracking, so one build serves later kernels of the same ABI), the upgrade
   excludes the kernel packages and says so, and a later run takes the kernel once AlmaLinux
   publishes its module. The reboot gate therefore never meets a kernel without a module from
   ALPI's own upgrade. Nothing is locked: a manual `dnf upgrade` still brings the kernel (the
   `full-update` alias then warns). Optional `nvidia_kernel_versionlock`
   (off) holds the installed kernel packages with `dnf versionlock`; turning it off does not
   release them (`dnf versionlock delete kernel kernel-core kernel-modules kernel-modules-core`).
4. **Never mix driver sources.** With the driver selected, `resolve.yml` stops the run when
   packages from RPM Fusion (`akmod-nvidia*`, `xorg-x11-drv-nvidia*`) or NVIDIA's CUDA repository
   (`kmod-nvidia-latest-dkms`, `kmod-nvidia-open-dkms`, `nvidia-kmod*` except AlmaLinux's own
   `nvidia-kmod-common`) are installed, and names them (`alpi_nvidia_foreign`). ALPI removes
   nothing, and never through a `*nvidia*` glob, which takes `nvidia-gpu-firmware` (needed by
   `linux-firmware` and the kernel).
5. **Undo, by hand** (ALPI undoes items 1 and 2 when they stop applying, but never removes the
   driver):
   ```sh
   sudo sed -i '/^KWIN_DRM_DEVICES=/d' /etc/environment
   sudo rm -f /etc/udev/rules.d/61-igpu-card.rules /etc/udev/rules.d/61-nvidia-card.rules
   sudo rm -f /usr/local/bin/prime-run /usr/local/bin/nvidia-kmod-check /usr/local/bin/alpi-nvidia-check
   sudo systemctl disable --now switcheroo-control
   # the driver: dnf history undo <ID> of its install transaction, never a glob; then reboot
   ```

## Decisions (2026-11-24)

Fixed DNS servers (GitHub issue #3), the optional `dns` feature (off by default):

1. **Only on chosen connections.** The servers (`dns_ipv4`, `dns_ipv6` in
   `group_vars/all/dns.yml`: Cloudflare, then Google) go, with `ignore-auto-dns yes`, on the
   NetworkManager connections listed for it, never on whatever is active when ALPI runs: a run at
   work or in a hotel changes nothing there (internal DNS, captive portals). Dropped: a
   NetworkManager dispatcher for every connection, and systemd-resolved split DNS.
2. **The list lives in `selection.yml`.** When the feature is checked, `pick.py` offers the
   connection in use (the active one on the default route's device, read from `/proc/net`; VPNs,
   WireGuard, tunnels, bridges and loopback never, `dns_skip_types`) and saves it as
   `dns_connections_selection` (`{uuid: name}`). The picker writes only `selection.yml`, never the
   user's `custom.yml`; `dns_connections` there (UUIDs, or `{uuid: name}`) replaces the list, as
   `custom.yml` wins elsewhere. The list stays when the feature is turned off.
3. **`roles/dns`** (after `common`): reads each listed connection with `nmcli -g` and changes only
   what differs, then `nmcli device reapply` where it is active (a device that refuses it gets the
   servers at its next connect, with a message). An IP family whose method is `disabled` or
   `ignore` is left alone: NetworkManager refuses `ipv4.dns`/`ipv6.dns` there (checked with 1.52 on
   Debian 13 and 1.58 on AlmaLinux 10). A UUID with no connection only warns; NetworkManager not
   running skips the role. Values from `nmcli -g` come with `:` escaped (`2606\:4700\:…`), and
   addresses are compared in canonical form.
4. **Off undoes what is ALPI's.** With the feature off, a listed connection that still has exactly
   ALPI's servers and `ignore-auto-dns yes` gets its own DNS back (servers cleared,
   `ignore-auto-dns no`); servers the user changed since stay, with a message.
5. **`alpi-dns`** (`~/.local/bin`, the user's, rendered from the same variables; removed when the
   feature is off): `alpi-dns [<connection>]`, `alpi-dns --off [<connection>]`, the connection in
   use when none is named. ALPI never runs it. It does not edit ALPI's files: to have ALPI's runs
   keep a connection, run `./pick.py` while connected to it.
