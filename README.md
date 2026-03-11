# 🔬SIEM Home Lab

## 📌<ins>Objective</ins>

I built this lab to get hands-on experience with the tools and workflows that matter in a SOC analyst role, log collection, centralised monitoring, and detecting real attack patterns. Rather than following a guided walkthrough, I wanted to build the infrastructure myself, troubleshoot the problems that came up, and end up with something that genuinely works.

Everything runs in VirtualBox on a personal machine. Every component was configured from scratch.

---

## <ins>Lab Architecture</ins>

```
                    [ Kali Linux — 10.0.3.3 ]
                           Attacker VM
                                |
                      OPT1 — AttackerNet (10.0.3.0/24)
                                |
                 [ pfSense 2.8.1 — 192.168.56.9 ]
                   Firewall + Suricata IDS v7.0.8
                                |
                    LAN — Host-only (192.168.56.0/24)
               _________________|_________________
               |                |                |
      [ Windows AD ]    [ Linux Server ]    [ Splunk ]
      192.168.56.104    192.168.56.105    192.168.56.103
         win-ad           linux-server     Ubuntu 24.04
```

| VM | Role | IP |
|---|---|---|
| Splunk Enterprise (Ubuntu 24.04) | SIEM / Log Aggregator | 192.168.56.103 |
| pfSense 2.8.1 | Perimeter Firewall + IDS | 192.168.56.9 |
| Windows AD | Target / Log Source | 192.168.56.104 |
| Ubuntu Server 24.04 | Target / Log Source | 192.168.56.105 |
| Kali Linux | Attacker | 10.0.3.3 |

---

## <ins>Setup</ins>

### VirtualBox Network Configuration

Each VM was given two network adapters:

- **Adapter 1 - NAT:** Gives each VM internet access for package installs and updates
- **Adapter 2 - Host-only:** Creates the private internal network so VMs can communicate and forward logs to Splunk

Kali was placed on a separate NAT Network (AttackerNet - 10.0.3.0/24) so all its traffic routes through pfSense OPT1 before reaching internal hosts. This was important because VMs on the same subnet talk directly to each other, bypassing the firewall entirely. Isolating Kali onto its own network forces all attack traffic through pfSense and into Suricata.

---

### <ins>Splunk SIEM</ins>

Deployed Splunk Enterprise 10.2.1 on Ubuntu 24.04 LTS as the central log collection and monitoring platform.

### Download and install
`wget -O splunk-10.2.1-c892b66d163d-linux-amd64.deb "https://download.splunk.com/..."`  
`sudo dpkg -i splunk-10.2.1-c892b66d163d-linux-amd64.deb`  
<img width="981" height="392" alt="Screenshot 2026-03-02 162755" src="https://github.com/user-attachments/assets/ade1a24c-f45a-4dba-9f47-ab7e0f1b42eb" />  
<img width="1843" height="460" alt="Screenshot 2026-03-02 162851" src="https://github.com/user-attachments/assets/146f6c2a-748d-4178-87a1-3b26815b9ee0" />

### Enable at boot and start
`sudo ./splunk enable boot-start --accept-license`  
`./splunk start`  
<img width="1837" height="836" alt="Screenshot 2026-03-03 151041" src="https://github.com/user-attachments/assets/fdfd0b24-86d6-49b8-bc6c-e0ee14d7e031" />

Configured Splunk to receive logs on two ports:
- **Port 9997 (TCP)** - Universal Forwarder connections from Windows and Linux
- **Port 5514 (UDP)** - Syslog from pfSense

Standard syslog port 514 wasn't usable as Linux restricts ports below 1024 to root. Port 5514 was used instead with a matching firewall rule in pfSense to permit the traffic.

A static IP and default gateway were configured via netplan so Splunk could communicate reliably with pfSense across the lab network.

<img width="1011" height="297" alt="Screenshot 2026-03-07 084238" src="https://github.com/user-attachments/assets/ecfebf46-051a-48fb-be48-951ae5866138" />

<img width="733" height="268" alt="Screenshot 2026-03-07 085552" src="https://github.com/user-attachments/assets/f6ad1821-59aa-4809-884a-036ea047b747" />

<img width="730" height="324" alt="Screenshot 2026-03-07 093323" src="https://github.com/user-attachments/assets/fc293911-f8e5-4612-96eb-cf7f1be4ac0d" />

---

### <ins>Windows AD - Log Forwarding</ins>

Downloaded and installed Splunk Universal Forwarder (x64) on the Windows 10 VM. Before configuring, checked Event Viewer to identify which logs to forward. Application, System and Security were selected.  
<img width="841" height="609" alt="Screenshot 2026-03-06 171529" src="https://github.com/user-attachments/assets/8ec22784-506f-4469-bcb9-d9f461dc6bb9" />  
<img width="787" height="306" alt="Screenshot 2026-03-06 171859" src="https://github.com/user-attachments/assets/cd0257ba-2e55-4261-bd21-dcc43dd4cf3a" />

Configured `inputs.conf` to collect Windows Event Logs:

<img width="772" height="538" alt="Screenshot 2026-03-06 171920" src="https://github.com/user-attachments/assets/4c811db5-bef6-48c7-aa7b-cf4ee2e1ac61" />

#### Configured `outputs.conf` to point the forwarder at Splunk:

<img width="769" height="223" alt="Screenshot 2026-03-06 171937" src="https://github.com/user-attachments/assets/75765de7-f6d6-4998-b382-ed04c5eb6596" />  

#### Restarted the forwarder from an admin command prompt and verified Windows logs were appearing in Splunk with `index=main`.  

<img width="961" height="482" alt="Screenshot 2026-03-06 173011" src="https://github.com/user-attachments/assets/4e385a21-8a0b-40b9-9d82-e13aaa821f3d" />  
<img width="1899" height="731" alt="Screenshot 2026-03-06 170241" src="https://github.com/user-attachments/assets/75b296d1-f52e-4957-801e-5af6f12e6baf" />

---

### <ins>Linux Server - Log Forwarding</ins>

Downloaded and installed Splunk Universal Forwarder on Ubuntu Server 24.04, then added the forwarding server and configured log monitors:
<img width="1289" height="437" alt="Screenshot 2026-03-06 191652" src="https://github.com/user-attachments/assets/3ba1791b-9c9b-477b-a2e8-cc84654aea83" />  
<img width="1100" height="521" alt="Screenshot 2026-03-06 191917" src="https://github.com/user-attachments/assets/c67abc92-b690-4bb8-9458-35c51afe4692" />  
<img width="929" height="781" alt="Screenshot 2026-03-06 200031" src="https://github.com/user-attachments/assets/76941d27-998a-49e3-9ed9-3f91b7af15d6" />

```bash
sudo /opt/splunkforwarder/bin/splunk add forwarder-server 192.168.56.103:9997
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/kern.log
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/dpkg.log
```
<img width="1919" height="426" alt="Screenshot 2026-03-07 055843" src="https://github.com/user-attachments/assets/05ab330f-ec24-4c94-88d8-06e665b1f35a" />

| Log file | Why it matters |
|---|---|
| auth.log | SSH logins, sudo usage, failed authentication |
| syslog | General system activity |
| kern.log | Kernel-level events, network changes |
| dpkg.log | Software installs, useful for detecting unauthorised changes |

Static IP and gateway configured via netplan to ensure traffic routes through pfSense:

<img width="1767" height="768" alt="Screenshot 2026-03-07 081046" src="https://github.com/user-attachments/assets/057e8b9c-7a55-4741-a76c-3f401f15ccef" />

---

### <ins>pfSense Firewall</ins>

Installed pfSense 2.8.1 in VirtualBox with three interfaces, WAN (bridged to home network, DHCP 192.168.0.193), LAN (host-only, static 192.168.56.9), and OPT1 (AttackerNet for Kali). Set pfSense as the default gateway for all internal VMs.

<img width="721" height="383" alt="Screenshot 2026-03-11 224105" src="https://github.com/user-attachments/assets/748dfd6b-ae25-405b-a1bb-8e06f4735f9f" />  
<img width="381" height="258" alt="Screenshot 2026-03-09 162652" src="https://github.com/user-attachments/assets/68ed77f0-1c04-418b-8e18-2e75478bfeba" />

Remote syslog forwarding configured to send System Events and Firewall Events to Splunk on port 5514  
Firewall rule added on the LAN interface to permit UDP traffic destined for Splunk on port 5514:  

<img width="926" height="575" alt="Screenshot 2026-03-07 094653" src="https://github.com/user-attachments/assets/65e88133-f159-491e-87fd-2053e16a5c34" />

<img width="973" height="531" alt="Screenshot 2026-03-07 094747" src="https://github.com/user-attachments/assets/b0d4522e-a44d-4aaa-89e2-4a2c1a3dd545" />

---

### <ins>Suricata IDS</ins>

Installed Suricata v7.0.8 via the pfSense Package Manager and configured it across WAN, LAN and OPT1 interfaces. Blocking mode was left disabled as the goal here was detection and visibility, not active prevention.

<img width="997" height="497" alt="Screenshot 2026-03-07 081637" src="https://github.com/user-attachments/assets/fc141b76-3d81-4355-a5dc-fffde46f7345" />  
<img width="992" height="424" alt="Screenshot 2026-03-11 225208" src="https://github.com/user-attachments/assets/f265a370-500e-46ee-8e9e-f647a84b70c2" />

Enabled EVE JSON output to file for structured alert logging, along with HTTP and TLS logging:

<img width="938" height="611" alt="Screenshot 2026-03-07 082941" src="https://github.com/user-attachments/assets/5fec3b24-0ce5-450e-8bc2-11f03575f70d" />

Enabled the following Emerging Threats Open rule categories:

| Rule Category | What it detects |
|---|---|
| emerging-scan | Port scans, Nmap reconnaissance |
| emerging-exploit | Exploit attempts |
| emerging-dos | ICMP floods and DoS attacks |
| emerging-malware | Malware callbacks and C2 traffic |
| emerging-shellcode | Reverse shells, port 4444 |
| emerging-dns | DNS exfiltration |
| emerging-telnet | Telnet usage |

<img width="971" height="595" alt="Screenshot 2026-03-08 162841" src="https://github.com/user-attachments/assets/d7873e54-deba-4b42-99bb-18218f5b3827" />

---

## <ins>Log Sources Confirmed in Splunk</ins>

With all forwarders running and syslog ingestion active, Splunk was collecting 8 sourcetypes across three hosts:

| Sourcetype | Source |
|---|---|
| syslog | pfSense / Linux Server |
| WinEventLog:Security | Windows AD |
| kern | Linux Server |
| dpkg | Linux Server |
| auth | Linux Server |
| WinEventLog:Application | Windows AD |
| WinEventLog:System | Windows AD |
| pfsense | pfSense |

<img width="1002" height="896" alt="Screenshot 2026-03-07 100750" src="https://github.com/user-attachments/assets/71c4607b-7d4e-4487-8df3-c184a7217190" />

<img width="1822" height="836" alt="Screenshot 2026-03-11 171148" src="https://github.com/user-attachments/assets/567b6d56-adec-4e7f-80b7-2e8cc8858cc3" />

---

## <ins>Attacks Simulated</ins>

### 🐉SSH Brute Force - Hydra

Ran Hydra from Kali against the Linux Server SSH service using the rockyou wordlist. Hydra ran 14 parallel tasks achieving around 246 attempts per minute, generating a sustained high-volume authentication failure pattern.

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt -t 4 192.168.56.105 ssh
```
<img width="636" height="296" alt="Screenshot 2026-03-09 160033" src="https://github.com/user-attachments/assets/3a40783a-3a57-42da-94d0-873d191ff7b1" />

Every failed attempt was captured in `/var/log/auth.log` on the Linux Server, forwarded to Splunk, and logged as a PAM authentication failure. The attacker's IP is clearly visible in every event:

<img width="944" height="727" alt="Screenshot 2026-03-09 160142" src="https://github.com/user-attachments/assets/2de52212-0b2d-4f5c-9163-0e15ca412128" />

---

### 🗺️Network Reconnaissance - Nmap

Ran aggressive SYN scans and service version detection against internal VMs to simulate the reconnaissance phase of an attack. All scan traffic passed through pfSense OPT1 because Kali sits on the isolated AttackerNet.

```bash
sudo nmap -sS -sV -A -T4 192.168.56.104
sudo nmap -sS -sV -A -T4 192.168.56.105
```

---

## <ins>Splunk Dashboard</ins>

Built a real-time SOC monitoring dashboard [**Alphins Home Lab**] covering the visibility a Level 1 analyst needs day to day.

<img width="1839" height="748" alt="Screenshot 2026-03-11 215945" src="https://github.com/user-attachments/assets/705bf344-4c50-47ff-bfe7-e7efca2e5ddb" />  
<img width="1837" height="478" alt="Screenshot 2026-03-11 215955" src="https://github.com/user-attachments/assets/4d989378-184a-4219-8430-37b5c8eaa1d3" />


| Panel | What it shows |
|---|---|
| Total Count | 18,522 events collected across all sources |
| Critical | Priority 1 alerts |
| Warning & Error | Priority 2 events for investigation |
| Log Sources | Per-host event count which confirms all sources feeding in |
| Events Over Time | Timeline with a visible spike during the attack simulation |
| Windows Security Events | Key Event IDs from Windows AD |
| Source IPs | Most active hosts across the lab network |
| Top Attacked Destinations | Which internal hosts received the most traffic |
| Failed Login Attempts | ~4,000 brute force attempts on Linux Server clearly visible |

---

## <ins>Key Findings</ins>

- **18,522 events** collected across three log sources during the lab period
- Hydra generated approximately **4,000 failed SSH login attempts** all captured in auth.log and visible in Splunk with the attacker's source IP in every event, demonstrating end-to-end brute force detection
- Windows AD produced **4,059 Security Event Log entries** including Event ID 5379 (credential reads), 4907 (audit policy changes), 4624 (successful logons) and 4672 (special privilege logons). The types SOC analysts monitor for insider threats and privilege abuse
- pfSense firewall logs and Suricata IDS alerts forwarded to Splunk via syslog, providing network-level visibility alongside host-based logs
- The Events Over Time panel shows a clear spike during the attack window which is the kind of anomaly that would trigger investigation in a real environment.

---

## <ins>Skills Demonstrated</ins>

- SIEM deployment and configuration from scratch (Splunk Enterprise + Universal Forwarder)
- Log collection from Windows, Linux and network device sources
- Network security monitoring with pfSense firewall and Suricata IDS
- Attack simulation through SSH brute force (Hydra), network reconnaissance (Nmap)
- Threat detection by identifying brute force patterns and recon activity in Splunk
- Network segmentation via isolated attacker network forcing all traffic through the firewall
- SOC dashboard building across multiple log sources
- Linux administration, Ubuntu Server, netplan, UFW, service management
- Windows administration, Event Log configuration, Splunk UF deployment

---

## <ins>Tools & Technologies</ins>

| Tool | Version | Purpose |
|---|---|---|
| Splunk Enterprise | 10.2.1 | SIEM, log aggregation, dashboarding |
| Splunk Universal Forwarder | 10.2.1 | Log shipping from Windows and Linux |
| pfSense | 2.8.1 | Perimeter firewall, network routing |
| Suricata | 7.0.8 | Network intrusion detection (IDS) |
| Kali Linux | - | Attack simulation platform |
| Hydra | 9.6 | SSH brute force simulation |
| Nmap | - | Network reconnaissance |
| VirtualBox | - | Lab virtualisation |
| Ubuntu | 24.04 LTS | Splunk host and Linux target |
| Windows 10 | - | Active Directory and log source |

---

## <ins>Reflections</ins>

Routing was the thing that took longest to get right. VMs on the same subnet communicate directly with each other, which means attack traffic bypasses the firewall completely. The fix was isolating Kali onto a separate network so every packet had to pass through pfSense before reaching anything internal. That's not something you find explained in most lab guides — you only learn it by watching where packets actually go.

There was also more Splunk troubleshooting than expected like port conflicts between TCP and UDP listeners, netplan YAML formatting errors, and working out why pfSense syslog was arriving at the host but not being indexed. Each one had a specific cause that required reading logs and understanding how the components talk to each other. That process of diagnosing real problems is closer to day-to-day SOC work than any certification exercise I've done.

---
