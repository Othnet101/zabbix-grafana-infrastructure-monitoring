# 📊 Zabbix + Grafana Infrastructure Monitoring

A hands-on lab that builds a complete monitoring stack using **Zabbix 7.0 LTS** and **Grafana** on two Ubuntu 24.04 VMs running in VMware Workstation.

Zabbix collects metrics, stores them, and raises alerts. Grafana reads from the Zabbix API and renders live dashboards.

---

## 🧰 Stack & Versions

| Component | Version |
|-----------|---------|
| OS (both VMs) | Ubuntu 24.04.3 LTS |
| Hypervisor | VMware Workstation (NAT — VMnet8) |
| Zabbix Server | 7.0 LTS |
| Zabbix Agent | Agent 2 (Go-based) |
| Database | MySQL (bundled with Zabbix) |
| Web server | Apache 2 |
| Grafana | Latest stable (≥ 11.6) |
| Grafana plugin | alexanderzobnin-zabbix-app |

---

## 🗺️ Architecture

```
INTERNET / Your PC
      │
      │  VMware NAT (VMnet8)
      ├─────────────────────────────────────────────────┐
      │                                                 │
  VM 1 — zbx-server (Ubuntu Server 24.04)         VM 2 — zbx-desktop (Ubuntu Desktop 24.04)
  ┌──────────────────────────────────┐             ┌─────────────────────────────┐
  │  Zabbix Server     :10051        │◄──TCP 10050─┤  Zabbix Agent 2   :10050    │
  │  MySQL DB          :3306 (local) │             │                             │
  │  Apache + Zabbix UI:80           │             │  Firefox (browser)          │
  │  Grafana           :3000         │◄──API───────│    http://SERVER_IP/zabbix  │
  │  Zabbix Agent 2    :10050        │             │    http://SERVER_IP:3000    │
  └──────────────────────────────────┘             └─────────────────────────────┘
```

> Grafana runs on the **same server** and reads Zabbix data through the Zabbix API (`/zabbix/api_jsonrpc.php`). The Desktop VM is both the **supervised machine** and the **browser client**.

---

## 🎯 Lab Goals

- [x] Both VMs communicate over VMnet8 (NAT)
- [x] Zabbix server + MySQL + Apache web UI running
- [x] Zabbix Agent 2 installed and configured on the Desktop VM
- [x] Host visible with a green **ZBX** icon in the web UI
- [x] Real alert triggered via CPU stress test (`stress-ng`)
- [x] Grafana dashboard with live CPU, RAM, disk, uptime and active problems

---

## ⚙️ VM Specifications

| VM | CPU | RAM | Disk | Role |
|----|-----|-----|------|------|
| zbx-server (Ubuntu Server) | 2 vCPU | 4 GB | 25 GB | Monitoring server |
| zbx-desktop (Ubuntu Desktop) | 2 vCPU | 4 GB | 25 GB | Supervised machine + browser |

---

## 🚀 Setup Steps

### Step 0 — Prepare the VMs

```bash
# On BOTH VMs
sudo apt update && sudo apt -y upgrade

# Set hostnames
sudo hostnamectl set-hostname zbx-server    # on the Server VM
sudo hostnamectl set-hostname zbx-desktop   # on the Desktop VM

# Find your IP address (look for 192.168.x.y)
ip -br a

# Verify connectivity
ping -c 3 <SERVER_IP>    # run on Desktop
ping -c 3 <DESKTOP_IP>   # run on Server
```

---

### Step 1 — Install Zabbix Server (on zbx-server)

```bash
# 1.1 Add the Zabbix 7.0 official repository
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update

# 1.2 Install all components
sudo apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf \
     zabbix-sql-scripts zabbix-agent2 zabbix-get mysql-server

zabbix_server --version    # verify: 7.0.x

# 1.3 Create the database
sudo mysql <<'EOF'
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user 'zabbix'@'localhost' identified by 'ZbxLab_2026_Pass';
grant all privileges on zabbix.* to 'zabbix'@'localhost';
set global log_bin_trust_function_creators = 1;
quit;
EOF

# 1.4 Import the schema (takes ~2 minutes)
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz \
  | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
sudo mysql -e "set global log_bin_trust_function_creators = 0;"

# 1.5 Set the DB password in zabbix_server.conf
sudo sed -i 's/^# DBPassword=/DBPassword=ZbxLab_2026_Pass/' /etc/zabbix/zabbix_server.conf

# 1.6 Start and enable services
sudo systemctl restart zabbix-server zabbix-agent2 apache2
sudo systemctl enable  zabbix-server zabbix-agent2 apache2
systemctl status zabbix-server --no-pager    # expect: active (running)
```

#### Optional — firewall rules (if ufw is active)

```bash
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # Zabbix web UI
sudo ufw allow 3000/tcp    # Grafana
sudo ufw allow 10051/tcp   # Zabbix server (active agents)
```

---

### Step 2 — Zabbix Web Wizard (from Desktop Firefox)

Go to `http://<SERVER_IP>/zabbix` and complete the 6-screen wizard:

| Screen | Settings |
|--------|----------|
| Welcome | Language: English |
| Prerequisites | All lines must say **OK** |
| Database | MySQL · localhost · port 0 · name `zabbix` · user `zabbix` · password from Step 1 |
| Settings | Server name: `Lab` · set your timezone |
| Summary | Review, click **Next step** |
| Install | Click **Finish** |

Then log in with **Admin / zabbix** and immediately change the password.

---

### Step 3 — Zabbix Agent on zbx-desktop

```bash
# 3.1 Add repo and install
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update && sudo apt install -y zabbix-agent2

# 3.2 Configure (replace SERVER_IP with your real value)
sudo sed -i 's/^Server=.*/Server=<SERVER_IP>/'             /etc/zabbix/zabbix_agent2.conf
sudo sed -i 's/^ServerActive=.*/ServerActive=<SERVER_IP>/' /etc/zabbix/zabbix_agent2.conf
sudo sed -i 's/^Hostname=.*/Hostname=zbx-desktop/'         /etc/zabbix/zabbix_agent2.conf

# Verify
grep -E '^(Server|ServerActive|Hostname)=' /etc/zabbix/zabbix_agent2.conf

# 3.3 Start
sudo systemctl enable --now zabbix-agent2
ss -tlnp | grep 10050    # a line = agent is listening

# 3.4 Test from the SERVER (not the desktop)
zabbix_get -s <DESKTOP_IP> -k agent.ping           # → 1
zabbix_get -s <DESKTOP_IP> -k system.hostname      # → zbx-desktop
zabbix_get -s <DESKTOP_IP> -k system.cpu.load[all,avg1]
```

Then in the Zabbix web UI: **Data collection › Hosts › Create host**

| Field | Value |
|-------|-------|
| Host name | `zbx-desktop` |
| Templates | `Linux by Zabbix agent` |
| Host groups | `Linux servers` |
| Interfaces | Agent · IP `<DESKTOP_IP>` · port `10050` |

---

### Step 4 — Trigger a Real Alert

```bash
# CPU stress test (on Desktop) — runs for 10 minutes
sudo apt install -y stress-ng
stress-ng --cpu 0 --timeout 600s

# In another terminal, watch the load:
top

# Agent availability test: stop then restart
sudo systemctl stop zabbix-agent2    # wait → ZBX turns red
sudo systemctl start zabbix-agent2   # wait → resolves automatically
```

Watch **Monitoring › Problems** in the Zabbix UI for the alert.

---

### Step 5 — Install Grafana (on zbx-server)

```bash
# 5.1 Add Grafana repository
sudo apt install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | \
   sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
   sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install -y grafana
sudo systemctl enable --now grafana-server

# 5.2 Install the Zabbix plugin
sudo grafana-cli plugins install alexanderzobnin-zabbix-app
sudo systemctl restart grafana-server
```

Then in Firefox → `http://<SERVER_IP>:3000` (login: **admin / admin**, change on first login):

1. **Administration › Plugins** → search `Zabbix` → **Enable**
2. **Connections › Data sources › Add** → choose `Zabbix`
3. Fill in:

| Field | Value |
|-------|-------|
| URL | `http://localhost/zabbix/api_jsonrpc.php` |
| Auth type | Username and password |
| Username / Password | Your Zabbix Admin credentials |

4. Click **Save & test** → green ✅ with Zabbix API version = success.

---

### Step 6 — Build the Dashboard

Create a new dashboard (**Dashboards › New › New dashboard**) and add these panels:

| Panel | Type | Query (Group / Host / Item) |
|-------|------|-----------------------------|
| CPU utilization | Time series | Linux servers / zbx-desktop / CPU utilization |
| Memory used | Gauge (0–100%) | Linux servers / zbx-desktop / Memory utilization |
| Uptime | Stat (seconds) | Linux servers / zbx-desktop / Uptime |
| Disk used on / | Bar gauge | Linux servers / zbx-desktop / Space: Used, in % on / |
| Active problems | Table (Problems) | Group: Linux servers · Host: zbx-desktop |

Set time range to **Last 1 hour**, refresh to **10 s**.

---

## 🔧 Key Ports Reference

| Port | Service | Direction |
|------|---------|-----------|
| 10050 | Zabbix Agent 2 (passive) | Server → Agent |
| 10051 | Zabbix Server (active) | Agent → Server |
| 80 | Zabbix Web UI (Apache) | Browser → Server |
| 3000 | Grafana | Browser → Server |
| 3306 | MySQL | Local only |

---

## 🩺 Troubleshooting Cheat Sheet

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| ZBX icon red or grey | Agent stopped / wrong IP / firewall / wrong `Server=` | `zabbix_get -s <DESKTOP_IP> -k agent.ping` from server |
| "Zabbix server is not running" on dashboard | Server crashed or wrong DBPassword | `systemctl status zabbix-server` → check log |
| Web page doesn't open | Apache stopped or wrong URL | `systemctl status apache2`; use `/zabbix` suffix |
| Items show "Not supported" | Wrong key or agent can't read metric | Hover red icon; test with `zabbix_get` |
| Active checks give no data | `Hostname=` doesn't match host name in UI | Make both identical exactly; check agent log |
| Grafana plugin not in list | Not restarted or not enabled | `systemctl restart grafana-server` then enable in Plugins |
| Data source test fails | Wrong URL or credentials | URL must end with `/api_jsonrpc.php` |

---

## 📁 Repo Structure

```
zabbix-grafana-infrastructure-monitoring/
├── configs/
│   ├── zabbix_server.conf.example     # Server config key lines
│   └── zabbix_agent2.conf.example     # Agent config (Desktop VM)
├── screenshots/                       # Your lab screenshots go here
│   └── README.md
├── docs/                              # Extra documentation
└── README.md
```

---

## 📚 References

- [Zabbix 7.0 Documentation](https://www.zabbix.com/documentation/7.0/)
- [Grafana Documentation](https://grafana.com/docs/)
- [alexanderzobnin-zabbix-app plugin](https://grafana.com/grafana/plugins/alexanderzobnin-zabbix-app/)
