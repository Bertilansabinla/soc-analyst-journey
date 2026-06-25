https://www.linkedin.com/posts/bertilansabinla_from-zero-to-siem-how-i-spent-8-hours-fighting-ugcPost-7475846222969356289-PmUf/?highlightedUpdateUrn=urn%3Ali%3Aactivity%3A7475846223590133760&origin=SOCIAL_SHARE&utm_source=social_share_send&utm_medium=member_desktop_web&rcm=ACoAACGu8iMBtZvxR9246aeP9oJJND7ytnHlSmE
# From Zero to SIEM: How I Spent 8 Hours Fighting My Own Laptop to Build a Cybersecurity Lab — And What I Found Inside

*Day 1 of my public journey to become a SOC Analyst by July 2026. ISC2 CC certified. No prior hands-on experience. No budget. Just a MacBook Air M1, three virtual machines, and a lot of error messages.*

---

## Why I'm Documenting This

I recently earned my ISC2 Certified in Cybersecurity (CC) certification. That was the theory. Now comes the harder part — the hands-on skills that actually get you hired.

I want to be job-ready for entry-level SOC Analyst roles by the end of July 2026. That's five weeks. The job descriptions I'm targeting ask for experience with SIEM platforms like Splunk and QRadar, EDR tools like CrowdStrike Falcon and SentinelOne, and NDR/NGFW tools like Zeek, Palo Alto, and Corelight. None of these are things you can learn by reading. You have to touch them.

So I built a home lab. And on Day 1, everything went wrong in the most educational way possible.

This article is for anyone learning cybersecurity on consumer hardware, with no budget, figuring things out as they go. I'm writing it for recruiters who want to see not just what I know, but how I think when things break. And I'm writing it for my future self, so I never forget how hard the beginning was.

---

## My Setup Going In

**Hardware:** MacBook Air M1, 2020 — 8GB RAM, 245GB storage

**Virtual Machines running in VMware Fusion:**
- Kali Linux (my attack/security testing OS)
- Ubuntu Server 24.04 (my server VM, where I planned to install Splunk)
- Windows 11 (for Active Directory learning — essential for cybersecurity)

**Goal for Day 1:** Install Splunk SIEM on Ubuntu Server, load sample security data, run my first SPL queries.

**What actually happened:** I spent the entire day fighting disk space, architecture incompatibilities, Docker failures, DNS issues, copy-paste problems, and a partition layout that blocked every fix I tried. And I still made it to Splunk by the end of the day.

Let me walk you through every step.

---

## Part 1: Before I Could Even Start — Everything Was Full

The first thing I wanted to do was update my Kali Linux VM. Standard practice before starting any new work. I ran `sudo apt upgrade` and immediately hit this:

> `Warning: More space needed than available: 1,152 MB > 330 MB, installation may fail`
> `Error: You don't have enough free space in /var/cache/apt/archives/.`

*[Screenshot 1: Kali terminal showing the disk full error — red warning and error text visible]*

My Kali VM had 29GB of total disk space and was sitting at 98% full with only 653MB available. And this was before I'd even started the day's actual work.

I ran `df -h` to understand the situation:

*[Screenshot 2: df -h output showing /dev/nvme0n1p3 at 98% — 29G total, 27G used, 653M available]*

I tried to clean up. `sudo apt clean` helped a little. `sudo apt autoremove --purge` found nothing to remove. I tried to clear journal logs with `sudo journalctl --vacuum-size=100m` — and got this:

> `Failed to parse vacuum size: 100m`

I had the wrong case. The correct command uses uppercase M. A small detail, but on Linux, small details like this cost you time when you're new.

Meanwhile, I checked my Mac's overall storage situation:

*[Screenshot 3: Mac storage settings showing 214.53GB of 245.11GB used — the bar is almost entirely red]*

My Mac had only **26.77GB free** across a 245GB drive. That's not much when you're running three virtual machines. The breakdown showed:
- Documents (my VMs): 86GB
- System Data: 56GB
- macOS itself: 41GB
- Applications: 27GB

I needed to understand what was inside that 86GB of Documents. Opening it up revealed my three VM bundles — Windows 11 at 41GB, something labeled "Debian 12" at 37GB, and Ubuntu Server at 13GB. There were also a bunch of loose `.vmdk` files (virtual disk chunks) that turned out to belong to the Windows 11 VM — VMware splits large virtual disks into 4GB chunks.

*[Screenshot 4: Mac large files view showing VM bundles and loose .vmdk files]*

This is also where I had an interesting discovery. The VM I'd been calling "Kali Linux" was showing up as "Debian 12.x 64-bit Arm" in VMware Fusion. I panicked briefly thinking I'd installed the wrong OS. But when I booted it, Kali opened normally. The reason: Kali Linux is built on top of Debian. When VMware scans the OS during installation, it detects the underlying Debian base and labels it accordingly. This is actually a useful thing to understand — Kali is essentially Debian with security tools pre-installed on top.

*[Screenshot 5: VMware Fusion showing the three VMs — Ubuntu Server, Windows 11, and "Debian 12.x 64-bit Arm" (which is actually Kali)]*

I tried to expand the Kali disk from VMware Fusion's settings, then ran the standard expansion commands inside Kali. Nothing worked. Running `sudo fdisk -l /dev/nvme0n1` revealed why:

```
Device            Start      End  Sectors  Size Type
/dev/nvme0n1p1     2048    34815    32768   16M Linux filesystem
/dev/nvme0n1p2    34816  2035711  2000896  977M EFI System
/dev/nvme0n1p3  2035712 63633407 61597696 29.4G Linux filesystem  ← Kali lives here
/dev/nvme0n1p4 63633408 67106815  3473408  1.7G Linux swap         ← This was the problem
```

Even though VMware had allocated more space, the swap partition (p4) was physically sitting between Kali's partition (p3) and the new free space. You can't just grow p3 sideways when there's another partition in the way. Fixing this would require deleting the swap partition, expanding p3, and recreating swap — a more involved operation I noted for later.

**Kali disk expansion: unresolved for now.** ⚠️

I ran `tmutil deletelocalsnapshots /` on my Mac to clear APFS local backups — these are invisible Time Machine snapshots macOS creates automatically. This freed about 8GB, giving me roughly 34GB free on my Mac. Enough breathing room to continue.

---

## Part 2: The Plan — Install Splunk on Ubuntu Server

Before diving into the failures, let me explain what I was actually trying to build and why. Understanding the goal makes the errors make more sense.

**What is a SIEM?**

A SIEM (Security Information and Event Management) system is the central nervous system of a SOC (Security Operations Center). In a real company, every computer, server, firewall, and application generates log files all day long — records of everything that happens. Logins, file changes, network connections, errors, process creations. Millions of lines every day.

No human can read raw log files from hundreds of machines simultaneously. So companies send all those logs to one central, searchable system — the SIEM. Analysts then search that data using query languages to find patterns that indicate something bad is happening. Splunk is one of the most widely used commercial SIEMs. Learning it directly maps to what the job descriptions ask for.

My plan was:
1. Set up Ubuntu Server as the SIEM server
2. Install Splunk on it
3. Load sample security log data
4. Practice writing SPL (Splunk Processing Language) queries
5. Eventually connect my Kali and Windows VMs to send real logs into it

Simple on paper.

---

## Part 3: The Internet Problem

First, I needed to get Splunk downloaded onto the Ubuntu VM. The VM was on a host-only network (isolated from the internet, which is good security practice for a lab) so I needed to temporarily give it internet access.

I confirmed network connectivity with `ip a`:

*[Screenshot 6: Ubuntu terminal showing ip a output — ens160 interface with IP 192.168.114.128]*

The network adapter was up and had an IP. Good. I opened Firefox inside the Ubuntu GUI (I had installed a lightweight XFCE desktop on what started as a server) and tried to navigate to splunk.com.

*[Screenshot 7: Ubuntu XFCE desktop with Firefox showing "Looks like there's a problem with your internet connection"]*

The IP was there but the internet wasn't working. I tried pinging Google:

```bash
ping google.com
```

> `ping: google.com: Temporary failure in name resolution`

*[Screenshot 8: Terminal showing ping failure with "Temporary failure in name resolution" error]*

The machine had a network connection but no DNS. DNS is the system that translates domain names (like google.com) into IP addresses. Without it, you can connect to the network but you can't find anything by name. The fix was simple:

```bash
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

This manually adds Google's DNS server (8.8.8.8) to the system's DNS configuration. After this, internet worked.

---

## Part 4: The Architecture Problem — The Wall I Hit Repeatedly

Now I could reach splunk.com. I went to the download page, clicked Linux, and saw the download options: `.rpm`, `.deb`, `.tgz`. All labeled "64-bit."

*[Screenshot 9: Splunk download page showing Linux tab with only x86 64-bit options — no ARM visible]*

I downloaded the `.deb` file. But before it could run, I hit a fundamental problem that I didn't fully understand yet and would spend hours battling.

**The ARM vs x86 problem:**

My MacBook Air has an Apple M1 chip. The M1 is an ARM processor — a different CPU architecture than the Intel/AMD x86 chips that most enterprise software is built for. When I create VMs in VMware Fusion on my M1 Mac, those VMs also run ARM architecture. They have to — the Mac's hardware is ARM.

Splunk Enterprise for Linux is compiled for x86 (Intel/AMD). There is no ARM64 Linux version of Splunk. This means you simply cannot install the standard Linux Splunk package natively on an M1 Mac's VMs.

I didn't know this going in. I found out through a series of failures.

**Docker as the workaround:**

The logical next move was Docker. Docker lets you run containers — lightweight isolated environments. With QEMU emulation, you can theoretically run x86 containers on ARM hardware by having the ARM processor simulate an Intel one. I installed Docker:

*[Screenshot 10: Ubuntu terminal showing Docker version 29.1.3 successfully installed, alongside Splunk verification email on the right side]*

First Docker attempt:
```bash
sudo docker run -d --platform=linux/amd64 -p 8000:8000 \
  -e SPLUNK_START_ARGS='--accept-license' \
  -e SPLUNK_PASSWORD='splunk123!' \
  --name splunk-enterprise splunk/splunk:latest
```

The image downloaded — 1.35GB pulled successfully. Then:

*[Screenshot 11: Terminal showing Docker download completing then failing with "no space left on device"]*

Ubuntu was 100% full. The 1.35GB Docker image had filled the remaining space. `df -h` confirmed it:

> `/dev/mapper/ubuntu--vg-ubuntu--lv   9.8G   9.8G     0  100%  /`

*[Screenshot 12: df -h output showing Ubuntu at 100% full — 9.8G used out of 9.8G]*

---

## Part 5: Expanding Ubuntu's Disk — The First Real Win

Unlike Kali, Ubuntu used LVM (Logical Volume Manager) for its disk layout. LVM is more flexible than standard partitioning — you can expand it while the system is running. I checked if there was free space to use:

```bash
sudo vgdisplay
```

*[Screenshot 13: vgdisplay output showing VG Size 17.32 GiB with Free PE / Size: 1873 / 7.32 GiB]*

There was 7.32GB of unallocated space sitting in the volume group. Time to use it. I had to type these commands manually in the Ubuntu terminal because copy-paste wasn't working (more on that shortly):

*[Screenshot 14: Ubuntu terminal showing lvextend command with errors on first attempts due to wrong syntax — multiple failed attempts visible]*

The first few attempts failed because of typos in the path. This is where not being able to paste from my Mac was really painful — I was transcribing long Linux paths by hand and getting them wrong. After correcting the syntax:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

*[Screenshot 15: Successful lvextend and resize2fs output — "Logical volume ubuntu-vg/ubuntu-lv successfully resized" — df -h showing 17G with 8.8G free at 46%]*

Ubuntu went from 9.8GB at 100% full → 17GB with 8.8GB free. **First real win of the day.** ✅

---

## Part 6: Docker Tries Again — And Keeps Failing

With space available, I tried Splunk in Docker again. This time it downloaded fully but the container kept exiting immediately. Checking the logs:

*[Screenshot 16: Docker ps -a showing container "Exited (1) 13 minutes ago"]*

*[Screenshot 17: Docker logs showing "License not accepted" error — Splunk updated their terms and needed a new environment variable]*

Splunk had updated their license terms. The old `--accept-license` flag wasn't enough anymore. They now required an additional `SPLUNK_GENERAL_TERMS` variable. I updated the command, deleted the old container, tried again.

Container still exited. New error in the logs:

```
sudo: effective uid is not 0, is /usr/bin/sudo on a file system 
with the 'nosuid' option set or an NFS file system without root privileges?
```

This is the real ARM emulation failure. Splunk's Docker container uses Ansible (an automation tool) internally to set itself up when it first runs. Ansible uses sudo internally. When you run an x86 Docker container on ARM using QEMU emulation, the emulation layer doesn't properly handle certain kernel-level privilege operations that sudo relies on. The result is that Splunk's internal setup process fails before it even starts.

I tried: adding `--privileged` flag. Still failed.
I tried: adding `--user root`. Still failed.
I tried: installing proper QEMU binfmt support with `tonistiigi/binfmt`. Still failed.

*[Screenshot 18: Docker ps -a showing container exiting with code 2, followed by docker logs showing the Ansible failure with sudo error]*

After several hours of this, the conclusion was clear: **running Splunk Enterprise via Docker on an ARM Ubuntu VM doesn't work due to how QEMU handles sudo privilege escalation in Splunk's Ansible provisioner.**

---

## Part 7: The Copy-Paste Problem

Running parallel to all of this was a frustrating quality-of-life issue: I couldn't copy and paste between my Mac and the Ubuntu VM terminal.

The normal Linux terminal paste shortcut `Ctrl+Shift+V` didn't work in this setup. Right-clicking showed a greyed-out Paste option.

*[Screenshot 19: VMware Fusion Edit menu showing Paste greyed out]*

I tried the Edit menu in VMware Fusion's menu bar — but that operates at the VMware level, not inside the VM's terminal.

The solution was SSH. Instead of using the VM's graphical terminal, I connected from my Mac Terminal directly to Ubuntu:

```bash
ssh administrator@172.16.23.135
```

*[Screenshot 20: Mac Terminal showing successful SSH connection to Ubuntu — "Welcome to Ubuntu 24.04.3 LTS" banner visible, administrator@ubuntu:~$ prompt]*

Once connected via SSH, I was operating inside my native Mac Terminal — which meant full Mac copy-paste with Cmd+V worked perfectly. From this point on, every command ran from SSH and the frustration of manual typing was over.

Setting up SSH required first enabling it on the Ubuntu side:
```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

*[Screenshot 21: SSH setup showing openssh-server install and systemctl commands — initial attempt failed because commands were pasted as one line, second attempt succeeded separately]*

This is also where I accidentally exposed my password in the terminal — I typed it in the wrong field. A learning moment: always be aware of which field is active before typing credentials.

---

## Part 8: The Right Solution Was There All Along

After hours of Docker attempts, I finally searched differently and found the answer I should have found first: **Splunk Enterprise has a native macOS ARM build.**

Going to splunk.com/download and clicking the Mac OS tab — not Linux — showed:

> ARM — macOS — .dmg — 840.8 MB

*[Screenshot 22: Splunk download page Mac OS tab showing ARM option with .dmg download]*

I downloaded the .dmg, double-clicked it, and the macOS installer opened:

*[Screenshot 23: Splunk 10.4.0 macOS installer showing "Writing files..." progress bar]*

After installation, Splunk's terminal prompted me to create admin credentials:

*[Screenshot 24: Mac terminal showing "This appears to be your first time running Splunk" — asking to create administrator username]*

Then a dialog appeared:

*[Screenshot 25: "Splunk's Little Helper" dialog asking "What would you like to do?" with "Start and Show Splunk" button]*

I clicked "Start and Show Splunk." macOS asked for keychain access (Splunk uses MongoDB internally which needs certificate signing). I entered my Mac password and clicked Always Allow.

*[Screenshot 26: macOS keychain dialog asking for password to allow mongod-8.0 to sign with imported private key]*

Browser opened automatically. Splunk Enterprise 10.4.0 was running.

*[Not shown — but the "Hello, Administrator" dashboard appeared at localhost:8000]*

The solution that worked: 4 clicks. Download ARM .dmg → Install → Create credentials → Start.

We spent hours on workarounds. The native solution was one tab click away.

---

## Part 9: The First Real SOC Investigation

With Splunk running, I clicked "Add Data" and uploaded `secure.log` — a Linux mail server security log from Splunk's official tutorial dataset, downloadable at `docs.splunk.com`.

*[Screenshot 27: Splunk Add Data screen — "File has been uploaded successfully" with Start Searching button]*

*[Screenshot 28: Splunk Input Settings page showing host as Bertilas-MacBook-Air.local and Index: Default]*

When the data loaded, even the preview was alarming. Every single event in the log was some variant of:

```
Failed password for invalid user [name] from [IP] port [number] ssh2
```

*[Screenshot 29: Splunk Set Source Type page showing raw log events — Failed password for invalid user appserver, root, testuser, apache, mongodb visible]*

Someone had been hammering this mail server trying to get in via SSH, trying username after username. This is called a brute force attack. And this was just the preview.

**Query 1: Who was attacking and how many times?**

```
source="secure.log" "Failed password" 
| rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)" 
| stats count by src_ip 
| sort -count
```

*[Screenshot 30: Splunk search results showing 181 unique attacking IPs — 87.194.216.51 at top with 218 attempts, 211.166.11.101 with 189, 194.215.205.19 with 186]*

**181 different IP addresses** had been trying to break in. The top attacker — 87.194.216.51 — had made 218 failed login attempts. This is a classic automated brute force: a script cycling through common usernames (guest, postgres, apache, mongodb, root, games, mail...) trying to find one that works.

**Query 2: Did any of them get in?**

This is the question that actually matters. Failed attempts are noise. Successful logins are incidents.

```
source="secure.log" "Accepted password" 
| rex "for (?P<user>\w+) from (?P<src_ip>\d+\.\d+\.\d+\.\d+)" 
| stats count by user src_ip 
| sort -count
```

*[Screenshot 31: Splunk results showing only 3 users with successful logins — djohnson from 10.3.10.46 with 237 logins, nsharpe from 10.2.10.163 with 104, myuan from 10.1.10.172 with 39]*

Three users had successful logins. But look at their IP addresses: 10.3.10.46, 10.2.10.163, 10.1.10.172. Those are all internal IPs — addresses that start with 10.x are private network addresses, meaning they're inside the organization's own network, not from the internet. These are legitimate employees logging in normally.

The external attackers — every single one of them — failed. The brute force was unsuccessful.

**My first incident report:**

> *Detected SSH brute force attack against mailsv1 from 181 unique external IP addresses. Top attacker 87.194.216.51 made 218 failed authentication attempts. No external IP address achieved successful authentication. Internal users djohnson, nsharpe, and myuan show normal login activity from internal network addresses (10.x.x.x range). No escalation required. Recommend blocking top attacker IPs at perimeter firewall as precautionary measure.*

That is a real SOC investigation. Detect the attack, quantify the scope, determine if it succeeded, write the summary. I did that today on Day 1.

---

## What I Actually Learned

Beyond the technical steps, here's what Day 1 taught me:

**1. ARM architecture is a real constraint, not just a footnote.** Most enterprise security tools are built for Intel/AMD processors. Running them on Apple Silicon requires workarounds, and some workarounds simply don't work. Plan around this — check host OS compatibility before assuming you need a VM.

**2. LVM is better than standard partitioning for lab environments.** Ubuntu's Logical Volume Manager let me expand disk space in two commands while the system was running. Kali's standard partition layout required a more complex procedure because a swap partition was in the way. When setting up VMs for labs, use LVM.

**3. SSH into your VMs from your host terminal.** Don't fight the VM's graphical terminal for copy-paste. Set up SSH immediately — it's faster, cleaner, and more realistic to how you'd actually administer a server.

**4. SOC analysis is about asking the right questions, not memorizing commands.** The two queries I ran today map to two fundamental SOC questions: "Who is attacking?" and "Did they succeed?" The syntax can be looked up. The questions are the skill.

**5. Error messages are a curriculum.** Every failure today — the disk space error, the DNS failure, the Docker sudo error, the wrong path in lvextend — explained exactly what was wrong when I read them carefully. The terminal is honest. It tells you what happened.

**6. The simple solution is often right there.** We spent hours on Docker workarounds. The macOS native ARM installer was one tab-click away on the same download page. Always check the obvious options before going deep on workarounds.

---

## What's Next — Day 2

Tomorrow: TryHackMe SOC Level 1 path — Splunk fundamentals room. I'll learn core SPL commands (`search`, `stats`, `table`, `eval`) and write five original queries against today's data. Goal is to be able to answer "walk me through how you'd investigate a SIEM alert" in an interview.

**The 5-week plan:**
- Week 1 (now): Splunk + SPL fundamentals
- Week 2: Google SecOps/Chronicle + LetsDefend SOC simulator
- Week 3: Wazuh open-source EDR + detection writeups
- Week 4: Zeek + Wireshark + pfSense NGFW concepts
- Week 5: Portfolio writeups, resume bullets, start applying

---

## For Recruiters Reading This

I know I don't have enterprise tool experience. I haven't used CrowdStrike Falcon or actual QRadar. What I do have is the ability to build environments, break things, diagnose them from error messages, and keep going until I get a result. That's what today was.

By the time I apply, I'll have five weeks of documented, screenshot-supported, narrative-logged hands-on work in Splunk, Google SecOps, Wazuh, Zeek, and pfSense. The tools are different. The logic is the same.

If you want to follow this journey, I'm documenting every day here on LinkedIn and in my GitHub repository.

---

*Day 1 complete. Splunk is running. The data is in. The first investigation is done.*

*Follow for Day 2: Learning SPL on TryHackMe.*

---

**Tags:** #Cybersecurity #SOCAnalyst #Splunk #SIEM #HomeLab #LearningInPublic #ISC2 #BlueTeam #CareerChange #AppleSilicon #CyberSecurityJourney #EntryLevel #ThreatHunting #SPL #VMware
