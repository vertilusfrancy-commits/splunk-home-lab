# SSH Brute Force Detection — Splunk Home Lab

Part of the [`splunk-home-lab`](../../README.md) series. This lab simulates a real SSH brute force attack against a Linux host and detects it end-to-end using Splunk Free: log ingestion, SPL detection queries, a dashboard, and an automated alert.

---

## 🎯 Objective

Most tutorials show Splunk detecting attacks in a clean, pre-built environment. This lab was built from a completely blank Ubuntu Server install — including VM provisioning, networking between two different hypervisors, and troubleshooting real infrastructure problems — to demonstrate not just SPL knowledge, but the full operational reality of running a SIEM.

Goals:
- Stand up Splunk Free on a headless (no GUI) Ubuntu server
- Generate a real SSH brute force attack from a separate attacker machine
- Detect, visualize, and alert on that attack using Splunk

---

## 🏗️ Lab Architecture

| Role | OS | Hypervisor | IP (lab) |
|---|---|---|---|
| Target + SIEM | Ubuntu 22.04 Server (headless) | VirtualBox (Windows host) | `192.168.100.15` |
| Attacker | Kali Linux | UTM (macOS host) | `192.168.100.14` |

```
┌─────────────────────┐         ┌─────────────────────┐
│   Kali Linux         │         │  Ubuntu 22.04 Server │
│   (UTM / macOS)       │  SSH    │  (VirtualBox)         │
│   Hydra ──────────────┼────────▶│  OpenSSH + Splunk     │
│   192.168.100.14      │  :22    │  192.168.100.15 :8000 │
└─────────────────────┘         └─────────────────────┘
```

Two hypervisors on two different physical machines (Windows + macOS) were bridged onto the same local network so both VMs could reach each other directly — a more realistic setup than running everything inside one hypervisor, and a good example of cross-platform lab design.

---

## ⚙️ Phase 1 — Splunk Installation (headless server)

The target is a **headless** Ubuntu Server — no desktop environment, no browser. This matters because it forces Splunk Web to be exposed over the network instead of accessed via `localhost`, which is how it's accessed in most tutorials.

```bash
# Download Splunk Enterprise Free (.deb package)
wget -O splunk.deb "https://download.splunk.com/products/splunk/releases/10.2.1/linux/splunk-10.2.1-c892b66d163d-linux-amd64.deb"

# Install the package
sudo dpkg -i splunk.deb
```

By default, Splunk Web only binds to `127.0.0.1`, which is invisible to any other machine on the network. Since there's no browser on this VM, that has to be changed before the first start:

```bash
sudo nano /opt/splunk/etc/system/local/web.conf
```

```ini
[settings]
server.socket_host = 0.0.0.0
```

This tells Splunk to listen on all network interfaces, not just loopback.

```bash
# First start (accepts license, creates admin account)
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root

# Persist Splunk across reboots
sudo /opt/splunk/bin/splunk enable boot-start

# Open the required ports
sudo ufw allow 8000/tcp   # Splunk Web
sudo ufw allow 22/tcp     # SSH
sudo ufw enable
```

Splunk Web is now reachable from any machine on the LAN at `http://192.168.100.15:8000`.

### 🪲 Troubleshooting: disk full, Splunk wouldn't start

The first start attempt failed silently with `Warning: web interface does not seem to be available`, and `splunk status` reported `splunkd is not running`. Diagnosis:

```bash
df -h
# /dev/mapper/ubuntu--vg-ubuntu--lv   12G   12G   0G   100%   /
```

The disk was at 100% — Splunk couldn't write its working files. The Ubuntu installer's default LVM setup had only allocated **half the virtual disk** (12G of 23G available) to the logical volume. Fix:

```bash
sudo apt clean                                              # free a little space to run LVM commands
sudo pvresize /dev/sda3                                     # expand the physical volume
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv # grow the logical volume to use all free space
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv             # grow the filesystem to match
```

Result: `/` went from 0G to 12G free, and Splunk started cleanly on the next attempt. This is a common gotcha with Ubuntu's default LVM partitioning on small lab disks and worth knowing for any future Linux server provisioning.

---

## 📥 Phase 2 — Log Input Configuration

SSH authentication events on Debian/Ubuntu systems are written to `/var/log/auth.log` (not `/var/log/secure`, which is the RHEL/CentOS equivalent — an easy mix-up). This file is configured as a monitored input so Splunk ingests it continuously:

```bash
sudo /opt/splunk/bin/splunk add monitor /var/log/auth.log \
  -index main \
  -sourcetype linux_secure
```

- **`-index main`** — stores the data in Splunk's default index, fine for a single-purpose lab
- **`-sourcetype linux_secure`** — tags the data with a sourcetype Splunk already understands, which improves field extraction (e.g. recognizing `Failed password` events automatically)

Verified ingestion with:
```spl
index=main sourcetype=linux_secure | head 20
```

---

## 💥 Phase 3 — Attack Simulation

From Kali, connectivity was confirmed first:
```bash
ping 192.168.100.15 -c 4
```

Then the brute force attack was launched with Hydra against the `root` account over SSH:
```bash
hydra -l root -P ~/rockyou.txt ssh://192.168.100.15 -t 4 -vV
```

- **`-l root`** — single target username
- **`-P ~/rockyou.txt`** — password wordlist to try
- **`-t 4`** — 4 parallel connection threads
- **`-vV`** — verbose output, shows every attempt in real time

This generated **276+ failed SSH login attempts** in a few minutes, all logged in `/var/log/auth.log` and ingested by Splunk in near real time.

### 🪲 Troubleshooting: Kali's clock was out of sync

Before Hydra could even be installed, `apt update` failed with:
```
Release file ... is not valid yet (invalid for another 4h 29min)
```
Kali's system clock was several hours ahead of the actual time, which `apt` flags as a possible replay/tampering attack on package repositories (repo metadata is timestamp-signed). Since the network-based NTP service wasn't syncing automatically inside the VM, the clock was corrected manually:
```bash
sudo date -s "2026-03-16 14:00:00"
```
Once the clock matched real UTC time, `apt` accepted the repositories and package installs worked normally.

---

## 🔍 Phase 4 — Detection with SPL

Four SPL queries were built to go from raw logs to actionable detection.

### Query 1 — All Failed Login Attempts
```spl
index=main sourcetype=linux_secure "Failed password" earliest=-24h latest=now
| table _time, host, src_ip, user
| sort -_time
```
Baseline visibility: every failed login, in order, with source IP and targeted account.

### Query 2 — Brute Force Detection by Source IP
```spl
index=main sourcetype=linux_secure "Failed password" earliest=-24h latest=now
| rex field=_raw "from (?<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by src_ip
| where count > 10
| sort -count
```
`rex` extracts the source IP from the raw log line into a new field `src_ip` (not natively parsed by the `linux_secure` sourcetype). Grouping by IP and filtering `count > 10` turns raw noise into an actual detection rule — this single query correctly isolated `192.168.100.14` (Kali) as the attacker.

### Query 3 — Attack Timeline
```spl
index=main sourcetype=linux_secure "Failed password" earliest=-24h latest=now
| rex field=_raw "from (?<src_ip>\d+\.\d+\.\d+\.\d+)"
| timechart count by src_ip
```
Same extraction, plotted over time — shows the attack's volume and duration, useful for distinguishing a short burst from a sustained campaign.

### Query 4 — Top Targeted Users
```spl
index=main sourcetype=linux_secure "Failed password" earliest=-24h latest=now
| rex field=_raw "for (?<user>\w+) from"
| top user
```
Confirms which accounts were targeted — in this run, exclusively `root`, consistent with the `-l root` flag used in Hydra.

---

## 📊 Phase 5 — Dashboard

A **Classic Dashboard** named `SSH Attack Detection` was built combining the three analytical queries into one view:

| Panel | Source Query | Visualization |
|---|---|---|
| Attacks by Source IP | Query 2 | Table |
| Attack Timeline | Query 3 | Line chart |
| Top Targeted Users | Query 4 | Bar chart |

This gives a single-pane-of-glass view an analyst could check at the start of a shift instead of running four separate searches.

---

## 🚨 Phase 6 — Automated Alert

| Setting | Value |
|---|---|
| Name | SSH Brute Force Detected |
| Base query | Query 2 (`count > 10` by `src_ip`) |
| Schedule | Cron `*/5 * * * *` (every 5 minutes) |
| Trigger condition | Number of results > 0 |
| Severity | Medium |
| Action | Add to Triggered Alerts |

Splunk Free doesn't expose a simple "every 5 minutes" dropdown for scheduled alerts — it requires a raw cron expression (`*/5 * * * *`), which is worth knowing going in since it isn't obvious from the UI.

---

## 📸 Screenshots

| Screenshot | What it shows |
|---|---|
| `hydra_attack.png` | Hydra running from Kali, generating failed login attempts |
| `splunk_realtime_events_1.png` / `_2.png` | Raw events landing in Splunk in real time during the attack |
| `splunk_event_detail.png` | A single expanded event showing `host`, `src_ip`, `sourcetype` |
| `spl_brute_force_by_ip.png` | Query 2 result — attacker IP isolated with attempt count |
| `spl_top_targeted_users.png` | Query 4 result — `root` confirmed as the only targeted account |
| `dashboard_overview.png` / `dashboard_timeline.png` / `dashboard_full.png` | The completed 3-panel dashboard |
| `alert_created.png` | Confirmation of the scheduled alert |

![Hydra attack](./screenshots/hydra_attack.png)
![Brute force by IP](./screenshots/spl_brute_force_by_ip.png)
![Dashboard](./screenshots/dashboard_full.png)
![Alert created](./screenshots/alert_created.png)

---

## 🧠 Key Findings

- Hydra generated **276+ failed SSH attempts** against `root` from a single source IP (`192.168.100.14`), fully captured by Splunk via `/var/log/auth.log`
- Unlike CrowdSec (which whitelists RFC 1918 / private IP ranges by default — see the [CrowdSec lab](https://github.com/vertilusfrancy-commits/crowdsec-home-lab)), **Splunk detected the attack from a private LAN address with no additional configuration**, since it has no concept of a trusted internal range built in
- A correctly tuned `stats count by src_ip | where count > 10` threshold is enough to separate a brute force campaign from normal authentication noise (cron jobs, legitimate logins) in the same log source

---

## 🛠️ Skills Demonstrated

- Linux server administration (headless Ubuntu, LVM disk management, firewall configuration with `ufw`)
- SIEM deployment and configuration (Splunk Free install, log onboarding, sourcetypes)
- SPL query writing: field extraction with `rex`, aggregation with `stats`, time-series analysis with `timechart`
- Offensive tooling fundamentals (Hydra) used safely in an isolated lab to generate realistic detection data
- Dashboarding and alerting for operational SOC use cases
- Cross-hypervisor lab networking (VirtualBox + UTM) and general infrastructure troubleshooting under real constraints (disk space, clock sync)

---

## 🔭 Future Improvements

- Add a second attack type (e.g. Nmap port scanning, RDP brute force) to the same dashboard
- Tune the alert to auto-block the source IP via a scripted response
- Forward logs to Splunk using a Universal Forwarder instead of direct file monitoring, closer to a production architecture

---

## 🔗 Related Projects

- [CrowdSec Home Lab](https://github.com/vertilusfrancy-commits/crowdsec-home-lab) — the same attack type detected with CrowdSec instead of Splunk, including the RFC 1918 whitelist finding referenced above

---

## ⚠️ Disclaimer

This lab was built entirely in an isolated virtual environment for educational purposes. All attack simulations were performed against systems owned and controlled by the author.
