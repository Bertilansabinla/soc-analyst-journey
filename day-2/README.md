# Day 2 — SIEM Fundamentals & Ransomware Attack Investigation

**Date:** June 29, 2026 | **Author:** Bertila Nsabinla | **Goal:** SOC Analyst by July 2026

---

## Rooms Completed

| Room | Platform | Points | Status |
|------|----------|--------|--------|
| Introduction to SIEM | TryHackMe | 32 | ✅ Complete |
| Splunk Basics — Did you SIEM? | TryHackMe | 40 | ✅ Complete |
| **Total** | | **72** | |

---

## What I Learned — SIEM Architecture

A SIEM is not a product. It is a 4-stage pipeline:

| Stage | What Happens | Splunk Component |
|-------|-------------|------------------|
| 1. Collect | Devices generate logs | Log sources |
| 2. Transport | Forwarder ships logs to SIEM | Universal Forwarder |
| 3. Process | Normalize + correlate patterns | Indexer |
| 4. Present | Analyst queries + alerts | Search Head |

---

## Ransomware Investigation — 18,744 Events Across 2 Sources

### Finding 1 — Attacker IP
```splunk
index=main sourcetype=web_traffic | stats count by client_ip
```
**Result:** `198.51.100.55` — 7,876 events (45.8% of all traffic). Every other IP had 3.

### Finding 2 — Peak Attack Day
```splunk
index=main sourcetype=web_traffic 
| timechart span=1d count | sort by count | reverse
```
**Result:** `2025-10-12` — 2,267 events (4x normal baseline)

### Finding 3 — SQL Injection Tools
```splunk
sourcetype=web_traffic client_ip="198.51.100.55" 
AND user_agent IN ("*sqlmap*", "*Havij*") 
| stats count by user_agent
```
**Result:** Havij/1.17 = 993 events | sqlmap/1.7.9 = 967 events

### Finding 4 — Path Traversal
```splunk
sourcetype=web_traffic client_ip="198.51.100.55" 
path="*../../*" | stats count by path
```
**Result:** 658 attempts to read `/etc/passwd`

### Finding 5 — Data Exfiltration (Firewall Log Pivot)
```splunk
sourcetype=firewall_logs src_ip="10.10.1.5" 
AND dest_ip="198.51.100.55" AND action="ALLOWED" 
| stats sum(bytes_transferred) by src_ip
```
**Result:** 126,167 bytes stolen to attacker C2 server

---

## Full Attack Chain
