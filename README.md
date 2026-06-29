#Detecting Suspicious Network Traffic using Suricata

## 📋 Overview

This project demonstrates how an **Endpoint Detection and Response (EDR)** setup can detect suspicious network traffic using **Suricata IDS**, with alerts visualized through **Wazuh**. The lab simulates a port scanning attack (Nmap SYN Scan) against a Linux endpoint, walks through configuring Suricata as a network intrusion detection engine, and integrates its alerts into the Wazuh SIEM dashboard.

---

## 🎯 Objective

Understand how an EDR solution can detect suspicious network traffic using Suricata IDS and visualize alerts on Wazuh:
- Install and configure Suricata to monitor network traffic in real time
- Simulate a port scanning (reconnaissance) attack using Nmap
- Detect the scan using Suricata signatures
- View and correlate the resulting alerts in the Wazuh dashboard

---

## 📚 Background

### What is Suspicious Network Traffic?
Abnormal or unexpected network activity that may indicate malicious intent or an ongoing attack, such as:
- A machine scanning multiple ports on the network (e.g., Nmap scan)
- Traffic to known malicious IPs (Command & Control)
- FTP uploads on non-standard ports
- HTTP connections to rare domains or IPs

### How IDS Helps
- Monitors and inspects network traffic in real time
- Triggers alerts when known attack patterns are detected
- Helps detect reconnaissance, malware communication, and lateral movement
- Provides visibility to SOC teams for early-stage attacks

### How IDS Works
1. **Packet Capture** – Captures and inspects each packet
2. **Rule Matching** – Matches traffic patterns against rule sets
3. **Alerting** – Triggers alerts when rules are matched
4. **Integration** – Sends logs/alerts to SIEM platforms like Wazuh

### What is Suricata?
An open-source, high-performance network IDS/IPS and monitoring engine featuring deep packet inspection, multi-threaded architecture, protocol identification (HTTP, TLS, DNS), JSON log output, and community-maintained detection rules.

---

## 🛠️ Lab Setup

| Component | Description |
|---|---|
| Wazuh Server | Manager & Dashboard |
| Wazuh Agent | CentOS/Ubuntu with Suricata installed |
| Attacker Machine | Kali Linux (Nmap) |

---

## ⚙️ Configuration Steps

### 1. Install Wazuh Agent
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update && sudo apt install wazuh-agent
```

> 💡 On CentOS/RHEL systems, use `dnf`/`yum` with the EPEL repository or the official OISF Suricata repo instead, since `apt` is Debian/Ubuntu-specific.

Configure the agent to point to the manager:
```bash
sudo nano /var/ossec/etc/ossec.conf
```
```xml
<address>wazuh-manager-ip</address>
```

Start the agent:
```bash
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### 2. Install Suricata
```bash
sudo dnf install epel-release -y
sudo dnf install suricata -y
```

### 3. Set the correct network interface
> ⚠️ **Critical step.** Suricata must listen on the actual active network interface, not a placeholder name like `eth0`.

Identify the real interface:
```bash
ip a
```

Update the interface in **two places**:
- `/etc/suricata/suricata.yaml` → under the `af-packet` section
- `/etc/sysconfig/suricata` → the `OPTIONS="-i <interface>"` line (this is what systemd actually passes to the binary on RHEL/CentOS)

```bash
sudo nano /etc/sysconfig/suricata
```
```
OPTIONS="-i ens15" ---> your interface 
```

### 4. Enable EVE JSON logging
In `/etc/suricata/suricata.yaml`, ensure:
```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
```

### 5. Update and install detection rules
```bash
sudo suricata-update
```

### 6. Add a custom detection rule for SYN scans
Several default "ET SCAN" Nmap signatures ship **disabled by default** (commented out with `#`) due to age and low confidence scoring. Rather than relying on them, a custom rule was added:

**File:** `/var/lib/suricata/rules/local.rules` *(must match the path defined in `default-rule-path`, not just any `rules/` folder)*

```
alert tcp any any -> $HOME_NET any (msg:"LOCAL SCAN Possible Nmap SYN Scan Detected"; flags:S; threshold:type both, track by_src, count 10, seconds 5; sid:1000001; rev:1; classtype:attempted-recon;)
```

Make sure it's referenced in `suricata.yaml`:
```yaml
rule-files:
  - suricata.rules
  - local.rules
```

### 7. Start Suricata
```bash
sudo systemctl daemon-reload
sudo systemctl enable suricata
sudo systemctl start suricata
sudo systemctl status suricata
```

### 8. Connect Suricata logs to the Wazuh Agent
```bash
sudo nano /var/ossec/etc/ossec.conf
```
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```
```bash
sudo systemctl restart wazuh-agent
```

---

## 💥 Attack Simulation

From the Kali attacker machine:
```bash
sudo nmap -sS -T4 <target-ip>
```

| Flag | Meaning |
|---|---|
| `-sS` | SYN scan ("half-open" scan) |
| `-T4` | Faster scan timing |

---

## 🔍 Detection & Analysis

### Verify Suricata caught the scan directly
```bash
sudo tail -f /var/log/suricata/eve.json | grep --line-buffered '"event_type":"alert"'
```
Sample alert captured:
```json
{"event_type":"alert","src_ip":"Attacker ip","dest_ip":"victim ip","dest_port":21,"proto":"TCP",
"alert":{"signature_id":1000001,"signature":"LOCAL SCAN Possible Nmap SYN Scan Detected","severity":2}}
```

### View alerts in Wazuh Dashboard
1. Open Wazuh Dashboard → **Security Events**
2. Select the target agent
3. Filter by:
```
rule.groups:"suricata"
```
4. Look for the alert signature, e.g.:
```
LOCAL SCAN Possible Nmap SYN Scan Detected
```

---

## 🐛 Troubleshooting Log (Real Issues Encountered)

| Issue | Root Cause | Fix |
|---|---|---|
| `apt: command not found` | Wrong package manager for CentOS | Use `dnf`/`yum` + EPEL repo |
| `Error opening file: "/var/log/suricata/" Is a directory` | `default-log-dir` accidentally set to a file path | Restore `default-log-dir` to `/var/log/suricata/` and keep `filename` as a relative name |
| `failed to find interface: No such device` | `eth0` hardcoded but real interface was `ens160` | Update interface in both `suricata.yaml` **and** `/etc/sysconfig/suricata` (`OPTIONS`) |
| No alerts despite traffic in `eve.json` | Default "ET SCAN Nmap" signatures were disabled (commented `#`) in the ruleset | Wrote a custom `local.rules` signature instead |
| `No rule files match the pattern /var/lib/suricata/rules/local.rules` | Rule file created in `/etc/suricata/rules/` instead of the actual `default-rule-path` | Create `local.rules` in the exact path defined by `default-rule-path` |

> 💡 **Key lesson learned:** A YAML file referencing a rule filename is not enough — Suricata resolves that filename relative to `default-rule-path`. Always verify the actual resolved path, not just the filename in `rule-files`.

---

## 🧠 Observation

This exercise reinforced that an IDS like Suricata is only as good as its active configuration — not its installed ruleset. Multiple layers had to be verified independently: the network interface actually being monitored, the log output paths, whether relevant signatures were even enabled, and whether the correct rule file path was being read by the engine. A single misconfigured layer (wrong interface, wrong path, or a disabled-by-default signature) silently breaks detection without any obvious error pointing to the real cause.

Once properly configured, the value of network-level detection became clear: Suricata caught the reconnaissance attempt (port scan) before any actual exploitation occurred — which is exactly the kind of early-stage visibility that allows a SOC team to intervene before an attacker moves further into the kill chain.

Key takeaways:
- **Don't assume default rules are active** – many community signatures are shipped disabled and need explicit enabling or replacement with custom rules.
- **Configuration paths matter as much as content** – a correctly written rule in the wrong directory is functionally the same as no rule at all.
- **Validate end-to-end, not just one layer** – confirming Suricata generates an alert in `eve.json` is a separate checkpoint from confirming Wazuh ingests and displays that same alert.
- **Network reconnaissance detection is a high-value, early warning signal** – catching a port scan gives defenders time to respond before an attacker identifies and exploits a vulnerable service.

---

## 🧩 Skills Demonstrated

- Suricata IDS installation and configuration on Linux
- Network interface and EVE JSON logging configuration
- Custom Suricata rule writing (signature syntax, thresholds, flags)
- Troubleshooting systemd service configuration (`sysconfig` overrides)
- Wazuh-Suricata log integration via `localfile` configuration
- Port scan (reconnaissance) attack simulation using Nmap
- SOC-style end-to-end detection validation

---

## 📚 References
- [Suricata Official Documentation](https://docs.suricata.io/)
- [Wazuh Official Documentation](https://documentation.wazuh.com/)
- [Emerging Threats Open Ruleset](https://rules.emergingthreats.net/)
