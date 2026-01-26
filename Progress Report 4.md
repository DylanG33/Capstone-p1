# Progress Report 4
## DNS SERVER VM CREATION

### 1. Network Configuration Verification

Verified network settings to document the server's network position within the lab environment.

**Network Details:**
| Setting | Value |
|---------|-------|
| Interface Name | `ens18` |
| IPv4 Address | `10.0.17.8/24` |
| Broadcast | `10.0.17.255` |
| Default Gateway | `10.0.17.2` |
| Protocol | DHCP (reserved) |

**Commands Executed:**
```bash
ip addr show
ip route | grep default
```

**Screenshot: Network Configuration Output**

<img width="1039" height="350" alt="image" src="https://github.com/user-attachments/assets/5403f4c9-3c36-497c-b787-28ae381874f1" />

*Description: Output of ip addr show and ip route commands displaying network interface configuration and default gateway.*

---

### 3. System Updates

Applied all pending system updates to ensure the server has the latest security patches and software versions.

**Commands Executed:**
```bash
sudo apt update
sudo apt upgrade -y
```
---

### 4. BIND9 DNS Server Installation

Installed the BIND9 DNS server software and associated utilities required for DNS resolution and RPZ implementation.

**Commands Executed:**
```bash
sudo apt install -y bind9 bind9utils bind9-doc dnsutils
```

**Packages Installed:**
- `bind9` - The DNS server daemon
- `bind9utils` - DNS utilities (rndc, named-checkconf, etc.)
- `bind9-doc` - Documentation
- `dnsutils` - DNS query tools (dig, nslookup)

---

### 5. BIND9 Service Verification

Confirmed that the BIND9 service started successfully and is enabled for automatic startup on boot.

**Commands Executed:**
```bash
sudo systemctl status bind9
sudo systemctl enable bind9
```

**Screenshot: BIND9 Service Status**

<img width="1320" height="486" alt="image" src="https://github.com/user-attachments/assets/a7026cc1-b674-402b-b3ec-925c55e91257" />


*Description: Output of systemctl status bind9 showing the DNS server is active and running.*

---

### 6. OpenSSH Server Installation

Installed OpenSSH server to enable remote management of the DNS server from client machines.

**Commands Executed:**
```bash
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

**Screenshot: SSH Service Status**

<img width="901" height="384" alt="image" src="https://github.com/user-attachments/assets/fd452aaa-e1a3-43d6-8f68-7c998c05690e" />

*Description: Output of systemctl status ssh confirming SSH server is running and accepting connections.*

---

### 7. BIND9 Configuration - Forwarders Setup

Configured BIND9 to forward DNS queries to upstream resolvers (Google DNS) and enabled query logging for monitoring purposes.

**Configuration File:** `/etc/bind/named.conf.options`

**Screenshot: BIND9 Configuration File**

<img width="993" height="679" alt="bind9config" src="https://github.com/user-attachments/assets/18673bec-0376-4b7b-b5d2-e59e59b1efe8" />


**Configuration Explanation:**
| Directive | Purpose |
|-----------|---------|
| `allow-query { any; }` | Accept DNS queries from all clients |
| `forwarders { 8.8.8.8; 1.1.1.1; }` | Forward queries to Google DNS and Cloudflare |
| `forward only` | Only use forwarders, don't attempt recursive resolution |
| `dnssec-validation no` | Disable DNSSEC (simplifies lab environment) |
| `listen-on { any; }` | Listen on all IPv4 interfaces |
| `listen-on-v6 { none; }` | Disable IPv6 (not configured in lab) |
| `querylog yes` | Enable logging of all DNS queries |

---

### 8. DNS Resolution Testing

Tested DNS resolution to confirm the server correctly forwards and resolves queries.

**Command Executed:**
```bash
dig @localhost google.com
```

**Results:**

<img width="817" height="585" alt="image" src="https://github.com/user-attachments/assets/6a3ffb28-892d-41e6-b4c4-7db006758f38" />


*Description: Output of dig command showing successful DNS resolution of google.com with IP address in the ANSWER SECTION.*

---

### 9. Proxmox Snapshot Created

Created a snapshot in Proxmox to preserve the working DNS server state before implementing RPZ blocking.

