---
title: "Configuration and Usage of Custom Apache 503 Error Pages for Web Application Instances"
date: 2025-04-03T00:15:58Z1
draft: false
tags: ['Infrastructure','Web Applications', 'Apache']
categories: ['Linux']
thumbnail: ""
summary: "This article outlines the implementation and usage of custom Apache 503 error pages on production Tomcat web application instances"
---
## Overview
This article outlines the implementation and usage of custom Apache 503 error pages on production Tomcat web application instances. These error pages are used to provide clear communication to users during service disruptions, with two supported scenarios:
- Unexpected Outage (default)
- Planned Maintenance

## 1 – Implementation
These actions were performed only once during the initial implementation and are **not required for routine operations**.

### Step 1 – Backup Original Apache Configuration

```bash
sudo mkdir -p /root/apache-config-backup
sudo cp -a --no-preserve=timestamps /etc/httpd/conf/httpd.conf /root/apache-config-backup/httpd.conf-$(date +"%Y%m%d")
```

### Step 2 – Update Apache Configuration File
To enable custom Apache 503 error pages, add the following directives directly in your Apache configuration file:
- **File**: /etc/httpd/conf/httpd.conf
- **Location**: Inside the `<VirtualHost _default_:443>` block.

#### Required Directives (in order):

```bash
RedirectMatch "^/(?!web/|admin/|custom_error_pages/).*$" /web/
```
If your configuration already uses a `RedirectMatch rule`, make sure to append `custom_error_pages/` to the allowed paths. This ensures error pages are not redirected away and can be served correctly by Apache.

#### Enable error page handling (place between proxy headers and backend proxy rules):

```bash
DocumentRoot "/var/www/html"
ProxyPass /custom_error_pages/ !
ErrorDocument 503 /custom_error_pages/503-unexpected-outage.html
# ErrorDocument 503 /custom_error_pages/503-planned-maintenance.html
```
Place these directives after `ProxyRequests Off` / `ProxyPreserveHost On` and before any `ProxyPass` rules to backend applications.

### Step 3 – Create Directory and Deploy HTML Files

Execute the following as the root user to create and secure the directory:

```bash
mkdir -p /var/www/html/custom_error_pages
chmod 755 /var/www/html/custom_error_pages
chcon -R --reference=/var/www/html /var/www/html/custom_error_pages
ls -ldZ /var/www/html/custom_error_pages
```

Create and edit the HTML pages:

```bash
vim /var/www/html/custom_error_pages/503-unexpected-outage.html
vim /var/www/html/custom_error_pages/503-planned-maintenance.html
```
 **Reference:** Example HTML files for both error scenarios can be found on in my GitHub repository:
 - [503-unepected-outatage.html](https://github.com/Mik3asg/custom-apache-503-error-pages/blob/main/503-unexpected-outage.html)
- [503-planned-maintenace.html](https://github.com/Mik3asg/custom-apache-503-error-pages/blob/main/503-planned-maintenance.html)

### Step 4 – Ensure Permissions and SELinux Context

Ensure both HTML files are accessible by Apache:
```bash
chmod 644 /var/www/html/custom_error_pages/503-*.html
chcon -u system_u -t httpd_sys_content_t /var/www/html/custom_error_pages/503-*.html
ls -lZh /var/www/html/custom_error_pages/503-*.html
```

## 2 – Operational Guide

This section covers routine operational procedures for switching between the two custom Apache 503 error pages.


### 2.1	– Default Behavior: Unexpected Outage Page (active now)
- The following page is active by default on all Tomcat web app instances in Production:
`/var/www/html/custom_error_pages/503-unexpected-outage.html`

- It is shown automatically if Tomcat becomes unavailable unexpectedly.

- No action is needed during normal operations.

### 2.2	– Switch to Planned Maintenance Page

#### Step 1 – Update the maintenance message content:

Before enabling the planned maintenance error page, manually edit the HTML file to include the correct date and time of the scheduled maintenance window: `/var/www/html/custom_error_pages/503-planned-maintenance.html`

**Important**:  This must be done proactively before the downtime.

#### Step 2 – Modify Apache configuration:
Edit the `/etc/httpd/conf/httpd.conf file`.

You need to **comment out** the line for the **unexpected outage** page (disabling it temporarily) and **uncomment** the line for the **planned maintenance** page (making it active during the maintenance window).

```bash
# ErrorDocument 503 /custom_error_pages/503-unexpected-outage.html
ErrorDocument 503 /custom_error_pages/503-planned-maintenance.html
```

#### Step 3 – Validate Apache configuration and restart:

```bash
sudo apachectl configtest
sudo systemctl restart httpd
sudo systemctl status httpd
```

#### Step 4 – Stop Tomcat (e.g., Tomcat-01):

```bash
sudo systemctl stop tomcat
```

#### Step 5 – Test
- Visit: https://www.example.com
- Clear your browser cache and refresh until you land on Tomcat-01

#### Step 6 – Verify:
You should see the custom Planned Maintenance custom error page.

#### Step 7 – Restart Tomcat:
Once the maintenance activity is complete, do not forget to restart Tomcat
```bash
sudo systemctl start tomcat
```

#### Step 7 – Repeat:
Repeat the above steps on all the required Tomcat web app instances (Tomcat-02, Tomcat-03, and Tomcat-04).

**Important**: Once the planned maintenance activity is complete, please immediately revert to the default Unexpected Outage page for any future 503 errors – see section below.

### 2.3	– Revert to Unexpected Outage Page

#### Step 1 – Modify Apache configuration
Edit the `/etc/httpd/conf/httpd.conf` file.
For this scenario, you now need to uncomment the line for the unexpected outage page (i.e. making it active again) and comment out the line for the planned maintenance page (i.e. disabling it until the next planned maintenance activity):

```bash
ErrorDocument 503 /custom_error_pages/503-unexpected-outage.html
# ErrorDocument 503 /custom_error_pages/503-planned-maintenance.html
```

#### Step 2 – Validate Apache configuration and restart:

```bash
sudo apachectl configtest
sudo systemctl restart httpd
sudo systemctl status httpd
```

**Note**: The steps described below are optional but recommended to confirm that the Unexpected Outage page has been successfully restored as the default page for all future 503 errors.

#### Step 3 – Stop Tomcat (e.g., Tomcat-01):

```bash
sudo systemctl stop tomcat
```

#### Step 4 – Test
- Visit: https://www.example.com
- Clear your browser cache and refresh until you land on Tomcat-01

#### Step 5 – Verify:
Validate if you see the custom Unexpected Outage error page. Now, the Unexpected Outage page will be the default page for all future 503 errors.

#### Step 6 – Restart Tomcat:

```bash
sudo systemctl start tomcat
```

#### Step 7 – Repeat:
Repeat the above steps on the required Tomcat web app instances.
