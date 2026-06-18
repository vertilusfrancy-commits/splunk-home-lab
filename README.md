# 🛡️ Splunk Home Lab

A growing collection of hands-on attack simulation and detection labs built on Splunk Free. Each lab starts from a blank VM, walks through real infrastructure setup (and the real problems that come with it), simulates an actual attack, and documents the full detection pipeline: log ingestion, SPL queries, dashboards, and alerting.

Built as part of a self-directed transition into cybersecurity, targeting SOC Analyst roles.

---

## 🎯 Why this repo exists

Most "Splunk lab" tutorials skip the messy parts — disk space running out, clocks drifting between VMs, two hypervisors on two different machines refusing to see each other. This repo documents those problems too, because troubleshooting infrastructure under pressure is exactly what a SOC analyst does daily, not just running SPL queries on a pre-built sandbox.

---

## 🧪 Lab Environment

| Component | Details |
|---|---|
| SIEM | Splunk Enterprise Free (v10.2.1) |
| Target host | Ubuntu 22.04 Server (headless, no GUI) |
| Attacker host | Kali Linux |
| Virtualization | VirtualBox (Windows host) + UTM (macOS host), bridged onto a shared network |

---

## 📁 Repository Structure

```
splunk-home-lab/
│
├── README.md                        # This file — repo overview
└── attacks/
    ├── ssh-brute-force/
    │   ├── README.md                # Full write-up: setup, attack, detection, troubleshooting
    │   ├── queries/
    │   │   ├── failed_logins.spl
    │   │   ├── brute_force_by_ip.spl
    │   │   ├── attack_timeline.spl
    │   │   └── top_targeted_users.spl
    │   └── screenshots/
    │       ├── hydra_attack.png
    │       ├── splunk_realtime_events_1.png
    │       ├── splunk_realtime_events_2.png
    │       ├── splunk_event_detail.png
    │       ├── spl_brute_force_by_ip.png
    │       ├── spl_top_targeted_users.png
    │       ├── dashboard_overview.png
    │       ├── dashboard_timeline.png
    │       ├── dashboard_full.png
    │       └── alert_created.png
    └── (more attacks coming)
```

---

## ⚔️ Attacks Documented

| # | Attack | Tools | Status | Write-up |
|---|---|---|---|---|
| 1 | SSH Brute Force | Hydra | ✅ Complete | [attacks/ssh-brute-force](./attacks/ssh-brute-force/README.md) |
| 2 | Port Scanning | Nmap | 🔜 Planned | — |
| 3 | TBD | — | 🔜 Planned | — |

---

## ⚙️ Base Splunk Setup (applies to every lab in this repo)

### Installation on a headless Ubuntu 22.04 server

```bash
wget -O splunk.deb "https://download.splunk.com/products/splunk/releases/10.2.1/linux/splunk-10.2.1-c892b66d163d-linux-amd64.deb"
sudo dpkg -i splunk.deb
```

Since the target server has no desktop environment, Splunk Web must be exposed to the network rather than accessed via `localhost`:

```bash
sudo nano /opt/splunk/etc/system/local/web.conf
```
```ini
[settings]
server.socket_host = 0.0.0.0
```

```bash
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
sudo /opt/splunk/bin/splunk enable boot-start
sudo ufw allow 8000/tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

Splunk Web is then reachable at `http://<ubuntu-ip>:8000` from any machine on the LAN.

### Standard log input — auth.log

Every attack in this repo starts from the same baseline input: SSH/auth events on Ubuntu live in `/var/log/auth.log` (not `/var/log/secure`, which is the RHEL/CentOS path):

```bash
sudo /opt/splunk/bin/splunk add monitor /var/log/auth.log \
  -index main \
  -sourcetype linux_secure
```

> Setup problems specific to each lab (disk space, network bridging, clock sync, etc.) are documented in that lab's own README, not duplicated here.

---

## 🔗 Related Projects

- [CrowdSec Home Lab](https://github.com/vertilusfrancy-commits/crowdsec-home-lab) — the same kind of attack/detection lab, built with CrowdSec instead of Splunk, including a finding that CrowdSec's default RFC 1918 whitelist suppresses detection of LAN-originated attacks

---

## ⚠️ Disclaimer

All labs in this repo were built in isolated virtual environments for educational purposes only. Every attack simulation was performed against systems owned and controlled by the author.
