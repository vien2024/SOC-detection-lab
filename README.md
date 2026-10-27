# Network Security & DMZ Lab with pfSense, Docker, Suricata & Kali Linux

![Lab Status](https://img.shields.io/badge/Lab-Active-brightgreen)
![Firewall](https://img.shields.io/badge/Firewall-pfSense_2.7.2-blue)
![IDS](https://img.shields.io/badge/IDS-Suricata-red)
![Platform](https://img.shields.io/badge/Platform-VMware_Workstation-orange)

A multi-zone virtual network security environment demonstrating perimeter security, DMZ segregation, NAT port forwarding, Intrusion Detection System (IDS) monitoring, and vulnerability assessment.

---

## 📐 Network Architecture Overview

```
                          +-------------------------------+
                          |        WAN (VMnet8 NAT)       |
                          |      192.168.133.0/24         |
                          +-------------------------------+
                                     |          |
                                     |          |
                                     v          v
                       +-------------------+   +--------------------+
                       | pfSense Firewall  |   | Kali Linux         |
                       | IP: .133 (DHCP)   |   | IP: .160           |
                       +-------------------+   +--------------------+
                           /           \
                          /             \
    +------------------------+       +------------------------+
    |    LAN (VMnet2)        |       |    DMZ (VMnet3)        |
    |  192.168.10.0/24       |       |  192.168.20.0/24       |
    +------------------------+       +------------------------+
                |                                |
                v                                v
    +------------------------+       +------------------------+
    | Windows Management VM  |       | Ubuntu Server (.10)    |
    | IP: 192.168.10.100     |       | - Docker DVWA (:8080)  |
    +------------------------+       +------------------------+
```

---

## ⚡ Quick Specs & Zone Layout

| Zone | Interface | Subnet | Included Assets | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **WAN** | `em0` | `192.168.133.0/24` | Kali Linux (`.160`) | Simulated external internet / attacker network |
| **LAN** | `em1` | `192.168.10.0/24` | Windows Workstation (`.100`) | Internal management & pfSense WebGUI access |
| **DMZ** | `em2` | `192.168.20.0/24` | Ubuntu Server (`.10`) + Docker DVWA | Segregated public web services layer |

---

## 🚀 Main Features Tested

1. **Perimeter Segregation & Stateful Firewall Rules:** Strict separation between untrusted WAN, management LAN, and service DMZ.
2. **NAT Port Forwarding:** Forwarding external `WAN:8081` requests to internal DMZ Docker container `192.168.20.10:8080`.
3. **Suricata IDS Deployment:** Real-time signature detection and HTTP payload logging on DMZ interface (`VMNet3DMZ`).
4. **Penetration Testing Verification:** Automated Nikto web security assessment from Kali Linux triggering real-time Suricata alerts.

---

## 📖 Complete Documentation

For step-by-step installation walkthroughs, detailed firewall rule tables, Suricata configuration settings, and raw Nikto scan logs, see the full guide:

📄 **[View Full Lab Documentation (LAB_DOCUMENTATION.md)](./network_security_dmz_lab_documentation.md)**

---

## 🛠️ Requirements & Tools Used

* **Hypervisor:** VMware Workstation Pro / Player
* **Firewall OS:** pfSense 2.7.2 Release
* **Attacker OS:** Kali Linux
* **Target Services:** Ubuntu Server 22.04 LTS, Docker Engine, DVWA Container
* **IDS Engine:** Suricata IDS with ETOpen (Emerging Threats) rule set
