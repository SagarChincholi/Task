# Fortune Cloud Technologies — Day 1 Tasks

Linux networking fundamentals and AWS EC2 network troubleshooting — 5 practical tasks covering IP configuration, IPv4 analysis, dynamic IP behavior, cloud server connectivity, and systematic network troubleshooting.

All tasks were completed on an AWS EC2 Linux instance, accessed via SSH.

## Overview

| Task | Title | Environment |
|---|---|---|
| 1 | Linux IP Investigation | Linux (EC2) |
| 2 | IPv4 Address Analysis | Linux (EC2) |
| 3 | Dynamic IP Investigation | AWS EC2 |
| 4 | Cloud Linux Server & IP | AWS EC2 |
| 5 | Cloud Network Troubleshooting | AWS EC2 |

---

## Task 1 — Linux IP Investigation

**Objective:** Identify the system's IP address, active network interface, default gateway, DNS configuration, and full network configuration.

### Commands Used
```bash
# IP address
ip addr show
hostname -I

# Network interface
ip link show
ip route show default

# Default gateway
ip route show default

# DNS information
cat /etc/resolv.conf
resolvectl status

# Full network configuration
nmcli device show <interface>
```

### Result
Successfully identified the instance's private IP, active interface (`eth0`), default gateway, and DNS resolver configuration.

---

## Task 2 — IPv4 Address Analysis

**Objective:** Break the IPv4 address into its four octets, determine private vs. public status, and identify other reachable devices.

### Commands Used
```bash
# IPv4 address
ip -4 addr show

# Octet / subnet breakdown
ipcalc 172.31.20.15/20

# Public IP
curl ifconfig.me
curl -s ipinfo.io

# Other devices on the network
ip neigh
arp -a
```

### IP Address Breakdown

Example address: `172.31.20.15`

| Octet Position | Value | Binary |
|---|---|---|
| 1st | 172 | 10101100 |
| 2nd | 31 | 00011111 |
| 3rd | 20 | 00010100 |
| 4th | 15 | 00001111 |

### Private vs. Public
- **Private IP (inside Linux):** `172.31.20.15` — falls within the `172.16.0.0 – 172.31.255.255` RFC 1918 private range.
- **Public IP:** the instance's internet-facing address, confirmed against AWS's NAT/Internet Gateway.
- `ipcalc` confirmed this directly, labeling the address space as **Private Use**.

### Result
Confirmed the private IP `172.31.20.15/20` (network `172.31.16.0/20`, broadcast `172.31.31.255`, 4094 usable hosts) and its corresponding public IP.

---

## Task 3 — Dynamic IP Investigation

**Objective:** Observe how IP addressing behaves across an instance stop/start cycle and record a before/after comparison.

**Note:** Disconnecting the network interface directly over an active SSH session would sever the connection being used to run the command. Instead, this was demonstrated safely by stopping and starting the EC2 instance via the AWS Console — a more realistic simulation of dynamic IP behavior in a cloud environment.

### Commands Used
```bash
# Before — record current state
echo "BEFORE - Private IP: $(hostname -I) | Public IP: $(curl -s ifconfig.me)" > ~/ip_record.txt

# Check connection status
nmcli device status

# [Instance stopped and restarted via AWS Console]

# After — reconnect via SSH using new public IP, then record
echo "AFTER  - Private IP: $(hostname -I) | Public IP: $(curl -s ifconfig.me)" >> ~/ip_record.txt

# Display combined record
cat ~/ip_record.txt
```

### Terminal-Based Record

| State | Private IP | Public IP |
|---|---|---|
| Before stop | 172.31.20.15 | *(old public IP)* |
| After start | 172.31.20.15 | *(new public IP)* |

### Result
- **Public IP changed** after stop/start — AWS releases the public IP back to its pool on stop and assigns a new one on start (unless an Elastic IP is attached).
- **Private IP remained the same** — it is tied to the instance's network interface (ENI), not to its power state.

---

## Task 4 — Cloud Linux Server & IP

**Objective:** Launch an AWS EC2 Linux instance, connect via SSH, and compare the private/public IP shown inside Linux against the AWS Console.

### Commands Used
```bash
# Connect via SSH
chmod 400 your-key.pem
ssh -i "your-key.pem" ec2-user@<public-ip>

# Private IP (inside Linux)
hostname -I
ip -4 addr show

# Private IP (via instance metadata)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4

# Public IP
curl -s ifconfig.me
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
```

### IP Comparison

| Source | Private IP | Public IP |
|---|---|---|
| AWS Console (Details tab) | 172.31.20.15 | *(matches below)* |
| Inside Linux (`hostname -I` / `curl ifconfig.me`) | 172.31.20.15 | *(matches above)* |

### Result
The private and public IPs retrieved from inside the Linux instance matched exactly with the values shown in the AWS Console, confirming consistency between the OS-level and cloud-provider-level network views.

---

## Task 5 — Cloud Network Troubleshooting

**Objective:** Systematically diagnose why a service running on a Linux cloud server is unreachable, identify the root cause, fix it, and demonstrate the working result.

### Scenario
Nginx was installed and running on the EC2 instance, but the web page was unreachable from a browser at `http://<public-ip>`.

### Commands Used
```bash
# 1. IP address
ip -4 addr show

# 2. Network interface
ip link show

# 3. Default route
ip route show

# 4. Connectivity
ping -c 4 8.8.8.8
curl -I https://www.google.com

# 5. Running services
systemctl status nginx

# 6. Listening ports
sudo ss -tulnp

# 7. Firewall / security rules
sudo firewall-cmd --list-all      # or: sudo ufw status
# + check Security Group inbound rules in AWS Console

# 8. Fix — add inbound rule for port 80 in the Security Group (via Console)

# 9. Verify the fix
curl -I http://localhost
curl -I http://<public-ip>
```

### Troubleshooting Documentation

| Field | Detail |
|---|---|
| **Problem** | Web service on port 80 unreachable from outside the instance |
| **Command Used** | `ip addr show`, `ip link show`, `ip route show`, `ping 8.8.8.8`, `systemctl status nginx`, `sudo ss -tulnp` |
| **Output** | Interface UP, routing correct, outbound connectivity fine, nginx `active (running)`, port 80 listening on `0.0.0.0` — OS-level networking fully healthy |
| **Cause** | Security Group had no inbound rule allowing port 80 (HTTP) |
| **Solution** | Added inbound rule: HTTP, port 80, source `0.0.0.0/0` |
| **Final Result** | `curl -I http://<public-ip>` returns `HTTP/1.1 200 OK`; nginx welcome page loads in browser |

### Result
Root cause was an AWS Security Group missing an inbound rule for port 80. After adding the rule, the service became reachable, confirmed both via `curl` and a browser load of the nginx default page.

---

## Key Learnings

- `ip` and `nmcli` are the modern standard for Linux network diagnostics; `ifconfig`/`route`/`netstat` are legacy but still common in exam/reference material.
- A cloud instance's **private IP** is tied to its network interface (ENI) and typically persists across stop/start, while its **public IP** is drawn from AWS's pool and usually changes — unless an Elastic IP is attached.
- Cloud network troubleshooting requires checking **both layers**: the OS (interface, routing, service, listening ports) and the cloud provider's controls (Security Groups) — a fully healthy OS-level stack can still be unreachable due to a missing cloud firewall rule.

## Environment

- **Cloud Platform:** AWS EC2
- **OS:** Amazon Linux 2023 / Ubuntu 22.04
- **Access:** SSH (key pair authentication)
