# Network Security & DMZ Lab with pfSense, Docker, Suricata, and Kali Linux

A hands-on, multi-zone virtual network security lab demonstrating network segregation, perimeter firewall filtering, NAT port forwarding, public web service deployment in a DMZ, Intrusion Detection System (IDS) logging, and vulnerability assessment using Kali Linux. [Video link](https://www.youtube.com/watch?v=AWnq1vH39QQ) and [notes link](https://hackmd.io/@XhhLIvHZSgy56tx8mWoOWg/H1lOuCVDhGx)

---

## Table of Contents
1. [Executive Summary & Network Topology Architecture](#1-executive-summary--network-topology-architecture)
2. [VMware Virtual Network Editor Setup](#2-vmware-virtual-network-editor-setup)
3. [pfSense 2.7.2 Installation & Base Setup](#3-pfsense-272-installation--base-setup)
4. [LAN Management Client & WebGUI Configuration](#4-lan-management-client--webgui-configuration)
5. [DMZ Web Server Deployment (Ubuntu + Docker DVWA)](#5-dmz-web-server-deployment-ubuntu--docker-dvwa)
6. [Complete Firewall Rules & NAT Port Forwarding](#6-complete-firewall-rules--nat-port-forwarding)
7. [Intrusion Detection System (Suricata IDS) Configuration](#7-intrusion-detection-system-suricata-ids-configuration)
8. [Security Verification & Penetration Testing](#8-security-verification--penetration-testing)
9. [Configuration Matrix & Implementation Notes](#9-configuration-matrix--implementation-notes)

---

## 1. Executive Summary & Network Topology Architecture

This project builds a enterprise-grade virtual network topology that isolates critical internal assets from publicly exposed web applications. A **pfSense 2.7.2** firewall manages traffic flow between three distinct security zones:

* **WAN Zone (`VMnet8 NAT` - `192.168.133.0/24`):** Public/Untrusted segment hosting the attacker machine (**Kali Linux**, assigned `192.168.133.160` via DHCP).
* **LAN Zone (`VMnet2 Host-Only` - `192.168.10.0/24`):** Internal Trusted Management segment hosting the **Windows Management VM** (`192.168.10.100`).
* **DMZ Zone (`VMnet3 Host-Only` - `192.168.20.0/24`):** Demilitarized Semi-Trusted segment hosting publicly accessible services:
  * **Ubuntu Server** (`192.168.20.10`) running **Docker DVWA** (Damn Vulnerable Web Application) bound to port `8080`.
  * **Windows IIS Server** (`192.168.20.100`).

### Architecture Topology Diagram

```
                      +----------------------------------+
                      |         Internet / WAN           |
                      |  Network: 192.168.133.0/24 (NAT) |
                      +----------------------------------+
                                 |          |
                                 |          | (Attacker)
                 +---------------+          v
                 |                   +------------------------+
                 |                   | Kali Linux Machine     |
                 |                   | IP: 192.168.133.160    |
                 |                   +------------------------+
                 v
      +-------------------------------------------------------+
      |               pfSense Firewall / Router               |
      |   WAN Interface (em0): 192.168.133.133 (DHCP / NAT)   |
      |   LAN Interface (em1): 192.168.10.1 /24               |
      |   DMZ Interface (em2): 192.168.20.1 /24               |
      +-------------------------------------------------------+
            |                                         |
            | (LAN Zone)                              | (DMZ Zone)
            v                                         v
+-----------------------+                 +-----------------------+
|  LAN Network (VMnet2) |                 |  DMZ Network (VMnet3) |
|  Subnet:              |                 |  Subnet:              |
|  192.168.10.0/24      |                 |  192.168.20.0/24      |
+-----------------------+                 +-----------------------+
            |                                 |               |
            v                                 v               v
+-----------------------+     +--------------------+ +--------------------+
| Windows Management VM |     | Ubuntu Server      | | Windows IIS Server |
| IP: 192.168.10.100    |     | Docker DVWA        | | IP: 192.168.20.100 |
| Gateway: 192.168.10.1 |     | IP: 192.168.20.10  | | Gateway:           |
| DNS: 192.168.10.1     |     | Port: 8080         | | 192.168.20.1       |
+-----------------------+     +--------------------+ +--------------------+
```

---

## 2. VMware Virtual Network Editor Setup

Before launching any virtual machines, virtual switches must be created using VMware Virtual Network Editor:

1. **VMnet8 (NAT - WAN Zone):**
   * **Subnet IP:** `192.168.133.0` / **Subnet Mask:** `255.255.255.0`
   * **DHCP:** Enabled (automatically assigns IP addresses to pfSense WAN interface and Kali Linux VM).
2. **VMnet2 (Custom / Host-Only - LAN Zone):**
   * **Subnet IP:** `192.168.10.0` / **Subnet Mask:** `255.255.255.0`
   * **Connect host adapter to this network:** Unchecked (isolates LAN from physical host).
   * **Use local DHCP service:** Unchecked (disabled; static network configuration used).
3. **VMnet3 (Custom / Host-Only - DMZ Zone):**
   * **Subnet IP:** `192.168.20.0` / **Subnet Mask:** `255.255.255.0`
   * **Connect host adapter to this network:** Unchecked.
   * **Use local DHCP service:** Unchecked.

---

## 3. pfSense 2.7.2 Installation & Base Setup

### Firewall VM Hardware Specifications
* **RAM:** 2 GB
* **CPU:** 2 Cores
* **Disk:** 50 GB (UFS File System)
* **Network Adapters:** 3 Adapters assigned sequentially:
  1. `Adapter 1` -> **VMnet8 (NAT)** (`em0` - WAN)
  2. `Adapter 2` -> **VMnet2 (Custom)** (`em1` - LAN)
  3. `Adapter 3` -> **VMnet3 (Custom)** (`em2` - DMZ)

### Console Setup Steps
1. Boot from the pfSense ISO, accept licensing, select **Install pfSense**, choose **UFS**, **Entire Disk**, and **MBR Partition Scheme**.
2. Upon booting into the main console menu:
   * Select **Option 2 (Set interface(s) IP address)**:
     * Configure **LAN (`em1`)**: Set IPv4 address to `192.168.10.1` | Subnet bit length: `24` | Disable DHCP Server.
     * Configure **DMZ (`em2`)**: Set IPv4 address to `192.168.20.1` | Subnet bit length: `24` | Disable DHCP Server.

---

## 4. LAN Management Client & WebGUI Configuration

### Windows Client Network Configuration
1. Attach the Windows Management VM network adapter to **VMnet2**.
2. Configure static TCP/IPv4 settings:
   * **IP Address:** `192.168.10.100`
   * **Subnet Mask:** `255.255.255.0`
   * **Default Gateway:** `192.168.10.1` (pfSense LAN address)
   * **Preferred DNS:** `192.168.10.1`

### WebGUI Initial Setup Wizard
1. Open a browser on the Windows client and navigate to `https://192.168.10.1`.
2. Log in using default credentials: `admin` / `pfsense`.
3. Set Primary DNS Server to `8.8.8.8`.
4. Under **WAN Configuration**, **Uncheck**:
   * *Block RFC1918 Private Networks*
   * *Block Non-Internet Routed Networks (Bogon networks)*
5. **DNS Forwarder Service Setup:**
   * Navigate to **Services > DNS Resolver** -> **Disable**.
   * Navigate to **Services > DNS Forwarder** -> **Enable**.
   * *Purpose:* Enables internal LAN/DMZ clients to resolve external domains (e.g., `google.com`) through pfSense DNS forwarding.

---

## 5. DMZ Web Server Deployment (Ubuntu + Docker DVWA)

### Ubuntu Server Network Configuration
1. Attach network adapter to **VMnet3** (DMZ).
2. Configure netplan `/etc/netplan/01-netcfg.yaml`:
   ```yaml
   network:
     version: 2
     ethernets:
       ens33:
         addresses:
           - 192.168.20.10/24
         routes:
           - to: default
             via: 192.168.20.1
         nameservers:
           addresses: [192.168.20.1, 8.8.8.8]
   ```
3. Apply settings: `sudo netplan apply`

### Docker DVWA Container Deployment
Pull and start the DVWA container mapping host port `8080` to container HTTP port `80`:
```bash
sudo docker run -d -p 8080:80 --name dvwa vulnerabilities/web-dvwa
```
Verify container status using `sudo docker ps` (confirm `0.0.0.0:8080->80/tcp`).

---

## 6. Complete Firewall Rules & NAT Port Forwarding

### A. WAN Interface Rules (`Firewall > Rules > WAN`)
| Action | Protocol | Source | Port | Destination | Port | Gateway | Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Pass** (Green Check) | IPv4 TCP | `*` | `*` | `WAN address` | `8081` | `*` | *NAT Port forward for dvwa in ubuntu server dmz* (Auto-generated linked rule) |

### B. LAN Interface Rules (`Firewall > Rules > LAN`)
| Action | Protocol | Source | Port | Destination | Port | Gateway | Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Pass** | `*` | `LAN net` | `*` | `LAN address` | `443, 80, 22` | `*` | *Anti-Lockout Rule* (Prevents admin lockout from WebGUI/SSH) |
| **Pass** | IPv4 `*` | `LAN net` | `*` | `*` | `*` | `*` | *Default allow LAN to any rule* |
| **Pass** | IPv6 `*` | `LAN net` | `*` | `*` | `*` | `*` | *Default allow LAN IPv6 to any rule* |

### C. DMZ Interface Rules (`Firewall > Rules > DMZ`)
| Action | Protocol | Source | Port | Destination | Port | Gateway | Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Pass** | IPv4 `*` | `DMZ net` | `*` | `*` | `*` | `*` | *Allow DMZ to any (ops for testing)* (Permits outbound traffic from DMZ subnet `192.168.20.0/24`) |

### D. NAT Port Forwarding Rules (`Firewall > NAT > Port Forward`)
| Interface | Proto | Source Addr / Port | Dest Addr / Port | Redirect Target IP | Redirect Target Port | Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **WAN** | TCP | `*` / `*` | `WAN address` / `8081` | `192.168.20.10` | `8080` | *Port forward for dvwa in ubuntu server dmz* |

---

## 7. Intrusion Detection System (Suricata IDS) Configuration

Suricata operates as an Intrusion Detection System inspecting DMZ-bound traffic:

1. **Interface Mapping:** Configured on interface **`VMNet3DMZ`** (`em2`).
2. **Pattern Match Mode:** Set to `AUTO`.
3. **Block IPs Setting:** `DISABLED` (Operates in pure IDS alert mode for logging and threat intelligence without dropping active TCP streams).
4. **Rule Set Deployment:**
   * Installed **ETOpen (Emerging Threats Open)** rule set.
   * Signature updates verified under **Services > Suricata > Updates**.

---

## 8. Security Verification & Penetration Testing

### Step 1: External Access Verification (Kali Linux)
1. Kali Linux assigned to **VMnet8** receiving dynamic WAN IP `192.168.133.160`.
2. Open browser on Kali and navigate to `http://192.168.133.133:8081`.
3. **Result:** DVWA Login interface renders through pfSense NAT port forwarding. Login with default credentials (`admin` / `password`).

### Step 2: Vulnerability Scanning with Nikto
Execute a web vulnerability scan against the forwarded service from Kali Linux terminal:

```bash
nikto -h 192.168.133.133 -p 8081
```

**Nikto Scan Findings Output:**
```text
- Target IP: 192.168.133.133
- Target Port: 8081
- Server: Apache/2.4.25 (Debian)
+ Cookie PHPSESSID created without the httponly flag
+ The Anti-Clickjacking X-Frame-Options header is not present.
+ The X-Content-Type-Options header is not set.
+ Entry '/config/' retrieves a directory listing.
+ Entry '/admin/' retrieves a directory listing.
+ Directory indexing found: /config/
+ /config/config.inc.php: Configuration file contains sensitive info.
+ /phpinfo.php: Output from phpinfo() discovered.
```

### Step 3: Real-Time IDS Alerts Capture (`Services > Suricata > Alerts`)
Inspect real-time alert output generated during Nikto's assessment on interface **`(DMZ) VMNet3DMZ`**:

* **Traffic Source:** `192.168.133.160` (Kali Linux)
* **Traffic Destination:** `192.168.20.10:8080` (Ubuntu DVWA Container)
* **Captured Signature Alerts:**
  1. `ET POLICY Python-urllib / Nikto Web Scanner User-Agent Detected`
  2. `ET WEB_SERVER Possible HTTP Attack / Directory Traversal`
  3. `ET WEB_SERVER /config/ access / Sensitive File Disclosure`
  4. `ET WEB_SERVER phpinfo.php Access`
  5. `ET ATTACK_RESPONSE / Command Injection / SQL Injection`

---

## 9. Configuration Matrix & Implementation Notes

| Component / Parameter | Theoretical Notes | Actual Verified Lab Execution |
| :--- | :--- | :--- |
| **pfSense WAN Addressing** | Static range notation (`192.168.133.0/24`) | Dynamic IP assignment via DHCP (`192.168.133.133`) from VMware VMnet8 NAT engine. |
| **Port Forward Target** | Broad port range (`8080-16082`) | Precise Single Port Forward: External `WAN:8081` maps to Internal `192.168.20.10:8080`. |
| **Firewall Rulesets** | Single generic pass concept | Fully defined WAN pass rule, LAN anti-lockout + dual outbound rules, and DMZ explicit allow rule. |
| **Suricata Monitoring** | Unspecified network interface | Explicitly monitoring interface **`VMNet3DMZ`** in non-blocking IDS alert mode. |
| **Web Service Stack** | Bare-metal web host | Ubuntu 22.04 LTS host running Docker engine hosting isolated DVWA web container. |