---
title: "Case Study - Automating Let's Encrypt Wildcard Certificates with Cloudflare DNS and Full (Strict) TLS Encryption"
date: 2026-01-03T23:32:31Z
draft: false
tags: ['TLS', 'SSL', 'Encryption', 'Certificate', 'Automation', 'Cloudflare']
categories: ['Security']
thumbnail: "images/letsencrypt-thumbnail.png"
summary: "This article demonstrates how to implement Full (Strict) SSL/TLS encryption by combining Cloudflare's managed frontend certificates with Let's Encrypt wildcard certificates on your origin server."
---

## Overview

This case study demonstrates how to implement **Full (Strict) SSL/TLS encryption** by combining Cloudflare's managed frontend certificates with Let's Encrypt wildcard certificates on your origin server. This architecture provides:

- **End-to-end encryption** from browser to origin server
- **Wildcard coverage** for unlimited subdomains with a single certificate
- **Automatic renewal** every 60-90 days with zero downtime
- **Free SSL certificates** with industry-standard security
- **Cloudflare proxy protection** (orange cloud) maintained

### Why This Approach?

- **Enhanced Security**: Full (strict) mode ensures encrypted connections throughout the entire path
- **Cost-Effective**: Let's Encrypt certificates are free and auto-renew
- **Scalability**: One wildcard certificate covers all current and future subdomains
- **Reliability**: Cloudflare manages frontend certs while you control backend security

---

## Architecture Overview

```
┌─────────┐           ┌────────────┐           ┌───────────────┐
│ Browser │           │ Cloudflare │           │ Origin Server │
└─────────┘           └────────────┘           └───────────────┘
https://virtualscale.dev
or your domain
     │                      │                         │
     │                      │                         │
     └─────HTTPS────────────┘                         │
        (TLS 1.3)                                     │
     (Cloudflare Cert)                                │
     Full (Strict) Mode                               │
                            │                         │
                            └──────HTTPS──────────────┘
                               (TLS 1.2/1.3)
                            (Let's Encrypt Cert)
                            Auto-renews
                            Validates Certificate
```

**Key Points:**
- Frontend: Cloudflare manages TLS certificates automatically
- Backend: Let's Encrypt ECDSA wildcard certificate (90-day validity, auto-renewal after 60 days)
- DNS Validation: Uses Cloudflare API for DNS-01 challenge (allows keeping orange cloud enabled)

---

## Prerequisites

Before starting, ensure you have:

- [ ] Domain managed by Cloudflare DNS
- [ ] Nginx web server (or Apache)
- [ ] Root/sudo access to AlmaLinux server
- [ ] Cloudflare API token with DNS edit permissions
- [ ] Ports 80 and 443 allowed in firewall (see below)

### Firewall Configuration

```bash
# Allow HTTP (needed for initial setup and renewals)
# Source: Server IP → Destination: ANY → Port: TCP 80 → Action: Allow

# Allow HTTPS (production traffic)
# Source: Server IP → Destination: ANY → Port: TCP 443 → Action: Allow
```

---

## Step 1: Install EPEL Repository

EPEL (Extra Packages for Enterprise Linux) provides Certbot packages.

```bash
# Check if EPEL is already installed
sudo dnf repolist | grep epel

# If no output, install EPEL
sudo dnf install epel-release -y
```

---

## Step 2: Install Certbot and Cloudflare DNS Plugin

### What is Certbot?

**Certbot** is the official Let's Encrypt client that automates the entire certificate lifecycle:
- **Requests** certificates from Let's Encrypt's Certificate Authority (CA)
- **Validates** domain ownership using various challenge methods
- **Installs** certificates on your web server
- **Renews** certificates automatically before expiration

Think of Certbot as your automated certificate manager that handles all communication with Let's Encrypt's API.

### For Nginx:

```bash
# Check for existing Certbot installation
rpm -qa | grep certbot

# Install Certbot for Nginx
sudo dnf install certbot python3-certbot-nginx -y

# Install Cloudflare DNS plugin (required for DNS-01 challenge)
sudo dnf install python3-certbot-dns-cloudflare -y

# Verify installation
certbot --version
certbot plugins
```

### For Apache (Alternative):

```bash
# Install Certbot for Apache
sudo dnf install certbot python3-certbot-apache -y

# Install Cloudflare DNS plugin
sudo dnf install python3-certbot-dns-cloudflare -y

# Optional: Verify SSL module is loaded
sudo httpd -M | grep ssl_module

# If not present, install SSL module
sudo dnf install -y mod_ssl
```

> **Note:** Apache's default `ssl.conf` can conflict with custom vhost configurations. Review and adjust as needed.

---

## Step 3: Create Cloudflare API Token

The API token allows Certbot to create DNS records for domain validation.

### Steps:

1. Go to **Cloudflare Dashboard** → **My Profile** → **API Tokens**
2. Click **Create Token** → Use **Edit zone DNS** template
3. Configure token:
   - **Token name**: `api-token-tls-letsencrypt-webserver`
   - **Permissions**: `Zone → DNS → Edit`
   - **Zone Resources**: `Include → Specific zone → yourdomain.com`
   - **IP Filtering** (optional): Add server's public IP for extra security
   - **TTL** (optional): Set expiration if desired

4. Click **Continue to summary** → **Create Token**
5. **Copy and save the token immediately** (you won't see it again!)

### Test Your Token

Verify the token works before proceeding:

```bash
curl "https://api.cloudflare.com/client/v4/user/tokens/verify" \
  -H "Authorization: Bearer YOUR_CLOUDFLARE_API_TOKEN"
```

Expected output should show `"status": "active"`

---

## Step 4: Store Cloudflare Credentials Securely

**Security Best Practice:** Generate one API token per webserver for better access control.

```bash
# Create secure directory
sudo mkdir -p /root/.secrets

# Create credentials file
sudo vim /root/.secrets/cloudflare.ini
```

Add this content:

```ini
# Cloudflare API Token for TLS/SSL Let's Encrypt
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN_HERE
```

Secure the file (critical step):

```bash
# Set restrictive permissions (owner read/write only)
sudo chmod 600 /root/.secrets/cloudflare.ini

# Verify permissions (should show -rw-------)
sudo ls -l /root/.secrets/cloudflare.ini
```

---

## Step 5: Obtain Wildcard TLS Certificate

### Understanding DNS-01 Challenge

**What is DNS-01 Challenge?**

DNS-01 is one of several domain validation methods Let's Encrypt uses to verify you control a domain. Unlike HTTP-01 (which requires port 80 access), DNS-01 validates ownership by checking for a specific DNS TXT record.

**How DNS-01 Works:**

```
1. You request a certificate from Let's Encrypt
   ↓
2. Let's Encrypt generates a unique challenge token
   ↓
3. Certbot creates a TXT record: _acme-challenge.yourdomain.com
   ↓
4. Let's Encrypt's servers query DNS for this TXT record
   ↓
5. If the record exists with the correct token → Validation successful
   ↓
6. Certificate is issued
   ↓
7. Certbot automatically removes the TXT record (cleanup)
```

**Why We Use DNS-01 for This Setup:**

- **Wildcard Support**: DNS-01 is the *only* method that supports wildcard certificates (`*.domain.com`)  
- **Cloudflare Proxy Compatible**: Works even with Cloudflare's orange cloud (proxy) enabled  
- **No Port Requirements**: Doesn't need port 80/443 exposed to the internet  
- **Server Location Flexible**: Works regardless of server location or firewall rules


### How It Works with Let's Encrypt and Cloudflare

**The Flow in Detail:**

1. **Request**: Certbot contacts Let's Encrypt API requesting a certificate for `*.yourdomain.com`
2. **Challenge**: Let's Encrypt responds with a unique challenge string
3. **DNS Record Creation**: Certbot uses the Cloudflare API (via your API token) to create:
```
   _acme-challenge.yourdomain.com TXT "random_challenge_string"
```
4. **Propagation Wait**: Certbot waits for DNS propagation (default 10s, we set it to 60s)
5. **Validation**: Let's Encrypt queries public DNS servers to verify the TXT record exists
6. **Certificate Issuance**: Upon successful validation, Let's Encrypt issues the certificate
7. **Cleanup**: Certbot removes the temporary TXT record via Cloudflare API

**Why This Requires a Cloudflare API Token:**

Certbot needs programmatic access to create/delete DNS records in your Cloudflare account during the validation process. The API token provides secure, limited access specifically for DNS editing.

### Request the Certificate

This step requests the certificate using DNS-01 validation.
```bash
# Verify Certbot location
which certbot
# Expected: /usr/bin/certbot

# Request wildcard certificate with ECDSA encryption
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  --key-type ecdsa \
  --elliptic-curve secp256r1 \
  -d '*.virtualscale.dev' \
  -d virtualscale.dev
```

**What happens during this process:**

1. Certbot creates temporary TXT records in Cloudflare DNS (`_acme-challenge.yourdomain.com`)
2. Let's Encrypt verifies domain ownership via DNS
3. Wildcard certificate is issued (valid for 90 days)
4. TXT records are automatically cleaned up
5. Certificate files are saved to `/etc/letsencrypt/live/yourdomain/`

### Verify Certificate Installation

```bash
# List certificate files
sudo ls -la /etc/letsencrypt/live/virtualscale.dev/

# Expected output:
# cert.pem -> ../../archive/virtualscale.dev/cert1.pem
# chain.pem -> ../../archive/virtualscale.dev/chain1.pem
# fullchain.pem -> ../../archive/virtualscale.dev/fullchain1.pem
# privkey.pem -> ../../archive/virtualscale.dev/privkey1.pem
```

### Check Certificate Details

```bash
# View certificate information
sudo certbot certificates

# Or check from the server itself
openssl s_client -connect localhost:443 2>/dev/null | \
  openssl x509 -noout -text -dates -issuer -subject
```

**Key Certificate Details:**
- **Issuer**: Let's Encrypt (E7)
- **Validity**: 90 days
- **Algorithm**: ECDSA with SHA-384
- **Key Size**: 256-bit (P-256 curve)
- **Coverage**: `*.virtualscale.dev` and `virtualscale.dev`

---

## Step 6: Install and Configure Nginx

### Install Nginx

```bash
sudo dnf install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

### Create Basic Webpage

```bash
# Create web root directory
sudo mkdir -p /var/www/virtualscale.dev/html

# Create index page
sudo vim /var/www/virtualscale.dev/html/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>VirtualScale.dev</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 50px auto; padding: 20px; }
        h1 { color: #2c3e50; }
        .secure { color: #27ae60; font-weight: bold; }
    </style>
</head>
<body>
    <h1>Welcome to VirtualScale.dev</h1>
    <p class="secure">🔒 Secured with Let's Encrypt SSL</p>
    <p>This site uses <strong>ECDSA (Elliptic Curve Digital Signature Algorithm)</strong> 
       with SHA-384 hashing for enhanced security.</p>
    <p><small>ECDSA provides strong security with smaller key sizes compared to traditional RSA encryption.</small></p>
</body>
</html>
```

### Configure Nginx for TLS

```bash
sudo vim /etc/nginx/conf.d/virtualscale.dev.conf
```

```nginx
# Redirect all HTTP traffic to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name virtualscale.dev www.virtualscale.dev;
    
    # 301 permanent redirect
    return 301 https://$server_name$request_uri;
}

# HTTPS server with Let's Encrypt certificate
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    
    server_name virtualscale.dev www.virtualscale.dev;
    
    root /var/www/virtualscale.dev/html;
    index index.html;
    
    # SSL Certificate Configuration (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/virtualscale.dev/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/virtualscale.dev/privkey.pem;
    
    # Modern SSL Configuration (Security Best Practices)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # Enable HSTS (forces HTTPS for 2 years)
    add_header Strict-Transport-Security "max-age=63072000" always;
    
    # Additional security headers (optional but recommended)
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Test and Apply Configuration

```bash
# Test configuration for syntax errors
sudo nginx -t

# If test passes, restart Nginx
sudo systemctl restart nginx

# Verify Nginx is running
sudo systemctl status nginx

# Check listening ports
sudo ss -tlnp | grep -E ':(80|443)'
```

---

## Step 7: Set Up Automatic Certificate Renewal

### How Auto-Renewal Works

Let's Encrypt certificates expire after 90 days for security reasons. Certbot handles automatic renewal through a systemd timer.

**The Auto-Renewal Process:**

```
Every 12 hours, the certbot-renew.timer triggers:
   ↓
1. Certbot checks all installed certificates
   ↓
2. If certificate expires in < 30 days → Renewal triggered
   ↓
3. Same DNS-01 challenge process runs automatically:
   - Create TXT record via Cloudflare API
   - Let's Encrypt validates
   - New certificate issued
   - Old TXT record removed
   ↓
4. Deploy hooks run (reload Nginx)
   ↓
5. Your server now uses the new certificate (seamless!)
```

**Key Points:**

- **Renewal Window**: Starts attempting renewal at 60 days (30 days before expiry)
- **Retry Logic**: If renewal fails, it retries automatically on the next scheduled run
- **Zero Downtime**: Nginx reload takes milliseconds; users don't notice
- **No Manual Intervention**: Fully automated as long as Cloudflare API token remains valid

**Timeline Example:**

```
Day 0:   Certificate issued (valid for 90 days)
Day 60:  First renewal attempt
Day 61:  If Day 60 failed, retry
Day 62:  If Day 61 failed, retry
...
Day 89:  Final renewal attempts (1 day before expiry)
Day 90:  Certificate expires (if all renewals failed - rare!)
```

### Enable Auto-Renewal Timer

```bash
# Enable timer to start on boot
sudo systemctl enable certbot-renew.timer

# Start timer immediately
sudo systemctl start certbot-renew.timer

# Verify timer status
sudo systemctl status certbot-renew.timer
```

### Configure Nginx Reload After Renewal

**Why This Is Needed:**

When Certbot renews a certificate, it writes new certificate files to disk, but Nginx is still using the *old* certificates loaded in memory. The deploy hook ensures Nginx reloads and starts using the new certificates immediately after renewal.

**Without the hook**: New certificates sit unused until you manually restart Nginx  
**With the hook**: Nginx automatically picks up new certificates within seconds

Create a deploy hook to automatically reload Nginx when certificates renew:

```bash
# Create deploy hooks directory
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy

# Create reload script
sudo vim /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

```bash
#!/bin/bash
# Reload Nginx after Let's Encrypt certificate renewal
# This ensures the web server uses the newly renewed SSL certificates

systemctl reload nginx
```

```bash
# Make script executable
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

### Test Auto-Renewal (Dry Run)

```bash
# Perform dry run (doesn't actually renew, just tests the process)
sudo certbot renew --dry-run
```

If successful, you should see: `Congratulations, all simulated renewals succeeded`

---

## Step 8: Increase DNS Propagation Timeout (Optional but Recommended)

Prevents timeout failures during renewal by allowing more time for DNS changes to propagate.

```bash
# Edit renewal configuration
sudo vim /etc/letsencrypt/renewal/virtualscale.dev.conf

# Add this line in the [renewalparams] section:
dns_cloudflare_propagation_seconds = 60
```

Verify the change:

```bash
grep "propagation" /etc/letsencrypt/renewal/virtualscale.dev.conf
```

Expected output:
```
dns_cloudflare_propagation_seconds = 60
```

---

## Troubleshooting

### Check Logs

```bash
# View Certbot logs
sudo tail -n 100 /var/log/letsencrypt/letsencrypt.log

# View renewal service logs
sudo journalctl -u certbot-renew.service -n 200

# View Nginx error logs
sudo tail -n 50 /var/log/nginx/error.log
```

### Common Issues

**Issue**: DNS validation timeout  
**Solution**: Increase `dns_cloudflare_propagation_seconds` to 60 or 120

**Issue**: Permission denied accessing Cloudflare credentials  
**Solution**: Verify file permissions are 600 and owned by root

**Issue**: Nginx won't start after renewal  
**Solution**: Check for syntax errors with `sudo nginx -t`

**Issue**: Certificate not covering subdomain  
**Solution**: Verify both `*.domain.com` and `domain.com` are in certificate

---

## Verification and Testing

### Verify SSL Certificate

```bash
# Check certificate on local server
openssl s_client -connect localhost:443 2>/dev/null | \
  openssl x509 -noout -dates -issuer -subject

# Expected output should show:
# - Issuer: Let's Encrypt
# - Subject: CN=*.virtualscale.dev
# - Valid dates (90 days from issue)
```

### Test External Access

1. Visit your domain in a browser: `https://virtualscale.dev`
2. Check certificate details (click lock icon in address bar)
3. Verify:
   - Certificate issuer: Let's Encrypt
   - Encryption: TLS 1.3 (if supported by browser)
   - Connection: Secure

#### Visual Confirmation: Successful TLS Implementation

Once configured correctly, your browser will display a secure connection indicator. Here's what a successful implementation looks like:

![Successful TLS Certificate Implementation - Pic 01](images/virtualscale-tls-verification-01.png)
![Successful TLS Certificate Implementation- Pic 02](images/virtualscale-tls-verification-02.png)

**What to look for in the browser:**
- **Padlock icon** in the address bar (indicates HTTPS)
- **"Connection is secure"** message in certificate viewer
- **"Certificate is valid"** status
- **No browser warnings** about insecure connections

This confirms your Let's Encrypt certificate is properly installed, trusted by browsers, and providing end-to-end encryption with Cloudflare's Full (strict) SSL/TLS mode.

### Test Auto-Renewal

```bash
# Force renewal (for testing only - has rate limits!)
sudo certbot renew --force-renewal

# Better: Use dry run
sudo certbot renew --dry-run
```

---

## Security Benefits Summary

**End-to-End Encryption**: Full (strict) mode ensures TLS encryption from browser to origin  
**Modern Cryptography**: ECDSA provides strong security with smaller key sizes  
**Automatic Updates**: Certificates renew automatically before expiration  
**Wildcard Coverage**: One certificate secures all subdomains  
**Zero Downtime**: Renewal happens in background without service interruption  
**Cloudflare Protection**: Maintains DDoS protection and CDN benefits  
**Free Solution**: No certificate costs while maintaining enterprise-grade security

---

## Key Takeaways

1. **Architecture**: Cloudflare handles frontend certificates, Let's Encrypt secures origin server
2. **Validation Method**: DNS-01 challenge allows wildcard certificates with Cloudflare proxy enabled
3. **Automation**: Systemd timer + deploy hooks ensure hands-off operation
4. **Security**: Full (strict) TLS mode provides maximum encryption throughout the connection
5. **Scalability**: Wildcard certificates eliminate the need for per-subdomain certificate management

---

## Additional Resources

- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Cloudflare TLS Full Strict Mode Documentation](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/)
- [Certbot Documentation](https://eff-certbot.readthedocs.io/)
- [Nginx SSL Configuration Generator](https://ssl-config.mozilla.org/)

---

**Need Help?** Check the Troubleshooting section above or review the log files for specific error messages.