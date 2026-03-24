# ozols — ASUS ROG Zephyrus G14 (2021) Post-Install Configuration
**Debian 13 (Trixie) · LUKS/LVM · GNOME Wayland · NVIDIA 550 · Kernel 6.12.74+deb13+1-amd64**
*Documented: 2026-03-24*

---

## Hardware
| Component | Detail |
|-----------|--------|
| CPU | AMD Ryzen 9 5900HS |
| RAM | 40 GB DDR4 |
| iGPU | AMD Radeon Vega (amdgpu) — primary display |
| dGPU | NVIDIA RTX 3060 Mobile (GA106M) — PRIME offload |
| Storage | Sabrent Rocket Q 4TB NVMe (nvme0n1) |
| Display | 3072×1728 14" |
| Wi-Fi | Mediatek MT7921 |
| Audio | Realtek ALC289 |

---

## LVM Layout
| LV | VG | Size | Mount |
|----|-----|------|-------|
| root | ozols-vg | ~3.6T | / |
| swap_1 | ozols-vg | 39.4G | [SWAP] |

LUKS container: `/dev/nvme0n1p3`
Resume path: `/dev/mapper/ozols--vg-swap_1`

---

## Key Configuration Files

### `/etc/modules-load.d/nvidia.conf`
```
nvidia-current
nvidia-current-modeset
nvidia-current-drm
nvidia-current-uvm
```
**Purpose:** Loads all NVIDIA DKMS modules early via systemd-modules-load, before GDM starts.
Note: Debian packages NVIDIA modules as `nvidia-current-*` not `nvidia-*`.

---

### `/etc/udev/rules.d/61-gdm.rules`
Override of `/usr/lib/udev/rules.d/61-gdm.rules`.

**Lines commented out to enable Wayland:**
- Line 43: `#TEST{0711}!="/usr/bin/nvidia-sleep.sh", GOTO="gdm_disable_wayland"`
- Line 44: `#TEST{0711}!="/usr/lib/systemd/system-sleep/nvidia", GOTO="gdm_disable_wayland"`
- Line 46: `#ENV{NVIDIA_PRESERVE_VIDEO_MEMORY_ALLOCATIONS}!="1", GOTO="gdm_disable_wayland"`
- Line 48: `#ENV{NVIDIA_HIBERNATE}!="enabled", GOTO="gdm_disable_wayland"`
- Line 50: `#ENV{NVIDIA_RESUME}!="enabled", GOTO="gdm_disable_wayland"`
- Line 52: `#ENV{NVIDIA_SUSPEND}!="enabled", GOTO="gdm_disable_wayland"`
- Line 67: `#LABEL="gdm_virt_passthrough_check"` (label commented to disable block)
- Line 71: `#GOTO="gdm_disable_wayland"` (virt passthrough false positive on hybrid GPU)

**Root cause documented:** GDM's udev rule detects hybrid graphics (AMD+NVIDIA) and
misidentifies the machine as a VM with GPU passthrough, disabling Wayland. The
`gdm_virt_passthrough_check` block fires because all three marker files exist:
`gdm-machine-has-hybrid-graphics`, `gdm-machine-has-virtual-gpu`,
`gdm-machine-has-hardware-gpu`.

**If a package update overwrites `/usr/lib/udev/rules.d/61-gdm.rules`:**
The override in `/etc/udev/rules.d/61-gdm.rules` takes precedence and will not
be affected. However, if the override is lost, re-copy and re-apply the commented lines.

---

### `/etc/udev/rules.d/70-nvidia-uvm.rules`
```
KERNEL=="nvidia", RUN+="/usr/bin/nvidia-modprobe -c0 -u"
KERNEL=="nvidia_uvm", RUN+="/usr/bin/nvidia-modprobe -c0 -u"
SUBSYSTEM=="module", ACTION=="add", DEVPATH=="/module/nvidia_uvm", RUN+="/usr/bin/nvidia-modprobe -c0 -u"
```
**Purpose:** Creates `/dev/nvidia-uvm` and `/dev/nvidia-uvm-tools` device nodes at boot.
Required for PyTorch CUDA access. Without this, `torch.cuda.is_available()` returns False.

---

### `/etc/udev/rules.d/99-disable-fingerprint.rules`
```
SUBSYSTEM=="usb", ATTR{idVendor}=="27c6", ATTR{idProduct}=="521d", ATTR{authorized}="0"
```
**Purpose:** Disables Goodix fingerprint reader (poor Linux support, not needed).

---

### `/etc/modprobe.d/nvidia-preserve.conf`
```
options nvidia NVreg_PreserveVideoMemoryAllocations=1
```
**Purpose:** Required for GDM Wayland — satisfies the
`NVIDIA_PRESERVE_VIDEO_MEMORY_ALLOCATIONS` check in 61-gdm.rules (even though
that line is commented out, this is correct to have for suspend/resume stability).

---

### `/etc/modprobe.d/nvidia-drm.conf`
```
options nvidia_drm modeset=1
```
**Purpose:** Enables NVIDIA DRM modesetting, required for Wayland.

---

### `/etc/default/grub`
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_pstate=active module.sig_enforce=0 resume=/dev/mapper/ozols--vg-swap_1"
```
| Parameter | Purpose |
|-----------|---------|
| `amd_pstate=active` | AMD P-State EPP for Zen 3 frequency scaling |
| `module.sig_enforce=0` | Required — removing this broke the system |
| `resume=...` | Hibernate resume from encrypted swap LV |

---

### `/etc/gdm3/daemon.conf`
```ini
[daemon]
WaylandEnable=true
InitialSetupEnable=false
GdmXserverTimeout=30
```

---

### `/etc/tlp.conf` (relevant lines)
```
STOP_CHARGE_THRESH_BAT0=80
```
Battery charge threshold handled natively by `asus_wmi` driver via TLP.
Verify with: `sudo tlp-stat -b | grep -i threshold`

---

## NVIDIA Services (must remain enabled)
```bash
systemctl is-enabled nvidia-hibernate.service nvidia-resume.service nvidia-suspend.service
# All three must return: enabled
```

---

## DO NOT touch
- `NVreg_DynamicPowerManagement` — caused black screen twice. GPU idles at ~9W, acceptable.
- `scale-monitor-framebuffer` experimental feature — caused GNOME Shell crash on cold boot.

---

## ML Environment
```bash
source ~/ml-env/bin/activate
```
**Packages:** torch (cu124), torchvision, torchaudio, scikit-learn, pandas, numpy,
matplotlib, seaborn, jupyterlab, transformers, datasets, accelerate,
sentence-transformers, elasticsearch, bitsandbytes

**Verify CUDA:**
```bash
python3 -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
# Expected: True / NVIDIA GeForce RTX 3060 Laptop GPU
```

**VRAM:** 6GB — fits small/medium models. Use bitsandbytes quantization for 7B+ parameter models.

---

## prime-run wrapper
```bash
# /usr/local/bin/prime-run
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia __VK_LAYER_NV_optimus=NVIDIA_only exec "$@"
```
Usage: `prime-run steam`, `prime-run glxinfo`, etc.
Steam launch option: `prime-run %command%`

---

## Recovery Reference

### If Wayland breaks after kernel/package update
1. Check `/run/gdm3/custom.conf` — if `WaylandEnable=false` is present, udev rule is firing
2. Verify `/etc/udev/rules.d/61-gdm.rules` override is intact
3. Check `systemctl is-enabled nvidia-hibernate nvidia-resume nvidia-suspend`
4. Check `/etc/modules-load.d/nvidia.conf` has all four `nvidia-current-*` entries

### If NVIDIA modules fail to load after kernel update
```bash
sudo apt install --reinstall nvidia-kernel-dkms
sudo update-initramfs -u
sudo reboot
```

### If CUDA stops working (torch.cuda.is_available() = False)
```bash
ls /dev/nvidia-uvm*  # should exist
sudo modprobe nvidia-current-uvm
sudo mknod -m 666 /dev/nvidia-uvm c $(grep nvidia-uvm /proc/devices | awk '{print $1}') 0
```

### Chroot recovery from live USB (LUKS)
```bash
sudo cryptsetup open /dev/nvme0n1p3 crypt_root
sudo lvs  # confirm VG name: ozols-vg
sudo mount /dev/ozols-vg/root /mnt
sudo mount /dev/nvme0n1p2 /mnt/boot
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt
```
