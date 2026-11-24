# Zabbix + Grafana Infrastructure Monitoring

A hands-on lab that builds a complete monitoring stack using **Zabbix 7.0 LTS** and **Grafana** on two Ubuntu 24.04 VMs running in VMware Workstation.

Zabbix collects metrics, stores them, and raises alerts. Grafana reads from the Zabbix API and renders live dashboards.

---

## Componennts & Versions

| Component | Version |
|-----------|---------|
| OS (both VMs) | Ubuntu 24.04.3 LTS |
| Hypervisor | VMware Workstation (NAT — VMnet8) |
| Zabbix Server | 7.0 LTS |
| Zabbix Agent | Agent 2 |
| Database | MySQL |
| Web server | Apache 2 |
| Grafana | Latest stable (>= 11.6) |
| Grafana plugin | alexanderzobnin-zabbix-app |

---

## Architecture

```
INTERNET / Your PC
      │
      │  VMware NAT (VMnet8)
      ├─────────────────────────────────────────────────┐
      │                                                 │
  VM 1 — ubuntus (Ubuntu Server 24.04)         VM 2 — ubuntud (Ubuntu Desktop 24.04)
  ┌──────────────────────────────────┐             ┌─────────────────────────────┐
  │  Zabbix Server     :10051        │◄──TCP 10050─┤  Zabbix Agent 2   :10050    │
  │  MySQL DB          :3306 (local) │             │                             │
  │  Apache +          :80           │             |    WEB UI                   |
  │  Grafana           :3000         │◄──API───────│    http://SERVER_IP/zabbix  │
  │  Zabbix Agent 2    :10050        │             │    http://SERVER_IP:3000    │
  └──────────────────────────────────┘             └─────────────────────────────┘
```

> Grafana runs on the **same server** and reads Zabbix data through the Zabbix API (`/zabbix/api_jsonrpc.php`). The Desktop VM is both the **supervised machine** and the **browser client**.

---

## Lab Goals

- [x] Zabbix server + MySQL + Apache web UI running
- [x] Zabbix Agent 2 installed and configured on the Desktop VM
- [x] Host visible and available to supervise in the web UI
- [x] Real alert triggered via CPU stress test (`stress-ng`)
- [x] Grafana dashboard with live CPU Usage & Problems

---

## VM Specifications

| VM | CPU | RAM | Disk | Role |
|----|-----|-----|------|------|
| ubuntus (Ubuntu Server) | 2 vCPU | 4 GB | 60 GB | Monitoring server |
| ubuntud (Ubuntu Desktop) | 3 vCPU | 6 GB | 60 GB | Supervised machine + browser |

---

## Setup Steps

### Step 0 — Prepare the VMs

```bash
# On BOTH VMs set hostnames
sudo hostnamectl set-hostname ubuntus    # Server 
sudo hostnamectl set-hostname ubuntud   # Desktop

# Verify connectivity
ping -c 3 <SERVER_IP>    
ping -c 3 <DESKTOP_IP>   
```

---

### Step 1 — Install Zabbix Server (on ubuntus)

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
create user 'zabbix'@'localhost' identified by '"your password"';
grant all privileges on zabbix.* to 'zabbix'@'localhost';
set global log_bin_trust_function_creators = 1;
quit;
EOF

# 1.4 Import the schema 
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz \
  | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
sudo mysql -e "set global log_bin_trust_function_creators = 0;"

# 1.5 Set the DB password in zabbix_server.conf
sudo sed -i 's/^# DBPassword=/DBPassword="your password"/' /etc/zabbix/zabbix_server.conf

# 1.6 Start and enable services
sudo systemctl restart zabbix-server zabbix-agent2 apache2
sudo systemctl enable  zabbix-server zabbix-agent2 apache2
systemctl status zabbix-server --no-pager    # expect: active (running)
```

#### firewall rules (if ufw is active)

```bash
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # Zabbix web UI
sudo ufw allow 3000/tcp    # Grafana
sudo ufw allow 10051/tcp   # Zabbix server 
```

---

### Step 2 — Zabbix Web Wizard (from Desktop Browser)

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

### Step 3 — Zabbix Agent on ubuntud

```bash
# 3.1 Add repo and install
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update && sudo apt install -y zabbix-agent2

# 3.2 Configure "zabbix_agent2.conf"
sudo sed -i 's/^Server=.*/Server=<SERVER_IP>/'             /etc/zabbix/zabbix_agent2.conf
sudo sed -i 's/^ServerActive=.*/ServerActive=<SERVER_IP>/' /etc/zabbix/zabbix_agent2.conf
sudo sed -i 's/^Hostname=.*/Hostname=ubuntud/'         /etc/zabbix/zabbix_agent2.conf

# Verify
grep -E '^(Server|ServerActive|Hostname)=' /etc/zabbix/zabbix_agent2.conf

# 3.3 Start
sudo systemctl enable --now zabbix-agent2
ss -tlnp | grep 10050    # test if agent is listening

# 3.4 Test from the SERVER
zabbix_get -s <DESKTOP_IP> -k agent.ping           # output = 1
zabbix_get -s <DESKTOP_IP> -k system.hostname      # → ubuntud
zabbix_get -s <DESKTOP_IP> -k system.cpu.load[all,avg1]
```

Then in the Zabbix web UI: **Data collection › Hosts › Create host**

| Field | Value |
|-------|-------|
| Host name | `ubuntud` |
| Templates | `Linux by Zabbix agent` |
| Host groups | `Linux servers` |
| Interfaces | Agent · IP `<DESKTOP_IP>` · port `10050` |

---

### Step 4 — Trigger a Real Alert

```bash
# CPU stress test 
sudo apt install -y stress-ng
stress-ng --cpu 0 --timeout 600s

# Agent availability test: stop then restart
sudo systemctl stop zabbix-agent2    # wait = ZBX turns red
sudo systemctl start zabbix-agent2   # wait = resolves automatically
```

Watch **Monitoring › Problems** in the Zabbix UI for the alert.

---

### Step 5 — Install Grafana (on ubuntus)

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

Then in browser : `http://<SERVER_IP>:3000` (login: **admin / admin**, change on first login):

1. **Administration › Plugins** → search `Zabbix` → **Enable**
2. **Connections › Data sources › Add** → choose `Zabbix`
3. Fill in:

| Field | Value |
|-------|-------|
| URL | `http://localhost/zabbix/api_jsonrpc.php` |
| Auth type | Username and password |
| Username / Password | Your Zabbix Admin credentials |

4. Click **Save & test** = green with Zabbix API version = success.

---

### Step 6 — Build the Dashboard

Create a new dashboard (**Dashboards › New › New dashboard**) and add these panels:

| Panel | Type | Query (Group / Host / Item) |
|-------|------|-----------------------------|
| CPU utilization | Time series | Linux servers / ubuntud / CPU utilization |
| Memory used | Gauge (0–100%) | Linux servers / ubuntud / Memory utilization |
| Uptime | Stat (seconds) | Linux servers / ubuntud / Uptime |
| Disk used on / | Bar gauge | Linux servers / ubuntud / Space: Used, in % on / |
| Active problems | Table (Problems) | Group: Linux servers · Host: ubuntud |

Set time range to **Last 1 hour**, refresh to **10 s**.

---

## 📚 References

- [Zabbix 7.0 Documentation](https://www.zabbix.com/documentation/7.0/)
- [Grafana Documentation](https://grafana.com/docs/)
- [alexanderzobnin-zabbix-app plugin](https://grafana.com/grafana/plugins/alexanderzobnin-zabbix-app/)
