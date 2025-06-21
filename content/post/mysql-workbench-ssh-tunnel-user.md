---
title: "Standardised MySQL Access via SSH Tunnel for Workbench in Non-Production Environments"
date: 2025-06-21T15:37:38+01:00 
draft: true # Sets whether to render this page. Draft of true will not be rendered.
thumbnail: "/images/path/thumbnail.png" # Sets thumbnail image appearing inside card on homepage.
categories:
  - Technology
tags:
  - Tag_name1
  - Tag_name2
---
## Current Issue & Security Risk
In the current UAT setup, a shared system account `vmadmin` is used for both SSH login to the VM and SSH tunneling for MySQL Workbench. Each trusted internal user (e.g., DevOps and Software Engineers, Support Team) uses a dedicated private SSH key with this account.

This account is configured with passwordless sudo privileges:

```bash
vmadmin ALL=(ALL) NOPASSWD: ALL
```
This effectively grants full root-level access to the system.

Extending this model to external users (e.g., third-party vendors) introduces a serious security risk. Even if access is intended only for MySQL tunneling, using `vmadmin` would allow full VM login and administrative privileges — violating the principle of least privilege (POLP) and increasing the risk of accidental or malicious misuse.

## Secure Approach & Standardisation

To mitigate this, a dedicated, non-interactive SSH user (`workbench-user`) has been introduced, designed specifically for MySQL access via SSH tunneling. This user:

- Has no interactive shell (/bin/false)
- Does not use SSH keys
- Is restricted to password-authenticated tunneling only
- Cannot log into the VM interactively

This setup, already implemented in pre-production, is now being standardised across all non-production environments (DEV, QA, UAT).

Each user is also provided with a dedicated MySQL account tied to an access profile (basic, standard, or maintenance), ensuring a clear separation between VM access and database access.

Therefore, this approach cleanly separates SSH login to the VM itself from MySQL Workbench access, ensuring consistent and secure practices across all non-production environments.

## Implementation Steps
1. Create a non-interactive user for MySQL SSH tunneling only
   
```bash
sudo useradd -m -s /bin/false workbench-user   # no shell access
sudo passwd workbench-user                     # enter a strong password
```

2. Verify user creation:
```bash
grep workbench-user /etc/passwd
# Expected output: workbench-user:x:1005:1005::/home/workbench-user:/bin/false
```
**Note**: `/bin/false` means no interactive shell session, i.e., the user cannot log in to the VM via a terminal session.

3. Restrict SSH access:
Edit `/etc/ssh/sshd_config` and add:

```bash
# Restrict workbench-user access to trusted VPN IP only
AllowUsers workbench-user@<VPN_IP_address>

## BEGIN WORKBENCH CONFIGURATION ###
# Exclusive use of Workbench for 3rd-party users - add [date]
Match User workbench-user
  PasswordAuthentication yes
  AuthenticationMethods password
## END WORKBENCH CONFIGURATION ###
```

**Note**: In this setup, the MySQL database server is only reachable from a specific VPN IP address. This directive enforces IP whitelisting, ensuring that workbench-user can only authenticate from that trusted source.

4. Restart SSH securely:
```bash
sudo sshd -t  # Test config
sudo systemctl restart sshd
sudo systemctl status sshd
```