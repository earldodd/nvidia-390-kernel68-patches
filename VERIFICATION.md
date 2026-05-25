# Final Verification — NVIDIA 390.157 on Kernel 6.8

**Date:** 2026-05-25  
**System:** earldodd-ThinkPad-T530, Linux Mint (Ubuntu 24.04 base)

---

## `uname -r`
```
6.8.0-117-generic
```

---

## `dkms status nvidia/390.157`
```
nvidia/390.157, 5.15.0-179-generic, x86_64: installed
nvidia/390.157, 6.8.0-117-generic, x86_64: installed
```

---

## `nvidia-smi`
```
Mon May 25 16:02:31 2026
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.157                Driver Version: 390.157                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  NVS 5400M           Off  | 00000000:01:00.0 N/A |                  N/A |
| N/A   49C    P8    N/A /  N/A |      4MiB /   964MiB |     N/A      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0                    Not Supported                                       |
+-----------------------------------------------------------------------------+
```

---

## Summary

| Check | Result |
|-------|--------|
| Kernel | 6.8.0-117-generic |
| DKMS modules built | nvidia, nvidia-uvm, nvidia-modeset, nvidia-drm |
| DKMS status (6.8) | installed |
| DKMS status (5.15) | installed |
| GPU | NVS 5400M (GF108M, Fermi) |
| Driver version | 390.157 |
| nvidia-smi | functional |
| dpkg state | clean (`dpkg --audit` returns nothing) |
| Desktop | LightDM + Cinnamon loading normally |
| Xorg driver | modesetting (nvidia_drv.so disabled — ABI incompatible with Xorg 21.x) |
| Kernel parameter | nvidia-drm.modeset=1 |
| gpu-manager | masked |
