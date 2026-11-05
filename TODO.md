# TODO — ALPI

Work plan: open items only. A done item moves to `TODO-HISTORY.md` (date, short summary, commit)
in the commit that finishes it; a section goes once it is empty. Architecture, rationale and
decisions in `PROPOSAL.md`. Every item ships with its Molecule scenarios / unit tests (see
`CLAUDE.md`).

## 5. New distros (after parity, only with a test VM)

Debian 13, then Ubuntu 26.04 LTS as a delta. Decisions, targets and checked facts in `PROPOSAL.md`
("Decisions (2026-10-24)"). Facts marked *(check)* are not confirmed yet: check them in a
container or VM before code relies on them. Consequences to keep in mind:
- The cedilla XWayland workaround turns on for both KDEs (KWin < 6.7).
- Debian's 6.12 kernel needs `/etc/modprobe.d/alpi-kvm.conf` (VirtualBox 7.2 shares VT-x with
  KVM only on 6.16+); on Ubuntu's 7.0 it is insurance, as on Fedora.
- Ubuntu's major version (`26`) is the same for 26.04 and 26.10: its support entry compares the
  full `distribution_version`.
- More controller lines: Debian's ansible-core 2.19 and Ubuntu's 2.20 (Data Tagging) next to
  EL10's 2.16 and Molecule's 2.21; `debian13-personal` and `ubuntu2604-personal` resolve with the
  container's own ansible-core, as `el10-work` does (`shared/verify_native_controller.yml`).

### Phase 5 — tests and release

- [ ] **VM checklist:** run `tests/vm-checklist-apt.md` on Debian 13 GNOME and KDE, Ubuntu 26.04
      and Kubuntu 26.04 (EFI + Secure Boot) and send the results: what containers cannot check
      (MOK enrollment and signed modules, `update-grub` and the menu, the reboot gate, the
      unattended-upgrades lock, the installer's own `sources.list`, cedilla and terminals in a
      session, the Antigravity IDE under AppArmor, `bootstrap.sh` without sudo). Fix what fails.
      Passed: Kubuntu 26.04 (2026-11-05). Left: Debian 13 GNOME and KDE, Ubuntu 26.04.
- [ ] **Steam on a hybrid NVIDIA laptop (Debian, Ubuntu):** ALPI's launcher copy with
      `PrefersNonDefaultGPU` (steam-installer's has none) starts Steam on the NVIDIA GPU: check on
      real hardware (`nvidia-smi` lists Steam) with Debian's 550 and Ubuntu's 615 drivers; drop the
      copy if switcheroo's offload does not work there. Steps: `tests/noir-checklist-apt.md`
      (external-drive installs on noir, MOK baseline and cleanup).
- [ ] **Release:** after `tests/run.sh full` and the VM checklist per distro, the next minor
      release says Debian 13 (and Ubuntu 26.04, or a release of its own) is VM-checked; `v1.2.0`
      and `v1.3.0` shipped them as a preview (decided 2026-10-25).

## 6. Ubuntu 24.04 LTS (after Phase 5's Ubuntu 26.04 VM checklist)

A second Ubuntu LTS next to 26.04 (added 2026-11-05). Estimate: 1–2 days of code and container
tests, plus Ubuntu 24.04 and Kubuntu 24.04 VMs. Checked in `ubuntu:24.04` (24.04.5) on
2026-11-05:
- `VERSION_ID` and Ansible's `distribution_version` are `24.04` on every point release (24.04.5
  included), so a `"24.04"` support entry covers them all; release `noble`.
- ansible-core 2.16.3 (as EL10: community.general 11.x), Python 3.12, classic `sudo` (sudo-rs
  0.2.2 is only an option), systemd-zram-generator 1.1.2, pipx 1.4.3, dkms 3.0.11, Node.js 18.
- Kubuntu: Plasma 5.27 (`kwin-wayland` 5.27.11, Konsole 23.08), whose default session is X11.
- GNOME 46 with gnome-terminal, no Ptyxis.
- Not packaged: `fastfetch`, `flatseal`. `nvidia-driver-615-open` exists (590–615).
- Kernel: GA 6.8 (`linux-headers-generic`); installs from later point-release ISOs may run the
  HWE kernel *(check)*.

### Phase 6 — port and tests

- [ ] **Support matrix:** let a distro entry list exact versions (`Ubuntu: {version: ["24.04",
      "26.04"]}`) in `alpi_support` and `pick.py`, with unit tests; `unsupported` keeps refusing
      the interim releases.
- [ ] **Per-version Ubuntu settings:** `alpi_catalog_keys` (`ubuntu2404, ubuntu, deb, all`) and
      Oracle's VirtualBox suite (`noble`) per version, out of `group_vars/os_Ubuntu/main.yml`
      (e.g. `group_vars/os_Ubuntu_24`/`_26`, from `env_setup`'s `group_by`).
- [ ] **Catalog:** `ubuntu2404` values: Flatseal as a Flatpak, fastfetch from upstream or `~`,
      the NVIDIA pin; check every name with apt (all four components, Oracle's `noble`).
- [ ] **HWE kernel:** check on a 24.04.5 install which kernel it runs; if HWE, the headers and
      Canonical's NVIDIA modules must follow the running kernel's flavour
      (`linux-headers-generic-hwe-24.04`, `linux-modules-nvidia-NNN-open-generic-hwe-24.04`).
      `alpi-kvm.conf` is needed there below 6.16, as on Debian.
- [ ] **Plasma 5.27:** decide the cedilla workaround for an X11 session (XWayland flags do
      nothing there) and for Plasma 5's Wayland; check the Konsole profile and `konsolerc` on
      Konsole 23.08.
- [ ] **ansible-core 2.16 with apt:** the apt modules, `deb822_repository` and `site.yml`'s apt
      defaults on Ubuntu's own 2.16 (`shared/verify_native_controller.yml`).
- [ ] **Tests:** scenarios `ubuntu2404` and `ubuntu2404-personal`, plus 24.04 instances in
      `bootstrap`, `secureboot` and `user`; `CLAUDE.md` updated.
- [ ] **VM checklist:** `tests/vm-checklist-apt.md` on Ubuntu 24.04.5 and Kubuntu 24.04.5 (EFI +
      Secure Boot), then a release.

## Maintenance

- [ ] After upgrading Molecule (and with it the venv's `ansible-core`, which `tests/run.sh`
      puts first in `PATH`) or Fedora's `ansible` package, rerun `tests/run.sh full` and recheck
      the expected log noise listed in `CLAUDE.md` (Molecule section): the `molecule/default`
      CRITICAL must still be non-fatal, the `containers.podman` in `~/.ansible/collections`
      must still be newer than the system copy, and the logs must have no `DEPRECATION`. Versions
      in the last green run, 2026-10-24 (AlmaLinux 10 work machine): Molecule 26.10.0 (host pipx),
      ansible-core 2.21.5 (Molecule's venv), `containers.podman` 1.21.1 and `community.general`
      11.4.9 (user; no system copy there). On 2026-10-06 (Fedora): ansible-core 2.20.7 (system),
      `containers.podman` 1.21.0 (user) vs 1.20.2 (system).
