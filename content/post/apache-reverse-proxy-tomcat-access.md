---
title: "Bypassing Security Group Restrictions: Using Apache as a Reverse Proxy to Access Tomcat in a Test Environment"
date: 2025-04-03T00:15:58Z
draft: true
tags: ['Infrastructure','Reverse Proxy', 'Apache']
categories: ['Linux']
thumbnail: "images/reverse-proxy.png"
summary: "This article outlines our server-side workaround in a non-production environment that enabled access to the TEST Tomcat application without modifying any network security rules, using Apache as a reverse proxy."
---
## Background
During a test environment setup, we encountered a common network security constraint. Our AlmaLinux 8.10 test VM was part of a security group that only allowed TCP port 80 access from our corporate VPN IP. We needed to access a test Tomcat application running on port 8080, but we couldn’t modify the existing security group to open that port, as we lacked the necessary permissions.

**Disclaimer:** This solution was implemented in a non-production environment specifically for testing purposes. Always follow your organisation's security policies when working with production systems.

## The Challenge
Our test environment had the following constraints:
- AlmaLinux 8.10 server running Tomcat on port 8080.
- Security group allowing only port 80 from our corporate VPN IP.
- No ability or desire to modify the security group.
- Need to access the Tomcat application from external browsers.

When attempting to access the Tomcat application directly through port 8080, we received timeout errors because the security group was blocking this traffic. Our browser attempts to access the server resulted in **"This site can't be reached" and "ERR_CONNECTION_TIMED_OUT"** errors, making it impossible to interact with the application.

## The Solution: Apache Reverse Proxy
We decided to use Apache HTTP Server as a reverse proxy to forward requests from the allowed port 80 to Tomcat running on port 8080. Here's how we set it up:

1. Install Apache HTTP Server

```bash
sudo dnf install httpd -y
```

2. Configure Apache as a Reverse Proxy

We created a proxy configuration file:

```bash
sudo vim /etc/httpd/conf.d/proxy.conf
```

And added the following configuration:

```bash
<VirtualHost *:80>
    ServerName <add_your_server_public_IP_address>
    
    # Enable proxy modules
    ProxyRequests Off
    ProxyPreserveHost On
    
    # Proxy all requests to Tomcat
    ProxyPass / http://localhost:8080/
    ProxyPassReverse / http://localhost:8080/
    
    ErrorLog /var/log/httpd/proxy_error.log
    CustomLog /var/log/httpd/proxy_access.log combined
</VirtualHost>
```

3. Configure SELinux to Allow the Proxy

SELinux was initially blocking the proxy connections, so we needed to adjust its policy:

```bash
sudo setsebool -P httpd_can_network_connect 1
```
4. Start and Enable Apache

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

## Results: From Failure to Success
Before implementing this solution, any attempt to browse to the server via port 8080 `(http://<your_server_public_IP>:8080)` would fail with timeout errors. The browser couldn't establish a connection due to the security group restrictions.
After setting up the Apache proxy, we were able to successfully access the Tomcat application by browsing to `http:<your_server_public_IP>` (on port 80). The Apache proxy seamlessly forwarded our requests to the Test Tomcat application, and we could interact with it normally.

## Why This Works
This solution effectively bypasses the security group restriction by:
- Accepting external connections on port 80 (which is allowed by the security group).
- Internally forwarding those requests to the Tomcat application on port 8080.
- Returning responses back through Apache to the client.

The key advantage is that all network traffic from external sources only uses port 80, which complies with our security group rules, while the internal communication between Apache and Tomcat happens entirely within the server using port 8080.

## Additional Considerations
- SELinux: We initially had to set SELinux to permissive mode. For a more secure environment, you should create proper SELinux policies.
- Logging: Apache logs in `/var/log/httpd/` can help diagnose any issues with the proxy.
- Performance: For production environments, you might want to tune Apache for better performance.
- Security: Consider implementing HTTPS if this were to be used beyond testing.

## Conclusion
This approach demonstrates an effective server-side workaround for network security constraints without requiring changes to network-level security groups. By using Apache as a reverse proxy, we successfully accessed our Test Tomcat application through the allowed port 80 while maintaining compliance with the existing security group rules, transforming a previously inaccessible application into one we could use normally through our browsers.
While this setup is useful for quick testing in non-production environments, **it is not recommended for production use**.