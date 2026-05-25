The system is fully working and documented. Here's what's left, roughly in order of priority:

Right Now (Optional but Useful)

Post to Linux Mint Forums — takes 5 minutes and will help others hitting this exact problem. 
Use the announcement text from earlier, post at: 
https://forums.linuxmint.com/viewforum.php?f=42.
Also worth posting a new thread in Hardware Support since the Wilma thread is in Releases & Announcements:
https://forums.linuxmint.com/viewforum.php?f=49

Routine Maintenance to Watch For

When apt upgrade runs next:
bash
If a new kernel is pulled in (e.g. 6.8.0-118-generic), DKMS will auto-rebuild using your patched source tree — it should just work. Verify with dkms status after the upgrade.

If nvidia-dkms-390 is ever reinstalled or upgraded, the source tree patches will be overwritten. Load up the README at ~/nvidia-390-kernel68-patches/README.md and reapply them.

Nothing Else Required

The system is:
•  ✅ Fully up to date
•  ✅ Kernel 6.8 running with NVIDIA driver functional
•  ✅ Desktop loading normally
•  ✅ dpkg clean
•  ✅ Work documented and backed up to GitHub

You're done — just use your computer.



-----
[SOLVED] NVIDIA 390.157 DKMS build fails + blank screen on kernel 6.8 (Ubuntu 24.04 / Mint 22)

------

If you upgraded to kernel 6.8 with a legacy NVIDIA GPU (Fermi generation —
NVS 5400M, Quadro K1000M, GeForce 630M/640M, etc.) and hit either of these:

  • sudo apt upgrade breaks dpkg with 4 unconfigured kernel packages
  • DKMS build fails in /var/lib/dkms/nvidia/390.157/build/make.log
  • System boots to a blank screen / hangs at splash after fixing DKMS

I've spent a full session diagnosing and fixing both problems and documented
everything here:

  https://github.com/earldodd/nvidia-390-kernel68-patches

SHORT VERSION OF THE FIX:

Problem 1 — DKMS build fails:
The driver source needs 15 patches to handle kernel API changes introduced
between kernel 5.9 and 6.8. The most critical are get_user_pages_remote()
losing its tsk and vmas arguments, vm_flags becoming read-only, and several
DRM API removals. Full patch details and reasoning in the README.

Problem 2 — Blank screen after reboot:
The NVIDIA 390.157 Xorg driver (nvidia_drv.so) crashes against Xorg 21.x
on Ubuntu 24.04 — it's a binary ABI incompatibility that can't be patched.
Fix: disable the two Xorg OutputClass configs that force it to load, and
let Xorg fall back to the modesetting driver over nvidia-drm KMS instead.

  sudo systemctl mask gpu-manager
  sudo mv /usr/share/X11/xorg.conf.d/10-nvidia.conf{,.disabled}
  sudo mv /usr/share/X11/xorg.conf.d/11-nvidia-prime.conf{,.disabled}

Also add nvidia-drm.modeset=1 to GRUB_CMDLINE_LINUX_DEFAULT in
/etc/default/grub and run sudo update-grub.

VERIFIED WORKING ON:
  GPU:    NVIDIA NVS 5400M (GF108M, Fermi / ThinkPad T530)
  Kernel: 6.8.0-117-generic
  OS:     Linux Mint 22 / Ubuntu 24.04
  Result: nvidia-smi functional, desktop loading normally

Full details, all 15 patches with explanations, and re-application
instructions at:

  https://github.com/earldodd/nvidia-390-kernel68-patches

Hope this saves someone a few hours.



