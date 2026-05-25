# NVIDIA 390.157 DKMS — Kernel 6.8 Compatibility Patches

## Background

**System:** Linux Mint (Ubuntu 24.04 base), x86_64  
**NVIDIA driver:** 390.157 (legacy, end-of-life 2022)  
**Kernel upgraded to:** 6.8.0-117-generic  
**Symptom:** `sudo apt upgrade` failed with exit code 100; four kernel packages left in broken/unconfigured state because the NVIDIA 390.157 DKMS module failed to build against the new kernel.

The NVIDIA 390.157 driver is the last release that supports certain legacy GPUs (GeForce 6xx/7xx/Fermi-era). It was never updated to support kernels beyond ~6.1. This document records the patches required to make it compile against kernel 6.8 and the reasoning behind each one.

---

## The Problem

When `apt upgrade` pulled in `linux-image-6.8.0-117-generic` and `linux-headers-6.8.0-117-generic`, the DKMS post-install hook tried to build the `nvidia/390.157` kernel module. It failed, leaving these four packages in an unconfigured state:

- `linux-headers-6.8.0-117-generic`
- `linux-headers-generic`
- `linux-generic`
- `linux-image-6.8.0-117-generic`

This put dpkg in a broken state where subsequent `apt` operations also failed with `E: Sub-process /usr/bin/dpkg returned an error code (1)`.

Build errors were logged to `/var/lib/dkms/nvidia/390.157/build/make.log`.

---

## How DKMS Works (Important)

The DKMS source tree lives at **`/usr/src/nvidia-390.157/`**. Each time `dkms build` runs, it:
1. Copies the source tree to a fresh build directory at `/var/lib/dkms/nvidia/390.157/build/`
2. Applies any `.patch` files from `/usr/src/nvidia-390.157/patches/`
3. Runs `conftest.sh` to probe the kernel's API and generate header files in `conftest/` within the build tree
4. Compiles the modules

**Critical implication:** Any edits made directly to the build tree at `/var/lib/dkms/nvidia/390.157/build/` are **overwritten** on the next `dkms build` run. All permanent fixes must go into `/usr/src/nvidia-390.157/`.

The conftest headers (`conftest/functions.h`, `conftest/types.h`, `conftest/generic.h`) are **generated at build time** by `conftest.sh`. They cannot be pre-edited in the source tree; instead `conftest.sh` itself must be patched.

---

## Iterative Diagnosis Process

Errors were exposed one layer at a time. Each `dkms build` run revealed the next set of errors after the previous ones were fixed. The layers, in order:

1. `get_user_pages_remote()` API mismatch (primary blocker, blocked all files)
2. `get_user_pages()` API mismatch (triggered once GUP remote was fixed)
3. `vma->vm_flags` read-only (kernel 6.3 change, two modules affected)
4. `objtool` retpoline check failure (precompiled binary blob)
5. `DRM_UNLOCKED` removed / `dumb_destroy` field removed (nvidia-drm)
6. `drm_helper_mode_fill_fb_struct()` removed (nvidia-drm-fb)

Secondary issues (missing prototypes, unused variables, filldir_t type) were patched proactively alongside the primary issues since they were already visible in the build log.

---

## All Patches Applied

All files are relative to `/usr/src/nvidia-390.157/`.

---

### 1. `conftest.sh` — `get_user_pages_remote` detection

**Why:** The conftest only had 4 test cases for `get_user_pages_remote`. Test #4 tried to define a version without `tsk` but still with `vmas`. In kernel 6.5, both `tsk` and `vmas` were removed (commit `ca5e863233e8`). The 4-arg prototype `(mm, start, nr_pages, gup_flags, pages, locked)` did not match any test case, so the script fell through to its default, which incorrectly set:
```
#define NV_GET_USER_PAGES_REMOTE_HAS_TSK_ARG
#undef  NV_GET_USER_PAGES_REMOTE_HAS_LOCKED_ARG
```
This caused `nv-mm.h` to call the kernel's `get_user_pages_remote` with the wrong number and types of arguments, triggering `-Werror=incompatible-pointer-types` in every file that included `nv-linux.h`.

**Fix:** Added a new `NV_GET_USER_PAGES_REMOTE_HAS_VMAS_ARG` flag. Added conftest #5 that tests the 6-arg kernel 6.5+ signature `(mm, start, nr_pages, gup_flags, pages, locked)`. Tagged all existing success cases (#2, #3, #4) with `#define NV_GET_USER_PAGES_REMOTE_HAS_VMAS_ARG`. Conftest #5 success sets `HAS_TSK_ARG=undef`, `HAS_LOCKED_ARG=define`, `HAS_VMAS_ARG=undef`.

**Kernel history:**
- v4.6: `get_user_pages_remote` added (8-arg with write/force)
- v4.9: write/force → gup_flags
- v4.10: `locked` arg added
- v5.9: `tsk` arg removed
- v6.5: `vmas` arg removed

---

### 2. `conftest.sh` — `get_user_pages` detection

**Why:** Same underlying issue. The `get_user_pages` conftest had 3 cases; kernel 6.5 removed `vmas` giving a 4-arg form `(start, nr_pages, gup_flags, pages)`. None of the 3 tests matched, causing the fallthrough default to incorrectly set:
```
#define NV_GET_USER_PAGES_HAS_WRITE_AND_FORCE_ARGS
#define NV_GET_USER_PAGES_HAS_TASK_STRUCT
```
This caused `nv-mm.h` to expand `NV_GET_USER_PAGES` as the old 8-arg call, failing with "too many arguments" in `os-mlock.c`.

**Fix:** Added `NV_GET_USER_PAGES_HAS_VMAS_ARG` flag. Added conftest #4 testing the 4-arg signature. Tagged existing cases with `HAS_VMAS_ARG=define`. New conftest #4 sets `HAS_WRITE_AND_FORCE_ARGS=undef`, `HAS_TASK_STRUCT=undef`, `HAS_VMAS_ARG=undef`.

---

### 3. `common/inc/nv-mm.h` — `get_user_pages_remote` call site

**Why:** The wrapper function `NV_GET_USER_PAGES_REMOTE` had two branches for LOCKED_ARG: one for `HAS_TSK_ARG` (8-arg with vmas) and one without (7-arg, still with vmas). Neither handles the kernel 6.5+ 6-arg form without vmas.

**Fix:** Added a new `#elif defined(NV_GET_USER_PAGES_REMOTE_HAS_VMAS_ARG)` branch for the old no-tsk-but-has-vmas case, and a new `#else` branch for kernel 6.5+:
```c
/* kernel 6.5+: tsk and vmas parameters removed */
return get_user_pages_remote(mm, start, nr_pages, flags, pages, NULL);
```

---

### 4. `common/inc/nv-mm.h` — `get_user_pages` call site

**Why:** The no-task_struct, no-write/force path called:
```c
return get_user_pages(start, nr_pages, flags, pages, vmas);
```
But kernel 6.5+ removed `vmas`, giving a 4-arg form.

**Fix:** Wrapped the call with `#if defined(NV_GET_USER_PAGES_HAS_VMAS_ARG)` / `#else` to call the 4-arg form on kernel 6.5+:
```c
return get_user_pages(start, nr_pages, flags, pages);
```

---

### 5. `nvidia/nv-mmap.c` — `vma->vm_flags` read-only

**Why:** Kernel 6.3 (commit `bc292ab00f6c`) made `vma->vm_flags` a `const vm_flags_t`, prohibiting direct assignment/modification. Kernel now requires `vm_flags_set(vma, flags)` and `vm_flags_clear(vma, flags)`.

**Fix:** Added `#include <linux/version.h>` and two helper macros near the top:
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 3, 0)
#define NV_VMA_FLAGS_SET(vma, f)   vm_flags_set((vma), (f))
#define NV_VMA_FLAGS_CLEAR(vma, f) vm_flags_clear((vma), (f))
#else
#define NV_VMA_FLAGS_SET(vma, f)   ((vma)->vm_flags |= (f))
#define NV_VMA_FLAGS_CLEAR(vma, f) ((vma)->vm_flags &= ~(f))
#endif
```
All six `vma->vm_flags |=` and `vma->vm_flags &= ~` expressions were replaced with `NV_VMA_FLAGS_SET` / `NV_VMA_FLAGS_CLEAR`.

---

### 6. `nvidia/nv-gpu-numa.c` — `filldir_t` return type change

**Why:** Kernel 6.1 (commit `a6cfd9d9b0c3`) changed `filldir_t` from `int (*)(...)` to `bool (*)(...)`. The driver's `filldir_get_memblock_id` function still returned `int`, causing a `-Wcast-function-type` warning treated as error.

**Fix:** Added `#include <linux/version.h>` and a set of conditional macros:
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 1, 0)
#define NV_FILLDIR_RETURN_TYPE bool
#define NV_FILLDIR_OK          true
#define NV_FILLDIR_ERR         false
#else
#define NV_FILLDIR_RETURN_TYPE int
#define NV_FILLDIR_OK          0
#define NV_FILLDIR_ERR         (-ERANGE)
#endif
```
The function signature was changed to `static NV_FILLDIR_RETURN_TYPE filldir_get_memblock_id(...)` and all `return 0` / `return -ERANGE` values replaced with the macros.

---

### 7. `nvidia/nv-acpi.c` — unused variable warning

**Why:** `struct acpi_device *device = NULL;` was declared at function scope in `nv_acpi_methods_uninit()` but only used inside an `#if defined(NV_ACPI_BUS_GET_DEVICE_PRESENT)` block. Since that macro is `#undef` on this system, the variable was never used, triggering a warning promoted to error.

**Fix:** Moved the `struct acpi_device *device = NULL;` declaration inside the `#if defined(NV_ACPI_BUS_GET_DEVICE_PRESENT)` block.

---

### 8. `nvidia/nv.c` — missing prototypes

**Why:** `-Wmissing-prototypes` (promoted to error by `-Werror=strict-prototypes`) fired because `nvidia_init_module` and `nvidia_exit_module` were defined without a prior prototype visible in the translation unit.

**Fix:** Added forward declarations just before the function definitions:
```c
int __init nvidia_init_module(void);
void nvidia_exit_module(void);
```

---

### 9. `nvidia/nv-instance.c` — missing prototype

**Why:** `nv_pci_register_driver` is defined in `nv-instance.c` but its prototype lives in `nv-instance.h`, which was not included by `nv-instance.c`.

**Fix:** Added `#include "nv-instance.h"` after `#include "nv-frontend.h"`.

---

### 10. `nvidia/nv-dma.c` — missing prototype

**Why:** `nv_load_dma_map_scatterlist` had no forward declaration before its definition, triggering `-Wmissing-prototypes`.

**Fix:** Added `void nv_load_dma_map_scatterlist(nv_dma_map_t *dma_map, NvU64 *va_array);` to the forward-declaration block at the top of the file.

---

### 11. `nvidia/nvidia.Kbuild` — objtool retpoline check

**Why:** Kernel 6.8's `objtool` checks all compiled objects for retpoline compliance in `MITIGATION_RETPOLINE` builds. The precompiled binary blob `nv-kernel.o_binary` contains indirect jumps without retpoline wrappers, causing `objtool` to exit with code 241 and fail the `nv.o` build step.

**Fix:** Added `OBJECT_FILES_NON_STANDARD := y` to `nvidia.Kbuild`, which instructs the kernel build system to skip `objtool` validation for all `nvidia.ko` objects. This is the standard approach for modules with pre-built binary components.

---

### 12. `nvidia-uvm/uvm8.c` — `vma->vm_flags` read-only

**Why:** Same kernel 6.3 `vm_flags` read-only issue as patch #5, in the UVM module.

**Fix:** Added `#include <linux/version.h>` and guarded the single `vma->vm_flags |= VM_MIXEDMAP | VM_DONTEXPAND;` assignment:
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 3, 0)
    vm_flags_set(vma, VM_MIXEDMAP | VM_DONTEXPAND);
#else
    vma->vm_flags |= VM_MIXEDMAP | VM_DONTEXPAND;
#endif
```

---

### 13. `nvidia-drm/nvidia-drm-drv.c` — `DRM_UNLOCKED` removed

**Why:** `DRM_UNLOCKED` was removed from kernel headers in kernel 6.8 (it had been a no-op since ~kernel 4.x when all DRM ioctls became unlocked by default). All `DRM_IOCTL_DEF_DRV(...)` macro invocations that passed `DRM_UNLOCKED` as a flag failed with "undeclared identifier."

**Fix:** Added a compatibility `#ifndef` guard before the `nv_drm_ioctls[]` array:
```c
#ifndef DRM_UNLOCKED
#define DRM_UNLOCKED 0
#endif
```
This defines it as 0 (a no-op bitmask value) when the kernel no longer provides it.

---

### 14. `nvidia-drm/nvidia-drm-drv.c` — `dumb_destroy` field removed

**Why:** The `dumb_destroy` callback was removed from `struct drm_driver` in kernel 6.8 (GEM dumb buffer destroy is now handled generically).

**Fix:** Guarded the assignment with a version check:
```c
#if defined(DRIVER_HAS_DUMB_DESTROY) || LINUX_VERSION_CODE < KERNEL_VERSION(6, 8, 0)
    nv_drm_driver.dumb_destroy = nv_drm_dumb_destroy;
#endif
```

---

### 15. `nvidia-drm/nvidia-drm-fb.c` — `drm_helper_mode_fill_fb_struct` removed

**Why:** `drm_helper_mode_fill_fb_struct()` was removed from kernel headers in kernel 5.14 (commit `2cb0d7f7ea98`). The function filled in a `struct drm_framebuffer` from a `struct drm_mode_fb_cmd2`. The conftest for this function works by trying to redefine it — if the kernel's header doesn't declare it, the redefinition compiles, and the conftest incorrectly reports it as "present with dev arg." The actual call site then gets an "implicit declaration" error.

**Fix:** Added `#include <linux/version.h>`, `#include <drm/drm_fourcc.h>`, and a static fallback implementation for kernels ≥ 5.14:
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 14, 0)
static void drm_helper_mode_fill_fb_struct(struct drm_device *dev,
                                            struct drm_framebuffer *fb,
                                            const struct drm_mode_fb_cmd2 *cmd)
{
    int i;
    fb->dev    = dev;
    fb->format = drm_get_format_info(dev, cmd);
    fb->width  = cmd->width;
    fb->height = cmd->height;
    for (i = 0; i < 4; i++) {
        fb->pitches[i] = cmd->pitches[i];
        fb->offsets[i] = cmd->offsets[i];
    }
    fb->modifier = cmd->modifier[0];
    fb->flags    = cmd->flags;
}
#endif
```

---

## Phase 2: Display / Xorg Failures After Boot

After the DKMS kernel modules built and installed successfully, the system booted into kernel 6.8 but **hung at the splash screen** with intermittent disk activity. The kernel modules loaded correctly, but the desktop environment never appeared.

### Root Cause: `nvidia_drv.so` ABI incompatibility with Xorg 21.x

Diagnosis from recovery mode (`journalctl -b -1 --no-pager -p err`) showed Xorg crashing repeatedly with a core dump:

```
Process 1242 (Xorg) dumped core.
  Module nvidia_drv.so without build-id.
  Module libnvidia-glcore.so.390.157 without build-id.
  Stack: libglx.so → AddCallback (Xorg) → tcache_get_n (libc) → SIGSEGV
```

The crash occurred inside `libglx.so` (NVIDIA's Xorg GLX extension, also a precompiled binary blob) when it called Xorg's internal `AddCallback`, which triggered a heap corruption signal. This is an **ABI mismatch** between the 390.157 binary userspace GLX extension (compiled for Xorg ~1.13 era) and the modern Xorg 21.x server shipped with Ubuntu 24.04.

The crash cannot be fixed by patching — `nvidia_drv.so` and `libglx.so` are closed-source precompiled binaries. The kernel module itself was healthy; `nvidia-smi` worked from recovery.

### Why Xorg Was Loading nvidia_drv.so

Two files in `/usr/share/X11/xorg.conf.d/` contain `OutputClass` sections that match any device driven by `nvidia-drm` and force the `nvidia` Xorg DDX driver:

- `/usr/share/X11/xorg.conf.d/10-nvidia.conf` — installed by `nvidia-driver-390`
- `/usr/share/X11/xorg.conf.d/11-nvidia-prime.conf` — auto-generated at boot by `gpu-manager`

With `nvidia-drm.modeset=1` in the kernel parameters, `nvidia-drm` exposes a proper DRM/KMS device, which triggers these OutputClass matches, which loads the crashing `nvidia_drv.so`.

An attempt to override these with `/etc/X11/xorg.conf.d/20-nvidia-modesetting.conf` failed because `/etc/X11/xorg.conf.d/` files are processed **before** `/usr/share/X11/xorg.conf.d/` files — the system configs overrode the override.

### Fix: Disable the Xorg NVIDIA OutputClass configs

**Step 1 — Mask `gpu-manager`** so it cannot regenerate `11-nvidia-prime.conf` at boot:

```bash
sudo systemctl disable gpu-manager
sudo systemctl mask gpu-manager
```

**Step 2 — Rename the OutputClass config files** so Xorg no longer loads them:

```bash
sudo mv /usr/share/X11/xorg.conf.d/10-nvidia.conf \
        /usr/share/X11/xorg.conf.d/10-nvidia.conf.disabled
sudo mv /usr/share/X11/xorg.conf.d/11-nvidia-prime.conf \
        /usr/share/X11/xorg.conf.d/11-nvidia-prime.conf.disabled
```

Without these OutputClass sections, Xorg automatically falls back to the `modesetting` driver, which uses the kernel's `nvidia-drm` DRM/KMS interface directly — no `nvidia_drv.so` involved.

### Additional GRUB Parameter Required

For the `modesetting` Xorg driver to access the NVIDIA GPU via DRM, the `nvidia-drm` kernel module must be loaded with KMS enabled. Add to `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1"
```

Then regenerate GRUB:
```bash
sudo update-grub
```

### Final Working Display Configuration

| Component | Details |
|-----------|----------|
| Xorg DDX driver | `modesetting` (via `nvidia-drm` KMS) |
| `nvidia_drv.so` | **Not loaded** (incompatible with Xorg 21.x) |
| Desktop 2D / video | Fully functional |
| `nvidia-smi` / compute | Functional (kernel module still loaded) |
| NVIDIA proprietary GLX | Not available (replaced by Mesa/software GLX) |
| `gpu-manager` | Masked (prevented from regenerating prime conf) |

### Important: If nvidia packages are upgraded

`apt upgrade` of `nvidia-driver-390` or related packages may restore the renamed `.conf` files and re-enable `gpu-manager`. After any nvidia package upgrade, re-check:

```bash
ls /usr/share/X11/xorg.conf.d/10-nvidia.conf 2>/dev/null && echo "WARNING: re-disable this file"
systemctl is-enabled gpu-manager && echo "WARNING: gpu-manager re-enabled"
```

If either appears, repeat the rename/mask steps above.

---

## Final Verification

After all patches and display fixes were applied — confirmed working on kernel 6.8.0-117:

```
uname -r
# → 6.8.0-117-generic

nvidia-smi --query-gpu=name,driver_version --format=csv,noheader
# → NVS 5400M, 390.157

dkms status nvidia/390.157 | grep 6.8.0-117
# → nvidia/390.157, 6.8.0-117-generic, x86_64: installed

apt-cache policy linux-image-generic | grep 'Installed\|Candidate'
# → Installed: 6.8.0-117.117   Candidate: 6.8.0-117.117  (fully up to date)

sudo dpkg --audit
# → (no output = clean)
```

Desktop environment (LightDM + Cinnamon) loads normally on every boot.

---

## Re-Applying These Patches

If DKMS needs to be rebuilt (e.g., after a source tree reinstall, or for a new kernel):

```bash
# Verify source tree patches are in place
grep -c 'conftest #5' /usr/src/nvidia-390.157/conftest.sh
grep -c 'NV_FILLDIR_RETURN_TYPE' /usr/src/nvidia-390.157/nvidia/nv-gpu-numa.c
grep -c 'OBJECT_FILES_NON_STANDARD' /usr/src/nvidia-390.157/nvidia/nvidia.Kbuild

# Rebuild for the current running kernel
sudo dkms build nvidia/390.157 -k $(uname -r)
sudo dkms install nvidia/390.157 -k $(uname -r)
sudo dpkg --configure -a
```

If the source tree gets reinstalled (e.g., by `apt reinstall nvidia-dkms-390`), all patches would need to be reapplied. The README and this context can serve as the reference.

Verify the display config is still intact after any nvidia package upgrade (see warning above).

---

## Key API / ABI Change Timeline (Relevant to 390.xx)

| Version | Change | Affected component |
|---------|--------|--------------------|
| Kernel 5.9 | `get_user_pages_remote`: `tsk` arg removed | `nv-mm.h`, `conftest.sh` |
| Kernel 5.14 | `drm_helper_mode_fill_fb_struct()` removed | `nvidia-drm-fb.c` |
| Kernel 6.1 | `filldir_t` return type changed `int` → `bool` | `nv-gpu-numa.c` |
| Kernel 6.3 | `vma->vm_flags` made `const` (read-only) | `nv-mmap.c`, `uvm8.c` |
| Kernel 6.5 | `get_user_pages[_remote]`: `vmas` arg removed | `nv-mm.h`, `conftest.sh` |
| Kernel 6.8 | `DRM_UNLOCKED` removed from DRM headers | `nvidia-drm-drv.c` |
| Kernel 6.8 | `dumb_destroy` removed from `struct drm_driver` | `nvidia-drm-drv.c` |
| Xorg 21.x | `AddCallback` internal ABI changed; breaks `nvidia_drv.so` 390.157 | Xorg display config |

---

## Notes and Caveats

- **This driver is EOL.** NVIDIA 390.xx does not support kernels beyond ~6.1 officially. These patches are best-effort compatibility shims. Stability is not guaranteed.
- **Two separate binary blob layers** cannot be patched: the kernel-side `nv-kernel.o_binary` (retpoline bypass needed) and the userspace `nvidia_drv.so`/`libglx.so` (incompatible with Xorg 21.x — worked around by disabling it entirely).
- **New kernel updates** that bump the kernel minor version (e.g., `6.8.0-118-generic`) will trigger a fresh DKMS build automatically and should succeed because the source patches are persistent. However, if a future kernel introduces additional API breaks, the build log at `/var/lib/dkms/nvidia/390.157/build/make.log` is the first place to look.
- **If you see dpkg broken state again**, run `sudo dpkg --audit` to confirm it's the same root cause, then check the DKMS make.log before proceeding.
- **No NVIDIA proprietary 3D/OpenGL** — Mesa software rendering is used for GLX. For this ThinkPad T530 (NVS 5400M, Fermi/GF108M), this is acceptable as the GPU is a workstation card used primarily for display/compute, not gaming.
