# TryHackMe: Take Over - Writeup & Documentation

![Take Over Banner](Asset%20Image/Pasted%20image%2020260807221507.png)

## 📌 Overview
Dokumentasi walkthrough dan solusi untuk tantangan **Take Over** dari platform TryHackMe.

* **Difficulty:** Easy
* **Category:** Web Security / Subdomain Enumeration / Virtual Host Fuzzing
* **Target IP:** `10.48.137.215`

---

## 🛠️ Step-by-Step Walkthrough

### Step 1: Host Resolution Setup
Menambahkan entri IP target ke `/etc/hosts` lokal:
```bash
sudo nano /etc/hosts
# Tambahkan entri:
10.48.137.215 futurevera.thm
```

### Step 2: Port Scanning & Reconnaissance
Melakukan pemindaian port menggunakan Nmap:
```bash
sudo nmap -sS 10.48.137.215
```
**Hasil Pemindaian:**
* `22/tcp` - SSH
* `80/tcp` - HTTP
* `443/tcp` - HTTPS

### Step 3: Initial HTTP Header Inspection
Inspeksi header HTTP/HTTPS target:
```bash
curl -I -k -H "Host: randomname.futurevera.thm" https://10.48.137.215
```

### Step 4: Subdomain / Virtual Host Enumeration
Menggunakan `ffuf` untuk fuzzing subdomain pada Virtual Host (`FUZZ.futurevera.thm`):
```bash
ffuf -H "Host: FUZZ.futurevera.thm" -u https://10.48.137.215 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 0,4605
```
**Subdomain Teridentifikasi:**
* `support.futurevera.thm`
* `blog.futurevera.thm`

### Step 5: Updating Local DNS Resolution
Menambahkan subdomain hasil temuan ke `/etc/hosts`:
```bash
echo "10.48.137.215 blog.futurevera.thm" | sudo tee -a /etc/hosts
echo "10.48.137.215 support.futurevera.thm" | sudo tee -a /etc/hosts
```

### Step 6: Blog Analysis & Verification
Membuka dan mengeksplorasi layanan pada `blog.futurevera.thm`:

![Blog Verification](Asset%20Image/Pasted%20image%2020260807232834.png)

---

## 🔒 Key Takeaways & Security Insights
1. **Virtual Host Fuzzing**: Sangat penting dalam pengujian penetrasi web ketika IP menampung banyak sub-domain/vhost tersembunyi.
2. **HTTPS & SNI**: Selalu periksa sertifikat SSL/TLS atau Subdomain tak terdaftar untuk memperluas *attack surface*.
