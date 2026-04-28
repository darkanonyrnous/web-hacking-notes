# 🌐 Subdomain Enumeration (Subdomenlarni Sanash)

> **Maqsad:** Asosiy domenga tegishli barcha subdomenlarni topish. Yashirin subdomenlar ko'pincha kam himoyalangan bo'ladi.

---

## 📖 Asosiy Tushunchalar

### Subdomen nima?
`target.com` asosiy domen bo'lsa:
- `admin.target.com` → subdomen
- `mail.target.com` → subdomen
- `dev.target.com` → subdomen (dasturchilar uchun, kam himoyalangan!)
- `staging.target.com` → test muhiti

**Nima uchun muhim?**
Asosiy sayt yaxshi himoyalangan bo'lishi mumkin, lekin `dev.target.com` yoki `old.target.com` zaif bo'lishi ehtimoli yuqori.

---

## 🔍 Subdomen Topish Usullari

### 1. SSL/TLS Sertifikat Tekshirish (Certificate Transparency)
Har bir HTTPS sertifikati olganda, u ochiq ro'yxatga kiritiladi. Bu ro'yxatdan subdomenlarni topish mumkin.

**crt.sh saytidan foydalanish:**
```
https://crt.sh/?q=%25.target.com
```
(% belgisi "wildcard" — har qanday subdomen demak)

```bash
# Buyruq satiri orqali
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sort -u
```

---

### 2. Brute Force (Zo'rlab Urinish)
Ko'p tarqalgan subdomen nomlari ro'yxatidan foydalanib, barchasini DNS orqali tekshirish.

#### dnsrecon
```bash
dnsrecon -t brt -d target.com -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

#### Amass
```bash
# Passiv topish (saytga so'rov yubormasdan)
amass enum -passive -d target.com

# Aktiv topish (DNS brute force bilan)
amass enum -active -d target.com -brute -w wordlist.txt

# Natijani faylga saqlash
amass enum -passive -d target.com -o amass_results.txt
```

#### Subfinder
```bash
subfinder -d target.com
subfinder -d target.com -o subfinder_results.txt
```

#### Gobuster (DNS rejimida)
```bash
gobuster dns -d target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Verbose — IP manzillarini ham ko'rish
gobuster dns -d target.com -w wordlist.txt -v
```

#### Ffuf (subdomen uchun)
```bash
ffuf -u http://FUZZ.target.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fc 404 -mc 200,301,302,403
```

---

### 3. OSINT — Ochiq Ma'lumot Manbalari

#### VirusTotal
```
https://www.virustotal.com/gui/domain/target.com/relations
```

#### DNSDumpster
```
https://dnsdumpster.com
```

#### Shodan
```
https://shodan.io
hostname:target.com
```

#### Google Dorks
```
site:*.target.com           → barcha subdomenlar
-site:www.target.com site:*.target.com   → www ni chiqarib
```

---

### 4. Virtual Host (VHost) Topish
Ba'zan subdomenlar DNS da ro'yxatda bo'lmaydi, lekin server ularni tushunadi. Bu `Host` sarlavhasi (header) orqali amalga oshiriladi.

```bash
# Ffuf bilan VHost brute force
ffuf -u http://target.com \
  -H "Host: FUZZ.target.com" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fc 404 \
  -fs 0    # Agar bir xil o'lchamli javoblar ko'p bo'lsa, o'lchamni filtirla

# Gobuster bilan VHost
gobuster vhost -u http://target.com -w wordlist.txt
```

---

## 📋 DNS Yozuvlari (DNS Records)

| Tur | Ma'nosi |
|-----|---------|
| **A** | Domen → IPv4 manzil |
| **AAAA** | Domen → IPv6 manzil |
| **CNAME** | Taxallus (alias) — boshqa domeni ko'rsatadi |
| **MX** | Mail server (pochta) |
| **TXT** | Matn ma'lumotlari (SPF, verification) |
| **NS** | Name server (DNS serveri) |

```bash
# DNS yozuvlarini ko'rish
dig target.com ANY
nslookup target.com
host target.com

# Faqat A yozuvi
dig target.com A

# Faqat MX
dig target.com MX

# Zone transfer urinish (AXFR) — zaif DNS serverlarda ishlaydi
dig axfr @ns1.target.com target.com
```

---

## 🎯 Amaliy Misol: To'liq Subdomen Skan

```bash
# 1-qadam: Sertifikat orqali passiv qidirish
curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq -r '.[].name_value' | sort -u > crt_results.txt

# 2-qadam: Subfinder bilan OSINT
subfinder -d target.com -o subfinder_results.txt

# 3-qadam: DNS brute force
gobuster dns -d target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -o gobuster_dns.txt

# 4-qadam: VHost tekshirish (agar IP manzili ma'lum bo'lsa)
ffuf -u http://10.10.x.x \
  -H "Host: FUZZ.target.com" \
  -w wordlist.txt \
  -fc 404

# 5-qadam: Barcha natijalarni birlashtirish va takrorlarni o'chirish
cat crt_results.txt subfinder_results.txt gobuster_dns.txt | sort -u > all_subdomains.txt

# 6-qadam: Har bir subdomenni tekshirish
for sub in $(cat all_subdomains.txt); do
  echo -n "$sub: "
  dig +short $sub A
done
```

---

## 💡 Eslab Qolish Uchun

| Usul | Vosita | Xususiyat |
|------|--------|-----------|
| SSL sertifikat | crt.sh | Passiv, tez |
| OSINT | Subfinder, Amass | Passiv |
| Brute Force | Gobuster, ffuf | Aktiv, to'liq |
| VHost | ffuf + Host header | Yashirin subdomenlar |
| DNS Zone Transfer | dig axfr | Eski, zaif serverlarda |

- **Passiv** = Maqsad serverga to'g'ridan-to'g'ri so'rov yubormaymiz
- **Aktiv** = Maqsad serverga so'rov yuboramiz (shovqin ko'p)
- `dev.`, `staging.`, `test.`, `old.`, `beta.` — eng qiziqarli subdomenlar
