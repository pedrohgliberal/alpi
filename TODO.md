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

## 7. Ubuntu 24.04 LTS (after Phase 5's Ubuntu 26.04 VM checklist)

A second Ubuntu LTS next to 26.04 (added 2026-11-05), in the support matrix and the container
tests since 2026-11-05 (`TODO-HISTORY.md`); what containers cannot check is left. Checked in
`ubuntu:24.04` (24.04.5) on 2026-11-05:
- `VERSION_ID` and Ansible's `distribution_version` are `24.04` on every point release (24.04.5
  included); release `noble`.
- ansible-core 2.16.3 (as EL10: community.general 11.x), Python 3.12, classic `sudo` (sudo-rs
  0.2.2 is only an option), systemd-zram-generator 1.1.2, pipx 1.4.3 (in every pocket), dkms
  3.0.11, Node.js 18; no Flatpak in the image (roles/packages installs it).
- Kubuntu: Plasma 5.27 (`kwin-wayland` 5.27.11, Konsole 23.08), whose default session is X11.
- GNOME 46 with gnome-terminal, no Ptyxis.
- Kernel: GA 6.8 (`linux-headers-generic`); installs from later point-release ISOs may run the
  HWE kernel *(check)*.

### Phase 6 — what containers cannot check

- [ ] **HWE kernel:** check on a 24.04.5 install which kernel it runs; if HWE, the headers and
      Canonical's NVIDIA modules must follow the running kernel's flavour
      (`linux-headers-generic-hwe-24.04`, `linux-modules-nvidia-NNN-open-generic-hwe-24.04`).
      `alpi-kvm.conf` is needed there below 6.16, as on Debian (set on every distro already).
- [ ] **Plasma 5.27:** decide the cedilla workaround for an X11 session (XWayland flags do
      nothing there) and for Plasma 5's Wayland; check the Konsole profile and `konsolerc` on
      Konsole 23.08.
- [ ] **VM checklist:** `tests/vm-checklist-apt.md` on Ubuntu 24.04.5 and Kubuntu 24.04.5 (EFI +
      Secure Boot), then a release.

## 8. Fedora 45

In the support matrix and the container tests since 2026-11-07 (`TODO-HISTORY.md`). Checked in
`fedora:45` on 2026-11-07: dnf5 5.4.7, KWin 6.8.2 (the cedilla workaround stays off), RPM Fusion
and the ASUS COPR publish 45.

- [ ] **VM checklist:** `tests/vm-checklist.md`'s Fedora KDE VM on Fedora 45 (EFI + Secure Boot):
      akmods key and signed `akmod-VirtualBox`/NVIDIA builds on 45's kernel, `dracut`, the reboot
      gate, Konsole and the cedilla in a Plasma 6.8 session; then a release.

## Maintenance

- [ ] RPM Fusion's `libheif-freeworld` 1.23.6 needs a `libheif` that Fedora and EPEL no longer
      publish (2026-11-05): `catalog`, `fedora44-personal` and `el10-work` fail at the name check
      until it is rebuilt. Rerun them then (`tests/run.sh catalog fedora44-personal el10-work`).
      Full run of 2026-11-07: EL10 still fails (`el10-work`, `alpi-catalog-el10`). Fedora 44
      passes the name check but `fedora44-personal` fails at the install: its updates ship
      `libheif-ffmpeg` 1.23.7, which obsoletes `libheif`, while RPM Fusion's stable
      `libheif-freeworld` 1.21.2-3 requires `libheif` 1.21.2 (depsolve error). Fedora 45 passes
      (`catalog`, `fedora45-personal`). Since 2026-11-07 (v1.5.0) the catalog marks it
      unavailable on Fedora 44 and EL (`{fedora44: ~, el: ~}`: skipped with a warning; Fedora
      45 keeps it). Once RPM Fusion rebuilds it: drop both keys, move it back to the native sets
      in `catalog`'s Fedora 44 and EL10 golden values and the `failures` strict case, and point
      `test_defaults_match_the_playbook_lists` back at Fedora 44 if wanted.

- [ ] After upgrading Molecule (and with it the venv's `ansible-core`, which `tests/run.sh`
      puts first in `PATH`) or Fedora's `ansible` package, rerun `tests/run.sh full` and recheck
      the expected log noise listed in `CLAUDE.md` (Molecule section): the `molecule/default`
      CRITICAL must still be non-fatal, the `containers.podman` in `~/.ansible/collections`
      must still be newer than the system copy, and the logs must have no `DEPRECATION`. Versions
      in the last green run, 2026-10-24 (AlmaLinux 10 work machine): Molecule 26.10.0 (host pipx),
      ansible-core 2.21.5 (Molecule's venv), `containers.podman` 1.21.1 and `community.general`
      11.4.9 (user; no system copy there). On 2026-10-06 (Fedora): ansible-core 2.20.7 (system),
      `containers.podman` 1.21.0 (user) vs 1.20.2 (system).
