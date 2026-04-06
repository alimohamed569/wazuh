
# SOC-Pentest LAB
## ⚙️ Lab Environment Setup

---

## 📌 Phase One: Install VMs and Requirements

### 🖥️ VM Specifications & Network

| VM             | vCPU | RAM   | Disk | Internet Access    | Bridged IP    |
| -------------- | ---- | ----- | ---- | ------------------ | ------------- |
| Ubuntu Server  | 2    | 6 GB  | 40GB | Yes (via Bridged)  | 192.168.1.50  |
| Windows Server | 2    | 6 GB  | 60GB | Yes (via Bridged)  | 192.168.1.60  |
| Windows 10     | 2    | 4 GB  | 50GB | Yes (via Bridged)  | 192.168.1.70  |
| Kali Linux     | 2    | 2–4GB | 40GB | Yes (via Bridged)  | 192.168.1.80  |

---

## 📌 Phase Two: Ubuntu Server Setup (Snort, Suricata, and Splunk)

- **Snort** → Network Intrusion Detection System (NIDS) to monitor VM traffic (Kali ↔ Windows ↔ Server) and generate alerts for attacks/scans.  
- **Suricata** → Multi-threaded NIDS/IDS that can output JSON logs — good for detailed packet logging and comparison with Snort.  
- **Splunk** → Centralized platform to collect/index/visualize logs from Snort & Suricata.

---

### 🌐 Ubuntu Network Setup

1. Identify network interfaces:
   ```bash
   ip a
   ```
   > Example: `enp0s3` → Bridged interface

2. Create or edit Netplan configuration:
   ```bash
   sudo nano /etc/netplan/01-network.yaml
   ```
   (Adjust values according to your lab network)  
   ![Netplan Config](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/yamlFile.png)

3. Apply changes:
   ```bash
   sudo netplan apply
   ```
   ✅ Test: Ping other hosts and `google.com` (via Bridged).

---

### 🔑 SSH Setup (Ubuntu Server)

1. Install SSH:
   ```bash
   sudo apt install -y openssh-server
   ```

2. Enable & start service:
   ```bash
   sudo systemctl start ssh
   sudo systemctl enable ssh
   sudo systemctl status ssh
   ```
   ![SSH Status](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/SSH.png)

3. Allow SSH through UFW:
   ```bash
   sudo ufw allow ssh
   sudo ufw reload
   ```

---

### 🛡️ Installing Suricata on Ubuntu

Follow the steps below to install and verify **Suricata** on Ubuntu.

#### 1️⃣ Update & Upgrade the System
```bash
sudo apt update && sudo apt upgrade -y
```

#### 2️⃣ Add Suricata Stable PPA & Update
```bash
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt update
```

#### 3️⃣ Install Suricata
```bash
sudo apt install suricata -y
```

#### 4️⃣ Verify Installation
```bash
suricata --build-info
```
✔️ If successful, you'll see build information (version, features, libraries).

#### 5️⃣ Enable & Start Suricata Service
```bash
sudo systemctl enable suricata
sudo systemctl start suricata
sudo systemctl status suricata
```
  ![Suricata_Service_Enable](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/Suricata_Service_Enable.png?raw=true)

  #### 6️⃣ Lets check if suricata running correctly 

```bash
sudo journalctl -u suricata --follow
```
 ![Suricata_Service_log](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/Suricata_Service_log.png?raw=true)

Got it! I can rewrite your Suricata instructions for your GitHub README and add Snort afterward as steps **7, 8, 9** in a clean, professional format. Here's how it could look:


#### 7️⃣ Suricata – Adding a Custom ICMP Rule

Create a new rules file called `local.rules`:

```bash
sudo nano /var/lib/suricata/rules/local.rules
```

Add the ICMP alert rule:

```text
alert icmp any any -> any any (msg:"ICMP Ping Detected"; sid:1000001; rev:1;)
```

> ⚡ Rule syntax:
> `alert <protocol> <src_ip> <src_port> -> <dst_ip> <dst_port> (<options>)`


#### 8️⃣ Update Suricata Configuration

Edit the main Suricata configuration to include your new rule file:

```bash
sudo nano /etc/suricata/suricata.yaml
```

Make sure these lines are set:

```yaml
default-rule-path: /var/lib/suricata/rules
rule-files:
  - local.rules
```

Save and exit.


#### 9️⃣ Test Suricata

Ping any VM in your network to trigger the ICMP alert, then check the logs:

```bash
tail -f /var/log/suricata/fast.log
```

You should see entries like:

```
[**] [1:1000001:1] ICMP Ping Detected [**]
```

---

## 🐍 Installing Snort on Ubuntu

#### 1️⃣ Update System
```bash
sudo apt update && sudo apt upgrade -y
```

#### 2️⃣ Install Snort
```bash
sudo apt install snort -y
```

#### 3️⃣ Check Interface
```bash
ip a
```

#### 4️⃣ Create Local Rules File
```bash
cd /etc/snort/rules/
sudo nano /etc/snort/rules/local.rules
```

Add rules:
```snort
alert icmp $HOME_NET any -> any any (msg:"Ping Detected on Ubuntu Server"; sid:1000001; rev:1;)
alert icmp any any -> any any (msg:"ICMP Ping Detected"; sid:1000002; rev:1; classtype:icmp-event;)
alert tcp any any -> $HOME_NET 22 (msg:"SSH connection attempt"; flags:S; sid:1000003; rev:1; threshold: type threshold, track by_src, count 10, seconds 120;)
```

#### 5️⃣ Edit Snort Config
```bash
sudo nano /etc/snort/snort.conf
```
Ensure:
```conf
include $RULE_PATH/local.rules
```

#### 6️⃣ Start Snort
```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i enp0s3
```

#### 7️⃣ Test ICMP Rule
```bash
ping 192.168.1.50
```

Expected output:
```
[**] [1:1000001:1] Ping Detected on Ubuntu Server [**]
```

#### 8️⃣ Test SSH Rule
From PowerShell (Windows):
```powershell
for ($i=1; $i -le 10; $i++) {
  ssh -o ConnectTimeout=2 -o BatchMode=yes admin@192.168.1.50
}
```

 ![SSH_Brute_Force](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/SHH-brute-force.png?raw=true)

Expected output:
```
[**] [1:1000003:1] SSH connection attempt [**]
```
 ![SSH_Brute_Force_Alert](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/ssh_brute_froce_alert.png?raw=true)

### Nessus Vulnerability Scanner

### 1. Download the Package

```bash
curl --request GET \
  --url 'https://www.tenable.com/downloads/api/v2/pages/nessus/files/Nessus-10.11.1-ubuntu1604_amd64.deb' \
  --output 'Nessus-10.11.1-ubuntu1604_amd64.deb'
```

### 2. Installation

```bash
# Install the package
sudo dpkg -i Nessus-10.11.1-ubuntu1604_amd64.deb

# Fix potential dependency issues
sudo apt --fix-broken install -y
```

### 3. Service Management

```bash
sudo systemctl start nessusd
sudo systemctl enable nessusd
sudo systemctl status nessusd
```
 ![nessus_service](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/neesus_service.png?raw=true)

### 4. Network & Firewall Configuration

**Verify Listening Port:**

```bash
sudo netstat -tulnp | grep 8834
```

**Configure Firewall:**

```bash
sudo ufw allow 8834/tcp
sudo ufw reload
sudo ufw status
```

### 5. Web Access and First Scan

```
https://<ServerIP>:8834
```

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `admin` |

 ![nessus_First_Scan](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/First_Scan_Nessus.png?raw=true)
---

## DVWA + Apache + ModSecurity WAF

### 🔧 Install Apache

```bash
sudo apt install apache2 -y
sudo systemctl enable apache2 && sudo systemctl start apache2
```

### 🐘 Install PHP + Extensions

```bash
sudo apt install -y php php-mysqli php-gd php-xml php-mbstring php-curl
```

**Enable mod_rewrite** (required for DVWA routing):

```bash
sudo a2enmod rewrite && sudo systemctl restart apache2
```

> **Why mod_rewrite?**  
> When you open `http://localhost/dvwa/vulnerabilities/sqli/`, Apache needs to rewrite that path and hand it to the correct PHP script. Without `mod_rewrite`, Apache can't process `.htaccess` rewrite rules and returns **404 Not Found**.

### 🗄️ Install MySQL

```bash
sudo apt install -y mysql-server
sudo systemctl enable mysql && sudo systemctl start mysql
```

**Create DVWA database and user:**

```bash
sudo mysql -u root -e "
  CREATE DATABASE dvwa;
  CREATE USER 'admin'@'localhost' IDENTIFIED BY 'admin';
  GRANT ALL PRIVILEGES ON dvwa.* TO 'admin'@'localhost';
  FLUSH PRIVILEGES;
"
```

### 🎯 Clone & Configure DVWA

```bash
cd /var/www/html && sudo git clone https://github.com/digininja/DVWA.git dvwa
sudo cp /var/www/html/dvwa/config/config.inc.php.dist /var/www/html/dvwa/config/config.inc.php
sudo nano /var/www/html/dvwa/config/config.inc.php
```

**Edit these values in the config file:**

```php
$_DVWA['db_user']              = 'admin';
$_DVWA['db_password']          = 'admin';
$_DVWA['db_database']          = 'dvwa';
$_DVWA['default_security_level'] = 'low';
```

**Set correct permissions:**

```bash
sudo chown -R www-data:www-data /var/www/html/dvwa
sudo chmod -R 755 /var/www/html/dvwa
```

Then complete setup via browser at `http://localhost/dvwa/setup.php`.

 ![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/DVWA_Setup.png?raw=true)
![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/dvwa2.webp?raw=true)
---

### 🛡️ Install & Configure ModSecurity

```bash
sudo apt install -y libapache2-mod-security2
sudo a2enmod security2 && sudo systemctl restart apache2
```

**Enable ModSecurity:**

```bash
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
```

**Route logs to Apache error log:**

```bash
sudo sed -i 's|SecAuditLog /var/log/modsec_audit.log|SecAuditLog /var/log/apache2/error.log|' \
  /etc/modsecurity/modsecurity.conf
```

**Switch from detection-only to blocking mode:**

```bash
sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/modsecurity/modsecurity.conf
sudo systemctl restart apache2
```

### 📦 Download OWASP Core Rule Set (CRS)

```bash
cd /tmp && wget https://github.com/coreruleset/coreruleset/archive/refs/tags/v3.3.5.tar.gz
tar -xvf v3.3.5.tar.gz && sudo mv coreruleset-3.3.5 /etc/modsecurity/crs
sudo cp /etc/modsecurity/crs/crs-setup.conf.example /etc/modsecurity/crs/crs-setup.conf
```

**Link CRS to Apache** (`/etc/apache2/mods-enabled/security2.conf`):

```apache
<IfModule security2_module>
    IncludeOptional /etc/modsecurity/crs/crs-setup.conf
    IncludeOptional /etc/modsecurity/crs/rules/*.conf
</IfModule>
```

```bash
sudo systemctl restart apache2
```

### 🧪 Test WAF Rules

**View ModSecurity logs:**

```bash
sudo tail -f /var/log/apache2/error.log | grep ModSecurity
```

**SQL Injection:**

```bash
curl -v "http://localhost/dvwa/vulnerabilities/sqli/?id=1'+OR+'1'='1&Submit=Submit"
```
![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/TEST_Sql.png?raw=true) 

**XSS:**

```bash
curl -v "http://localhost/dvwa/vulnerabilities/xss_r/?name=<script>alert(1)</script>"
```
![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/TEST%20XSS.png?raw=true) 


---

## Wazuh SIEM & Custom Rules

### 📥 Setup Wazuh (OVA)

Download the official OVA from:  
👉 [Wazuh Virtual Machine Documentation](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html)

**SSH into the Wazuh server:**

```bash
ssh wazuh-user@<WAZUH_SERVER_IP>
# password: wazuh
```

**Set a static IP** (before anything else):

```bash
sudo nano /etc/systemd/network/10-cloud-init-eth0.network
```

**Access the Wazuh dashboard:**

```
URL:      https://<WAZUH_SERVER_IP>
Username: admin
Password: admin
```
![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/WAZUH_Dashboard.png?raw=true) 

---

### 🔗 Connect Ubuntu Agent to Wazuh

![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/ADD_Agent.png?raw=true) 

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.2-1_amd64.deb \
  && sudo WAZUH_MANAGER='192.168.1.50' WAZUH_AGENT_NAME='Ubuntu-Server' \
  dpkg -i ./wazuh-agent_4.14.2-1_amd64.deb

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

### 📡 Log Collection Sources

Wazuh collects logs from:
- **Snort** — Network IDS
- **Suricata** — Network IDS/IPS
- **ModSecurity WAF** — Web application firewall events

Triggered detection examples:
- ✅ SSH Brute-Force (from IDS rule)
- ✅ ICMP flood detection
- ✅ SQL injecion 
- ✅ XSS
---
WAF triggered 
![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/Wazuh_rules_trigger.png?raw=true) 
![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/Wazuh_events.png?raw=true) 

IDS triggerd
![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/Wazuh_SSHIDS_Trigger.jpeg?raw=true) 
![DVWA](https://github.com/MohamedAshrafElRokh/SOC-Pentest-LAB/blob/main/Images/ICMP_Wazuh.jpeg?raw=true) 

### 📜 Custom Wazuh Rules — ModSecurity CRS

The following rules map ModSecurity CRS events to Wazuh alerts with MITRE ATT&CK tagging:

<details>
<summary>Click to expand full ruleset</summary>

```xml
<!-- ======================== -->
<!-- ModSecurity CRS Rules   -->
<!-- ======================== -->
<group name="modsecurity,web,attack,">

  <!-- Base catch rule -->
  <rule id="100101" level="6">
    <if_sid>30411</if_sid>
    <match>ModSecurity</match>
    <description>ModSecurity: Event detected</description>
    <group>modsecurity,</group>
  </rule>

  <!-- Access Denied / Blocked -->
  <rule id="100102" level="10">
    <if_sid>100101</if_sid>
    <match>Access denied</match>
    <description>ModSecurity: Access denied - request blocked</description>
    <group>modsecurity,blocked,</group>
    <mitre><id>T1190</id></mitre>
  </rule>

  <!-- REQUEST-913: Scanner Detection -->
  <rule id="100913" level="10">
    <if_sid>100101</if_sid>
    <match>id "913</match>
    <description>ModSecurity CRS 913: Security scanner or automated tool detected</description>
    <group>modsecurity,scanner,recon,</group>
    <mitre><id>T1595</id></mitre>
  </rule>

  <!-- REQUEST-930: LFI -->
  <rule id="100930" level="12">
    <if_sid>100101</if_sid>
    <match>id "930</match>
    <description>ModSecurity CRS 930: Local File Inclusion (LFI) attack detected</description>
    <group>modsecurity,lfi,web_attack,</group>
    <mitre><id>T1083</id></mitre>
  </rule>

  <!-- REQUEST-932: RCE -->
  <rule id="100932" level="15">
    <if_sid>100101</if_sid>
    <match>id "932</match>
    <description>ModSecurity CRS 932: Remote Code Execution (RCE) attack detected</description>
    <group>modsecurity,rce,critical,web_attack,</group>
    <mitre><id>T1059</id></mitre>
  </rule>

  <!-- REQUEST-941: XSS -->
  <rule id="100941" level="11">
    <if_sid>100101</if_sid>
    <match>id "941</match>
    <description>ModSecurity CRS 941: Cross-Site Scripting (XSS) attack detected</description>
    <group>modsecurity,xss,web_attack,</group>
    <mitre><id>T1059.007</id></mitre>
  </rule>

  <!-- REQUEST-942: SQLi -->
  <rule id="100942" level="14">
    <if_sid>100101</if_sid>
    <match>id "942</match>
    <description>ModSecurity CRS 942: SQL Injection (SQLi) attack detected</description>
    <group>modsecurity,sqli,web_attack,</group>
    <mitre><id>T1190</id></mitre>
  </rule>

  <!-- REQUEST-949: Inbound Anomaly Score Exceeded -->
  <rule id="100949" level="14">
    <if_sid>100101</if_sid>
    <match>id "949</match>
    <description>ModSecurity CRS 949: Inbound anomaly score exceeded - request blocked</description>
    <group>modsecurity,anomaly,blocked,</group>
    <mitre><id>T1190</id></mitre>
  </rule>

  <!-- 949 + SQLi -->
  <rule id="100950" level="14">
    <if_sid>100949</if_sid>
    <match>vulnerabilities/sqli</match>
    <description>ModSecurity CRS 949: Anomaly block triggered by SQL Injection</description>
    <group>modsecurity,anomaly,sqli,blocked,</group>
    <mitre><id>T1190</id></mitre>
  </rule>

  <!-- 949 + XSS -->
  <rule id="100951" level="14">
    <if_sid>100949</if_sid>
    <match>vulnerabilities/xss</match>
    <description>ModSecurity CRS 949: Anomaly block triggered by XSS attack</description>
    <group>modsecurity,anomaly,xss,blocked,</group>
    <mitre><id>T1059.007</id></mitre>
  </rule>

  <!-- 949 + RCE -->
  <rule id="100952" level="15">
    <if_sid>100949</if_sid>
    <match>vulnerabilities/exec</match>
    <description>ModSecurity CRS 949: Anomaly block triggered by RCE attempt</description>
    <group>modsecurity,anomaly,rce,blocked,critical,</group>
    <mitre><id>T1059</id></mitre>
  </rule>

  <!-- 949 + Unix Shell Injection -->
  <rule id="100956" level="15">
    <if_sid>100949</if_sid>
    <match>unix-shell</match>
    <description>ModSecurity CRS 949: Anomaly block triggered by Unix shell injection</description>
    <group>modsecurity,anomaly,rce,unix_shell,blocked,critical,</group>
    <mitre><id>T1059.004</id></mitre>
  </rule>

  <!-- ======================== -->
  <!-- Frequency / Brute Force -->
  <!-- ======================== -->

  <!-- Multiple blocks from same IP -->
  <rule id="100200" level="14" frequency="5" timeframe="60">
    <if_matched_sid>100102</if_matched_sid>
    <same_source_ip />
    <description>ModSecurity: Multiple blocked requests from same IP - possible attack campaign</description>
    <group>modsecurity,brute_force,blocked,</group>
    <mitre><id>T1190</id></mitre>
  </rule>

  <!-- Repeated SQLi -->
  <rule id="100201" level="15" frequency="3" timeframe="60">
    <if_matched_sid>100950</if_matched_sid>
    <same_source_ip />
    <description>ModSecurity: Repeated SQL Injection attempts from same IP</description>
    <group>modsecurity,sqli,brute_force,</group>
    <mitre><id>T1190</id></mitre>
  </rule>

  <!-- Repeated RCE -->
  <rule id="100203" level="15" frequency="2" timeframe="120">
    <if_matched_sid>100952</if_matched_sid>
    <same_source_ip />
    <description>ModSecurity: Repeated RCE attempts from same IP - CRITICAL</description>
    <group>modsecurity,rce,brute_force,critical,</group>
    <mitre><id>T1059</id></mitre>
  </rule>

</group>
```

</details>

---

### 🔍 Rule Severity Reference

| Level | Severity | Example |
|-------|----------|---------|
| 6 | Low | ModSecurity event detected |
| 10 | Medium | Access denied / blocked |
| 12–13 | High | LFI, RFI, Scanner |
| 14–15 | Critical | SQLi, RCE, Brute Force |

---

## 🧰 Tools Used

| Tool | Version | Purpose |
|------|---------|---------|
| Nessus | 10.11.1 | Vulnerability scanning |
| Apache2 | Latest | Web server |
| DVWA | Latest | Vulnerable web app target |
| ModSecurity | 2.x | Web Application Firewall |
| OWASP CRS | 3.3.5 | WAF ruleset |
| Wazuh | 4.14.2 | SIEM & log analysis |
| MySQL | Latest | DVWA backend database |
