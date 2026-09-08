# DevGuru Lab Setup — Ethical Hacking 

## Introduction to Ethical Hacking

Ethical hacking is the practice of legally and intentionally probing systems, networks, and applications for security weaknesses, using the same tools and techniques as malicious attackers, but with permission and the goal of improving security rather than causing harm. It's a core skill in cybersecurity, used for penetration testing, vulnerability assessment, and security research.

As part of learning these fundamentals, I set up a controlled lab environment using **VirtualBox**, with **Kali Linux** as the attack machine and **DevGuru** (a vulnerable machine from [VulnHub](https://www.vulnhub.com/)) as the target. This repository documents that setup process, the network configuration used to connect the two machines, and the real-world troubleshooting challenges I ran into along the way.

---

## Lab Environment

| Component | Role |
|---|---|
| Kali Linux | Attack machine |
| DevGuru (VulnHub) | Target / vulnerable machine |
| VirtualBox | Hypervisor |
| NAT Network ("LabNetwork") | Isolated internal network connecting both VMs |

---

## Installing DevGuru

DevGuru was downloaded as an `.ova` file from VulnHub and imported into VirtualBox via **File → Import Appliance**.

### Challenge 1: Import Failure — `VERR_ALREADY_EXISTS`

On my first import attempt, VirtualBox threw the following error:

```
Could not create the imported medium 'devguru-disk001.vdi'
VDI: cannot create image (VERR_ALREADY_EXISTS)
```

**Cause:** A leftover `.vdi` file from a previous failed import still existed on disk, even after the VM itself had been removed from VirtualBox Manager. VirtualBox checks the actual filesystem before writing a new disk image, so the stale file blocked the new import.

**Fix:**
- Located the leftover file directly in File Explorer at `C:\Users\<user>\VirtualBox VMs\devguru\`
- Deleted the folder/file manually
- Re-ran the import, which then completed successfully

---

## Network Configuration

For Kali to be able to reach DevGuru, both VMs needed to be attached to the same isolated network.

### Steps taken:
1. Created a custom **NAT Network** named `LabNetwork` via **File → Tools → Network Manager**
2. Set both DevGuru's and Kali's **Adapter 1** settings to:
   - Attached to: `NAT Network`
   - Name: `LabNetwork`

### Challenge 2: "LabNetwork" not appearing in the Name dropdown

Initially, the `LabNetwork` option wasn't available to select. This was because the NAT Network hadn't actually been created yet — it needs to exist in VirtualBox's Network Manager before it can be assigned to a VM's adapter.

---

## Boot Issue: Kernel Panic on DevGuru

### Challenge 3: `rcu_sched` stalls / kernel hang on boot

After powering on DevGuru, the boot process hung indefinitely at "Loading essential drivers," eventually producing kernel errors:

```
INFO: rcu_sched detected stalls on CPUs/tasks
rcu_sched kthread starved for 117410 jiffies
```

**Cause:** This is a known compatibility issue between older Linux kernels (common in older VulnHub VMs) and the CPU scheduling behavior of modern host hardware/VirtualBox versions.

**Fix:**
- Powered off the VM
- Went to **Settings → System → Acceleration**
- Changed the **Paravirtualization Interface** from Default to **KVM**
- Powered the VM back on — it booted cleanly to the login screen

---

## Discovering DevGuru on the Network

With both VMs on `LabNetwork`, I used `netdiscover` from Kali to find DevGuru's IP address:

```bash
sudo netdiscover -r 10.0.2.0/24
```

This initially only showed the NAT gateway (`10.0.2.2`) and DNS proxy (`10.0.2.3`) — DevGuru wasn't visible until after the kernel panic issue above was resolved and the machine fully booted. Once fixed, a third host appeared:

```
10.0.2.1   —  DevGuru
```

---

## Confirming Connectivity — Ping Test

The main objective of this exercise was to confirm that Kali could reach DevGuru over the network:

```bash
ping 10.0.2.1
```

**Result:**
```
20 packets transmitted, 20 received, 0% packet loss
rtt min/avg/max/mdev = 0.551/3.227/23.683/5.370 ms
```

![Ping success from Kali to DevGuru](ping-success.jpeg)



 **0% packet loss confirms Kali and DevGuru are successfully connected on the isolated lab network.**

---

## 🔎 Bonus: Initial Reconnaissance

Beyond the core connectivity task, I also ran some basic service enumeration for practice:

```bash
nmap -sV -T4 -F 10.0.2.1
```

**Result:**
```
135/tcp open  msrpc         Microsoft Windows RPC
445/tcp open  microsoft-ds
```

Attempted anonymous SMB enumeration:

```bash
smbclient -L //10.0.2.1/ -N
enum4linux -a 10.0.2.1
```

**Result:** Both attempts were rejected —

```
session setup failed: NT_STATUS_ACCESS_DENIED
Server doesn't allow session using username '', password ''
```

This confirmed the target does not allow anonymous SMB access, meaning further exploitation would require valid credentials or a different attack vector — a good next step for future practice.

---

## Summary of Challenges & Fixes

| Challenge | Fix |
|---|---|
| `VERR_ALREADY_EXISTS` on import | Deleted leftover `.vdi` file from a failed prior import |
| "LabNetwork" missing from dropdown | Created the NAT Network first in Network Manager |
| Kernel panic (`rcu_sched` stalls) on boot | Changed Paravirtualization Interface to KVM |
| DevGuru not appearing in netdiscover | Waited for full boot completion after fixing the kernel panic |
| Anonymous SMB access denied | Confirmed as expected hardening; noted for future credentialed testing |

---

## Tools Used
- VirtualBox
- Kali Linux
- DevGuru (VulnHub)
- `netdiscover`
- `nmap`
- `smbclient`
- `enum4linux`

---

*This lab was completed for educational purposes as part of an ethical hacking course, in an isolated virtual environment with no real-world targets involved.*
