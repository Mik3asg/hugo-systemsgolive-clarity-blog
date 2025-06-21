---
title: "MySQL User Profile Creation Guide" # Title of the blog post.
date: 2025-06-21T16:52:43+01:00 # Date of post creation.
description: "Article description." # Description used for search engine.
draft: false 
thumbnail: "images/mysql-user-profile-creation.png" 
summary: "Learn how to set up MySQL user accounts with role-based access profiles, secure configurations, and consistent privilege management across environments."
categories:
  - Database Admininistration
tags:
  - MySQL
  - User Creation
  - Priviledges
  - Role-Bases Access Profiles
  - Best Practices

---

## Overview
This guide explains how to securely create and manage MySQL user accounts using standard access profiles. It covers required configuration, user setup, privilege assignment, and best practices to ensure consistent and controlled access across environments.

## Prerequisites

### System Requirements

- MySQL 8.0.41 or later
- `partial_revokes = ON` must be enabled in MySQL configuration

### Check Configuration (One-Time Check Per Server)

```sql
-- Connect using a MySQL admin or root account
mysql -u root -p                  # or mickael_admin or mysql --login-path=client

-- Check both settings (should return values, not empty)
SHOW VARIABLES LIKE 'partial_revokes';        -- Should be: ON
SHOW VARIABLES LIKE 'validate_password.%';    -- Should return 8 rows
```

### Fix Missing Configuration (If Needed)

```sql
-- If partial_revokes = OFF:
SET GLOBAL partial_revokes = ON;

-- If password validation empty:
INSTALL COMPONENT 'file://component_validate_password';
```

### Make partial_revokes Permanent

Add to `/etc/mysql/my.cnf`:

```bash
[mysqld]
# Added on [DATE]
# Enable partial revokes for user profile management (basic, standard, maintenance, admin)
# Matches Pre-Prod config - allows REVOKE on specific databases after global GRANT
partial_revokes=ON
```

Then restart MySQL: `sudo systemctl restart mysql`

## User Profiles

Naming Convention: `firstname_profile` or `firstname_surname_profile`
Profiles: `_basic`, `_stand`, `_maint`, `_admin`

**Note:** Replace `username` in the commands below with actual names following the naming convention (e.g., `mickael_basic`, `mickael_stand`, `mickael_maint`, `mickael_admin`)


### Basic Profile (`_basic`) - Read Only

```sql
-- Create user with password security settings (5 failed attempts = 1 day lockout)
CREATE USER 'username_basic'@'localhost' IDENTIFIED BY 'secure_password' FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;

-- Grant read-only access to all databases
GRANT SELECT ON *.* TO 'username_basic'@'localhost';

-- Remove access to MySQL system database
REVOKE SELECT ON mysql.* FROM 'username_basic'@'localhost';

-- Apply all privilege changes
FLUSH PRIVILEGES;
```

### Standard Profile (`_stand`) - Data Access

```sql
-- Create user with password security settings (5 failed attempts = 1 day lockout)
CREATE USER 'username_stand'@'localhost' IDENTIFIED BY 'secure_password' FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;

-- Grant data manipulation access to all databases
GRANT SELECT, UPDATE, INSERT, DELETE ON *.* TO 'username_stand'@'localhost';

-- Remove access to MySQL system database
REVOKE SELECT, UPDATE, INSERT, DELETE ON mysql.* FROM 'username_stand'@'localhost';

-- Apply all privilege changes
FLUSH PRIVILEGES;
```

### Maintenance Profile (`_maint`) - Database Admin

```sql
-- Create user with password security settings (5 failed attempts = 1 day lockout)
CREATE USER 'username_maint'@'localhost' IDENTIFIED BY 'secure_password' FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;

-- Grant database administration privileges with ability to grant privileges to others
GRANT SELECT, UPDATE, INSERT, DELETE, CREATE, ALTER, DROP, REFERENCES, LOCK TABLES, CREATE TABLESPACE, CREATE TEMPORARY TABLES, CREATE VIEW, EVENT, EXECUTE, INDEX, SHOW VIEW, TRIGGER ON *.* TO 'username_maint'@'localhost' WITH GRANT OPTION;

-- Remove all administrative access to MySQL system database
REVOKE SELECT, UPDATE, INSERT, DELETE, CREATE, ALTER, DROP, REFERENCES, LOCK TABLES, CREATE TABLESPACE, CREATE TEMPORARY TABLES, CREATE VIEW, EVENT, EXECUTE, INDEX, SHOW VIEW, TRIGGER, GRANT OPTION ON mysql.* FROM 'username_maint'@'localhost';

-- Apply all privilege changes
FLUSH PRIVILEGES;
```

### Admin Profile (`_admin`) - Full Server Access

```sql
-- Create user with password security settings (5 failed attempts = 1 day lockout)
CREATE USER 'username_admin'@'localhost' IDENTIFIED BY 'secure_password' FAILED_LOGIN_ATTEMPTS 5 PASSWORD_LOCK_TIME 1;

-- Grant all traditional database and server administration privileges
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, SHUTDOWN, PROCESS, FILE, REFERENCES, INDEX, ALTER, SHOW DATABASES, SUPER, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER, CREATE TABLESPACE, CREATE ROLE, DROP ROLE ON *.* TO 'username_admin'@'localhost' WITH GRANT OPTION;

-- Grant all dynamic privileges and administrative roles
GRANT APPLICATION_PASSWORD_ADMIN, AUDIT_ABORT_EXEMPT, AUDIT_ADMIN, AUTHENTICATION_POLICY_ADMIN, BACKUP_ADMIN, BINLOG_ADMIN, BINLOG_ENCRYPTION_ADMIN, CLONE_ADMIN, CONNECTION_ADMIN, ENCRYPTION_KEY_ADMIN, FIREWALL_EXEMPT, FLUSH_OPTIMIZER_COSTS, FLUSH_STATUS, FLUSH_TABLES, FLUSH_USER_RESOURCES, GROUP_REPLICATION_ADMIN, GROUP_REPLICATION_STREAM, INNODB_REDO_LOG_ARCHIVE, INNODB_REDO_LOG_ENABLE, PASSWORDLESS_USER_ADMIN, PERSIST_RO_VARIABLES_ADMIN, REPLICATION_APPLIER, REPLICATION_SLAVE_ADMIN, RESOURCE_GROUP_ADMIN, RESOURCE_GROUP_USER, ROLE_ADMIN, SENSITIVE_VARIABLES_OBSERVER, SERVICE_CONNECTION_ADMIN, SESSION_VARIABLES_ADMIN, SET_USER_ID, SHOW_ROUTINE, SYSTEM_USER, SYSTEM_VARIABLES_ADMIN, TABLE_ENCRYPTION_ADMIN, TELEMETRY_LOG_ADMIN, XA_RECOVER_ADMIN ON *.* TO 'username_admin'@'localhost' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

## User Management

### View Users

```sql
-- All users
SELECT User, Host FROM mysql.user ORDER BY User;

-- By profile
SELECT User FROM mysql.user WHERE User LIKE '%_basic';   -- Basic
SELECT User FROM mysql.user WHERE User LIKE '%_stand';   -- Standard  
SELECT User FROM mysql.user WHERE User LIKE '%_maint';   -- Maintenance
SELECT User FROM mysql.user WHERE User LIKE '%_admin';   -- Admin
```

### View Privileges

```sql
SHOW GRANTS FOR 'username_profile'@'localhost';
```

### Modify User

```sql
-- Change password
ALTER USER 'username_profile'@'localhost' IDENTIFIED BY 'new_password';

-- Delete user
DROP USER 'username_profile'@'localhost';

-- Always flush after changes
FLUSH PRIVILEGES;
```

### Test User Access

```sql
-- Exit admin session
EXIT;

-- Login as new user
mysql -u username_profile -p

-- Test access
SHOW DATABASES;
```

## Troubleshooting

### "There is no such grant defined" Error

**Problem:** REVOKE fails after global GRANT

**Solution:** Enable `partial_revokes = ON` and restart MySQL

### Password Validation Errors

**Problem:** "Password does not satisfy policy requirements"

**Requirements:** 8+ chars, uppercase, lowercase, number, special character

**Example Valid Password:** `SecurePass123!`

### Check Configuration Issues

```sql
-- Verify settings
SHOW VARIABLES LIKE 'partial_revokes';
SHOW VARIABLES LIKE 'validate_password.policy';
SELECT VERSION();-- Should be 8.0.16+
```

## Security Guidelines

**Password Requirements:**

- 8+ characters with mixed case, numbers, special chars
- Cannot contain username
- Auto-lock after 5 failed attempts (1 day lockout)

**Profile Guidelines:**

- **Basic:** Application read-only access
- **Standard:** Application data manipulation (most common)
- **Maintenance:** Database administration tasks
- **Admin:** Server administration only (senior DBAs only)

**Best Practices:**

- Use strong passwords (Bitwarden recommended)
- Follow naming convention strictly
- Regular privilege audits
- Document user purpose
- Time-limited access for temporary users
- Limit admin profile creation

## Environment Consistency

Ensure all environments (UAT, Pre-Prod, Production) have:

- Same MySQL version
- `partial_revokes = ON`
- Password validation enabled
- Consistent user privilege patterns