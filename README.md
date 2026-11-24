# wazuh-lab

Home SOC lab: Wazuh SIEM monitoring Linux and Windows endpoints on an isolated virtual network, with simulated attacks and incident reports.

## Overview

This is a home security operations lab I built to practice the Tier 1 SOC workflow: get logs flowing, run attacks, catch them, and write them up.

Everything runs on a MacBook Pro (Apple Silicon) in UTM virtual machines on an isolated shared network. A Wazuh 4.14 server (indexer, manager and dashboard on one Ubuntu Server VM) collects logs from two endpoints: an Ubuntu Server machine standing in for a school's Linux server, and a Windows 11 Pro workstation. Each endpoint runs a Wazuh agent and reports back to the server.

## Lab layout

| Machine | Role | IP |
|---|---|---|
| `wazuh` | SIEM server (indexer, manager, dashboard) | 192.168.64.100 |
| `ubuntu-endpoint` | Monitored Linux server | 192.168.64.101 |
| `win11-endpoint` | Monitored Windows workstation | 192.168.64.102 |

## Status

- [x] Wazuh server deployed
- [x] Ubuntu endpoint reporting
- [x] Windows 11 endpoint reporting
- [ ] Kali attacker VM
- [ ] Attack 1: SSH brute force
- [ ] Attack 2: Backdoor admin account
- [ ] Attack 3: Tampering with fake student records
- [ ] Incident reports mapped to MITRE ATT&CK

## Wazuh dashboard

<img width="3838" height="2102" alt="Wazuh dashboard showing both agents active" src="https://github.com/user-attachments/assets/0f78fad6-eda5-4d3e-b68c-82d0f77a16ee" />

## Setup and issues

Build notes from November 23, 2026.

### Setup

- Mac (Apple Silicon), UTM, Shared Network (`192.168.64.0/24`, gateway `.1` = the Mac)
- Wazuh server VM: 6 CPU, 12 GB RAM, 64 GB disk, Ubuntu Server (ended up on 26.10, the minimized image, by accident)
- Wazuh 4.14.10, all-in-one install (`wazuh-install.sh -a`)
- Static IPs: server `.100`, ubuntu-endpoint `.101`, win11-endpoint `.102`. Kept them high because the Mac's DHCP hands out addresses starting at `.2`.

### Problem 1: Couldn't copy and paste into the VM

- Ubuntu Server has no desktop, so the UTM clipboard doesn't work
- **Fix:** SSH in from Mac Terminal and do everything from there

### Problem 2: Netplan file had a different name

- Guide said `50-cloud-init.yaml`, my VM had `00-installer-config.yaml`
- **Fix:** check `ls /etc/netplan` and edit whatever's there
- Opening `nano` on the wrong name created a second, empty `50-cloud-init.yaml`. Netplan reads every file and the higher number wins, so it overrode my config. Deleted it.

### Problem 3: `netplan try` rolled back my static IP

- Ran it over SSH. The IP changed, SSH dropped, I couldn't press Enter to confirm, so it reverted after 120 seconds
- **Fix:** run `sudo netplan apply` in the UTM console, then SSH to the new IP

### Problem 4: First install failed with "error while trying to get the API token"

- Installer retried for 12 minutes, then removed everything
- Dashboard log (`journalctl -u wazuh-dashboard`) showed `ENOSPC: no space left on device`
- Ubuntu's LVM setup only gave root 30 GB of the 64 GB disk
- **Fix:** grew root to 60 GB:

```bash
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

- **Lesson:** the real error was in the dashboard's log, not the installer output

### Problem 5: Second install failed on ports 1515 and 55000 already in use

- Leftover manager processes from the first attempt were still running even though the package was removed
- **Fix:** reboot (the service was disabled, so it didn't come back)

### Problem 6: Third install failed with "wazuh-manager could not be started"

- I'd deleted `/var/ossec` by hand, but dpkg still thought the manager was installed, so the installer skipped it
- Normal purge failed (exit 127) because the uninstall script called files I'd deleted
- **Fix:** replaced the broken uninstall scripts with `exit 0`, then purged:

```bash
echo 'exit 0' | sudo tee /var/lib/dpkg/info/wazuh-manager.prerm
echo 'exit 0' | sudo tee /var/lib/dpkg/info/wazuh-manager.postrm
sudo dpkg --purge wazuh-manager
```

- **Lesson:** don't delete a package's files before removing the package itself

**The fourth install worked.** Logged in to the dashboard at `https://192.168.64.100`.

### Problem 7: Dashboard gave "AxiosError 500" after rebooting to clone the VM

- Manager service failed: `wazuh-apid did not start correctly`
- Stale PID files from before the shutdown pointed at a process number another program had taken
- **Fix:**

```bash
sudo rm -f /var/ossec/var/run/*.pid
sudo systemctl restart wazuh-manager
```

- **Lesson:** always shut down with `sudo poweroff` before cloning

### Endpoints

- **ubuntu-endpoint:** static `.101`, agent deployed from the dashboard (Linux, DEB, aarch64), showing Active
- **win11-endpoint:** Windows 11 Pro ARM, static `.102`, agent deployed (Windows MSI), showing Active

### Backups

Each VM has a clean clone, made right after it was working:

| VM | Backup |
|---|---|
| `Wazuh` | `wazuh-clean` |
| `ubuntu-endpoint` | `ubuntu-endpoint-clean` |
| `win11-endpoint` | `win11-endpoint-clean` |

- **Never run a clone at the same time as its original.** A clone is an exact copy with the same static IP and, for the endpoints, the same Wazuh agent ID and key. Running both causes IP conflicts, and Wazuh sees two machines claiming to be the same agent.
- **Day to day:** run only `Wazuh`, `ubuntu-endpoint` and `win11-endpoint`.
- **If one breaks:** delete the broken VM, clone its `-clean` copy, and give the new copy the original name. Leave the `-clean` VM untouched so there's always a backup.
- **Shut down cleanly before cloning** (`sudo poweroff` on Linux, Shut down on Windows). Start the Wazuh server first, wait 2 to 3 minutes, then the endpoints.

### For next time

- Use Ubuntu 24.04 LTS, the standard install, and give root the full disk during install
- Back up (clone) after every working step
