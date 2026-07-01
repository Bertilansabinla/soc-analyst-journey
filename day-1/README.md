# Day 1 — From Zero to SIEM: Building a Cybersecurity Home Lab on Apple Silicon

**Date:** June 24, 2026 | **Author:** Bertila Nsabinla | **Goal:** SOC Analyst by July 2026

> *ISC2 CC certified. No prior hands-on experience. No budget. Just a MacBook Air M1, three virtual machines, and a lot of error messages.*

---

## Why I'm Documenting This

I recently earned my ISC2 Certified in Cybersecurity (CC) certification. That was the theory. Now comes the harder part — the hands-on skills that actually get you hired.

I want to be job-ready for entry-level SOC Analyst roles by the end of July 2026. The job descriptions I'm targeting ask for experience with SIEM platforms like Splunk and QRadar, EDR tools like CrowdStrike Falcon and SentinelOne, and NDR/NGFW tools like Zeek, Palo Alto, and Corelight. None of these are things you can learn by reading. You have to touch them.

So I built a home lab. And on Day 1, everything went wrong in the most educational way possible.

---

## My Setup

| Component | Details |
|-----------|---------|
| Hardware | MacBook Air M1, 2020 — 8GB RAM, 245GB storage |
| Hypervisor | VMware Fusion |
| VM 1 | Kali Linux ARM (shows as "Debian 12" in VMware — this is normal) |
| VM 2 | Ubuntu Server 24.04 ARM (XFCE GUI installed) |
| VM 3 | Windows 11 ARM (for Active Directory learning) |

**Goal for Day 1:** Install Splunk on Ubuntu Server, load sample security data, run first SPL queries.

**What actually happened:** Disk space failures, ARM architecture incompatibilities, Docker failures, DNS issues, copy-paste problems, and a swap partition blocking disk expansion. And I still made it to Splunk by end of day.

---

## Part 1 — Everything Was Full Before I Even Started

First task: update Kali Linux. Standard practice. Immediately hit:

```
Warning: More space needed than available: 1,152 MB > 330 MB
Error: You don't have enough free space in /var/cache/apt/archives/
```

Kali VM: 29GB total, 98% full, only 653MB free.

Tried to clean up — `sudo apt clean` helped slightly, `sudo apt autoremove --purge` found nothing. Tried to clear journal logs:

```bash
sudo journalctl --vacuum-size=100m   # WRONG — lowercase m fails
sudo journalctl --vacuum-size=100M   # CORRECT — uppercase M required
```

Small detail. On Linux, small details like this cost time when you're new.

**Root cause of Kali disk problem:** Running `sudo fdisk -l /dev/nvme0n1` revealed:

```
/dev/nvme0n1p3  2035712 63633407 61597696 29.4G  Linux filesystem  ← Kali lives here
/dev/nvme0n1p4 63633408 67106815  3473408  1.7G  Linux swap        ← This blocks expansion
```

The swap partition (p4) was physically sitting between Kali's partition and the new free space. Can't grow p3 sideways when another partition is in the way.

**Status: Kali disk expansion unresolved ⚠️**

**Mac fix:** Ran `tmutil deletelocalsnapshots /` to clear invisible APFS Time Machine snapshots — freed ~8GB.

---

## Part 2 — What Is a SIEM and Why Does It Matter?

A SIEM (Security Information and Event Management) system is the central nervous system of a SOC. Every device generates log files all day — logins, file changes, network connections, process creations. Millions of lines daily across hundreds of machines.

No human can read that raw. So companies send all logs to one central searchable system — the SIEM. Analysts query that data to find patterns indicating something bad is happening.

**Splunk** is one of the most widely used commercial SIEMs. Learning it directly maps to what job descriptions ask for.

My plan:
1. Set up Ubuntu Server as the SIEM server
2. Install Splunk on it
3. Load sample security log data
4. Practice writing SPL (Splunk Processing Language) queries
5. Eventually connect Kali and Windows VMs to send real logs

Simple on paper.

---

## Part 3 — The Internet Problem

Needed to download Splunk onto Ubuntu VM. The VM was on host-only network (isolated from internet — good security practice). Gave it temporary internet access, checked connectivity:

```bash
ip a   # IP was there
ping google.com   # ping: google.com: Temporary failure in name resolution
```

IP existed but no DNS. DNS translates domain names into IP addresses. Without it, you can connect to a network but can't find anything by name.

**Fix:**
```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

Manually added Google's DNS server. Internet worked.

---

## Part 4 — The ARM Architecture Problem

Downloaded Splunk `.deb` for Linux. Couldn't run it. Here's why:

**The ARM vs x86 problem:**
- MacBook Air M1 = ARM processor
- VMs on M1 Mac = also ARM (they have to be — the hardware is ARM)
- Splunk Enterprise for Linux = compiled for x86 (Intel/AMD) only
- No ARM64 Linux version of Splunk exists

You simply cannot install the standard Linux Splunk package natively on an M1 Mac's VMs.

**Docker as the workaround:**

Installed Docker, tried running Splunk in an x86 container with QEMU emulation:

```bash
sudo docker run -d --platform=linux/amd64 -p 8000:8000 \
  -e SPLUNK_START_ARGS='--accept-license' \
  -e SPLUNK_PASSWORD='splunk123!' \
  --name splunk-enterprise splunk/splunk:latest
```

Image downloaded (1.35GB). Then failed — Ubuntu hit 100% full from the download.

---

## Part 5 — Expanding Ubuntu's Disk (First Real Win ✅)

Unlike Kali, Ubuntu used **LVM (Logical Volume Manager)**. LVM is more flexible — you can expand it while the system is running.

```bash
sudo vgdisplay
# Output: Free PE / Size: 1873 / 7.32 GiB — 7.32GB unallocated, available to use
```

Commands run (typed manually — copy-paste not working yet):

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

**Result:** Ubuntu went from 9.8GB at 100% full → 17GB with 8.8GB free. ✅

**Lesson:** LVM is better than standard partitioning for lab environments. Ubuntu's LVM let me expand in two commands while running. Kali's standard partitions required deleting the swap partition first.

---

## Part 6 — Docker Keeps Failing

With space available, tried Docker again. Container downloaded fully but kept exiting immediately.

**Error 1:** License terms updated — needed new environment variable `SPLUNK_GENERAL_TERMS`

**Error 2 (the real problem):**
```
sudo: effective uid is not 0, is /usr/bin/sudo on a file system
with the 'nosuid' option set or an NFS file system without root privileges?
```

This is the ARM emulation failure. Splunk's Docker container uses Ansible internally for setup. Ansible uses `sudo`. When running x86 containers on ARM via QEMU emulation, the emulation layer doesn't properly handle kernel-level privilege operations that `sudo` relies on.

Tried: `--privileged` flag. Still failed.  
Tried: `--user root`. Still failed.  
Tried: QEMU binfmt support. Still failed.

**Conclusion:** Running Splunk Enterprise via Docker on ARM Ubuntu VM doesn't work due to how QEMU handles `sudo` privilege escalation in Splunk's Ansible provisioner.

---

## Part 7 — The Copy-Paste Problem

Running parallel to everything: couldn't copy-paste between Mac and Ubuntu VM terminal.

**Solution: SSH**

Instead of the VM's graphical terminal, connected from Mac Terminal directly:

```bash
ssh administrator@172.16.23.135
```

Inside native Mac Terminal = full Mac copy-paste with Cmd+V. Problem solved.

**Setup required on Ubuntu first:**
```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

**Lesson:** SSH into your VMs from your host terminal immediately. Don't fight the VM's graphical terminal.

---

## Part 8 — The Right Solution Was There All Along

After hours of Docker attempts, searched differently. Found it:

**Splunk Enterprise has a native macOS ARM build.**

On splunk.com/download → click **Mac OS tab** (not Linux) → ARM — macOS — .dmg — 840.8 MB

Downloaded .dmg → double-clicked → installer opened → created admin credentials → clicked "Start and Show Splunk" → browser opened automatically.

**Splunk Enterprise 10.4.0 running at localhost:8000. ✅**

The solution that worked: 4 clicks.  
We spent hours on workarounds. The native solution was one tab-click away on the same download page.

**Lesson:** Always check the obvious options before going deep on workarounds.

---

## Part 9 — First Real SOC Investigation

Loaded `secure.log` (Linux mail server authentication log from Splunk's official tutorial dataset) into Splunk via Add Data.

Even the preview was alarming — every event was a variant of:
```
Failed password for invalid user [name] from [IP] port [number] ssh2
```

Someone had been hammering the mail server trying usernames one after another. Classic brute force.

### Query 1 — Who was attacking and how many times?

```splunk
source="secure.log" "Failed password"
| rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by src_ip
| sort -count
```

**Result:** 181 different IP addresses. Top attacker — `87.194.216.51` — made 218 failed login attempts. Classic automated brute force: a script cycling through common usernames (guest, postgres, apache, mongodb, root).

### Query 2 — Did any of them get in?

```splunk
source="secure.log" "Accepted password"
| rex "for (?P<user>\w+) from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by user src_ip
| sort -count
```

**Result:** 3 users with successful logins — all from internal IPs (10.x.x.x range). Internal employees logging in normally. Zero external attackers succeeded.

### Incident Report — Day 1

> Detected SSH brute force attack against mailsv1 from 181 unique external IP addresses. Top attacker 87.194.216.51 made 218 failed authentication attempts. No external IP address achieved successful authentication. Internal users djohnson, nsharpe, and myuan show normal login activity from internal network addresses (10.x.x.x range). No escalation required. Recommend blocking top attacker IPs at perimeter firewall as precautionary measure.

That is a real SOC investigation: detect the attack, quantify the scope, determine if it succeeded, write the summary.

---

## Key Lessons from Day 1

| Lesson | Detail |
|--------|--------|
| ARM is a real constraint | Most enterprise security tools are x86 only. Check host OS compatibility before assuming you need a VM. |
| Use LVM for lab VMs | Expandable on the fly. Standard partitions with swap block expansion. |
| SSH immediately | Set up SSH from your host terminal on day one. Faster, cleaner, more realistic. |
| SOC analysis = right questions | "Who is attacking?" and "Did they succeed?" The syntax is lookupable. The questions are the skill. |
| Error messages are a curriculum | The terminal is honest. Read every error carefully — it tells you exactly what went wrong. |
| Check obvious solutions first | Native ARM .dmg was one tab-click away while we spent hours on Docker workarounds. |

---

## Lab Status After Day 1

| Component | Status |
|-----------|--------|
| Splunk Enterprise 10.4.0 | ✅ Running natively on macOS |
| SSH Mac → Ubuntu | ✅ Configured |
| Ubuntu disk expanded | ✅ 10GB → 17GB via LVM |
| secure.log loaded into Splunk | ✅ |
| First SPL queries written | ✅ |
| Kali disk expansion | ⚠️ Blocked by swap partition |
| Ubuntu pending updates (180) | ⚠️ Pending |

---

## 5-Week Plan

| Week | Focus | Status |
|------|-------|--------|
| Week 1 (Jun 24–30) | Splunk + SPL | 🟡 In Progress |
| Week 2 (Jul 1–7) | Google SecOps + LetsDefend | ⬜ Upcoming |
| Week 3 (Jul 8–14) | Wazuh EDR | ⬜ Upcoming |
| Week 4 (Jul 15–21) | Zeek + Wireshark + pfSense | ⬜ Upcoming |
| Week 5 (Jul 22–31) | Portfolio + Resume + Apply | ⬜ Upcoming |

---

*LinkedIn article: [From Zero to SIEM: How I Spent 8 Hours Fighting My Own Laptop](https://linkedin.com/in/bertilansabinla)*  
*GitHub: [github.com/bertilansabinla/soc-analyst-journey](https://github.com/bertilansabinla/soc-analyst-journey)*

#Cybersecurity #SOCAnalyst #Splunk #SIEM #HomeLab #LearningInPublic #ISC2 #BlueTeam #CareerChange #AppleSilicon
