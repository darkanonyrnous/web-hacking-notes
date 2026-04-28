# 🔍 Content Discovery (Kontent Kashfiyoti)

> **Maqsad:** Veb-saytda ko'rinmaydigan yashirin sahifalar, papkalar, fayllar va resurslarni topish.

---

## 📖 Asosiy Tushunchalar

### Content Discovery nima?
Sayt egasi ko'rsatmagan, lekin serverda mavjud bo'lgan sahifa va fayllarni topish jarayoni.

**3 xil usul bor:**
1. **Qo'lda (Manual)** — robots.txt, sitemap.xml, manba kodi
2. **OSINT** — ochiq ma'lumot bazalaridan foydalanish
3. **Avtomatik (Automated)** — maxsus vositalar bilan

---

## 🔑 Wordlist (So'zlar Ro'yxati) nima?
Keng tarqalgan papka va fayl nomlari yozilgan oddiy matn fayli. Vosita har bir so'zni serverga yuborib, qaysilari mavjudligini tekshiradi.

**Mashhur wordlist lar:**
- `/usr/share/wordlists/dirb/common.txt` — keng tarqalgan
- `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` — katta ro'yxat
- `big.txt`, `small.txt` — hajmiga qarab

---

## 🛠️ Asosiy Vositalar

### 1. Gobuster
Go tilida yozilgan, tez ishlaydigan directory/fayl topish vositasi.

```bash
# Asosiy ishlatish
gobuster dir -u http://target.com -w /usr/share/wordlists/dirb/common.txt

# Kengaytirilgan
gobuster dir \
  -u http://target.com \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -x php,html,txt \    # Fayl kengaytmalarini ham qidirish
  -t 50 \              # 50 ta parallel so'rov (tezlik)
  -o results.txt       # Natijani faylga saqlash

# Status kodlarini filtrlash
gobuster dir -u http://target.com -w wordlist.txt -s 200,301,302
```

### 2. Ffuf (Fuzz Faster U Fool)
Juda tez va moslashuvchan fuzzing vositasi.

```bash
# Directory qidirish
ffuf -u http://target.com/FUZZ -w /usr/share/wordlists/dirb/common.txt

# Fayl kengaytmasi bilan
ffuf -u http://target.com/FUZZ -w wordlist.txt -e .php,.html,.txt,.bak

# Natijalarni filtrlash (404 larni yashirish)
ffuf -u http://target.com/FUZZ -w wordlist.txt -fc 404

# Hajmiga qarab filtrlash
ffuf -u http://target.com/FUZZ -w wordlist.txt -fs 0
```

### 3. Dirb
Klass vosita, sodda va ishonchli.

```bash
dirb http://target.com /usr/share/wordlists/dirb/common.txt
```

---

## 📡 HTTP Status Kodlari (Javob Kodlari)

| Kod | Ma'nosi | Pentester uchun |
|-----|---------|-----------------|
| **200** | OK — sahifa mavjud | ✅ Topildi! |
| **301/302** | Redirect (yo'naltirish) | ✅ Tekshir! |
| **401** | Unauthorized (Ruxsat yo'q) | ✅ Login kerak — zaif bo'lishi mumkin |
| **403** | Forbidden (Taqiqlangan) | ⚠️ Mavjud lekin kirish yo'q — usul qidir |
| **404** | Not Found (Topilmadi) | ❌ Yo'q |
| **500** | Server Error | ⚠️ Server xatosi — qiziqarli! |

---

## 🌐 OSINT Usullari

### Google Dorks (Google orqali yashirin ma'lumot izlash)
```
site:target.com                    → faqat shu saytdagi sahifalar
site:target.com filetype:pdf       → PDF fayllar
site:target.com inurl:admin        → URL da "admin" so'zi bor sahifalar
site:target.com intitle:login      → Sarlavhada "login" bor sahifalar
cache:target.com                   → Google keshi
```

### Wayback Machine (web.archive.org)
Saytning eski versiyalarini ko'rish. Eski, o'chirilgan sahifalar ba'zan hali ham serverda qoladi.

```
https://web.archive.org/web/*/target.com/*
```

### GitHub qidirish
Dasturchilar ba'zan maxfiy ma'lumotlarni (API kalit, parol) GitHub ga yuklaydi.
```
site:github.com "target.com" password
site:github.com "target.com" api_key
```

---

## 📂 Tekshirilishi Kerak Bo'lgan Standart Joylar

```
/admin/             → Boshqaruv paneli
/administrator/     → Boshqaruv paneli (WordPress va boshqalar)
/wp-admin/          → WordPress admin
/login/             → Kirish sahifasi
/dashboard/         → Ko'rinish paneli
/backup/            → Zaxira nusxalar
/config/            → Konfiguratsiya fayllari
/api/               → API endpoint lar
/api/v1/            → API versiyasi
/.env               → Muhit o'zgaruvchilari (parollar!)
/config.php         → PHP konfiguratsiya
/web.config         → IIS konfiguratsiya
/.git/              → Git repository (manba kodi!)
/phpinfo.php        → PHP ma'lumotlari
/server-status      → Apache holati
```

---

## 🎯 Amaliy Misol: To'liq Skan

```bash
# 1-qadam: Tez va kichik wordlist bilan boshlash
gobuster dir -u http://10.10.x.x -w /usr/share/wordlists/dirb/common.txt -t 40

# Natija:
# /admin   (Status: 200)
# /login   (Status: 302)
# /backup  (Status: 403)

# 2-qadam: Topilgan joylarni chuqurroq tekshirish
gobuster dir -u http://10.10.x.x/admin -w /usr/share/wordlists/dirb/common.txt

# 3-qadam: Fayl kengaytmalari bilan skan
gobuster dir -u http://10.10.x.x -w wordlist.txt -x php,txt,html,bak,zip

# 4-qadam: .env va konfiguratsiya fayllarini tekshirish
curl http://10.10.x.x/.env
curl http://10.10.x.x/config.php.bak
```

---

## 💡 Eslab Qolish Uchun

- **200** = Topildi ✅ | **403** = Mavjud lekin taqiqlangan ⚠️ | **404** = Yo'q ❌
- Kichik wordlist → keyin katta wordlist (vaqt tejash)
- `-x` bilan kengaytmalar qo'sh: `.php`, `.txt`, `.bak`, `.zip`, `.old`
- **403** ko'rsansa, boshqa usul qidirish kerak (403 bypass)
- Google Dorks + Wayback Machine = OSINT ning asosi
