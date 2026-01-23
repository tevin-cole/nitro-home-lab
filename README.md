# Welcome to my Nitro Home Server Project

For a long thing I've wanted to repurpose an old Nitro 5 9th generation laptop that I no longer use due to getting a more specialized laptop (Macbook Air M1) for my needs and this year (2025) I decided to make the leap into exploring homelabbing so that I can become more functional and self-sufficent on the web (everything is turning into a subscription). This repo will document my journey for my own sake (just in case something breaks or I have to resetup any aspect of the homeserver) and so that it may be able to help someone else out there who is just like me but has no idea on how or where to get started.

### Considerations to bare in mind

 1. My nitro laptop has a broken screen that happened sometime when I was travelling internationally in my brief case so I had to come up with a solution that could work around that limitation and most importantly cheap (I'm very frugal)!
 2. I ultimately wanted to have my own NAS / deploy my own gallery website (because I am a photographer who consume massive amounts of data as the years go by aka. a Data Horder)

# Initial decisions I made
After doing some research, I learned of a software called proxmox which would act as the OS of my nitro 5 (who ill formally call Nitro going forward) to leverage the resources of my laptop.

# Proxmox Installation
Despite everything that I saw online about how I should run proxmox, the best solution is definitely bare-metal but I was not about to do that I was not able to navigate through the Proxmox installation without a functioning screen. I tried everything, from 

 1. Installing proxmox to my bootdrive from another laptop (did not work)
 2. Tried to install proxmox headless (since I had no screen) (could not make it through the install gui or terminal)
 3. Install proxmox through a USB but selecting the boot drive (could not make it through the install gui or terminal)
 
 I was frustrated, dejected, and ready to give up, this seemed like a task too big for me to overcome myself. I was just ready to bite the bullet and order a new Nitro screen from eBay and replace it myself when I decided to just try something for the fun of it and you would not believe me when I say that it actually worked!

## What actually worked for me
Installing Proxmox on a virtual machine (VMWARE Workstation 17 Pro) and allocating it as much resources as I could since the laptop was running Windows10 on nvme0 (boot drive) and then logging in from M1 and doing everything else from there (yipee!!)

I was so frustrated when it actually worked because everything I looked online spoke about potential issues of not having proxmox run on bare-metal or the limitations of device passthrough through a virtual machine but this taught me a really big lesson which is that **everyone's needs are not my needs**. And so I should take everything I learn on the internet with a grain of salt and just try it. Had I just done that from the beginning, I would not have almost given up on this hobby/dream of repurposing this laptop to now serve me!

#### So now we are off to actually realizing this dream

# My first proxmox LXC installation - Adguard Home
If its one thing you should know about me is that I am not a big fan of the state of the internet currently. Misinformation, AI Slop, decreasing attention spans, you name it, I don't even enjoy my time on social media anymore because I feel like I am always being pressured into thinking or feeling a kind of way and that is no more evident than with the amount of garbage ads there are EVERYTHING on the internet. 

Now I understand that ads are a necessary way of paying it forward for the content I consume but not the absolute cesspit that has plagued social media and the internet as of recently.

# Huawei HG8145X6-12 DNS Configuration for Adguard Home 
I am mainly making this tutorial because I basically trialed and errored my way into the right configuration but it took me alot longer that it should have (over 3 hours) and I want to make it easier for anyone who has the same modem/router model to figure it out quicker.

#### You're really only concerned with 2 tabs when you login to your router's admin panel which is:

 1. LAN -> DHCP Server Configuration - Change your Primary DNS Server to point to your Adguard Home IPv4 Address

![enter image description here](https://i.postimg.cc/8kh1jpMN/Screenshot-2025-10-10-at-12-29-49-AM.png)

 2. IPv6 -> DHCP LAN Address Configuration - Change DNS Information Setting, DNS Source on the LAN Side to **Static Configuration from DNS Agent** and fill in the Preferred DNS as your Adguard Home IPv6 Address

![enter image description here](https://i.postimg.cc/mZQbtLCx/Screenshot-2025-10-10-at-12-29-34-AM.png)

# Homelab Infrastructure Update: LenovoNAS Addition

## Project Overview
Expanded homelab infrastructure with a dedicated NAS to centralize storage, backups, and media management. This build focused on starting small with room for future expansion while maintaining budget constraints.

---

## Hardware Specifications

### LenovoNAS (m700s)
- **Model:** Lenovo ThinkCentre m700s SFF
- **CPU:** Intel i7-7700 (7th gen, 4c/8t, 3.6GHz base)
- **RAM:** 16GB DDR4-2400 (upgraded from 8GB stock)
  - Added: Crucial CT8G4DFS824A 8GB module
- **Storage Array:** 2x 4TB HDDs in UnRAID parity configuration
- **Network:** 1GbE via desktop switch to router
- **UPS:** APC BVX700LU-LM 700VA with USB NUT monitoring
- **Power:** 210W max (PSU limited)
- **Cost:** $1100 TTD (~$200 USD) + ~$150 USD for drives

### Storage Configuration
- **OS:** UnRAID (trial → licensed)
- **Array Structure:** 
  - Parity: 1x 4TB (single parity protection)
  - Data: 1x 4TB (usable capacity)
- **Usable Space:** ~4TB initially
- **Filesystem:** XFS on data disks
- **Expansion Path:** Can add drives incrementally, mix sizes
- **Legacy Drive:** 500GB HDD repurposed as cache (optional)

---

## Infrastructure Role & Integration

### Primary Functions
1. **Centralized Backup Target**
   - Immich photo backups (~250GB currently)
   - Proxmox VE VM/LXC backups (~26GB)
   - Photography/videography archive storage (~1.5TB)

2. **Media Server** (Future-ready)
   - Jellyfin for media playback
   - Sonarr/Radarr for media management
   - qBittorrent for public domain content acquisition

3. **Future Expansion**
   - Planned: Reolink security camera NVR backups
   - Scalable to 12 drives with HBA card addition

### Network Architecture
```
Internet
    │
  Router
    │
1GbE Switch
    ├── Acer Nitro 5 (Proxmox VE)
    ├── LenovoNAS (UnRAID)
    └── MacBook (client access)
```

**Access Methods:**
- **SMB Shares:** Mac clients (Finder integration)
- **NFS Shares:** Proxmox VE backups (mounted at `/mnt/unraid-backups`)
- **Remote Access:** Tailscale VPN for external access

---

## Build Journey & Key Decisions

### Storage Strategy Evolution

**Initial Constraint:** $150 USD budget for drives

**Considered Options:**
1. Traditional RAID1 mirroring
2. UnRAID parity array
3. TrueNAS Scale with ZFS

**Decision: UnRAID Parity Array**
- **Why:** Mix-and-match drive sizes for budget flexibility
- **Why:** Single-drive incremental expansion
- **Why:** Easy to start small (2 drives) and grow organically
- **Trade-off:** Slower parity writes vs RAID performance (acceptable for backup use case)

**Phased Expansion Plan:**
- **Phase 1 (Current):** 2x 4TB (1 parity + 1 data) = 4TB usable
- **Phase 2 (Future):** Add 1x 4TB data = 8TB usable total
- **Phase 3 (Long-term):** HBA card (~$50) + additional drives = 12 drive max capacity

### UPS Selection & Power Protection

**Requirements:**
- USB communication for UnRAID NUT (Network UPS Tools)
- 5-10 minutes runtime for graceful shutdown
- No proprietary software dependency

**Evaluated Models:**
- Forza HT-1000D / SBNB1200DR: Uncertain NUT compatibility
- Tripp Lite INTERNET750U/600U/AVR700U: Confirmed NUT support
- APC BVX700LU-LM: Initially recommended but lacked data port

**Critical Discovery:** APC BVX700LU-LM's USB port is charging-only, not data communication

**Final Choice:** FORZA Smart UPS 800VA/480W - Due to local availability and prohibitive cost of shipping one in

---

## Technical Implementation Details

### UnRAID Array Configuration

**Array vs Pool Decision:**
- **Chose:** Array with parity (not cache pool RAID1)
- **Reasoning:** 
  - Aligns with backup/archival use case
  - Preserves future drive size flexibility
  - UnRAID's core feature set optimized for this

**Setup Process:**
```
Parity:  4TB Drive A (WDC or Seagate)
Disk 1:  4TB Drive B (formatted XFS)
Cache:   500GB (optional, for Docker/VM acceleration)
```

**Parity Build:** 4-8 hours initial sync for 4TB drives

### Network Share Configuration

#### SMB Shares (Mac Access)
**Use Case:** General file access, Immich backups

**Configuration:**
- Protocol: SMB (Samba)
- Authentication: User-based (non-root user created)
- Permissions: Read/Write for authorized users
- Connection: `smb://[NAS-IP]/[share-name]` from Finder

#### NFS Shares (Proxmox Backups)
**Use Case:** Proxmox VE VM/LXC backup target

**UnRAID Export Settings:**
```bash
# Share: proxmox-backups
# NFS Security rule:
192.168.1.X(rw,sync,no_subtree_check,insecure,all_squash)
```

**Proxmox Mount:**
```bash
# Mount point
mkdir -p /mnt/unraid-backups

# Permanent mount in /etc/fstab
[NAS-IP]:/mnt/user/proxmox-backups /mnt/unraid-backups nfs defaults,_netdev 0 0
```

**Proxmox VE Storage Configuration:**
- Type: Directory
- Path: `/mnt/unraid-backups`
- Content: VZDump backup files
- Access: All nodes in cluster

### Media Stack Implementation

**Installed Services (Docker):**
- **Jellyfin:** Media server/playback
- **Radarr:** Movie management automation
- **Sonarr:** TV show management automation (planned)
- **qBittorrent:** Torrent client for public domain content

#### Docker Path Mapping Challenge

**Problem:** Path inconsistency between containers causing import failures

**Error Encountered:**
```
Radarr: "No files found eligible for import in /downloads/movies/..."
```

**Root Cause:** 
- qBittorrent saving to `/data/torrents/movies/`
- Radarr looking in `/downloads/movies/`
- Containers had different internal path mappings

**Solution: Standardized `/data` Structure**

**UnRAID Host Paths:**
```
/mnt/user/data/
├── torrents/
│   ├── movies/      (qBittorrent downloads)
│   └── tv/          (qBittorrent downloads)
└── media/
    ├── movies/      (Radarr organized library)
    └── tv/          (Sonarr organized library)
```

**All Containers Mapped Consistently:**
```
Container Path: /data
Host Path:      /mnt/user/data/
```

**qBittorrent Configuration:**
- Default Save Path: `/data/torrents/movies`
- Category-based routing: `movies` → `/data/torrents/movies/`

**Radarr Configuration:**
- Download Client Category: `movies`
- Root Folder: `/data/media/movies/`
- Remote Path Mapping: Not needed (paths now consistent)

**Jellyfin Library Paths:**
- Movies: `/data/media/movies/`
- TV Shows: `/data/media/tv/`

**Result:** Automated workflow with hardlinks (zero-copy) from torrents to media library

---

## Current System Status

### Active Services
- ✅ UnRAID array with single parity protection
- ✅ SMB shares accessible from Mac clients
- ✅ NFS share mounted on Proxmox VE
- ✅ UPS monitoring via NUT (graceful shutdown configured)
- ✅ Jellyfin media server operational
- ✅ Radarr/Sonarr/qBittorrent media automation stack
- ✅ Immich backup target configured

### Storage Utilization
- **Total Usable:** ~4TB
- **Current Usage:** ~1.75TB (photos/videos migrated)
- **Free Space:** ~2.25TB
- **Protection:** Single parity (1 drive failure tolerance)

### Backup Strategy
- **Primary:** Proxmox → UnRAID NFS (automated)
- **Primary:** Immich → UnRAID SMB (automated)
- **Future:** 3-2-1 backup with off-site component planned

---

## Lessons Learned

### Hardware Acquisition
- **Avoid scams:** Multiple false listings encountered in used market
- **Be flexible:** Ended up with better CPU (i7-7700) than originally targeted
- **Verify specs:** Check PCIe slots, RAM limits, drive capacity before purchase
- **Business systems:** Lenovo/Dell business desktops excellent for homelab repurposing

### Storage Architecture
- **Start simple:** Don't overcomplicate initial setup
- **Plan for growth:** UnRAID's flexibility justifies the learning curve
- **Single parity sufficient:** For <6 drives and non-critical data
- **XFS over BTRFS:** Simpler, more stable for array disks

### Docker & Path Management
- **Consistency is key:** All containers must see identical paths
- **Use `/data` standard:** Community best practice for *arr stack
- **Hardlinks save space:** Keep torrents seeding + organized library with zero duplication
- **Test one service first:** Get qBittorrent → Radarr → Jellyfin working before adding complexity

### Power & Reliability
- **UPS is essential:** NAS runs 24/7, dirty power kills drives
- **NUT integration:** Verify USB data port capability before purchase
- **Graceful shutdown:** 2-3 minutes runtime sufficient for safe array spindown

---

## Future Roadmap

### Short-term (3-6 months)
- [ ] Add 3rd 4TB drive (expand to 8TB usable)
- [ ] Implement Sonarr for TV show automation
- [ ] Configure Immich automated backups
- [ ] Set up regular Proxmox VM backup schedule

### Medium-term (6-12 months)
- [ ] Add HBA card (LSI 9211-8i) for >4 drive expansion
- [ ] Install 4th & 5th drives (12TB+ usable)
- [ ] Deploy Reolink camera system with NVR backup to NAS
- [ ] Implement off-site backup (3-2-1 strategy completion)

### Long-term (1+ years)
- [ ] Upgrade parity drive to 8TB+ (increase max data drive size)
- [ ] Consider dual parity for increased protection
- [ ] Evaluate 10GbE networking for media streaming performance
- [ ] Explore TrueNAS Scale migration if needs evolve

---

## Resource Links

### Documentation Referenced
- [UnRAID Documentation](https://docs.unraid.net/)
- [TRaSH Guides](https://trash-guides.info/) - Docker path mapping
- [ServeTheHome](https://www.servethehome.com/) - Hardware reviews
- [/r/unraid](https://reddit.com/r/unraid) - Community support

### Hardware Compatibility
- Lenovo m700s: Well-supported in Linux, excellent homelab platform
- APC/Tripp Lite UPS: NUT compatibility database
- WD Red / Seagate IronWolf: CMR drives recommended for NAS

### Software Stack
- UnRAID: OS/storage management
- Jellyfin: Media server
- Radarr/Sonarr: Media automation
- qBittorrent: Torrent client
- Tailscale: Zero-config VPN

---

## Conclusion

The LenovoNAS successfully fills the centralized storage and backup role in the homelab infrastructure. Key success factors:

1. **Budget-conscious hardware selection** with room for expansion
2. **UnRAID's flexibility** for incremental growth and mixed drive sizes
3. **Proper integration** with existing Proxmox infrastructure via NFS
4. **Automated media management** with standardized Docker path mapping
5. **Power protection** ensuring data integrity during outages

This foundation supports current backup needs (~2TB) while providing clear expansion path to 12+ drives and 40TB+ capacity as requirements grow.

**Total Investment:** ~$375 USD (system + drives + RAM + UPS)
**Value Delivered:** Centralized backup, media server, future camera storage, learning platform

The NAS operates as a complement to the Proxmox VE server, maintaining separation of concerns (compute vs storage) while enabling robust backup workflows and media serving capabilities.


*Last updated 23/01/2026
