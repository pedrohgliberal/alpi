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
  EL10's 2.16 and Molecule's 2.22; `debian13-personal` and `ubuntu2604-personal` resolve with the
  container's own ansible-core, as `el10-work` does (`shared/verify_native_controller.yml`).

### Phase 5 — tests and release

- [ ] **Steam on a hybrid NVIDIA laptop (Debian, Ubuntu):** ALPI's launcher copy with
      `PrefersNonDefaultGPU` (steam-installer's has none) starts Steam on the NVIDIA GPU: check on
      real hardware (`nvidia-smi` lists Steam) with Debian's 550 and Ubuntu's 615 drivers; drop the
      copy if switcheroo's offload does not work there. Steps: `tests/noir-checklist-apt.md`
      (external-drive installs on noir, MOK baseline and cleanup).

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

## 9. NVIDIA: AlmaLinux 10, and older GPUs on Ubuntu

The catalog leaves NVIDIA out on EL (`nvidia-driver`, `nvidia-build`, `nvidia-vulkan`,
`nvidia-vaapi`: `el: ~`, skipped with a warning), so an EL10 machine with an NVIDIA GPU keeps
nouveau (reported 2026-11-12: an AlmaLinux notebook with a GeForce MX450, Turing). AAPI never
installed it, and RPM Fusion's akmods path is Fedora's. Checked in `almalinux:10` (10.2) on
2026-11-12: AlmaLinux's own repository, enabled by `almalinux-release-nvidia-driver` (extras),
`almalinux-nvidia` ("AlmaLinux 10 - NVIDIA Driver & CUDA"), ships driver 620.70.06
(`nvidia-driver`, `nvidia-driver-cuda`) and prebuilt open modules (`nvidia-open-kmod` →
`kmod-nvidia-open`, signer "AlmaLinux NVIDIA Module CA"; Turing and newer only), with a
posttrans that rebuilds the initramfs for the GSP firmware.

- [ ] **Catalog and repository:** EL values for `nvidia-driver` (`nvidia-open-kmod`,
      `nvidia-driver`, `nvidia-driver-cuda`), and for `nvidia-vulkan`/`nvidia-vaapi` where the
      repository has them (check); `nvidia-build` stays `~` (nothing is built). The repository
      through `almalinux-release-nvidia-driver` (a `repo:` entry in `roles/repos`, as EL's other
      release packages). Golden values in `catalog` (EL10), `el10-work` installs it.
- [ ] **Secure Boot:** EL's `kmod_signer` stays `none` if the AlmaLinux NVIDIA Module CA is
      trusted under Secure Boot without a MOK *(check on real hardware)*; otherwise enroll its
      certificate through `roles/akmods_mok/tasks/enroll.yml`. `secureboot`'s EL instance checks
      the outcome.
- [ ] **roles/nvidia:** the power options (`nvidia_modprobe_conf`) on EL; no rebuild step (the
      modules are prebuilt: check that a missing module for the running kernel is reported, not
      built). Pre-Turing GPUs (no open-module support): decide whether to skip with a message.
- [ ] **Real hardware:** the AlmaLinux notebook (MX450, EFI + Secure Boot): nouveau gone,
      `nvidia-smi` lists the GPU, the module loads with Secure Boot on; then a release.
- [ ] **Chrome crashes with the driver:** reported 2026-11-12 on that notebook, after the manual
      install (`almalinux-release-nvidia-driver`, `nvidia-open-kmod`, `nvidia-driver`,
      `nvidia-driver-cuda`) on top of an ALPI run: the driver worked (`nvidia-smi` ran) and Google
      Chrome, fine before, started to crash (KDE; Chrome installed by hand after ALPI, so on
      native Wayland). ALPI now installs Chrome and, on a KWin older than 6.7 (EL10's), starts it
      on XWayland with the cedilla workaround (`feat(packages): Google Chrome from Google's
      package`): check on the notebook whether that alone stops the crashes. Otherwise find the
      cause there (`chrome://gpu`, `coredumpctl`, the journal) and whether disabling the hardware
      acceleration or keeping Chrome off the NVIDIA GPU (Vulkan/EGL vendor) avoids it; then
      decide what ALPI sets or documents. Check Brave and the Electron apps too.
- [ ] **Removal:** on the same notebook, `dnf remove` of the driver also took packages the
      desktop needed (dependents, and unused dependencies through `clean_requirements_on_remove`)
      and the next boot no longer reached the login; `dnf history undo` failed from a live USB's
      chroot (it wants the old versions). A second notebook of the same model also lost the login
      *(check: the same sequence?)*. Document a removal that leaves the desktop working: remove
      only the NVIDIA packages (`dnf remove --noautoremove`, or `rpm -e` of the exact list), drop
      what the install added to the kernel command line (`grubby`: nouveau blacklist,
      `nvidia-drm` options) and `/etc/modprobe.d`, `dracut -f --regenerate-all`; and the recovery
      (chroot with `/dev`, `/proc`, `/sys`, `/run` and `resolv.conf`; reinstall the removed
      non-NVIDIA packages by name from `dnf history info`; `dnf group upgrade` of the
      environment). Consider an ALPI way back (a feature turned off removes what it added).
- [ ] **Pre-Turing GPUs on Ubuntu (24.04, 26.04):** the pinned `nvidia-driver-615-open` with
      Canonical's `linux-modules-nvidia-615-open-generic` are open modules, which drive only
      Turing and newer (GTX 16xx, RTX 20xx, MX450 and later); a Maxwell, Pascal or Volta GPU
      (GTX 9xx/10xx…) gets a driver that cannot drive it *(check)*. Decide: a proprietary
      (non-open) branch for those GPUs (which branch still supports them, and Canonical's signed
      `linux-modules-nvidia-NNN-generic` for it), or skip with a message. Detection needs the
      GPU's generation (PCI device id, `lspci -nn`), not only "NVIDIA present"; the same applies
      to AlmaLinux's open modules (above). Debian's 550 (proprietary) and Fedora's RPM Fusion
      driver are not affected *(check Fedora's branch support for those GPUs)*.

## Maintenance

- [ ] After upgrading Molecule (and with it the venv's `ansible-core`, which `tests/run.sh`
      puts first in `PATH`) or Fedora's `ansible` package, rerun `tests/run.sh full` and recheck
      the expected log noise listed in `CLAUDE.md` (Molecule section): the `molecule/default`
      CRITICAL must still be non-fatal, the `containers.podman` in `~/.ansible/collections`
      must still be newer than the system copy, and the logs must have no `DEPRECATION`. Versions
      in the last green run, 2026-11-11 (AlmaLinux 10 work machine): Molecule 26.11.0 (host pipx,
      on uv's CPython 3.13.17: ansible-core 2.22 needs 3.13+, EL10 has 3.12), ansible-core 2.22.0
      (Molecule's venv; 2.21 cannot run modules on Python 3.15, the `fedora:45`/`fedora:46`
      images'), `containers.podman` 1.21.2 and `community.general` 11.4.10 (user; no system copy
      there). That run's new noise, still to clear: `[DEPRECATION WARNING]` for the `tree` and
      `oneline` callbacks at the top of every log (loaded outside ALPI's config: find who lists
      them), `ansible.module_utils.six` in the package and flatpak install tasks and
      `shared/pinned_pair.yml` (find the module), and `Encountered 1 template error` from
      `roles/akmods_mok`'s task names when the block is skipped (`_signer` is `{}`: no `label`). On 2026-10-06 (Fedora): ansible-core 2.20.7 (system),
      `containers.podman` 1.21.0 (user) vs 1.20.2 (system).
