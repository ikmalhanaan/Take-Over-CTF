![Take Over Banner](Asset%20Image/Pasted%20image%2020260807221507.png)

Step 1:
```
┌──(hanzcodedrop㉿kali)-[~/Documents/TryHackMe_CTF/Take_Over_Easy]
└─$ sudo nano /etc/host
```

masukkan ip dan domainnya ke dalam
```
10.48.137.215 futurevera.thm
```

step 2: 
```
┌──(hanzcodedrop㉿kali)-[~/Documents/TryHackMe_CTF/Take_Over_Easy]
└─$ nmap -sS 10.48.137.215        
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-07 23:12 +0700
Nmap scan report for futurevera.thm (10.48.137.215)
Host is up (0.081s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

step 3:
```
┌──(hanzcodedrop㉿kali)-[~/Documents/TryHackMe_CTF/Take_Over_Easy]
└─$ curl -I -k -H "Host: randomname.futurevera.thm" https://10.48.137.215
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

Step 4:
```
┌──(hanzcodedrop㉿kali)-[~/Documents/TryHackMe_CTF/Take_Over_Easy]
└─$ ffuf -H "Host: FUZZ.futurevera.thm" -u https://10.48.137.215 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs 0,4605


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://10.48.137.215
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

step 5:
```
┌──(hanzcodedrop㉿kali)-[~/Documents/TryHackMe_CTF/Take_Over_Easy]
└─$ echo "10.48.137.215 blog.futurevera.thm" | sudo tee -a /etc/hosts
echo "10.48.137.215 support.futurevera.thm" | sudo tee -a /etc/hosts

```

Step 6: buka blognya
![Blog Verification](Asset%20Image/Pasted%20image%2020260807232834.png)