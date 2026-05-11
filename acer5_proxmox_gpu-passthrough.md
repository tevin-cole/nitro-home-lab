# NVIDIA GTX 1650 Mobile GPU Passthrough to Proxmox LXC
### Acer Nitro 5 (AN515-54) — Complete Home Lab Guide

> Written from real experience getting GPU-accelerated LLM inference (Ollama) working on a Proxmox home lab running on an Acer Nitro 5 laptop with a discrete NVIDIA GTX 1650 Mobile GPU. This guide documents everything that was tried, what failed, and what ultimately worked — including all the dead ends — so future homelabers don't have to rediscover them.

---

## Hardware & Software Context

| Item | Detail |
|---|---|
| Laptop | Acer Nitro 5 AN515-54 |
| CPU | Intel Core i7 (Coffee Lake-H) |
| GPU | NVIDIA GeForce GTX 1650 Mobile / Max-Q (TU117M, 4GB VRAM) |
| GPU Architecture | Turing (Compute 7.5) |
| GPU Type | Discrete, Optimus/MUXless |
| Hypervisor | Proxmox VE 9 |
| Proxmox Kernel | 6.14.11-6-pve |
| LXC OS | Ubuntu Noble 24.04 |
| Goal | GPU-accelerated Ollama inference in an LXC container |

---

## Part 1: Key Concepts Before You Start

### MUXed vs MUXless (Why Laptop GPUs Are Tricky)

The Acer Nitro 5 uses **NVIDIA Optimus** in a **MUXless** configuration. This means the discrete GPU's framebuffer does not connect directly to any physical display output — all display output routes through the Intel iGPU first.

**What this means in practice:**
- Traditional VM-based GPU passthrough (VFIO) is complicated on MUXless laptops
- For compute-only workloads like LLM inference, you do **not** need VFIO passthrough
- LXC device passthrough works well and delivers near-native GPU performance
- The GPU will be visible to Proxmox even without a display connected (confirmed on this hardware)

### LXC Device Passthrough vs VM Passthrough

For running Ollama or other CUDA compute workloads, **LXC device passthrough is the right approach**, not VM (VFIO) passthrough. The difference:

- **LXC passthrough**: Share `/dev/nvidia*` device nodes directly into the container. The container uses the host's kernel module. Near-native performance with minimal setup.
- **VM passthrough (VFIO)**: Give a full VM exclusive ownership of the GPU via IOMMU. Requires IOMMU isolation, vBIOS extraction, and much more complex setup. Mainly useful for gaming VMs or Windows guests.

For inference workloads, LXC passthrough is the correct and simpler choice.

---

## Part 2: Verifying GPU Visibility on the Proxmox Host

Before installing anything, confirm Proxmox can see the GPU.

```bash
lspci | grep -i nvidia
```

Expected output (good sign):
```
01:00.0 VGA compatible controller: NVIDIA Corporation TU117M [GeForce GTX 1650 Mobile / Max-Q] (rev a1)
01:00.1 Audio device: NVIDIA Corporation Device 10fa (rev a1)
```

If nothing appears, the GPU may be powered down. Try connecting an HDMI display or HDMI dummy plug. On this hardware the GPU was visible without one.

### Get PCI IDs (needed later)
```bash
lspci -nn | grep -i nvidia
```
Note the IDs in brackets e.g. `[10de:1f91]`.

### Check IOMMU Status
```bash
dmesg | grep -i iommu
```

On this hardware, the GTX 1650 landed in **IOMMU Group 2** along with the Intel PCIe bridge (`00:01.0`). This means pure VM VFIO passthrough would require passing the bridge too — another reason LXC is the better approach for this hardware.

### Check IOMMU Groups
```bash
find /sys/kernel/iommu_groups/ -type l | sort -V | while read f; do
  echo "Group $(basename $(dirname $f)): $(lspci -nns $(basename $f))"
done
```

---

## Part 3: Installing NVIDIA Drivers on the Proxmox Host

### Step 1: Enable the non-free Repository

Proxmox uses the DEB822 format for apt sources. The default `debian.sources` file does not include the `non-free` component where NVIDIA drivers live.

Check your current sources:
```bash
cat /etc/apt/sources.list.d/debian.sources
```

Edit the file and add `non-free` to both `Components` lines:
```bash
nano /etc/apt/sources.list.d/debian.sources
```

Each `Components` line should read:
```
Components: main contrib non-free non-free-firmware
```

Update package lists:
```bash
apt update
```

Verify the driver package is now visible:
```bash
apt-cache search nvidia-driver
```
You should now see `nvidia-driver` in the results.

### Step 2: Install Kernel Headers and Driver

```bash
apt install pve-headers-$(uname -r)
apt install nvidia-driver
```

The second command triggers DKMS to compile the kernel module. This takes a few minutes and produces a lot of output — that is normal.

### Step 3: Deal with the Secure Boot Problem

After installation, reboot and run:
```bash
nvidia-smi
```

If you get:
```
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
```

Check Secure Boot status:
```bash
mokutil --sb-state
```

If `SecureBoot enabled` — this is your problem. The kernel refuses to load unsigned third-party modules.

**Acer Nitro 5 Secure Boot is grayed out in BIOS** — this is by design. To unlock it:

1. Boot into BIOS (F2 at startup)
2. Go to the **Security** tab
3. Set a **Supervisor Password** (this is required to unlock security settings — any password works)
4. Now go to the **Boot** tab — Secure Boot will be clickable
5. Set Secure Boot to **Disabled**
6. Save and exit (F10)

After rebooting with Secure Boot disabled:
```bash
modprobe nvidia && echo "loaded OK"
nvidia-smi
```

### Step 4: Confirm the Driver is Working

A successful `nvidia-smi` output looks like:
```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.142             Driver Version: 580.142       CUDA Version: 12.x        |
|-----------------------------------------+------------------------+----------------------+
|   0  NVIDIA GeForce GTX 1650        On  |   00000000:01:00.0 Off |                  N/A |
| N/A   47C    P8              2W /   50W |       1MiB /   4096MiB |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

Confirm device nodes exist:
```bash
ls -la /dev/nvidia*
```

You should see:
- `/dev/nvidia0` — major 195
- `/dev/nvidiactl` — major 195
- `/dev/nvidia-modeset` — major 195
- `/dev/nvidia-uvm` — major 507
- `/dev/nvidia-uvm-tools` — major 507
- `/dev/nvidia-caps/nvidia-cap1` — major 511
- `/dev/nvidia-caps/nvidia-cap2` — major 511

---

## Part 4: The Driver Version Mismatch Problem (Important)

This is the most painful part of the process and deserves its own section.

### The Problem

The Proxmox host runs **Debian Trixie**. Debian's `non-free` repo at the time of writing provides NVIDIA driver **550.163.01**.

The Ollama LXC runs **Ubuntu Noble 24.04**. Ubuntu's `restricted` repo provides driver **580.142**.

When you install `nvidia-utils-550` or `libnvidia-compute-550` on Ubuntu Noble, they are **transitional stub packages** (~11KB each) that pull in the 580 versions as the actual implementation. Ubuntu has moved the 550 series to use 580 libraries.

This causes a mismatch:
- Host kernel module: 550.x
- Container userspace libraries: 580.x
- Result: `Failed to initialize NVML: Driver/library version mismatch`

### The Solution: Upgrade the Host to Match the Container

Since Debian's repos don't have 580, install it on the Proxmox host using NVIDIA's official `.run` installer:

**Remove the apt-installed driver first:**
```bash
apt remove --purge nvidia-driver nvidia-open-kernel-dkms nvidia-open-kernel-source glx-alternative-nvidia
apt autoremove
```

Verify everything is clean:
```bash
dpkg -l | grep -i nvidia  # should return nothing or only rc entries
```

**Download and install the matching version:**
```bash
cd /root
wget https://download.nvidia.com/XFree86/Linux-x86_64/580.142/NVIDIA-Linux-x86_64-580.142.run
chmod +x NVIDIA-Linux-x86_64-580.142.run
./NVIDIA-Linux-x86_64-580.142.run --dkms
```

The `--dkms` flag registers the driver with DKMS so it rebuilds automatically on kernel updates.

**If the installer complains about a previously installed package:**

The installer may detect leftover files. Force it past the check:
```bash
./NVIDIA-Linux-x86_64-580.142.run --dkms --no-check-for-alternate-installs
```

Reboot after installation and confirm with `nvidia-smi`.

### The `.run` Installer Pitfall in the LXC

At some point a previous attempt involved running the NVIDIA `.run` installer **inside the LXC container**. This is incorrect for LXC passthrough — the container needs only userspace libraries, not kernel modules.

The `.run` installer also does not integrate with apt, so `apt remove` won't clean it up. To uninstall a `.run`-installed driver:
```bash
./NVIDIA-Linux-x86_64-XXX.run --uninstall
# or
nvidia-uninstall
```

---

## Part 5: Configuring the LXC Container for GPU Passthrough

### Step 1: Stop the Container

```bash
pct stop <CONTAINER_ID>
```

### Step 2: Edit the Container Configuration

```bash
nano /etc/pve/lxc/<CONTAINER_ID>.conf
```

Add the following lines at the bottom:

```
# NVIDIA GPU passthrough
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 507:* rwm
lxc.cgroup2.devices.allow: c 511:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap1 dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap2 dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file
```

The major numbers (195, 507, 511) correspond to the device node groups above. If your system shows different major numbers from `ls -la /dev/nvidia*`, update these accordingly.

### Step 3: Start the Container

```bash
pct start <CONTAINER_ID>
pct enter <CONTAINER_ID>
```

### Step 4: Install NVIDIA Userspace Libraries in the Container

For an Ubuntu Noble container, install only the userspace libraries — **not** kernel modules (the container shares the host's kernel):

```bash
apt install nvidia-utils-580 libnvidia-compute-580
```

> Use the version number that matches your host driver. If your host runs 580.142, install the 580 packages.

Verify inside the container:
```bash
nvidia-smi
```

This should display the same GPU table as the host. If you get a version mismatch error, ensure the package version in the container matches the kernel module version on the host exactly.

---

## Part 6: Confirming Ollama Uses the GPU

### Check Ollama Detects the GPU at Startup

```bash
journalctl -u ollama --no-pager | grep -i -E "gpu|cuda|nvidia|inference compute"
```

Look for a line like:
```
msg="inference compute" description="NVIDIA GeForce GTX 1650" library=CUDA total="4.0 GiB" available="3.6 GiB"
```

This confirms Ollama found the GPU via CUDA.

### Run a Test

```bash
ollama pull phi3:mini
ollama run phi3:mini "hello, what are you running on?"
```

### Watch GPU Utilization in Real Time

In a second terminal session:
```bash
nvidia-smi dmon -s u -d 1
```

The `sm%` (shader/compute utilization) and `mem%` columns should spike during inference. Seeing activity here confirms the model is running on GPU, not CPU.

---

## Part 7: Notes, Tips, and Gotchas

### Kernel 6.17 Incompatibility (If You Upgrade)

NVIDIA drivers (550.x from Debian) fail to build against Proxmox kernel 6.17 due to DRM API changes. If you upgrade the Proxmox kernel to 6.17, you will need to use NVIDIA's `.run` installer with a newer driver version (565+) or pin back to 6.14.

### OLLAMA_KEEP_ALIVE

By default Ollama unloads models from VRAM after 5 minutes of inactivity. To keep a model permanently loaded:

Edit the Ollama systemd service:
```bash
systemctl edit ollama
```

Add:
```
[Service]
Environment="OLLAMA_KEEP_ALIVE=-1"
```

### Model Size Recommendations for 4GB VRAM

| Model | VRAM Usage | Notes |
|---|---|---|
| phi3:mini (3.8B Q4) | ~2.3 GiB | Excellent, recommended starting point |
| phi3.5:mini | ~2.3 GiB | Slightly better than phi3:mini |
| gemma2:2b | ~1.6 GiB | Fast, good for lightweight tasks |
| llama3.2:3b | ~2.0 GiB | Good general purpose |
| mistral:7b Q4 | ~4.1 GiB | Borderline — may partially CPU offload |

Avoid 7B+ models without Q3 or lower quantization as they will exceed VRAM and spill to system RAM.

### Initial Load Delay is Normal

The first inference after loading a model takes 20-40 seconds — this is the model being loaded from disk into VRAM. Subsequent prompts in the same session are much faster.

### After a Proxmox Host Driver Update

Any time the host driver is updated, restart the LXC container so it picks up the new `/dev/nvidia*` device nodes:
```bash
pct stop <ID>
pct start <ID>
```

---

## Summary: What Actually Worked

1. GPU is visible on Proxmox host without external display (Optimus didn't block it)
2. Enable `non-free` in Debian apt sources → install `pve-headers` + `nvidia-driver`
3. Unlock Secure Boot toggle via Supervisor Password in Acer BIOS → disable Secure Boot
4. Upgrade host driver to 580.142 via `.run` installer to match Ubuntu container's libraries
5. Add cgroup device rules and bind mounts to LXC config
6. Install matching userspace libraries inside the container (no kernel modules)
7. Restart Ollama service so it detects the GPU at startup
8. Confirm with `nvidia-smi` inside the container and `nvidia-smi dmon` during inference
