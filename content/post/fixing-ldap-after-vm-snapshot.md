---
title: "LDAP Service Failure After VM Snapshot – Configuration Mismatch Resolution"
date: 2026-03-03T14:30:34Z
draft: false
tags: ["LDAP", "OpenLDAP", "Linux", "AlmaLinux", "Troubleshooting", "SLAPD", "CRC32", "VM Snapshot"]
categories: ["System Administration", "Troubleshooting", "Infrastructure"]
thumbnail: "images/ldap-failure-after-vm-snapshot.png"
summary: "Resolving an OpenLDAP service failure on a VM snapshot — fixing hostname identity mismatches, loopback address issues, olcServerID with CRC32 recalculation, and removing leftover replication config from the source server."
---

## Environment

| Component | Version |
|---|---|
| OS | AlmaLinux 8.10 (Cerulean Leopard) |
| LDAP | OpenLDAP 2.5.18 |

---

## Background

Recently I was asked to help troubleshoot an LDAP service that wasn't starting on a newly created VM. The VM in question, `SNAP-198`, was a snapshot of our Pre-Release server, spun up to test something in isolation — hence the name. Sounds straightforward — but as soon as the snapshot was booted, LDAP was in a failed state.

---

## The Problem

The issue was a classic snapshot trap. When you snapshot a running server, the new VM inherits everything — including the source server's identity. LDAP is particularly sensitive to this because it uses its own hostname to identify itself in the configuration.

The errors in the logs made it clear:

```text
daemon: bind() failed errno=99 (Cannot assign requested address)
read_config: no serverID / URL match found
```

LDAP was trying to bind to an address that didn't exist on this new server, and couldn't match its own identity in the config.

---

## Root Causes

After digging in, there were four things that needed fixing:

- `SLAPD_URLS` still referencing the Pre-Release server hostname
- `olcServerID` in the LDAP config still pointing to the Pre-Release server
- `SNAP-198`'s hostname resolving to `127.0.0.1` (loopback) instead of its actual public IP
- Replication config still pointing back to Pre-Release — not needed since `SNAP-198` is a standalone server

---

## The Fix

### 1. Update SLAPD URLs — `/etc/sysconfig/slapd`

```bash
sudo vim /etc/sysconfig/slapd
```

```text
SLAPD_URLS="ldapi:/// ldap://SNAP-198 ldap://localhost"
```

Verify the change:

```bash
sudo cat /etc/sysconfig/slapd
```

```text
## Reason: Give access to file to specific group. This can now be edited as required ##
#SLAPD_URLS="ldapi:/// ldap://Pre-Prod ldap://localhost"
SLAPD_URLS="ldapi:/// ldap://SNAP-198 ldap://localhost"
```

### 2. Fix `/etc/hosts`

Remove the loopback entry for the hostname and map it to the actual public IP:

```bash
sudo vim /etc/hosts
```

```text
<public-ip> SNAP-198 SNAP-198
```

### 3. Fix `olcServerID`

This is where it got interesting. The right way to modify LDAP config is through `ldapmodify` — the config files explicitly say do not edit directly. So I prepared the ldif file and tried to apply it:

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /tmp/fix_serverid.ldif
```

But this failed immediately — `ldapmodify` requires `slapd` to be running, and `slapd` wouldn't start because of the bad config. A classic chicken and egg situation.

The only way out was to edit the config file directly and manually recalculate the CRC32 checksum that OpenLDAP uses to validate the file:

```bash
sudo vim /etc/openldap/slapd.d/cn=config.ldif
```

Change:

```text
olcServerID: 0 ldap://Pre-Release
olcServerID: 1 ldap://Other-Server
```

To:

```text
olcServerID: 0 ldap://SNAP-198
```

OpenLDAP stores a CRC32 checksum at the top of `cn=config.ldif`. When `slapd` starts, it recalculates the checksum from the file content and compares it against the stored value. If they don't match, `slapd` rejects the file:

```text
ldif_read_file: checksum error on "/etc/openldap/slapd.d/cn=config.ldif"
```

Editing the file directly changes the content but leaves the stored checksum pointing to the old version. Recalculating it and updating the value at the top of the file tells `slapd` the content is valid.

Then recalculate the CRC32:

```bash
sudo python3 -c "
import binascii
with open('/etc/openldap/slapd.d/cn=config.ldif', 'r') as f:
    lines = f.readlines()
content = ''.join(lines[2:])
crc = binascii.crc32(content.encode()) & 0xffffffff
print(f'{crc:08x}')
"
```

Update the `# CRC32 <value>` line at the top of the file with the new value.

### 4. Remove Replication Config

Now that `slapd` was up, I could use `ldapmodify` properly to remove the unwanted replication entries.

Create the ldif files:

```bash
sudo vim /tmp/fix_config_syncrepl.ldif
```

```text
dn: olcDatabase={0}config,cn=config
changetype: modify
delete: olcSyncrepl
```

```bash
sudo vim /tmp/fix_mdb_syncrepl.ldif
```

```text
dn: olcDatabase={1}mdb,cn=config
changetype: modify
delete: olcSyncrepl
```

Apply both:

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /tmp/fix_config_syncrepl.ldif
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /tmp/fix_mdb_syncrepl.ldif
```

---

## Verification

```bash
sudo systemctl restart slapd
sudo systemctl status slapd
sudo grep -r "olcSyncrepl" /etc/openldap/slapd.d/
```

No output on the last command — `SNAP-198` is now running as a fully standalone LDAP server with no dependency on Pre-Release.

---

## Key Takeaway

When creating a VM snapshot from a running LDAP server, always update the server identity configuration before starting LDAP. The config doesn't just hold data — it holds the server's identity, and LDAP will refuse to start if that identity doesn't match the environment it's running in.
