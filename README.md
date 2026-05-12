# Zabbix Template — Ruijie AP820-L(V3) SNMP Monitoring

![Zabbix](https://img.shields.io/badge/Zabbix-7.0-red?style=flat-square&logo=zabbix)
![SNMP](https://img.shields.io/badge/Protocol-SNMPv2c-blue?style=flat-square)
![Device](https://img.shields.io/badge/Device-Ruijie%20AP820--L(V3)-orange?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

A production-ready Zabbix 7.0 SNMP monitoring template for **Ruijie RG-AP820-L(V3)** and **RG-AP720-L** Wi-Fi access points.

> **All OIDs in this template were verified from a live `snmpwalk` on a real Ruijie AP820-L(V3) running RGOS 11.9(6)W3B1P3.** No theoretical OIDs — everything actually works.

---

## 📋 Overview

This template monitors Ruijie Wi-Fi access points via SNMPv2c, covering reachability, uptime, memory, and radio/uplink traffic — all confirmed from a real device snmpwalk.

### Confirmed Device Details
| Parameter | Value |
|---|---|
| Device Model | Ruijie RG-AP820-L(V3) |
| Firmware | RGOS 11.9(6)W3B1P3 |
| Wi-Fi Standard | Wi-Fi 6 (802.11ax) |
| SNMP Version | v2c |

---

## ✅ What This Template Monitors

### 25 Items across 7 categories:

| Category | Items |
|---|---|
| **ICMP** | Ping reachability, round-trip latency (ms), packet loss (%) |
| **SNMP Agent** | Agent availability |
| **System Identity** | Description (model + firmware), uptime, hostname, location, contact |
| **Ruijie OIDs** | RGOS firmware version, serial number, model name, total RAM, free RAM, device uptime |
| **Wired Uplink `Gi0/1` (ifIndex 1)** | Operational status, traffic in (bps), traffic out (bps) |
| **2.4GHz Radio `Do1/0` (ifIndex 3)** | Operational status, traffic in (bps), traffic out (bps) |
| **5GHz Radio `Do2/0` (ifIndex 4)** | Operational status, traffic in (bps), traffic out (bps) |
| **Calculated** | RAM used (%) |

> ⚠️ **Important:** On the RG-AP820-L(V3), the 2.4GHz radio (`Dot11radio 1/0`) is at **ifIndex 3**, not ifIndex 2. ifIndex 2 is `MTGigabitEthernet 0/2` (internal management interface with no traffic). This template uses the correct ifIndex 3.

### Interface Map (confirmed from live device)

```
ifIndex 1  =  GigabitEthernet 0/1   →  Wired PoE uplink
ifIndex 2  =  MTGigabitEthernet 0/2 →  Internal (no traffic)
ifIndex 3  =  Dot11radio 1/0        →  2.4GHz radio  ← correct
ifIndex 4  =  Dot11radio 2/0        →  5GHz radio
```

---

## 🔔 Triggers (9 total)

| Trigger | Severity | Condition |
|---|---|---|
| AP is UNREACHABLE | **DISASTER** | ICMP fails 3 consecutive checks |
| SNMP agent unreachable | **HIGH** | SNMP unavailable while ICMP is up |
| Wired uplink (Gi0/1) is DOWN | **HIGH** | `ifOperStatus = 2` |
| 5GHz radio (Do2/0) is DOWN | **HIGH** | `ifOperStatus = 2` |
| 2.4GHz radio (Do1/0) is DOWN | **AVERAGE** | `ifOperStatus = 2` |
| Device rebooted | **AVERAGE** | `sysUpTime < {$REBOOT_UPTIME}` |
| RAM usage above threshold | **WARNING** | RAM used % > `{$RAM_USED_MAX}` |
| High ICMP packet loss | **WARNING** | Loss > `{$ICMP_LOSS_WARN}`% for 3 min |
| High ICMP latency | **WARNING** | RTT > `{$ICMP_LATENCY_WARN}`ms for 3 min |

> All lower-severity triggers are suppressed when the DISASTER trigger is active — no alert floods.

---

## 📊 Graphs (5 total)

- AP: ICMP Latency and Packet Loss
- AP: RAM Usage
- AP: Uplink (Gi0/1) Traffic
- AP: 2.4GHz Radio Traffic
- AP: 5GHz Radio Traffic

---

## ⚙️ Macros

| Macro | Default | Description |
|---|---|---|
| `{$SNMP_COMMUNITY}` | `public` | SNMPv2c community string — **override per host** |
| `{$ICMP_LOSS_WARN}` | `20` | Packet loss % threshold for WARNING |
| `{$ICMP_LATENCY_WARN}` | `150` | Round-trip latency ms threshold for WARNING |
| `{$REBOOT_UPTIME}` | `600` | Uptime in seconds below which reboot alert fires |
| `{$RAM_USED_MAX}` | `80` | RAM used % threshold for WARNING |

---

## 📦 Requirements

- **Zabbix:** 7.0 LTS or later
- **SNMP:** v2c enabled on the AP
- **Network:** UDP port 161 reachable from Zabbix server to AP management IP
- **No MIB files required** — all items use numeric OIDs verified directly from the device

---

## 🚀 Installation

### Step 1 — Enable SNMP on the Ruijie AP

Connect via SSH and run:

```bash
Ruijie> enable
Ruijie# configure terminal
Ruijie(config)# snmp-server enable
Ruijie(config)# snmp-server community <your-community-string> ro
Ruijie(config)# snmp-server location "Your AP Location"
Ruijie(config)# end
Ruijie# write memory
```

Verify:
```bash
Ruijie# show snmp
```
Expected: `SNMP agent: enabled` and `0 Unknown community name`.

### Step 2 — Test SNMP from Zabbix Server

```bash
snmpget -v2c -c <your-community-string> <AP-IP> 1.3.6.1.2.1.1.1.0
```

Expected output:
```
SNMPv2-MIB::sysDescr.0 = STRING: Ruijie AP820-L(V3) (802.11a/n/ac/ax) By Ruijie Networks.
```

### Step 3 — Import Template into Zabbix

1. Go to **Data Collection → Templates**
2. Click **Import** (top-right)
3. Upload `ruijie_ap820_zabbix7_template.xml`
4. Check all boxes → click **Import**
5. Confirm `Template SNMP Ruijie AP820-L V3` appears in the list

### Step 4 — Add Your AP as a Host

1. **Data Collection → Hosts → Create Host**
2. Fill in:
   - **Host name:** e.g. `AP820-Floor2-Room201`
   - **Templates:** `Template SNMP Ruijie AP820-L V3`
   - **Host groups:** `Wireless APs`
3. Under **Interfaces → Add → SNMP:**
   - IP: your AP management IP
   - Port: `161`
   - SNMP version: `SNMPv2`
   - Community: `{$SNMP_COMMUNITY}`
4. Under **Macros** tab, add:
   - `{$SNMP_COMMUNITY}` → your community string

### Step 5 — Verify Data Collection

After 2–3 minutes, go to **Monitoring → Latest Data**, filter by your AP host. All 25 items should show values.

```bash
# Debug from Zabbix server if items show NOTSUPPORTED
sudo apt install zabbix-get -y
zabbix_get -s <AP-IP> -p 161 -c <your-community-string> -k "system.uptime"
```

---

## 🔁 Using with Multiple APs

Once the first AP is working:

1. **Data Collection → Hosts** → click your working AP
2. Scroll to bottom → click **Clone**
3. Change only: **Host name** and **IP address**
4. Update `{$SNMP_COMMUNITY}` macro if the community string differs
5. Click **Add**

> The template works identically on **RG-AP720-L** (Wi-Fi 5). Same OIDs, same ifIndex map.

---

## ⚠️ Known Behaviour

### 2.4GHz ifIndex
On RG-AP820-L(V3), the 2.4GHz radio traffic is on **ifIndex 3**, not ifIndex 2. This template already accounts for that. If you find your AP uses a different ifIndex, clone the template and update the three 2.4GHz OIDs:

| Item | OID to update |
|---|---|
| 2.4GHz Operational Status | `1.3.6.1.2.1.2.2.1.8.X` |
| 2.4GHz Traffic In | `1.3.6.1.2.1.31.1.1.1.6.X` |
| 2.4GHz Traffic Out | `1.3.6.1.2.1.31.1.1.1.10.X` |

Replace `X` with your correct ifIndex. Find it with:
```bash
snmpwalk -v2c -c <your-community-string> <AP-IP> 1.3.6.1.2.1.2.2.1.2
snmpwalk -v2c -c <your-community-string> <AP-IP> 1.3.6.1.2.1.31.1.1.1.6
```

---

## 📁 Repository Structure

```
.
├── ruijie_ap820_zabbix7_template.xml   # Zabbix 7.0 import-ready template
└── README.md                           # This file
```

---

## 🧪 Tested On

| Device | Firmware | Zabbix | Status |
|---|---|---|---|
| Ruijie RG-AP820-L(V3) | RGOS 11.9(6)W3B1P3 | 7.0.21 | ✅ Confirmed working |
| Ruijie RG-AP720-L | RGOS (compatible) | 7.0.21 | ✅ Compatible (same OIDs) |

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

Contact me if you got a problem
Email : rheza.yudhistira.r@gmail.com
Phone : (+62) 82213983749

## 🙏 Acknowledgements

- OIDs sourced from a live `snmpwalk` on a real Ruijie RG-AP820-L(V3) device
- Built and validated for **Zabbix 7.0.21** schema
