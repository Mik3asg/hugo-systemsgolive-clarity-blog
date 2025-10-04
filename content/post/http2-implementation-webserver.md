---
title: "Implementing HTTP/2 with Zero Downtime: A Blue-Green Deployment Case Study"
date: 2025-04-03T00:15:58Z
draft: false
tags: ['Infrastructure','HTTP/2', 'Apache']
categories: ['Linux']
thumbnail: "images/reverse-proxy.png"
summary: "This case study demonstrates how we implemented HTTP/2 on production Apache servers with zero downtime using a blue-green deployment strategy. After discovering that HAProxy health checks only support HTTP/1.1, we designed a dual-port architecture separating customer traffic (port 443, HTTP/2) from health checks (port 8443, HTTP/1.1), then deployed it through staged phases including pilot testing, parallel infrastructure rollout, and seamless cutover across our four-server environment"
---


# Implementing HTTP/2 with Zero Downtime: A Blue-Green Deployment Case Study

This case study demonstrates how we implemented HTTP/2 on production Apache servers with zero downtime using a blue-green deployment strategy. After discovering that HAProxy health checks only support HTTP/1.1, we designed a dual-port architecture separating customer traffic (port 443, HTTP/2) from health checks (port 8443, HTTP/1.1), then deployed it through staged phases including pilot testing, parallel infrastructure rollout, and seamless cutover across our four-server environment

> **Note:** Domain names have been anonymized for this article. All references to `login.companyabc.com` are used as examples and do not reflect actual production domain names.

## The Problem

We attempted to enable HTTP/2 on our Apache web servers (running on Tomcat VMs) to improve performance for our production application `login.companyabc.com`. However, when we configured Apache with `Protocols h2 http/1.1` on port 443, our ANS load balancer's health checks immediately failed, marking all servers as DOWN and causing a complete service outage.

**Root Cause:** The HAProxy health check process only supports HTTP/1.1. When it attempted to communicate with HTTP/2-enabled ports, the protocol mismatch caused all health checks to fail with "invalid response" errors.

Our pre-production environment (hosted on Vultr without a load balancer) worked perfectly with the same Apache HTTP/2 configuration, confirming the issue was specific to the load balancer health check mechanism.

---

## Deployment Journey Overview

The following diagram illustrates our complete deployment journey from the initial problem through to the final HTTP/2-enabled state. 

Each phase shows the traffic flow from Cloudflare through the ANS Load Balancer to our four Tomcat backend servers, with clear visualization of customer traffic and health check paths, ports, and protocols at each stage.

![HTTP/2 Deployment Phases](images/http2-01.png)
![HTTP/2 Deployment Phases](images/http2-02.png)
![HTTP/2 Deployment Phases](images/http2-03.png)
![HTTP/2 Deployment Phases](images/http2-04.png)

*Figure 1: Complete HTTP/2 deployment progression showing seven phases: The Problem (protocol mismatch causing outage), 

- Phase 1 (initial HTTP/1.1 state).
- Phase 2 (Tomcat-01 isolated in DRAIN mode).
- Phase 3 (dual-port configuration tested).
- Phase 4 (parallel infrastructure rolled out).
- Phase 5 (seamless listener cutover).
- Phase 6 (HTTP/2 enabled with dedicated health check port). 

The diagram clearly distinguishes between customer traffic (port 443) and health check traffic (port 8443), showing how the solution separates these concerns to prevent protocol mismatch.*


---

## The Solution: Dual-Port Architecture

We designed a workaround that separates customer traffic from health check infrastructure by creating **two dedicated VirtualHosts**:

- **Port 443:** Customer traffic (eventually HTTP/2-enabled)
- **Port 8443:** Dedicated HTTP/1.1-only health checks with a lightweight `/healthcheck` endpoint

This approach allows the load balancer to monitor server health using HTTP/1.1 while enabling HTTP/2 for actual user traffic.

## Implementation Strategy: Blue-Green Deployment

Rather than a risky in-place upgrade, we adopted a parallel infrastructure approach:

1. Build new infrastructure alongside existing production
2. Validate thoroughly while production continues unchanged
3. Perform seamless cutover with zero downtime
4. Enable HTTP/2 as a separate, final enhancement

### Phase 1: Pilot Testing on Drained Server

**Understanding "DRAIN Mode":**
When a Tomcat server is set to DRAIN mode in the load balancer:
- No new user sessions are created on that server
- Existing sessions continue until they naturally expire (12-hour sticky session timeout in our configuration)
- The server becomes isolated from production traffic, making it safe for testing
- Health checks continue to monitor the server

**Why We Didn't Enable HTTP/2 in Phase 1:**

The key principle in our approach was **separation of concerns**. We deliberately kept HTTP/1.1 on both ports during pilot testing to:
- Validate the dual-VirtualHost infrastructure works correctly
- Prove the server can exist in both target groups simultaneously
- Eliminate variables during testing (infrastructure change only, no protocol change)
- Ensure easy rollback if issues arose

This staged approach meant we could confidently validate the health check solution before introducing the complexity of HTTP/2 protocol negotiation.

**Test Configuration on Tomcat-01:**

We created two dedicated VirtualHosts with identical application logic but different purposes:

```apache
# Port listening configuration
Listen 443   # Customer traffic
Listen 8443  # Health checks

# HTTP to HTTPS redirect (not shown for brevity)
<VirtualHost *:80>
   ServerName login.companyabc.com
   Redirect / https://login.companyabc.com/
</VirtualHost>

# VirtualHost 1: Customer Traffic Port (HTTP/1.1 for testing)
<VirtualHost _default_:443>
    ServerName login.companyabc.com
    Protocols http/1.1  # Deliberately kept as HTTP/1.1 for validation
    
    Include common-conf.d/ssl-vhost.conf
    Include common-conf.d/gzip.conf
    
    # Security headers and SSL configuration
    RequestHeader set X-Forwarded-Proto "https"
    Header edit Location ^http:// https://
    Header always set X-Frame-Options SAMEORIGIN
    Header always set X-Content-Type-Options nosniff
    
    ProxyRequests off
    ProxyPreserveHost on
    DocumentRoot "/var/www/html"
    
    # Application routing (simplified for clarity)
    RedirectMatch "^/(?!web/|admin/|custom_error_pages/).*$" /web/
    
    # Proxy to Tomcat application
    ProxyPass /web/ http://127.0.0.1:8080/web/ timeout=600
    ProxyPassReverse /web/ http://127.0.0.1:8080/web/
    ProxyPass /admin/ http://127.0.0.1:8080/admin/
    ProxyPassReverse /admin/ http://127.0.0.1:8080/admin/
</VirtualHost>

# VirtualHost 2: Dedicated Health Check Port (HTTP/1.1 only)
<VirtualHost _default_:8443>
    ServerName login.companyabc.com
    Protocols http/1.1  # Critical: Must remain HTTP/1.1 for health checks
    
    Include common-conf.d/ssl-vhost.conf
    Include common-conf.d/gzip.conf
    
    # Security headers and SSL configuration
    RequestHeader set X-Forwarded-Proto "https"
    Header edit Location ^http:// https://
    Header always set X-Frame-Options SAMEORIGIN
    Header always set X-Content-Type-Options nosniff
    
    ProxyRequests off
    ProxyPreserveHost on
    DocumentRoot "/var/www/html"
    
    # Lightweight health check endpoint - bypasses Tomcat for better performance
    <Location "/healthcheck">
        ProxyPass !              # Don't proxy to Tomcat
        SetHandler none          # Serve directly from Apache
        Require all granted      # Allow access
    </Location>
    
    # Standard application proxying (same as port 443)
    ProxyPass /web/ http://127.0.0.1:8080/web/ timeout=600
    ProxyPassReverse /web/ http://127.0.0.1:8080/web/
    ProxyPass /admin/ http://127.0.0.1:8080/admin/
    ProxyPassReverse /admin/ http://127.0.0.1:8080/admin/
</VirtualHost>
```

**Health Check Endpoint Setup:**
```bash
# Create dedicated health check directory and simple response page
mkdir -p /var/www/html/healthcheck
echo "OK" > /var/www/html/healthcheck/index.html
```

**Graceful Configuration Reload:**
```bash
httpd -t  # Validate syntax
systemctl reload httpd  # Graceful reload - no connection drops
```

We created a new target group `tomcatshttp2` with health checks pointing to `GET login.companyabc.com:8443/healthcheck/`, and we validated that Tomcat-01 passed health checks in both the old (`tomcats`) and new (`tomcatshttp2`) target groups simultaneously.

### Phase 2: Production Rollout

After successful pilot testing, we rolled out the same configuration to production servers (Tomcat-02, 03, 04):

**Rollout Process:**
1. Applied identical dual-VirtualHost configuration to each server
2. Created `/healthcheck` directory and endpoint on each server
3. Used `systemctl reload httpd` for graceful, zero-downtime updates
4. We added each server to `tomcatshttp2` target group after completion
5. Verified health check status for each server before proceeding to the next

**Infrastructure State After Phase 2:**
- **Target Group `tomcats`:** All servers, health checks via port 443 `/web/login`, handling all customer traffic
- **Target Group `tomcatshttp2`:** All servers, health checks via port 8443 `/healthcheck`, monitoring but not serving traffic

This parallel infrastructure allowed us to thoroughly validate the new health check system while production traffic continued unaffected.

### Phase 3: The Seamless Cutover

As illustrated in Phase 5 of the deployment diagram, the listener cutover was the critical moment where we switched from the old infrastructure to the new.

**Load Balancer Configuration Change:**

We switched the listener's default target group from `tomcats` to `tomcatshttp2` - a single dropdown change in the load balancer UI that took approximately 30 seconds.

**Why Zero Downtime:**
- Same servers in both target groups
- Same port 443 configuration (HTTP/1.1)
- Same application responses
- Existing connections continued normally
- New connections immediately routed to new target group
- Only operational change: health checks switched from port 443 to port 8443

**Note on Session Stickiness:** The cutover reset session cookies (different between target groups), potentially logging out active users. We performed this change during a low-traffic period to minimize impact.

**Monitoring During Cutover:**
```bash
# Watched Apache logs on all servers during cutover
tail -f /var/log/httpd/access_log

# Customer traffic continued on port 443:
10.0.0.7 - - [29/Sep/2025:11:15:33 +0100] "GET /web/login HTTP/1.1" 200 3507

# Health checks now appeared on dedicated port 8443:
10.0.0.7 - - [29/Sep/2025:11:15:35 +0100] "GET /healthcheck/ HTTP/1.1" 200 88
```

### Phase 4: HTTP/2 Enablement

With stable infrastructure in place and the new health check system proven reliable, we enabled HTTP/2 on the customer-facing port 443.

**Final Production Configuration:**

```apache
Listen 443
Listen 8443

# VirtualHost 1: Customer Traffic Port - HTTP/2 NOW ENABLED
<VirtualHost _default_:443>
    ServerName login.companyabc.com
    Protocols h2 http/1.1  # HTTP/2 enabled! Falls back to HTTP/1.1 for older clients
    
    Include common-conf.d/ssl-vhost.conf
    Include common-conf.d/gzip.conf
    
    # Security headers and SSL configuration
    RequestHeader set X-Forwarded-Proto "https"
    Header edit Location ^http:// https://
    Header always set X-Frame-Options SAMEORIGIN
    Header always set X-Content-Type-Options nosniff
    
    ProxyRequests off
    ProxyPreserveHost on
    DocumentRoot "/var/www/html"
    
    RedirectMatch "^/(?!web/|admin/|custom_error_pages/).*$" /web/
    
    ProxyPass /web/ http://127.0.0.1:8080/web/ timeout=600
    ProxyPassReverse /web/ http://127.0.0.1:8080/web/
    ProxyPass /admin/ http://127.0.0.1:8080/admin/
    ProxyPassReverse /admin/ http://127.0.0.1:8080/admin/
</VirtualHost>

# VirtualHost 2: Health Check Port - REMAINS HTTP/1.1 ONLY
<VirtualHost _default_:8443>
    ServerName login.companyabc.com
    Protocols http/1.1  # Must stay HTTP/1.1 for health checks
    
    Include common-conf.d/ssl-vhost.conf
    Include common-conf.d/gzip.conf
    
    RequestHeader set X-Forwarded-Proto "https"
    Header edit Location ^http:// https://
    Header always set X-Frame-Options SAMEORIGIN
    Header always set X-Content-Type-Options nosniff
    
    ProxyRequests off
    ProxyPreserveHost on
    DocumentRoot "/var/www/html"
    
    # Lightweight health check endpoint
    <Location "/healthcheck">
        ProxyPass !
        SetHandler none
        Require all granted
    </Location>
    
    ProxyPass /web/ http://127.0.0.1:8080/web/ timeout=600
    ProxyPassReverse /web/ http://127.0.0.1:8080/web/
    ProxyPass /admin/ http://127.0.0.1:8080/admin/
    ProxyPassReverse /admin/ http://127.0.0.1:8080/admin/
</VirtualHost>
```

We enabled HTTP/2 on port 443 one server at a time, using graceful Apache reloads, and validating each change before proceeding to the next.

**Verification:**
```bash
# Apache access logs showing HTTP/2 for customer traffic:
10.0.0.7 - - [29/Sep/2025:11:33:41 +0100] "GET /web/login HTTP/2.0" 200 3507

# Health checks still using HTTP/1.1 on dedicated port:
10.0.0.7 - - [29/Sep/2025:11:33:42 +0100] "GET /healthcheck/ HTTP/1.1" 200 88
```

## Results and Lessons Learned

**Achievements:**
- Zero downtime throughout entire implementation
- HTTP/2 successfully enabled for all customer traffic
- Improved health check reliability with dedicated, lightweight endpoint
- Clear separation of concerns (customer traffic vs. operational monitoring)

**Key Success Factors:**
1. **Pilot testing on drained server** reduced risk before production rollout
2. **Parallel infrastructure** allowed thorough validation without affecting production
3. **Staged approach** separated infrastructure changes from protocol changes
4. **Graceful Apache reloads** eliminated service interruptions during configuration changes
5. **Dedicated VirtualHosts** provided clear separation between customer traffic and health checks
6. **Strong vendor partnership** with ANS provided expert guidance throughout

**Post-Implementation:**
- Kept legacy `tomcats` target group for one week as rollback safety net
- Restricted DMZ network policy to allow only TCP/8443 for health checks (security best practice)
- Load balancer OS upgrade postponed until HTTP/2 implementation proven stable

This approach demonstrates how enterprise-grade infrastructure changes can be executed with zero customer impact by leveraging proven deployment patterns, careful planning, and staged implementation. The key insight was recognizing that infrastructure validation and feature enablement are separate concerns that benefit from being tackled independently.