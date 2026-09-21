# TryHackMe: Take Over - Penetration Testing Walkthrough

![Take Over Banner](Asset%20Image/Pasted%20image%2020260807221507.png)

---

## 📋 Machine Overview

- **Room Name:** Take Over
- **Platform:** [TryHackMe](https://tryhackme.com/room/takeover)
- **Target OS:** Linux (Ubuntu)
- **Difficulty:** Easy
- **Target IP:** `IP-TARGET`
- **Primary Domain:** `futurevera.thm`
- **Discovered Subdomains:** `blog.futurevera.thm`, `support.futurevera.thm`, `secrethelpdesk934752.support.futurevera.thm`
- **Attack Vectors:** Virtual Host (VHost) Fuzzing, TLS/SSL Certificate Inspection (Subject Alternative Name - SAN Disclosure), Subdomain Enumeration

---

## 🎯 Executive Summary

The **Take Over** challenge on TryHackMe tests essential web reconnaissance, virtual host enumeration, and TLS/SSL certificate metadata analysis skills.

1. **Reconnaissance & Port Scanning:** An initial TCP SYN stealth scan revealed three active listening ports: OpenSSH (port 22), Apache HTTP (port 80), and Apache HTTPS (port 443).
2. **Virtual Host (VHost) Fuzzing:** Using `ffuf` with SecLists wordlists to fuzz the HTTP `Host` header, two virtual hosts were identified: `blog.futurevera.thm` and `support.futurevera.thm`.
3. **Information Disclosure via TLS/SSL Certificate Inspection:** While inspecting the services hosted on `support.futurevera.thm`, an in-depth analysis of the X.509 TLS certificate using `openssl s_client` revealed an unpublicized internal subdomain embedded within the **Subject Alternative Name (SAN)** field: `secrethelpdesk934752.support.futurevera.thm`.
4. **Flag Retrieval:** Appending the hidden subdomain to `/etc/hosts` and querying it over HTTPS using `curl -v` revealed the hidden web interface containing the flag.

---

## 📝 Room Questions & Answers

| Question | Answer |
| :--- | :--- |
| **What is the flag?** | `flag{beea0d6edfcee06a59b83fb50ae81b2f}` |

---

## 🔍 Phase 1: Reconnaissance & Environment Setup

### 1. Local DNS Resolution Setup

Because the target application relies on name-based virtual hosting on an internal domain (`futurevera.thm`), we must configure local host resolution by adding an entry to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add the following entry:
```text
IP-TARGET futurevera.thm
```

Alternatively, append using `tee`:
```bash
echo "IP-TARGET futurevera.thm" | sudo tee -a /etc/hosts
```

---

### 2. Network Port Scanning

We perform a TCP SYN stealth scan (`-sS`) against the target host to determine all open ports and running network daemons:

```bash
sudo nmap -sS IP-TARGET
```

**Scan Output:**
```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-07 23:12 +0700
Nmap scan report for futurevera.thm (IP-TARGET)
Host is up (0.081s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

**Port Analysis:**
- **Port 22 (SSH):** Secure Shell service (OpenSSH).
- **Port 80 (HTTP):** Standard unencrypted HTTP web server.
- **Port 443 (HTTPS):** Encrypted HTTPS web server utilizing TLS certificates.

---

### 3. Baseline HTTP Header & Response Analysis

Before executing virtual host fuzzing, we probe the HTTPS endpoint with a dummy `Host` header to determine baseline response behavior and filter out generic catch-all responses:

```bash
curl -I -k -H "Host: randomname.futurevera.thm" https://IP-TARGET
```

**Response Output:**
```http
HTTP/1.1 200 OK
Date: Fri, 07 Aug 2026 16:13:16 GMT
Server: Apache/2.4.41 (Ubuntu)
Last-Modified: Sun, 13 Mar 2022 08:48:57 GMT
ETag: "11fd-5da15a2613040"
Accept-Ranges: bytes
Content-Length: 4605
Vary: Accept-Encoding
Content-Type: text/html
```

**Key Observation:** The default web server responds with `HTTP 200 OK` and a `Content-Length` of `4605` bytes for invalid or non-existent virtual hosts. This baseline size (`-fs 4605`) will be filtered out to eliminate false positives.

---

## 🔎 Phase 2: Virtual Host (VHost) Fuzzing

### 1. Subdomain Discovery via `ffuf`

Using `ffuf` (Fuzz Faster U Fool), we fuzz the HTTP `Host` header against the HTTPS target using the `subdomains-top1million-110000.txt` wordlist from SecLists:

```bash
ffuf -H "Host: FUZZ.futurevera.thm" -u https://IP-TARGET -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 0,4605
```

**Flag Breakdown:**
- `-H "Host: FUZZ.futurevera.thm"`: Injects fuzzing payloads into the HTTP `Host` header for name-based virtual host enumeration.
- `-u https://IP-TARGET`: Target URL specifying the HTTPS scheme.
- `-w /usr/share/seclists/...`: Wordlist path containing over 110,000 common subdomain strings.
- `-fs 0,4605`: Filters out responses with content sizes of `0` or `4605` bytes (eliminating empty and default catch-all responses).

**Fuzzing Results:**
```text
        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://IP-TARGET
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.futurevera.thm
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 0,4605
________________________________________________

support                 [Status: 200, Size: 1522, Words: 367, Lines: 34, Duration: 82ms]
blog                    [Status: 200, Size: 3838, Words: 1326, Lines: 81, Duration: 90ms]
:: Progress: [114442/114442] :: Job [1/1] :: 455 req/sec :: Duration: [0:04:45] :: Errors: 0 ::
```

Two valid virtual hosts were discovered:
1. `support.futurevera.thm` (Response Size: 1522)
2. `blog.futurevera.thm` (Response Size: 3838)

---

### 2. Updating Local Hosts Configuration

We add both discovered virtual hosts to `/etc/hosts` to enable direct domain resolution:

```bash
echo "IP-TARGET blog.futurevera.thm" | sudo tee -a /etc/hosts
echo "IP-TARGET support.futurevera.thm" | sudo tee -a /etc/hosts
```

---

## 🌐 Phase 3: Service Analysis & Certificate Inspection

### 1. Inspecting Discovered Services

Navigating to `https://blog.futurevera.thm`:

![Blog Verification](Asset%20Image/Pasted%20image%2020260807232834.png)

Next, inspecting `https://support.futurevera.thm`:

![Support Page](Asset%20Image/Screenshot%202026-09-21%20232517.png)

The support interface indicates generic support instructions, hinting that an internal or restricted helpdesk portal exists.

---

### 2. SSL/TLS Certificate Analysis & SAN Extraction

Digital certificates frequently expose internal infrastructure through the **Subject Alternative Name (SAN)** X.509 extension, which lists all fully qualified domain names (FQDNs) secured by the certificate.

We query and inspect the SSL/TLS certificate on `support.futurevera.thm` using `openssl s_client`:

```bash
openssl s_client -connect support.futurevera.thm:443 -servername support.futurevera.thm < /dev/null | openssl x509 -text -noout | grep -A 1 "Subject Alternative Name"
```

**Command Breakdown:**
- `openssl s_client -connect support.futurevera.thm:443`: Establishes a TLS session on port 443.
- `-servername support.futurevera.thm`: Passes the Server Name Indication (SNI) header to retrieve the matching virtual host certificate.
- `< /dev/null`: Pipes EOF into stdin to automatically terminate the interactive connection after handshake negotiation.
- `openssl x509 -text -noout`: Decodes the raw X.509 certificate into structured plain text.
- `grep -A 1 "Subject Alternative Name"`: Filters the output to display the SAN field and the subsequent line containing all listed domains.

**Execution Output:**

![SSL SAN Certificate Inspection](Asset%20Image/Screenshot%202026-09-21%20232817.png)

The certificate inspection exposes a hidden sub-subdomain:
```text
X509v3 Subject Alternative Name: 
    DNS:secrethelpdesk934752.support.futurevera.thm
```

---

## 🚩 Phase 4: Exploitation & Flag Retrieval

### 1. Mapping the Secret Subdomain

We append the discovered internal helpdesk subdomain to `/etc/hosts`:

```bash
echo "IP-TARGET secrethelpdesk934752.support.futurevera.thm" | sudo tee -a /etc/hosts
```

---

### 2. Querying the Secret Endpoint

We send a verbose HTTPS request to the secret helpdesk domain using `curl`:

```bash
curl -v https://secrethelpdesk934752.support.futurevera.thm
```

**Output & Verification:**

![Flag Captured](Asset%20Image/Screenshot%202026-09-21%20233054.png)

The HTTP response contains the challenge flag:

🎉 **Final Flag:**
```text
flag{beea0d6edfcee06a59b83fb50ae81b2f}
```

---

## 🛡️ Remediation & Hardening Recommendations

1. **Segregate Public and Internal Certificates:**
   - Never include sensitive, non-public, or internal administrative subdomains in public-facing SSL/TLS certificates (SAN fields).
   - Issue separate, dedicated certificates for public applications and internal portals, or utilize wildcard certificates properly to avoid exposing specific hostnames.

2. **Network-Level Access Control for Sensitive Portals:**
   - Internal administrative services (e.g., helpdesk systems, staging environments) should be restricted to private subnets, administrative VPNs, or protected by strong authentication proxies.
   - Do not bind private administrative virtual hosts on the same publicly accessible web server interfaces.

3. **Harden Virtual Host Configurations:**
   - Implement strict default catch-all virtual hosts in Apache/Nginx that return `404 Not Found` or `400 Bad Request` when accessed with unrecognized or unexpected `Host` headers.
   - Disable unnecessary directory indexing and ensure error pages do not disclose server versions or internal hostnames.