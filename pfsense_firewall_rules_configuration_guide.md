# Comprehensive Guide to Configuring Firewall Rules on pfSense

This document outlines the core concepts, execution logic, and step-by-step procedures for configuring firewall and NAT rules on pfSense.

---

## 1. Core Principles of pfSense Firewall Rules

### Traffic Direction & Interface Rules
* **Inbound Filtering:** Firewall rules on any interface tab match traffic **entering** that interface from attached network devices.
  * **LAN Rules:** Control traffic originating from machines on the LAN network (e.g., LAN client trying to reach WAN or DMZ).
  * **WAN Rules:** Control traffic originating from the outside Internet trying to enter your network.
  * **DMZ Rules:** Control traffic originating from machines inside the DMZ.

### Processing Logic
* **Top-to-Bottom Execution:** Rules are processed sequentially from top to bottom.
* **First Match Wins:** The first rule that matches the packet criteria (Source, Destination, Port, Protocol) is applied. All rules below it are ignored.
* **Implicit Deny All:** Any packet that does not explicitly match an **ALLOW (Pass)** rule is automatically blocked by pfSense's hidden default rule.

### Default Interface Behaviors
* **LAN Interface:** Created during installation with default "Allow All" rules.
* **DMZ / OPT Interfaces:** Default to "Implicit Deny All" (zero rules created). Traffic from these interfaces is blocked until you manually add `Pass` rules.

---

## 2. Parameter Definitions (Rule Configuration Options)

When creating or editing a rule (**Firewall** $\rightarrow$ **Rules** $\rightarrow$ **[Interface]**), configure the following parameters:

| Parameter | Options / Format | Technical Description |
| :--- | :--- | :--- |
| **Action** | `Pass`, `Block`, `Reject` | **Pass:** Allows packet.<br>**Block:** Drops packet silently.<br>**Reject:** Drops packet and notifies sender (TCP RST / ICMP Unreachable). |
| **Interface** | `LAN`, `WAN`, `DMZ`, etc. | Selects the interface where traffic enters the firewall. |
| **Address Family** | `IPv4`, `IPv6`, `IPv4+IPv6` | Specifies the IP version. |
| **Protocol** | `Any`, `TCP`, `UDP`, `ICMP`, etc. | Transport layer protocol filter. |
| **Source** | `Any`, `Single Host or Alias`, `Network`, `[Interface] net`, `[Interface] address` | **Single Host:** Exactly 1 IP address.<br>**Network:** A custom CIDR block (e.g., `192.168.10.0/24`).<br>**`[Interface] net`:** Macro matching the full interface subnet.<br>**`[Interface] address`:** Macro matching *only* the firewall IP itself. |
| **Destination** | Same as Source options | Target network/host address. |
| **Destination Port** | Ports / Ranges (e.g., `80`, `443`, `22`, `53`) | Specific application port or range. |

---

## 3. Network Address Translation (NAT) Rules

### A. Port Forwarding (Inbound NAT)
Directs inbound traffic from the external WAN interface to an internal host in your LAN or DMZ.

* **Path:** **Firewall** $\rightarrow$ **NAT** $\rightarrow$ **Port Forward**
* **Configuration Steps:**
  1. **Interface:** `WAN`
  2. **Protocol:** `TCP` (or required protocol)
  3. **Destination:** `WAN address`
  4. **Destination Port Range:** External listening port (e.g., `80` for HTTP)
  5. **Redirect Target IP:** Private IP of target host (e.g., `192.168.20.10`)
  6. **Redirect Target Port:** Internal port on target host (e.g., `80`)
  7. **Filter Rule Association:** Set to `Pass` or `Add associated filter rule` (automatically generates matching WAN firewall rule).

### B. Outbound NAT
Translates internal private network IPs into the public/WAN interface IP address when reaching external networks.

* **Path:** **Firewall** $\rightarrow$ **NAT** $\rightarrow$ **Outbound**
* **Modes:**
  * **Automatic Outbound NAT (Default):** Auto-generates translation rules for all internal subnets.
  * **Hybrid Outbound NAT:** Combines automatic rules with user-defined overrides.
  * **Manual Outbound NAT:** Disables automatic generation; requires manual rules for every network.

---

## 4. Step-by-Step Configuration Workflows

### Workflow 1: Allow Internet Access for DMZ Subnet
Since newly added interfaces block all traffic by default, follow these steps to enable Internet access for DMZ hosts:

1. Navigate to **Firewall** $\rightarrow$ **Rules** $\rightarrow$ **DMZ**.
2. Click **Add** (top button).
3. Set the following options:
   * **Action:** `Pass`
   * **Interface:** `DMZ`
   * **Address Family:** `IPv4`
   * **Protocol:** `Any`
   * **Source:** `DMZ net`
   * **Destination:** `Any`
4. Click **Save**, then click **Apply Changes**.

---

### Workflow 2: Isolate DMZ from Accessing LAN (Security Isolation)
To allow DMZ hosts to access the Internet while preventing them from initiating connections into your internal LAN network:

1. Navigate to **Firewall** $\rightarrow$ **Rules** $\rightarrow$ **DMZ**.
2. **Rule 1 (Top Rule - Block LAN Access):**
   * Click **Add** (top arrow to place at position 1).
   * **Action:** `Block` (or `Reject`)
   * **Interface:** `DMZ`
   * **Source:** `DMZ net`
   * **Destination:** `LAN net`
   * Click **Save**.
3. **Rule 2 (Bottom Rule - Allow Internet):**
   * Click **Add**.
   * **Action:** `Pass`
   * **Interface:** `DMZ`
   * **Source:** `DMZ net`
   * **Destination:** `Any`
   * Click **Save**.
4. Click **Apply Changes**.

---

## 5. Troubleshooting & Verification

* **Firewall Live Logs:** Navigate to **Status** $\rightarrow$ **System Logs** $\rightarrow$ **Firewall** to inspect allowed (green) or blocked (red) traffic in real time.
* **Filter by Interface/IP:** Click the funnel icon in the firewall logs to filter entries by interface name or specific source/destination IP addresses.
* **State Table Inspection:** Go to **Diagnostics** $\rightarrow$ **States** to verify active translated connections and existing session states.