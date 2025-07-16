---
title: "A Guide to Generating TLS Ed25519 (Elliptic Curve Cryptography) Certificates Using Private CA"
date: 2025-07-10T11:37:03+01:00
draft: false
tags: ['TLS', 'SSL', 'Encryption', 'Certificate', 'Ed25519']
categories: ['Security']
thumbnail: "images/tls-ed25519.png"
summary: "This guide demonstrates how to set up secure TLS 1.3 communication using Ed25519 elliptic curve certificates and a private Certificate Authority (CA). It covers encrypted client-server communication with modern, efficient cryptographic standards — ideal for internal systems, microservices, and zero-trust network architectures."
---
## Overview

This guide demonstrates how to set up secure TLS 1.3 communication using Ed25519 elliptic curve certificates and a private Certificate Authority (CA). It covers encrypted client-server communication with modern, efficient cryptographic standards — ideal for internal systems, microservices, and zero-trust network architectures.

### What Is Ed25519 and How Does It Compare to RSA?

Ed25519 is a modern elliptic-curve signature algorithm that offers fast operations, smaller keys, and strong security while reducing operational overhead in your infrastructure.

Below is a summary of the key aspects in a comparison table:


| **Aspect**                  | **Ed25519**                        | **RSA (2048)**                           |
| --------------------------- | ---------------------------------- | ---------------------------------------- |
| **Key Size**                | 256 bits                           | 2048 bits                                |
| **Signature Size**          | 64 bytes                           | \~256 bytes                              |
| **Security Level**          | \~128-bit (RSA-3072 equivalent)    | \~112-bit                                |
| **Key Generation**          | 10–20x faster                      | Slower                                   |
| **Signing Speed**           | 3–5x faster                        | Slower                                   |
| **Verification Speed**      | 2–3x faster                        | Slower                                   |
| **Bandwidth**               | Lower (smaller certs & signatures) | Higher                                   |
| **Implementation**          | Simple, deterministic, no padding  | Complex, padding required                |
| **Side-Channel Resistance** | Resistant                          | Susceptible if not carefully implemented |
| **Compatibility**           | Limited on legacy systems          | Broad support                            |
| **Quantum Resistance**      | Not quantum-safe                   | Not quantum-safe                         |


### Pros and Cons of Using Ed25519

| **Pros**                                         | **Cons**                                          |
| ------------------------------------------------ | ------------------------------------------------- |
| Fast key generation, signing, and verification | Limited support on legacy systems              |
| Smaller certs & signatures reduce bandwidth    | Not quantum-resistant, future migration needed |
| Strong security (RSA-3072 equivalent)          |                                                   |
| Resistant to side-channel attacks              |                                                   |
| Simple, deterministic implementation           |                                                   |
| Lower CPU and memory usage                     |                                                   |


### Environment setup

I built this environment locally on Fedora 42 using `libvirt` (Linux virtualisation API) and `Vagrant` (VM automation tool) to provision 3 x `AlmaLinux OS 9` nodes. 

- Virtualisation Platform: `libvirt`.
- Orchestration VM tool: `Vagrant`.
- VM OS: 3 x `AlmaLinux 9`.
- Network: Private isolated network (`192.168.56.x` range).

**Note**: This setup is flexible and can be replicated using any virtualisation platform in a non-production environment — including `VMware`, `VirtualBox`, `WSL`, `cloud-based VMs`. The only requirement is having three systems that can communicate over a network.

- node-03 — Certificate Authority (IP address: 192.168.56.103)
    - Generates Ed25519 root certificate and signs CSRs.

- node-02 — Server (IP address: 192.168.56.102)
   - Generates Ed25519 private key and CSR.
   - Receives signed certificate from CA.
   - Hosts HTTPS service using NGINX with TLS 1.3.

- node-01 — Client (192.168.56.101)
   - Trusts CA certificate.
   - Initiates TLS connection to the server and validates the ed25519 certificate.
   - Verifies encrypted communication.

### Architecture Flow Diagram

The diagram below illustrates the complete TLS 1.3 setup flow using Ed25519 and a private Certificate Authority (CA), showing each step across the client, server, and CA nodes.

 For each step shown in the diagram, the corresponding implementation commands are provided in the sections below.

![TLS 1.3 Setup Flow Using Ed25519 and a Private Certificate Authority](images/tls-setup-ca-ed25519.png)

## Detailed Implementation

The following implementation steps assume you are operating as the `root` user in `/root` directory.

### Step 1–2: Private Root Certificate Authority Setup (almalinux9-node-03)

Create the private Root Certificate Authority (CA) that will issue certificated for TLS.

```bash
# Switch to root shell with root environment
sudo -i

# Install OpenSSL
dnf install -y openssl

# Create CA directory structure
mkdir -p ~/ca/{certs,crl,newcerts,private}
chmod 700 ~/ca/private
cd ~/ca

# Create CA configuration file
cat > openssl.cnf << 'EOF'
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = /root/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem

default_md        = sha256
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 365
preserve          = no
policy            = policy_strict

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
x509_extensions     = v3_ca

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
EOF

# Initialize CA database
touch index.txt
echo 1000 > serial
echo 1000 > crlnumber

# Generate Ed25519 CA private key
openssl genpkey -algorithm Ed25519 -out private/ca.key.pem
chmod 400 private/ca.key.pem

# Create self-signed CA certificate
openssl req -config openssl.cnf -key private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.cert.pem -subj "/C=GB/ST=England/L=Bristol/O=MyOrg/CN=MyCA"

chmod 444 certs/ca.cert.pem
```

### Step 3–4: Server Key & CSR (almalinux9-node-02)
Generate the server's private key and certificate signing request (CSR).

```bash
# Switch to root shell with root environment
sudo -i

# Install OpenSSL and NGINX
dnf install -y openssl nginx

# Generate Ed25519 server private key
openssl genpkey -algorithm Ed25519 -out server.key.pem

# Create certificate signing request
openssl req -new -key server.key.pem -out server.csr -subj "/C=GB/ST=England/L=Bristol/O=MyOrg/CN=almalinux9-node-02"
```

### Step 5: CSR File Transfer (node-02 → node-03)

Transfer the server's CSR to the CA for signining 

```bash
# On almalinux9-node-02 (server)
# Start temporary HTTP server to share the CSR
python3 -m http.server 8000
```

### Step 6–7: Certificate Signing (almalinux9-node-03)

Download and sign the CSR with the CA private key.

```bash
# Switch to root shell with root environment
sudo -i

# On almalinux9-node-03 (CA)
cd ~/ca

# Download CSR from node-02
wget http://192.168.56.102:8000/server.csr      # 192.168.56.102 is node-02 (server) IP

# Sign the server certificate
openssl ca -config openssl.cnf -extensions server_cert -days 365 -notext -md sha256 -in server.csr -out server.cert.pem

# Start HTTP server to distribute signed certificates to other nodes
python3 -m http.server 8000
```

### Step 8–9: Certificate Distribution (node-03 → node-02)

Download the signed certificate and CA certificate to the server.

```bash
# On almalinux9-node-02

# Download signed server certificate from CA
wget http://192.168.56.103:8000/server.cert.pem     # 192.168.56.103 is node-03 (CA) IP

# Download CA certificate 
wget http://192.168.56.103:8000/certs/ca.cert.pem   # 192.168.56.103 is node-03 (CA) IP
```

### Step 10: Certificate Verification (almalinux9-node-02)

Verify the signed certificate is valid and properly configured before using it with NGINX.

```bash
# 1. Verify certificate was signed by your CA
openssl verify -CAfile ca.cert.pem server.cert.pem

# Output expected
server.cert.pem: OK
```

```bash
# 2. Check certificate details
openssl x509 -in server.cert.pem -noout -subject -issuer -dates -fingerprint

# Output expected:
subject=C=GB, ST=England, O=MyOrg, CN=almalinux9-node-02
issuer=C=GB, ST=England, L=Bristol, O=MyOrg, CN=MyCA
notBefore=Jul 14 20:00:35 2025 GMT
notAfter=Jul 14 20:00:35 2026 GMT
SHA1 Fingerprint=40:B3:BD:5D:3E:76:9C:FE:AE:08:92:5A:58:0B:53:35:CB:62:05:8E
```

``` bash
# 3. Confirm Ed25519 is being used
openssl x509 -in server.cert.pem -noout -text | grep "Public Key Algorithm"

# Output expected
Public Key Algorithm: ED25519
```

```bash
# 4. View complete certificate information (optional)
openssl x509 -in server.cert.pem -text -noout

# Output expected:
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4096 (0x1000)
        Signature Algorithm: ED25519
        Issuer: C=GB, ST=England, L=Bristol, O=MyOrg, CN=MyCA
        Validity
            Not Before: Jul 14 20:00:35 2025 GMT
            Not After : Jul 14 20:00:35 2026 GMT
        Subject: C=GB, ST=England, O=MyOrg, CN=almalinux9-node-02
        Subject Public Key Info:
            Public Key Algorithm: ED25519
                ED25519 Public-Key:
                pub:
                    9c:03:54:87:8c:4b:b2:39:ca:4c:79:b7:39:6e:40:
                    eb:aa:3b:fb:4e:a7:7b:c1:7d:f9:3b:99:29:89:4b:
                    e6:68
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            X509v3 Subject Key Identifier: 
                48:6C:A7:A9:8B:72:29:1E:90:75:3B:6C:FE:4C:DF:75:1F:0F:59:5D
            X509v3 Authority Key Identifier: 
                keyid:C4:72:73:3D:9D:CA:4A:CB:EF:77:D6:3F:D7:58:C9:B5:4A:A5:8C:DE
                DirName:/C=GB/ST=England/L=Bristol/O=MyOrg/CN=MyCA
                serial:43:D8:98:80:38:39:6E:1A:92:1E:31:CF:F8:1B:5B:E0:D0:BA:43:E7
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication
    Signature Algorithm: ED25519
    Signature Value:
        fb:4d:f9:d3:68:bb:ab:34:d0:c1:6c:e8:9f:fe:4a:57:57:64:
        a4:2c:90:60:04:0b:c5:10:e8:2c:a6:99:eb:4a:25:c8:ba:39:
        54:7c:60:bd:e4:0f:d3:f1:e6:99:c4:06:9b:8e:01:bf:f1:05:
        14:62:0f:3d:cb:3b:6f:35:90:08
```
### Certificate Validation Checklist
The outputs above confirm that our Ed25519 certificate setup is fully operational and secure:

- Certificate Verification: server.cert.pem: OK - Valid CA signature
- Identity Match: Subject CN=almalinux9-node-02 matches server hostname
- Trusted Issuer: Certificate signed by our private CA CN=MyCA
- Ed25519 Confirmed: Modern elliptic curve cryptography active
- TLS Ready: Proper extensions for HTTPS server authentication

**Result**: Server Certificate successfully validated - ready for NGINX TLS 1.3 deployment.

### Step 11–12: NGINX Setup (almalinux9-node-02)

Configure NGINX with TLS 1.3 with the Ed25519 certificate.

```bash
# Install certificates
mkdir -p /etc/nginx/ssl
cp server.cert.pem /etc/nginx/ssl/
cp server.key.pem /etc/nginx/ssl/
cp ca.cert.pem /etc/nginx/ssl/
chmod 600 /etc/nginx/ssl/server.key.pem

# Configure NGINX for TLS 1.3
tee /etc/nginx/conf.d/https-server.conf << 'EOF'
# HTTP Server (unencrypted) - for testing
server {
    listen 80;
    server_name almalinux9-node-02;

    location / {
        return 200 "UNENCRYPTED HTTP Server - this data is visible!\n";
        add_header Content-Type text/plain;
    }

    location /secret {
        return 200 "SECRET DATA: password123 - credit card: 1234-5678-9012-3456\n";
        add_header Content-Type text/plain;
    }
}

# HTTPS Server (encrypted) - TLS 1.3
server {
    listen 443 ssl http2;
    server_name almalinux9-node-02;

    ssl_certificate /etc/nginx/ssl/server.cert.pem;
    ssl_certificate_key /etc/nginx/ssl/server.key.pem;

    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;

    location / {
        return 200 "HTTPS Server - TLS 1.3 + Ed25519 Encrypted!\n";
        add_header Content-Type text/plain;
    }

    location /secret {
        return 200 "ENCRYPTED SECRET DATA: admin_password=SuperSecret789 - credit_card=9876-5432-1098-7654 - ssn=123-45-6789 - bank_account=987654321\n";
        add_header Content-Type text/plain;
    }
}
EOF

# Start NGINX
systemctl start nginx
systemctl enable nginx

# Check NGINX status
systemctl status nginz
```

### Step 13: Client Setup (almalinux9-node-01)

Configure the client to trust the CA and establish secure TLS.

```bash
# Install client tools
dnf install -y openssl curl

# Create certificate directory
mkdir -p ~/certs

# Download CA certificate
wget http://192.168.56.103:8000/certs/ca.cert.pem -O ~/certs/ca.cert.pem
```

### Steps 14–18: Secure Communication Phase

Test secure HTTPS access, inspect handshake, and verify traffic encryption.

```bash
# 13. Test basic HTTPS connection first (establish that TLS works)
curl --cacert ~/certs/ca.cert.pem https://almalinux9-node-02/secret

# 14. Verify TLS handshake details
curl -v --cacert ~/certs/ca.cert.pem https://almalinux9-node-02/

# 15. Encrypted Packet Capture (HTTPS)
echo "=== Capturing HTTPS Traffic (Encrypted) ==="
tcpdump -i any -A -c 10 host almalinux9-node-02 and port 443 &
sleep 2  # Give tcpdump time to start
curl --cacert ~/certs/ca.cert.pem https://almalinux9-node-02/secret

# 16. Plaintext Packet Capture (HTTP)
echo "=== Capturing HTTP Traffic (Unencrypted) ==="
tcpdump -i any -A -c 10 host almalinux9-node-02 and port 80 &
sleep 2  # Give tcpdump time to start
curl http://almalinux9-node-02/secret
```

## Expected Results and Evidence
### Test 1: Basic HTTPS Connection

```bash
curl --cacert ~/certs/ca.cert.pem https://almalinux9-node-02/secret
```

### Expected Output:

```bash
ENCRYPTED SECRET DATA: admin_password=SuperSecret789 - credit_card=9876-5432-1098-7654 - ssn=123-45-6789 - bank_account=987654321
```
Client successfully receives decrypted data over secure TLS connection.

### Test 2: TLS Handshake Verification

```bash
curl -v --cacert ~/certs/ca.cert.pem https://almalinux9-node-02/
```

### Key Output Indicators:

```bash
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* Server certificate:
*  subject: C=GB; ST=England; O=MyOrg; CN=almalinux9-node-02
*  issuer: C=GB; ST=England; L=Bristol; O=MyOrg; CN=MyCA
*  common name: almalinux9-node-02 (matched)
*  SSL certificate verify ok.
```

TLS 1.3 active, Ed25519 certificate verified, hostname matched.

### Test 3: HTTPS Network Traffic Analysis
### Network Capture (tcpdump):

```bash
E..<..@.@.....8e..8f.h..Lv.o.........J.........
..q.........
22:38:52.489995 eth1  In  IP almalinux9-node-02.https > almalinux9-node-01.34408: Flags [S.], seq 133835979, ack 1282837360, win 65160, options [mss 1460,sackOK,TS val 1778032372 ecr 2642506154,nop,wscale 7], length 0
E..<..@.@.H...8f..8e...h..,.Lv.p.....J.........
i.....q.....
22:38:52.490029 eth1  Out IP almalinux9-node-01.34408 > almalinux9-node-02.https: Flags [.], ack 1, win 502, options [nop,nop,TS val 2642506155 ecr 1778032372], length 0
E..4..@.@.....8e..8f.h..Lv.p..,......B.....
..q.i...
22:38:52.493578 eth1  Out IP almalinux9-node-01.34408 > almalinux9-node-02.https: Flags [P.], seq 1:518, ack 1, win 502, options [nop,nop,TS val 2642506159 ecr 1778032372], length 517
E..9..@.@.....8e..8f.h..Lv.p..,......G.....
..q.i..............X-.:7...akx...PI[".w..0..&.C..#. ......p....A.c.o....Y...*........H.........,.0.......+./...#.'.
...	...........=.<.5./...........k.g.9.3.....k.........almalinux9-node-02.........
........................3t.........h2.http/1.1.........1.....". ...........	.
...................+........-.....3.&.$... o.."!......;.......T..H.|.....d6.............................................................................................................................................................................
22:38:52.494024 eth1  In  IP almalinux9-node-02.https > almalinux9-node-01.34408: Flags [.], ack 518, win 506, options [nop,nop,TS val 1778032376 ecr 2642506159], length 0
E..4.|@.@..+..8f..8e...h..,.Lv.u.....B.....
i.....q.
22:38:52.494745 eth1  In  IP almalinux9-node-02.https > almalinux9-node-01.34408: Flags [P.], seq 1:1015, ack 518, win 506, options [nop,nop,TS val 1778032377 ecr 2642506159], length 1014
E..*.}@.@..4..8f..8e...h..,.Lv.u.....8.....
i.....q.....z...v..../.{.z...&..t}....g.p.PJ.i.).t. ......p....A.c.o....Y...*.............+.....3.$... N${......s....k....O...A.*...i,g..........$a7.......4\u....h.Z..=...WrY/ar...v..........-H6.!tc.0.../.8.ca^.f.|...r....f|..Rm.....sE&n.4Yz3.:..y...1.[x|.....L'....Y..=...0....q.YiM.-b.3..z.....}..E+,.{..........Hdj.u.J.B...4M3bj.h........eisf..rS.R.G.K..7.!.K.VD._.otr.Y}..%..m#+3..).E......5U...M.r.9..3so....X
..S	..'.X,q.........*_.&................Et..D...V........}.eK...~.Gj..I_..0ue.e..y.....B.V^S....}b6%.g3.P..s...e..w......,$.e.$.>oU....b..l.!;p.=.+...5....9bW2H1.$..........|...J;.)\.v!h`.C.sY.....
WCGh.Xj..].....`...e....W.....(:E'......;.3.....@..B....K-....y&3.B.-.H...d.w6Z.YX[4..m....Xx.a.Xbx..VLQ1?Uw...._.....X....k^..
..+.......1;Q..6.]N.1..1...SG..Zz...vM.2.P..@...}..u..,.....E..;.bQ...Q.............G.-..v...$...&.'.m.].n.Eb.YA1....Ye..ij....D..w.	ez$.....K........3.x....bG...I.c_...*..$...U5..-.Sz!.36.......".......g!......E...l...........-..X<z.W...kgw..............3....g.....C.|O.K+.......A
22:38:52.494777 eth1  Out IP almalinux9-node-01.34408 > almalinux9-node-02.https: Flags [.], ack 1015, win 524, options [nop,nop,TS val 2642506160 ecr 1778032377], length 0
E..4..@.@.....8e..8f.h..Lv.u..0......B.....
..q.i...
22:38:52.496125 eth1  Out IP almalinux9-node-01.34408 > almalinux9-node-02.https: Flags [P.], seq 518:598, ack 1015, win 524, options [nop,nop,TS val 2642506161 ecr 1778032377], length 80
E.....@.@.....8e..8f.h..Lv.u..0............
..q.i.............E.KR...>xL...R....K..C.#.......).)..x....?rS.a..b..{.S....T*.t....{..<
22:38:52.496230 eth1  Out IP almalinux9-node-01.34408 > almalinux9-node-02.https: Flags [P.], seq 598:644, ack 1015, win 524, options [nop,nop,TS val 2642506161 ecr 1778032377], length 46
E..b..@.@.....8e..8f.h..Lv....0......p.....
..q.i.......)9.C.P...Rb.~.....l.	=....np.<.p'.J!|.... 
22:38:52.496242 eth1  Out IP almalinux9-node-01.34408 > almalinux9-node-02.https: Flags [P.], seq 644:693, ack 1015, win 524, options [nop,nop,TS val 2642506161 ecr 1778032377], length 49
E..e..@.@.....8e..8f.h..Lv....0......s.....
..q.i.......,.,......;.hT.j..bPu.E.|@.....<.=gU2?=....z.	
10 packets captured
27 packets received by filter
0 packets dropped by kernel
```
Evidence: Network traffic shows encrypted binary data - sensitive information is completely protected.

### Test 4: HTTP Traffic Analysis (Comparison)
### Network Capture (tcpdump):
```bash
GET /secret HTTP/1.1
Host: almalinux9-node-02
User-Agent: curl/7.76.1

HTTP/1.1 200 OK
Server: nginx/1.20.1
Content-Type: text/plain
SECRET DATA: password123 - credit card: 1234-5678-9012-3456
```
 Evidence: Network traffic exposes all sensitive data in plain text - completely insecure.

## Security Comparison Results

 | Protocol (Client)         | Experience           | Network Security     | Attacker Visibility         |
|----------------------------|-----------------------|------------------------|-----------------------------|
| HTTPS (TLS 1.3 + Ed25519) | Normal data access | Fully encrypted     | Encrypted gibberish only |
| HTTP (Unencrypted)        | Normal data access | No protection       | All secrets visible      |

## Conclusion

This implementation demonstrates secure TLS 1.3 communication using Ed25519 elliptic curve cryptography and a private Certificate Authority (CA). 

The key outcomes are summarised below:

- **Ed25519**: Strong cryptographic security with fast key generation and smaller signatures.
- **TLS 1.3**: Enforces modern encryption with forward secrecy and reduced handshake overhead.
- **Private CA**: Enables full control over certificate issuance and trust boundaries.
- **Verified Encryption**: Network traffic analysis confirms all sensitive data remains encrypted in transit.

Compared to RSA-based setups, Ed25519 simplifies key handling while improving performance and side-channel resistance—making it well-suited for secure internal systems and service-to-service communication.