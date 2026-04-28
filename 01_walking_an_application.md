# 🚶 Walking An Application (Ilovani Aylanib Chiqish)

> **Maqsad:** Hech qanday maxsus vosita ishlatmasdan, faqat brauzer yordamida veb-ilovaning zaif tomonlarini qo'lda topish.

---

## 📖 Asosiy Tushunchalar

### View Source (Manba Kodini Ko'rish) nima?
Brauzer sizga sahifaning HTML kodini ko'rish imkonini beradi. Bu yerda dasturchilar ba'zan:
- Maxfiy izohlar (`<!-- -->` ichida)
- Parollar yoki API kalitlari
- Yashirin formalar
- Eski yoki test sahifalariga havolalar qoldirishi mumkin.

**Qanday ochiladi:** `CTRL+U` yoki sichqonchaning o'ng tugmasi → "View Page Source"

---

### Developer Tools (Dasturchi Vositalari) nima?
Brauzerga o'rnatilgan kuchli vosita. Pentester uchun eng muhim bo'limlari:

| Bo'lim | Nima uchun kerak |
|--------|-----------------|
| **Inspector** | HTML/CSS ni real vaqtda ko'rish va tahrirlash |
| **Network** | Barcha HTTP so'rovlarini kuzatish |
| **Console** | JavaScript xatolarini ko'rish, JS kodni ishlatish |
| **Storage** | Cookie, localStorage, sessionStorage ni ko'rish |
| **Debugger** | JavaScript fayllarini ko'rish (yashirin endpoint lar uchun) |

**Qanday ochiladi:** `F12` yoki `CTRL+SHIFT+I`

---

### Robots.txt nima?
Sayt egalari qaysi sahifalarni qidiruv tizimlari (Google va boshqalar) indekslamasligini shu faylda yozadi. Lekin bu fayl hammaga ochiq! Pentester uchun bu **yashirin sahifalar ro'yxati** bo'lishi mumkin.

**Manzil:** `https://target.com/robots.txt`

**Namuna:**
```
User-agent: *
Disallow: /admin/
Disallow: /backup/
Disallow: /secret-panel/
```
Yuqoridagi misolda `/admin/`, `/backup/`, `/secret-panel/` — tekshirishga arziydi!

---

### Sitemap.xml nima?
Saytdagi barcha sahifalar ro'yxati. Qidiruv tizimlariga mo'ljallangan, lekin pentester uchun ham foydali.

**Manzil:** `https://target.com/sitemap.xml`

---

### Favicons (Sayt Belgisi) nima?
Brauzer tabida ko'rinadigan kichik ikonka. Ba'zan sayt freymwork (framework — dastur skeleti) ishlab chiquvchisi standart ikonkani almashtirmasligi mumkin. Bu orqali qaysi freymwork ishlatilgani aniqlanishi mumkin.

**OWASP Favicon Database:** MD5 hash orqali freymworkni aniqlash mumkin.

```bash
# Favicon hash aniqlash
curl https://target.com/favicon.ico | md5sum
```

---

## 🛠️ Cheatsheet (Tezkor Yo'riqnoma)

```
# Tekshiriladigan standart manzillar
https://target.com/robots.txt
https://target.com/sitemap.xml
https://target.com/favicon.ico
https://target.com/.well-known/
https://target.com/humans.txt

# Developer Tools yorliqlari
F12                  → Developer Tools ochish
CTRL+U               → Sahifa manba kodini ko'rish
CTRL+SHIFT+I         → Inspector ochish
CTRL+SHIFT+J         → Console ochish

# Network tabida nima izlash kerak
- API endpoint lar (masalan /api/v1/users)
- Token yoki cookie lar
- Yashirin parametrlar
- JavaScript fayllarida maxfiy ma'lumotlar
```

---

## 🎯 Amaliy Misol: Qadam-ba-qadam

**Vaziyat:** `http://target.com` saytini tekshiryapmiz.

### 1-qadam: Manba kodini ko'rish
```
CTRL+U bosing → HTML kodini izlab chiqing:
<!-- Admin panel: /secret-admin-page -->  ← TOPILDI!
```

### 2-qadam: Robots.txt tekshirish
```
http://target.com/robots.txt ni oching:

Disallow: /staff-portal/   ← Bu sahifani tekshir!
Disallow: /old-login/      ← Bu ham qiziq!
```

### 3-qadam: Developer Tools → Network
```
1. F12 bosing
2. Network tabini oching
3. Sahifani yangilang (F5)
4. Barcha so'rovlarni ko'ring:
   - /api/getUserData  ← API endpoint topildi!
   - Authorization: Bearer eyJ...  ← Token ko'rindi!
```

### 4-qadam: JS fayllarini ko'rish (Debugger tab)
```
1. F12 → Debugger (yoki Sources) tab
2. JS fayllarini oching
3. Qidiring: password, secret, api_key, token, admin
```

### 5-qadam: Storage tekshirish
```
F12 → Application/Storage tab:
- Cookies: session ID, auth token
- localStorage: foydalanuvchi ma'lumotlari
- sessionStorage: vaqtinchalik ma'lumotlar
```

---

## 💡 Eslab Qolish Uchun

- **View Source** = Statik HTML kodni ko'rish
- **Inspector** = Jonli, o'zgaruvchan HTML ni ko'rish (JS tomonidan o'zgartirilganlar ham)
- **robots.txt** = "Bormang" deyilgan joylar → aynan o'sha joylarni tekshir!
- **Network tab** = Sahifa nimalar bilan "gaplashmoqda" — API va so'rovlarni kuzat
- **Favicon hash** = Freymworkni aniqlash uchun

---

## ⚠️ Muhim Eslatma

Bu texnikalar faqat **ruxsat berilgan** tizimlarni test qilishda ishlatilishi kerak. TryHackMe laboratoriyalari bu uchun maxsus yaratilgan xavfsiz muhitdir.
