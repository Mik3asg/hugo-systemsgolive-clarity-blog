---
date: '2025-02-04T16:03:45Z'
draft: false
title: 'Essential Linux Network Commands: A Practical Guide'
tags: ['Networking']
categories: ['Linux']
thumbnail: "images/linux-networking.png"
summary: 'Network troubleshooting and configuration are crucial skills for Linux system administrators. This guide covers essential network commands for AlmaLinux/RHEL systems.'
---

Network troubleshooting and configuration are crucial skills for Linux system administrators or DevOps Engineer. This guide covers essential network commands for AlmaLinux8/RHEL8 systems.
> **Note:** Use `sudo` for commands requiring elevated privileges

## 1. IP Command

Modern replacement for `ifconfig`. Manages network interfaces and routing.

```shell
# Show network interfaces
ip addr show

# Enable/disable interface
ip link set eth0 up
ip link set eth0 down

# Set IP address
ip addr add 192.168.1.100/24 dev eth0

# Show routing table
ip route show

# Add static route
ip route add 10.0.0.0/24 via 192.168.1.1
```

## 2. Socket Statistics (ss)

Replaces `netstat`, provides socket information.

```shell
# Show TCP connections
ss -ta

# Show listening ports
ss -ltpn

# Display processes using port
ss -ltpn | grep :80

# Show UDP sockets
ss -ua
```

## 4. Network Manager

Manages network connections and interfaces.

```shell
# Check NetworkManager status
systemctl status NetworkManager

# View all devices
nmcli device status

# Show connection details
nmcli connection show

# View real-time logs
journalctl -fu NetworkManager

# Restart NetworkManager
sudo systemctl restart NetworkManager

# Connect specific interface
nmcli device connect eth0
```

## 5. Ping

Tests network connectivity.

```shell
# Basic ping
ping google.com

# Limit to 5 packets
ping -c 5 192.168.1.1

# Set interval (seconds)
ping -i 0.5 8.8.8.8

# Set packet size
ping -s 1500 google.com
```

## 6. Traceroute

Identifies network path issues.

```shell
# Basic trace
traceroute google.com

# Use TCP
traceroute -T google.com

# Specify port
traceroute -p 443 google.com
```

## 7. Nmap

Network exploration and security scanning.

```shell
# Scan host
nmap 192.168.1.100

# Scan network range
nmap 192.168.1.0/24

# Scan specific ports
nmap -p 80,443 192.168.1.100

# OS detection
nmap -O 192.168.1.100
```

## 8. cURL

Tests web services and APIs.

```shell
# GET request
curl http://example.com

# Download file
curl -O http://example.com/file.zip

# POST request
curl -X POST -d "data=value" http://api.example.com

# Show headers
curl -I https://example.com
```

## 9. dig

DNS lookup utility.

```shell
# Basic lookup
dig example.com

# Query MX records
dig example.com MX

# Use specific DNS server
dig @8.8.8.8 example.com

# Trace resolution
dig +trace example.com
```

## 10. tcpdump

Network packet analyzer.

```shell
# Capture interface packets
tcpdump -i eth0

# Capture specific port
tcpdump -i eth0 port 80

# Save to file
tcpdump -w capture.pcap

# Read from file
tcpdump -r capture.pcap
```

## 11. iptables

Firewall management.

```shell
# List rules
iptables -L

# Allow SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Block IP
iptables -A INPUT -s 192.168.1.100 -j DROP
```

## 12. netcat (nc)

Network utility tool.

```shell
# Listen on port
nc -l 8080

# Connect to port
nc example.com 80

# Port scanning
nc -zv example.com 20-30
```

## 13. host

Simple DNS lookup.

```shell
# Basic lookup
host example.com

# Reverse lookup
host 8.8.8.8

# Query MX records
host -t MX example.com
```

## Common Use Cases

### Web Server Troubleshooting

```bash
# Check if web server is listening
ss -ltpn | grep :80

# Test local web server
curl -I http://localhost

# Check server certificate
openssl s_client -connect yourdomain.com:443

# Monitor HTTP traffic
tcpdump -i eth0 port 80 -A

# Check error logs
tail -f /var/log/httpd/error_log
```

### Network Connectivity

```bash
# Test DNS and internet connectivity
ping -c 3 8.8.8.8
ping -c 3 google.com

# Check interface status and IP
ip addr show
nmcli device status

# Verify routing
ip route show
traceroute problematic-server.com

# Monitor interface traffic
tcpdump -i eth0 -n

# Check DNS resolution
dig +trace yourdomain.com
```

### Security Checks

```bash
# Port scanning
nmap -sS -p- 192.168.1.0/24

# Check open connections
ss -tapn

# Monitor connections in real-time
watch ss -tapn

# Review firewall rules
firewall-cmd --list-all
iptables -L -n -v

# Check SELinux status
sestatus
getenforce
```

### Performance Analysis

```bash
# Network interface statistics
ip -s link show

# Monitor bandwidth by process
iftop -i eth0

# Track network latency
mtr google.com

# Check network load
netstat -s

# Monitor network errors
watch -n1 'netstat -i'

# NetworkManager real-time logs
journalctl -fu NetworkManager
```

### Service Status

```bash
# Check system services
systemctl status NetworkManager
systemctl status firewalld

# View network services
ss -tulpn

# DNS resolution check
cat /etc/resolv.conf
dig google.com

# Review network logs
journalctl -u NetworkManager
```

> Note: Many commands require root privileges. Use `sudo` as needed.