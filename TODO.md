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

AlmaLinux 10 installs AlmaLinux's NVIDIA driver with prebuilt open modules since 2026-11-16
(PROPOSAL "Decisions (2026-11-16)"); a GPU older than Turing is skipped with a message there.
Hybrid laptops get KWin on the iGPU, `prime-run`, switcheroo-control, the kernel/module check
(the upgrade holds a kernel the module cannot load) and `alpi-nvidia-check` since 2026-11-17
(PROPOSAL "Decisions (2026-11-17)"). What is left needs the AlmaLinux notebook
(GeForce MX450, Turing, EFI + Secure Boot; reported 2026-11-12; the KWin fix checked there by
hand on 2026-11-17).

- [ ] **Secure Boot:** EL's `kmod_signer` stays `none` if the AlmaLinux NVIDIA Module CA is
      trusted under Secure Boot without a MOK *(check on real hardware)*; otherwise enroll its
      certificate through `roles/akmods_mok/tasks/enroll.yml`. `secureboot`'s EL instance checks
      the outcome.
- [ ] **Real hardware:** an ALPI run on the AlmaLinux notebook (MX450, EFI + Secure Boot), then
      a reboot, then `alpi-nvidia-check` inside Plasma: every line PASS (driver, no nouveau, the
      iGPU link and its driver, `KWIN_DRM_DEVICES` in `kwin_wayland`'s environment, `prime-run`
      with glxinfo, eglinfo and vulkaninfo). Also by hand: Plasma comes up on the built-in screen;
      the module loads with Secure Boot on; "Run using dedicated graphics card" in Plasma's menus;
      a second run changes nothing. Then a release.
- [ ] **prime-run's EGL line** (`__EGL_VENDOR_LIBRARY_FILENAMES`): the file is the driver's
      (`el10-work` checks it); that a Wayland app renders on NVIDIA through it is not validated
      yet (`alpi-nvidia-check`'s eglinfo line on the notebook). If it breaks an app, drop the two
      `egl_json` lines from `roles/nvidia/files/prime-run`.
- [ ] **External monitors on the NVIDIA GPU:** with the iGPU alone in `KWIN_DRM_DEVICES`, a port
      wired to the NVIDIA GPU stays dark *(check on the notebook which ports hang off which
      GPU)*. `nvidia_kwin_dgpu: true` (custom.yml; run ALPI again after the reboot that loads the
      driver) lists the NVIDIA card second: check that Plasma still starts on the built-in screen
      and the external port lights up; if so, consider turning it on by default on EL.
- [ ] **Kernel held back by the module check:** `roles/update` leaves the kernel out of the
      upgrade while the newest `kmod-nvidia-open` lacks symbols the new kernel provides (PROPOSAL
      "Decisions (2026-11-17)", item 3; tested with stand-ins in `update`). Check on the notebook
      at the next AlmaLinux kernel that the comparison matches what `weak-modules` does (no hold
      when the module loads, a hold when it would not).
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
      environment). Consider an ALPI way back (a feature turned off removes what it added). The
      hybrid fix, `prime-run` and switcheroo-control have their manual undo in PROPOSAL
      "Decisions (2026-11-17)", item 5.
- [ ] **Pre-Turing GPUs on Ubuntu (24.04, 26.04):** the pinned `nvidia-driver-615-open` with
      Canonical's `linux-modules-nvidia-615-open-generic` are open modules, which drive only
      Turing and newer (GTX 16xx, RTX 20xx, MX450 and later); a Maxwell, Pascal or Volta GPU
      (GTX 9xx/10xx…) gets a driver that cannot drive it *(check)*. Decide: a proprietary
      (non-open) branch for those GPUs (which branch still supports them, and Canonical's signed
      `linux-modules-nvidia-NNN-generic` for it), or skip with a message: setting
      `nvidia_open_modules_only` in `group_vars/os_Ubuntu` does that already (the generation
      comes from the PCI device id, `is_nvidia_pre_turing`, as on AlmaLinux). Debian's 550 (proprietary) and Fedora's RPM Fusion
      driver are not affected *(check Fedora's branch support for those GPUs)*.

## Maintenance

- [ ] **`services` flake:** twice on 2026-11-17, in `tests/run.sh full` (scenarios in parallel),
      idempotence failed on "Enable switcheroo-control" (the service no longer active on the
      second run); alone and on reruns it passes, and the service stays up with no restarts.
      Not reproduced: if it recurs, keep the containers (`molecule converge`, then
      `journalctl -u switcheroo-control -b`) and find what stops it.
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
