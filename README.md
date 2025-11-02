# Unraid Backup & Recovery Plan

_Last revised:_ 2025-11-01  
_Server name:_ Gyarados

---

## System Overview

| Component | Details |
|------------|----------|
| **Array** | 4 drives total (3 data + 1 parity) |
| **Parity** | 1 drive (single parity protection) |
| **Pools** | - `cache_protected` → 500GB RAID1 (btrfs)  <br> - `cache_appdata` → 500GB RAID1 (btrfs) <br> - `cache` → 4TB XFS (no redundancy)|
| **Important Shares** | `appdata`, `system`, `backups` |
| **Unraid Flash Drive** | Backed up daily by CA Backup/Restore plugin |

---

## Backup Strategy

### Local / Onsite
| Backup Type | Tool | Frequency | Destination | Notes |
|--------------|------|------------|--------------|-------|
| **Appdata + Flash** | CA Backup/Restore | Daily | `/mnt/user/backups/unraid_backups` on array | Stored in “backups” share, included in offsite sync |
| **Array Data Backup** | `rsync` | Monthly | External 28TB USB drive | Plugged in manually, includes flash + appdata backups |
| **Parity Check** | Unraid checker | Monthly | N/A | Non-correcting, same day as backup |

### Offsite
| Backup Type | Tool | Frequency | Destination | Notes |
|--------------|------|------------|--------------|-------|
| **1st Critical Data Backup** | Duplicacy | Nightly | Backblaze B2 | Backs up `/mnt/user/backups` share (includes appdata + flash backups) |
| **2nd Critical Data Backup** | rclone | Weekly | Backblaze B2 | Backs up `/mnt/user/backups` share (includes appdata + flash backups) |
| **Offsite Verification** | Scripted Duplicacy restore | Monthly | Local `test_restore` folder | Verifies integrity with checksum comparison |

---

## Maintenance Schedule

| Task | Frequency | Tool / Command | Notes |
|------|------------|----------------|-------|
| **Parity Check (non-correcting)** | Monthly | Unraid Checker | Run before external backup |
| **External Backup (rsync)** | Monthly | 28TB External Userscript | Rotate drive offsite if possible |
| **btrfs Scrub (cache pools)** | Monthly | `/mnt/cache_protected` and `/mnt/cache_appdata` | Detects and repairs bitrot, just click the pool and then "scrub" |
| **Duplicacy Check** | Monthly | `duplicacy check -a -tabular` | Confirms remote chunks are valid |
| **File Integrity Verification** | Monthly | Offsite Verify Userscript | Grab random files, checksum them and compare vs. backup |
| **Recovery Test** | Every 90 days | Manual | Restore 3-5 files from each backup source |

---

## Recovery Procedures

### Array or Disk Failure
1. Confirm only one disk is disabled and parity is valid before proceeding with rebuild.
2. Identify the failed drive by serial number. Stop the array.  
3. Replace the failed drive with a new, precleared one and assign it to the same slot.  
4. Start the array — Unraid will begin rebuilding the disk from parity.    
5. Run a **non-correcting parity check** to confirm parity accuracy post-rebuild.

### Flash Drive Failure
1. Replace flash with new USB stick.  
2. Copy contents of latest flash backup (gyarados-flash-backup.zip) from `/mnt/user/backups/unraid_backups` or Backblaze.
3. Transfer license using Unraid’s automated license replacement system (Tools → Registration).

### Appdata Loss / Corruption
1. Stop the Docker service before restoring appdata.
2. Install “CA Backup/Restore Appdata” plugin (if not already).  
3. Select most recent backup under `/mnt/user/backups/unraid_backups/`.  
4. Click "Restore" at the top of the Appdata Restore plugin and follow instructions.  
5. Start the Docker service and verify containers.

### Full Array Loss / Catastrophic Event
1. Reinstall Unraid on new flash drive if needed.  
2. Recreate array and pools matching original layout from system overview.  
3. Restore data from:
   - External 28TB backup drive (via rsync)
   - Duplicacy or rclone (Backblaze) for critical data if External is damaged



