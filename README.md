# 🛡️ Splunk Home Lab

A home lab using Splunk Free to simulate, detect, and analyze real-world cyberattacks. Each attack is documented with setup instructions, SPL detection queries, dashboards, and automated alerts — built to demonstrate SOC analyst skills.

---

## 🧪 Lab Environment

| Component | Details |
|---|---|
| SIEM | Splunk Enterprise Free (v10.2.1) |
| Target / Splunk Host | Ubuntu 22.04 Server (headless) |
| Attacker | Kali Linux |
| Virtualization | VirtualBox (Ubuntu) + UTM (Kali on macOS) |

---

## 📁 Repository Structure

```
splunk-home-lab/
│
├── README.md                        # This file
└── attacks/
    ├── ssh-brute-force/
    │   ├── README.md                # Full documentation
    │   ├── queries/
    │   │   ├── failed_logins.spl
    │   │   ├── brute_force_by_ip.spl
    │   │   ├── attack_timeline.spl
    │   │   └── top_targeted_users.spl
    │   └── screenshots/
    │       ├── hydra_attack.png
    │       ├── spl_failed_logins.png
    │       ├── spl_brute_force_by_ip.png
    │       ├── spl_attack_timeline.png
    │       ├── spl_top_users.png
    │       ├── dashboard_full.png
    │       └── alert_created.png
    └── (more attacks coming)
```

---

## ⚔️ Attacks Documented

| # | Attack | Tools | Status |
|---|---|---|---|
| 1 | [SSH Brute Force](./attacks/ssh-brute-force/README.md) | Hydra | ✅ Complete |
| 2 | Port Scanning | Nmap | 🔜 Coming soon |
| 3 | More TBD | — | 🔜 Coming soon |

---

## ⚙️ Splunk Setup

### Installation on Ubuntu 22.04 (headless)

```bash
# Download Splunk Enterprise Free (.deb)
wget -O splunk.deb "https://download.splunk.com/products/splunk/releases/10.2.1/linux/splunk-10.2.1-c892b66d163d-linux-amd64.deb"

# Install
sudo dpkg -i splunk.deb

# Configure to accept external connections
sudo nano /opt/splunk/etc/system/local/web.conf
```

`web.conf` content:
```ini
[settings]
server.socket_host = 0.0.0.0
```

```bash
# Start Splunk
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root

# Enable on boot
sudo /opt/splunk/bin/splunk enable boot-start

# Open firewall ports
sudo ufw allow 8000/tcp
sudo ufw allow 22/tcp
sudo ufw enable
```

> Splunk Web is accessible from any machine on the network at `http://<ubuntu-ip>:8000`

### Log Input — auth.log

```bash
sudo /opt/splunk/bin/splunk add monitor /var/log/auth.log \
  -index main \
  -sourcetype linux_secure
```

---

## 🔗 Related Projects

- [CrowdSec Home Lab](https://github.com/vertilusfrancy-commits/crowdsec-home-lab) — SSH brute force detection with CrowdSec, including RFC 1918 whitelist analysis

---

## ⚠️ Disclaimer

This lab was built in an isolated virtual environment for educational purposes only. All attack simulations were performed against systems owned and controlled by the author.
