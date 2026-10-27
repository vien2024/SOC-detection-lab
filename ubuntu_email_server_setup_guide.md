\---

title: Complete Ubuntu Email Server Setup Guide (Postfix \& Dovecot)
description: Step-by-step guide to installing and configuring Postfix, Dovecot, and SSL/TLS certificates on Ubuntu.
tags: Ubuntu, MailServer, Postfix, Dovecot, Linux, Networking, Guide
robots: noindex, nofollow
---

# ✉️ Ubuntu Email Server Setup Guide (Postfix \& Dovecot)

\[TOC]

\---

## 📚 Key Concepts \& Terminology

:::info
💡 **New to Mail Servers?** Here are the foundational terms used throughout this guide:
:::

* **MTA (Mail Transfer Agent):** Software responsible for routing and transferring emails between servers across the network using the SMTP protocol (e.g., **Postfix**).
* **MDA (Mail Delivery Agent) / MRA (Mail Retrieval Agent):** Software that accepts delivered email from the MTA, stores it in local mailboxes, and lets clients retrieve it via IMAP or POP3 protocols (e.g., **Dovecot**).
* **SMTP (Simple Mail Transfer Protocol):** Protocol used for sending email from a client to a server, or between mail servers.

  * *Unencrypted default port:* `25`
  * *Submission port:* `587`
  * *SMTPS port:* `465`
* **IMAP (Internet Message Access Protocol):** Protocol used by email clients (e.g., Thunderbird) to view and manage emails directly on the server without downloading the full messages locally.

  * *Default port:* `143` | *IMAPS port:* `993`
* **POP3 (Post Office Protocol 3):** Protocol used by clients to download emails from the server to local storage, usually deleting them from the server afterward.

  * *Default port:* `110` | *POP3S port:* `995`
* **FQDN (Fully Qualified Domain Name):** The complete domain name for a specific machine on the network (e.g., `daidus.example.com`).
* **Maildir:** A mailbox storage format where each email is stored as an individual file inside a designated folder (`\\\~/Maildir`). It prevents file corruption issues found in legacy single-file `mbox` formats.
* **SASL (Simple Authentication and Security Layer):** A framework providing authentication services to internet protocols. Postfix uses SASL to delegate client login validation to Dovecot.
* **TLS/SSL Certificates:** Encryption keys (`.crt` and `.key`) used to encrypt communication channels between clients and servers to prevent eavesdropping and data tampering.

\---

## 🛠️ Step 1: Virtual Machine Network \& Hostname Setup

### 1\. Network Adapters Setup

When creating your Ubuntu VM in VirtualBox or VMware, configure two network adapters:

1. **NAT Adapter:** Enables outbound internet access to download system packages via `apt`.
2. **Host-Only Adapter:** Creates a private local network between host machine and the VM. Provides a stable IP address for SSH access and local mail client testing.

### 2\. Configure System Hostname and DNS Resolution

Check current hostname details:

```bash
# Display short hostname
hostname

# Display Fully Qualified Domain Name (FQDN)
hostname -f

# Set permanent system hostname
sudo hostnamectl set-hostname daidus.example.com
```

Edit the local host resolution file:

```bash
sudo nano /etc/hosts
```

Ensure the loopback entry maps your FQDN and short hostname:

```text
127.0.0.1   localhost daidus.example.com daidus
```

Reboot the virtual machine to apply system-wide changes:

```bash
sudo reboot
```

\---

## 📦 Step 2: Install and Configure Postfix (MTA)

### 1\. Package Installation

Install Postfix along with SASL authentication tools:

```bash
sudo apt update
sudo apt install -y postfix sasl2-bin
```

:::warning
⚠️ **Prompt Selection:** When prompted during installation, select **No configuration** so we can build a custom configuration file manually.
:::

### 2\. Configure `main.cf`

Copy the template distribution file to create a clean main configuration:

```bash
sudo cp /usr/share/postfix/main.cf.dist /etc/postfix/main.cf
sudo nano /etc/postfix/main.cf
```

Paste and adjust the following parameters:

```ini
mail\\\_owner = postfix

# Server Identity
myhostname = daidus.example.com
mydomain = daidus.example.com
myorigin = $mydomain
inet\\\_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
local\\\_recipient\\\_maps = unix:passwd.byname $alias\\\_maps

# Network \\\& Relay Restrictions
mynetworks\\\_style = subnet
# Note: Update 192.168.133.0/24 to match your VM's Host-Only interface subnet (check using `ip a`)
mynetworks = 127.0.0.0/8, 192.168.133.0/24

# Mailbox Storage \\\& Command Paths
alias\\\_maps = hash:/etc/aliases
alias\\\_database = hash:/etc/aliases
home\\\_mailbox = Maildir/
smtpd\\\_banner = $myhostname ESMTP
sendmail\\\_path = /usr/sbin/postfix
newaliases\\\_path = /usr/bin/newaliases
mailq\\\_path = /usr/bin/mailq
setgid\\\_group = postdrop
inet\\\_protocols = ipv4

# Security Hardening
disable\\\_vrfy\\\_command = yes
smtpd\\\_helo\\\_required = yes
message\\\_size\\\_limit = 10240000

# SASL Authentication Settings (Delegated to Dovecot)
smtpd\\\_sasl\\\_type = dovecot
smtpd\\\_sasl\\\_path = private/auth
smtpd\\\_sasl\\\_auth\\\_enable = yes
smtpd\\\_sasl\\\_security\\\_options = noanonymous
smtpd\\\_sasl\\\_local\\\_domain = $myhostname
smtpd\\\_recipient\\\_restrictions = 
  permit\\\_mynetworks,
  permit\\\_sasl\\\_authenticated,
  reject\\\_unauth\\\_destination
```

### 3\. Build Database \& Restart Postfix

```bash
# Rebuild binary database from /etc/aliases
sudo newaliases

# Restart and verify service status
sudo systemctl restart postfix
sudo systemctl status postfix
```

\---

## 📬 Step 3: Install and Configure Dovecot (MDA)

### 1\. Package Installation

Install Dovecot core binaries alongside POP3 and IMAP server modules:

```bash
sudo apt install -y dovecot-core dovecot-pop3d dovecot-imapd
```

### 2\. Configure Dovecot Files

#### A. Global Listener Configuration (`/etc/dovecot/dovecot.conf`)

```bash
sudo nano /etc/dovecot/dovecot.conf
```

Update line:

```ini
listen = \\\*, ::
```

*(Forces Dovecot to listen on all IPv4 and IPv6 network interfaces.)*

#### B. Authentication Settings (`/etc/dovecot/conf.d/10-auth.conf`)

```bash
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

Update parameters:

```ini
disable\\\_plaintext\\\_auth = no
auth\\\_mechanisms = plain login
```

#### C. Mail Storage Location (`/etc/dovecot/conf.d/10-mail.conf`)

```bash
sudo nano /etc/dovecot/conf.d/10-mail.conf
```

Update parameter:

```ini
mail\\\_location = maildir:\\\~/Maildir
```

#### D. Postfix SASL Socket Integration (`/etc/dovecot/conf.d/10-master.conf`)

```bash
sudo nano /etc/dovecot/conf.d/10-master.conf
```

Locate `unix\\\_listener /var/spool/postfix/private/auth` block and set:

```text
unix\\\_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
}
```

### 3\. Restart Dovecot and Postfix

```bash
sudo systemctl restart dovecot.service
sudo systemctl restart postfix.service
```

\---

## 👤 Step 4: Add Email Accounts

Mail accounts on standard Linux mail servers correspond directly to local Linux user accounts.

```bash
# Create a new system user
sudo adduser daidus

# Set user password
sudo passwd daidus
```

:::success
🎉 **Directory Creation:** The `\\\~/Maildir` directory structure will automatically be generated in `/home/daidus/Maildir/` as soon as the account receives its first email.
:::

\---

## 🖥️ Step 5: Setup Windows / Client Testing

### 1\. Windows Hosts File Resolution

Edit the host resolution file on your host/client Windows machine so it can resolve the local VM domain without a DNS server:

1. Open Notepad or text editor as **Administrator**.
2. Open file path: `C:\\\\Windows\\\\System32\\\\drivers\\\\etc\\\\hosts`
3. Add the line:

```text
   192.168.133.156 daidus.example.com
   ```

### 2\. Thunderbird Configuration

1. Install **Mozilla Thunderbird**.
2. Add a Mail Account with custom parameters:

   * **Username:** Set to local account name (`daidus`) — *Do not append domain name*.
   * **Incoming Server (IMAP):** `daidus.example.com`, Port `143`, SSL `None`.
   * **Outgoing Server (SMTP):** `daidus.example.com`, Port `25`, SSL `None`.

\---

## 🔒 Step 6: SSL/TLS Certificate Encryption Setup

### 1\. Generate Local Certificates (`mkcert`)

```bash
# Install SSL tooling dependencies
sudo apt update \\\&\\\& sudo apt install -y libnss3-tools

# Download mkcert binary executable
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v\\\*-linux-amd64
sudo cp mkcert-v\\\*-linux-amd64 /usr/local/bin/mkcert

# Install Local Certificate Authority (CA)
mkcert -install

# Generate domain certificate and private key
mkcert -cert-file server.crt -key-file private.key localhost 192.168.133.156 daidus.example.com

# Copy generated certificates to standard system SSL locations
sudo cp server.crt /etc/ssl/certs/
sudo cp private.key /etc/ssl/private/

# Grant user permissions for file transfer
sudo cp server.crt /home/daidus/
sudo chown daidus:daidus /home/daidus/server.crt
```

### 2\. Configure SSL/TLS in Postfix

Edit `/etc/postfix/main.cf`:

```bash
sudo nano /etc/postfix/main.cf
```

Add parameters:

```ini
smtpd\\\_use\\\_tls = yes
smtp\\\_tls\\\_mandatory\\\_protocols = !SSLv2, !SSLv3
smtpd\\\_tls\\\_mandatory\\\_protocols = !SSLv2, !SSLv3
smtpd\\\_tls\\\_cert\\\_file = /etc/ssl/certs/server.crt
smtpd\\\_tls\\\_key\\\_file = /etc/ssl/private/private.key
smtpd\\\_tls\\\_session\\\_cache\\\_database = btree:${data\\\_directory}/smtpd\\\_scache
```

Enable Submission (587) and SMTPS (465) ports in `/etc/postfix/master.cf`:

```bash
sudo nano /etc/postfix/master.cf
```

Uncomment and adjust blocks:

```text
submission inet n       -       y       -       -       smtpd
  -o syslog\\\_name=postfix/submission
  -o smtpd\\\_sasl\\\_auth\\\_enable=yes

smtps     inet  n       -       y       -       -       smtpd
  -o syslog\\\_name=postfix/smtps
  -o smtpd\\\_tls\\\_wrappermode=yes
  -o smtpd\\\_sasl\\\_auth\\\_enable=yes
```

### 3\. Configure SSL/TLS in Dovecot

Edit `/etc/dovecot/conf.d/10-ssl.conf`:

```bash
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```

Update parameters:

```ini
ssl = yes
ssl\\\_cert = </etc/ssl/certs/server.crt
ssl\\\_key = </etc/ssl/private/private.key
```

Restart services:

```bash
sudo systemctl restart postfix dovecot
```

### 4\. Client Certificate Installation \& Port Override

1. Transfer `server.crt` to Windows host using FileZilla / SFTP.
2. Double click `server.crt` -> **Install Certificate** -> **Local Machine**.
3. Choose **Place all certificates in the following store** -> Select **Trusted Root Certification Authorities**.
4. **Thunderbird Port Unblocking:**

   * In Thunderbird: Open **Settings** -> **General** -> Scroll to bottom -> **Config Editor...**
   * Search for `network.security.ports.banned.override`
   * Select **String** type and assign value `465`.

\---

## 📑 Configuration Files Summary Cheat Sheet

|File Path|Service|Main Purpose \& Modifications|
|-|-|-|
|`/etc/hosts`|System|Maps domain names locally to IP addresses without external DNS.|
|`/etc/postfix/main.cf`|Postfix|Controls domains, allowed subnets, mailbox formats, SASL, and TLS keys.|
|`/etc/postfix/master.cf`|Postfix|Configures listening ports/daemons (e.g., submission on 587, smtps on 465).|
|`/etc/aliases`|Postfix|Maps system mail aliases to real user accounts.|
|`/etc/dovecot/dovecot.conf`|Dovecot|Main Dovecot configuration; defines listening IP interfaces (`listen = \\\*, ::`).|
|`/etc/dovecot/conf.d/10-auth.conf`|Dovecot|Controls authentication methods and plain text authorization permissions.|
|`/etc/dovecot/conf.d/10-mail.conf`|Dovecot|Defines mail storage format and location (`maildir:\\\~/Maildir`).|
|`/etc/dovecot/conf.d/10-master.conf`|Dovecot|Opens authentication sockets accessible by Postfix (`unix\\\_listener`).|
|`/etc/dovecot/conf.d/10-ssl.conf`|Dovecot|Defines SSL certificate path locations and activates encryption.|



