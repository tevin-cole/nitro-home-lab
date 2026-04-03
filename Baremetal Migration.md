**Proxmox VE: VMware to Baremetal Migration**

Acer Nitro 5 Homelab - Full Migration Guide & Incident Log

April 2026

# **1\. Overview**

This document covers the full migration of a Proxmox VE installation from a VMware Workstation-hosted VM to a baremetal install on an Acer Nitro 5 laptop. It includes pre-migration planning, disk management, configuration backup/restore procedures, problems encountered, and their resolutions.

## **1.1 Hardware Specs - Acer Nitro 5**

| **Component**  | **Details**                                                                        |
| -------------- | ---------------------------------------------------------------------------------- |
| CPU            | Intel Core i5/i7 (Acer Nitro 5)                                                    |
| GPU            | Intel Integrated (00:02.0) + Nvidia Discrete (01:00.0)                             |
| NVMe 1 (256GB) | Proxmox VE install target - was split: 128GB Windows To Go, 128GB Proxmox local    |
| NVMe 2 (512GB) | VM/LXC storage - fully passed through VMware to Proxmox                            |
| HDD (1TB)      | PBS datastore + LXC storage - passed through VMware to Proxmox                     |
| Network        | USB-A to Ethernet adapter (enx00e04c366b89) - temporary until screen/NIC available |

## **1.2 Services Running**

| **LXC ID** | **Hostname** | **Purpose**                    | **Primary Storage**         |
| ---------- | ------------ | ------------------------------ | --------------------------- |
| 100        | adguard      | DNS ad-blocking (AdGuard Home) | nvme512                     |
| 101        | -            | LXC service                    | nvme512                     |
| 102        | -            | LXC service                    | nvme512                     |
| 103        | -            | LXC service                    | nvme512                     |
| 105        | immich       | Photo management (Immich)      | tbhdd-full (rootfs + media) |
| 106        | -            | LXC service                    | nvme512                     |
| 107        | pbs          | Proxmox Backup Server          | nvme512 + tbhdd-full        |
| 108        | -            | LXC service                    | nvme512                     |
| 109        | -            | LXC service                    | nvme512                     |
| 110        | -            | LXC service                    | nvme512                     |

# **2\. Pre-Migration Preparation**

## **2.1 Backup Strategy**

Before touching anything, the following backups were verified:

- PBS backups confirmed current with NAS sync completed
- Manual vzdump run for Immich and any PBS-excluded LXCs to NAS
- Proxmox host config files archived to NAS
- PBS LXC config files archived to NAS

## **2.2 Backing Up Host Config Files**

Run the following from the Proxmox host shell to back up all critical configuration:

tar -cvpzf /mnt/pve/&lt;nas-storage-name&gt;/pve-backup-\$(hostname)-\$(date +%Y%m%d).tar.gz \\

/etc/pve \\

/etc/network/interfaces \\

/etc/fstab \\

/etc/modprobe.d/

Note: /etc/pve/lxc/ contains symlinks to /etc/pve/nodes/pve/lxc/ - the actual config files live in the nodes path. The tar command captures both correctly.

## **2.3 Backing Up PBS Config (LXC)**

Since PBS runs as an LXC, its config lives inside the container, not on the host. Extract it from inside the container:

pct exec &lt;pbs-lxc-id&gt; -- tar -cvpzf /tmp/pbs-config-backup.tar.gz /etc/proxmox-backup/

Then pull it to the NAS:

pct pull &lt;pbs-lxc-id&gt; /tmp/pbs-config-backup.tar.gz /mnt/pve/&lt;nas-storage&gt;/pbs-config-backup.tar.gz

⚠ Always create the destination directory first with mkdir -p before running pct pull or tar will throw 'No such file or directory'.

## **2.4 Pre-Migration Disk Reorganisation**

The 1TB HDD originally had two partitions: OldWindowsPartition (460GB) and ProxmoxPartition (470GB). The Windows data was backed up to NAS and the Windows partition was deleted to free up the full disk for Proxmox use.

⚠ Never use Windows Disk Manager to interact with disks that Proxmox owns. Even creating a partition entry without formatting (RAW/Basic Healthy Partition) can cause confusion. If this happens accidentally, verify the ext4 filesystem is still intact from within Proxmox before proceeding.

✓ Windows will show Proxmox ext4 partitions as RAW because it cannot read the filesystem. This does not mean data is lost.

# **3\. Baremetal Proxmox VE Installation**

## **3.1 BIOS Preparation**

Critical BIOS setting that must be changed before the Proxmox installer will detect NVMe drives:

⚠ Intel RST (Rapid Storage Technology) must be DISABLED in BIOS. With RST enabled, the Proxmox installer will only see drives presented in RST mode (typically the HDD) and will miss the NVMe drives entirely.

To disable: Enter BIOS → Advanced → Storage → SATA Mode → change from RST/RAID to AHCI.

## **3.2 Install Target Selection**

⚠ The Proxmox installer can silently target multiple disks. Always explicitly select only NVMe 1 (256GB) as the install target. Do not let the installer touch the 512GB NVMe or the 1TB HDD.

After installation, secondary drives are added manually as storage directories.

## **3.3 Wiping the Install Target NVMe**

If the target NVMe has existing partitions (e.g. old Windows To Go + Proxmox partitions), the installer may not list it. Wipe it first using one of these methods:

### **From Proxmox Debug Mode (Ctrl+Alt+F2 in installer):**

wipefs -a /dev/nvme0n1

sgdisk --zap-all /dev/nvme0n1

### **From a Mac using USB adapter (if debug shell is unavailable):**

diskutil list # identify the drive

diskutil eraseDisk FREE EMPTY /dev/diskN

✓ The Mac method is a reliable fallback if the Proxmox installer debug shell is not responding.

## **3.4 Network Configuration**

On baremetal, Proxmox creates a Linux bridge (vmbr0) and expects a physical NIC to be enslaved to it. WiFi adapters cannot be bridged in Linux - use a wired connection.

### **Temporary USB Ethernet Setup:**

With a USB-A to Ethernet adapter, the interface will appear as enx&lt;mac-address&gt;. Edit /etc/network/interfaces to bridge it correctly:

auto lo

iface lo inet loopback

iface wlp9s0 inet manual

iface enx00e04c366b89 inet manual

auto vmbr0

iface vmbr0 inet static

address 192.168.100.217/24

gateway 192.168.100.1

bridge-ports enx00e04c366b89

bridge-stp off

bridge-fd 0

Apply changes with:

systemctl restart networking

⚠ Do not use bridge-ports wlp9s0 - WiFi cannot be bridged. The bridge interface column will show blank in 'brctl show' if the wrong interface is used.

# **4\. Storage Configuration**

## **4.1 Filesystem Choice**

Use ext4 for the 1TB HDD. Reasons: PBS is designed and tested against ext4, LXC bind mounts work seamlessly, and fsck/recovery tooling is superior for spinning disk workloads. XFS has no meaningful advantage at HDD speeds for this use case.

## **4.2 Mounting Secondary Drives**

After baremetal install, the secondary drives will not be mounted. Identify them first:

lsblk -f

Create mount points and mount:

mkdir -p /mnt/pve/nvme512

mkdir -p /mnt/pve/tbhdd-full

mount /dev/nvme0n1p1 /mnt/pve/nvme512

mount /dev/sda1 /mnt/pve/tbhdd-full

Add to /etc/fstab using UUIDs (get UUIDs from lsblk -f output):

echo "UUID=&lt;nvme-uuid&gt; /mnt/pve/nvme512 ext4 defaults 0 2" >> /etc/fstab

echo "UUID=&lt;hdd-uuid&gt; /mnt/pve/tbhdd-full ext4 defaults 0 2" >> /etc/fstab

## **4.3 Restoring storage.cfg**

The backed-up storage.cfg can be copied directly to restore all storage definitions including PBS and NFS in one step:

cp /path/to/backup/storage.cfg /etc/pve/storage.cfg

Verify all storage is active:

pvesm status

✓ PBS storage will show inactive until the PBS LXC is restored and running. This is expected.

## **4.4 Moving LXC Volumes Between Storage**

If a volume needs to be moved between storage directories, use the CLI - the GUI can fail with a read-only loop device error:

pct move-volume &lt;lxc-id&gt; rootfs --target-storage &lt;storage-id&gt;

⚠ Always shut down the LXC before moving volumes.

# **5\. LXC and VM Restoration**

## **5.1 Why Fresh Install + Config Migration is Better Than Host Restore**

A Proxmox VE host backup captures the VM's virtual hardware layer and expects VMware's virtual devices on restore. Network interfaces, storage paths, and device IDs will all differ on baremetal. A fresh install with manual config migration is faster and more reliable.

## **5.2 Restoring LXC Config Files**

LXC configs live at /etc/pve/nodes/pve/lxc/ (with symlinks at /etc/pve/lxc/). Extract them from the backup tar:

tar -xOf /path/to/pve-backup.tar.gz --wildcards "etc/pve/nodes/pve/lxc/\*.conf" -C /

If disk images are already present on the storage drives from the previous setup, Proxmox will recognise them immediately and LXCs will appear in the UI without needing a full restore from PBS.

## **5.3 Verifying Disk Image Locations**

Before starting any LXC, verify the disk images referenced in its config actually exist:

ls /mnt/pve/nvme512/images/&lt;lxc-id&gt;/

ls /mnt/pve/tbhdd-full/images/&lt;lxc-id&gt;/

When multiple disk images exist for the same LXC (e.g. disk-0 and disk-1 for the same role), check timestamps to identify the most recent:

ls -lh /mnt/pve/tbhdd-full/images/&lt;lxc-id&gt;/

## **5.4 Updating LXC Config After Proxmox Version Change**

After a fresh install, Proxmox may be a different version than the previous setup. Community Scripts-generated LXC configs contain HTML comment headers that can cause parse issues. Additionally, newer Proxmox hook scripts may not recognise older Debian versions.

Fix: Update Proxmox immediately after install before starting any containers:

apt update && apt dist-upgrade -y

reboot

# **6\. Proxmox Backup Server Restoration**

## **6.1 Restoring PBS Config into LXC**

After starting the PBS LXC, push the backed-up config into it and extract:

pct push 107 /mnt/pve/unraid-nfs/pbs-config-backup.tar.gz /tmp/pbs-config-backup.tar.gz

pct exec 107 -- tar -xvf /tmp/pbs-config-backup.tar.gz -C /

pct exec 107 -- systemctl restart proxmox-backup

## **6.2 Reconnecting PBS to PVE (401 Unauthorized Fix)**

After restoring PBS config files, the API credentials stored in PVE's storage.cfg will be invalid because the PBS instance was recreated. A new token must be generated and the fingerprint updated.

Steps:

- In PBS UI: Configuration → Access Control → API Tokens → create new token for proxmox-backup@pbs
- Copy the token secret (shown only once)
- Get new fingerprint from PBS LXC:

pct exec 107 -- proxmox-backup-manager cert info | grep Fingerprint

- In PVE: Datacenter → Storage → proxmox → Edit → update username (proxmox-backup@pbs!tokenname), password (token secret), and fingerprint

# **7\. Issues Encountered & Resolutions**

## **Issue 1: Proxmox Installer Only Sees HDD, Not NVMe Drives**

Symptom: Only the 1TB HDD appeared in the installer's target selection. NVMe drives were not listed.

Cause: Intel RST (Rapid Storage Technology) was enabled in BIOS, presenting NVMe drives differently to the OS.

Resolution: Disable Intel RST in BIOS → Advanced → Storage → SATA Mode → AHCI. Reboot and re-run installer.

## **Issue 2: Proxmox Debug Shell Unresponsive**

Symptom: Ctrl+Alt+F2 in the Proxmox installer opened a terminal but no commands were accepted.

Resolution: Wipe the NVMe externally using a Mac with a USB-to-M.2 NVMe adapter:

diskutil eraseDisk FREE EMPTY /dev/diskN

## **Issue 3: vmbr0 Has No Bridge Interface (No Network)**

Symptom: Could not reach Proxmox web UI. brctl show showed vmbr0 with blank interfaces column.

Cause: /etc/network/interfaces had bridge-ports wlp9s0 (WiFi) which cannot be bridged in Linux.

Resolution: Edit /etc/network/interfaces and change bridge-ports to the USB ethernet interface (enx00e04c366b89). Restart networking.

## **Issue 4: LXC Fails to Start - lxc.hook.pre-start Error Status 25**

Symptom: All LXCs failed with 'Failed to run lxc.hook.pre-start for container' and exit status 25.

Root cause (from lxc-start debug log):

Script exec /usr/share/lxc/hooks/lxc-pve-prestart-hook produced output: unsupported debian version '13.1'

Cause: The fresh Proxmox install had a hook script that did not recognise Debian 13 (Trixie), which the LXCs were running.

Resolution: Update Proxmox fully before starting any containers:

apt update && apt dist-upgrade -y && reboot

## **Issue 5: LXC Volume Move Fails in GUI**

Symptom: Moving an LXC volume via the Proxmox GUI resulted in 'cannot mount /dev/loopX read-only' error.

Resolution: Use the CLI instead:

pct move-volume &lt;lxc-id&gt; rootfs --target-storage &lt;storage-id&gt;

## **Issue 6: Immich LXC GPU Passthrough Fails - /dev/dri/card0 Does Not Exist**

Symptom: Immich LXC (105) failed to start with 'Device /dev/dri/card0 does not exist'.

Cause: Under VMware, the GPU was presented as card0. On baremetal with both Intel and Nvidia GPUs present, device numbering changed.

Resolution: Identify the correct device:

ls -la /dev/dri/by-path/

Intel integrated GPU is always at PCI address 00:02.0, which mapped to card2 and renderD129 on this system. Update the LXC config accordingly (replacing card0 with card2 and renderD128 with renderD129).

## **Issue 7: PBS Shows 401 Unauthorized in PVE After Restore**

Symptom: PBS storage entry in PVE showed 'error fetching datastores - 401 Unauthorized'.

Cause: Restoring PBS config files does not restore API tokens or TLS certificates. The fingerprint and credentials stored in PVE's storage.cfg are no longer valid after a PBS reinstall.

Resolution: Generate a new API token in PBS, get the new TLS fingerprint, and update the PBS storage entry in PVE with both.

## **Issue 8: Windows Disk Manager Wrote Partition Entry to Proxmox-Owned HDD**

Symptom: After passing the 1TB HDD through to Proxmox and formatting it as ext4, Windows Disk Manager was accidentally opened and a 'New Volume' was created on the unallocated space. The disk showed as 'Basic Healthy Partition' with type RAW.

Cause: Windows Disk Manager wrote a partition table entry but did not format the partition, leaving the underlying ext4 filesystem intact.

Resolution: Verify from within Proxmox that the disk is still mounted and data is accessible. Do not interact with Proxmox-owned disks via Windows Disk Manager again.

⚠ Windows Disk Manager should never be used to view or modify disks owned by Proxmox, even passively. Any interaction risks corrupting the partition table.

# **8\. Quick Reference**

## **8.1 Key IP Addresses**

| **Service**                     | **IP Address**  | **Port** |
| ------------------------------- | --------------- | -------- |
| Proxmox VE (baremetal)          | 192.168.100.x | 8006     |
| Proxmox Backup Server (LXC 107) | 192.168.100.x | 8007     |
| Home Assistant VM               | 192.168.100.x | 8123     |
| Unraid NAS                      | 192.168.100.x | -        |
| AdGuard (LXC 100)               | 192.168.100.x | -        |

## **8.2 Storage Layout**

| **Storage ID** | **Path**            | **Device**                | **Used For**                 |
| -------------- | ------------------- | ------------------------- | ---------------------------- |
| local          | /var/lib/vz         | nvme1n1 (Proxmox install) | ISOs, templates, snippets    |
| nvme512        | /mnt/pve/nvme512    | nvme0n1p1                 | VM/LXC disk images           |
| tbhdd-full     | /mnt/pve/tbhdd-full | sda1                      | PBS datastore + Immich media |
| unraid-nfs     | /mnt/pve/unraid-nfs | NFS 192.168.100.x       | NAS backups, vzdumps         |

## **8.3 Essential Commands**

| **Task**              | **Command**                                                          |
| --------------------- | -------------------------------------------------------------------- |
| Check storage status  | pvesm status                                                         |
| Check disk layout     | lsblk -f                                                             |
| Check bridge config   | brctl show                                                           |
| Start LXC with debug  | lxc-start -n &lt;id&gt; -l DEBUG -o /tmp/lxc-debug.log               |
| Move LXC volume (CLI) | pct move-volume &lt;id&gt; rootfs --target-storage &lt;storage&gt;   |
| Push file into LXC    | pct push &lt;id&gt; /host/path /container/path                       |
| Pull file from LXC    | pct pull &lt;id&gt; /container/path /host/path                       |
| Exec command in LXC   | pct exec &lt;id&gt; -- &lt;command&gt;                               |
| Get PBS fingerprint   | pct exec 107 -- proxmox-backup-manager cert info \| grep Fingerprint |
| Restart pveproxy      | systemctl restart pveproxy                                           |
| Apply network changes | systemctl restart networking                                         |

## **8.4 Post-Migration Checklist**

- All LXCs and VMs start successfully
- PBS LXC (107) is running and datastores are visible
- PBS is connected to PVE (no 401 errors in Datacenter → Storage)
- PBS sync job to NAS is re-enabled and tested
- Immich LXC (105) GPU passthrough working (card2/renderD129)
- NFS mount to Unraid NAS is active
- AdGuard DNS is resolving correctly
- Reverse proxy (pve.internal.tevincole.com) resolving to 192.168.0.x
- fstab verified - all mounts survive a reboot
- Replace USB ethernet adapter with permanent solution when screen/NIC arrives

# **9\. Outstanding Items**

The following items were identified during migration but not yet resolved:

| **Item**                                          | **Priority** | **Notes**                                                                                                        |
| ------------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------- |
| Replace USB ethernet adapter with permanent NIC   | High         | USB adapters drop intermittently under load - pveproxy may appear to crash when it is actually a network dropout |
| Investigate intermittent pveproxy/network dropout | High         | Run: journalctl -f \| grep enx00e04c366b89 during a dropout to confirm if it is adapter or service               |
| Re-enable PBS NAS sync job                        | High         | Verify sync job config survived PBS restore and re-enable                                                        |
| Update SMLIGHT SLZB-06M coordinator firmware      | Medium       | Radio FW 20250220 has known ACK timeout lockup issues                                                            |
| Enable z2m watchdog in Home Assistant addon       | Medium       | Currently disabled - logs show 'Starting Zigbee2MQTT without watchdog'                                           |
| Enable persistent logging in Home Assistant       | Medium       | Required to capture events across restarts                                                                       |
| Add reconnect_period to z2m configuration.yaml    | Low          | Improves self-healing on coordinator disconnect                                                                  |
