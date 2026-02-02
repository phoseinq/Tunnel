# راهنمای کامل ساخت تانل 6to4 + GRE6 + HAProxy

## 📋 معرفی

این آموزش به شما کمک می‌کنه تا بین دو سرور (یکی در ایران و یکی در خارج) یک تانل امن بسازید و با استفاده از HAProxy ترافیک رو منتقل کنید.

### چرا از این روش استفاده می‌کنیم؟

- دور زدن فیلترینگ و سانسور
- انتقال ترافیک از سرور ایران به خارج
- بهبود سرعت و پایداری اتصال

---

## 🛠️ پیش‌نیازها

- دو سرور لینوکس (یکی ایران، یکی خارج)
- دسترسی root به هر دو سرور
- IP عمومی هر دو سرور

---

## 📝 مرحله 0: آماده‌سازی IP ها

برای این آموزش از IP های زیر استفاده می‌کنیم (شما باید IP های سرورهای خودتون رو جایگزین کنید):

```
IP سرور ایران: YOUR_IRAN_IP
IP سرور خارج: YOUR_KHAREJ_IP
```

IPv6 های لوکال که استفاده می‌کنیم:
```
IPv6 ایران: fd12:3456:789a::1/64
IPv6 خارج: fd12:3456:789a::2/64
```

IPv4 های لوکال GRE:
```
IPv4 GRE ایران: 10.10.10.1/30
IPv4 GRE خارج: 10.10.10.2/30
```

---

## 🇮🇷 قسمت اول: راه‌اندازی سرور ایران

### مرحله 1: پاک کردن تانل‌های قبلی (در صورت وجود)

ابتدا SSH به سرور ایران بزنید و این دستورات رو اجرا کنید:

```bash
# پاک کردن تانل‌های قبلی
ip link set GRE6Tun_To_KH down 2>/dev/null
ip link set 6to4_To_KH down 2>/dev/null
ip -6 tunnel del GRE6Tun_To_KH 2>/dev/null
ip tunnel del 6to4_To_KH 2>/dev/null

# پاک کردن قوانین فایروال قبلی
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
ip6tables -F
ip6tables -X
```

### مرحله 2: لود کردن ماژول‌های کرنل

```bash
modprobe sit
modprobe ip6_gre
modprobe ip6_tunnel
```

### مرحله 3: ساخت تانل 6to4

**مهم:** در خط اول، `YOUR_KHAREJ_IP` را با IP سرور خارج و `YOUR_IRAN_IP` را با IP سرور ایران جایگزین کنید.

```bash
ip tunnel add 6to4_To_KH mode sit remote YOUR_KHAREJ_IP local YOUR_IRAN_IP
ip -6 addr add fd12:3456:789a::1/64 dev 6to4_To_KH
ip link set 6to4_To_KH mtu 1280
ip link set 6to4_To_KH up
```

### مرحله 4: ساخت تانل GRE6

```bash
ip -6 tunnel add GRE6Tun_To_KH mode ip6gre remote fd12:3456:789a::2 local fd12:3456:789a::1
ip addr add 10.10.10.1/30 dev GRE6Tun_To_KH
ip link set GRE6Tun_To_KH mtu 1240
ip link set GRE6Tun_To_KH up
```

### مرحله 5: تنظیم فایروال

```bash
iptables -A INPUT -p gre -j ACCEPT
iptables -A OUTPUT -p gre -j ACCEPT
ip6tables -A INPUT -p ipv6-icmp -j ACCEPT
ip6tables -A OUTPUT -p ipv6-icmp -j ACCEPT
```

### مرحله 6: بررسی تانل‌ها

```bash
# مشاهده تانل‌ها
ip tunnel show
ip -6 tunnel show

# مشاهده IP ها
ip addr show dev GRE6Tun_To_KH
```

---

## 🌍 قسمت دوم: راه‌اندازی سرور خارج

### مرحله 1: پاک کردن تانل‌های قبلی

SSH به سرور خارج بزنید:

```bash
# پاک کردن تانل‌های قبلی
ip link set GRE6Tun_To_IR down 2>/dev/null
ip link set 6to4_To_IR down 2>/dev/null
ip -6 tunnel del GRE6Tun_To_IR 2>/dev/null
ip tunnel del 6to4_To_IR 2>/dev/null

# پاک کردن فایروال
iptables -F
iptables -X
ip6tables -F
ip6tables -X
```

### مرحله 2: لود کردن ماژول‌ها

```bash
modprobe sit
modprobe ip6_gre
modprobe ip6_tunnel
```

### مرحله 3: ساخت تانل 6to4

**مهم:** `YOUR_IRAN_IP` و `YOUR_KHAREJ_IP` را جایگزین کنید.

```bash
ip tunnel add 6to4_To_IR mode sit remote YOUR_IRAN_IP local YOUR_KHAREJ_IP
ip -6 addr add fd12:3456:789a::2/64 dev 6to4_To_IR
ip link set 6to4_To_IR mtu 1280
ip link set 6to4_To_IR up
```

### مرحله 4: ساخت تانل GRE6

```bash
ip -6 tunnel add GRE6Tun_To_IR mode ip6gre remote fd12:3456:789a::1 local fd12:3456:789a::2
ip addr add 10.10.10.2/30 dev GRE6Tun_To_IR
ip link set GRE6Tun_To_IR mtu 1240
ip link set GRE6Tun_To_IR up
```

### مرحله 5: تنظیم فایروال

```bash
iptables -A INPUT -p gre -j ACCEPT
iptables -A OUTPUT -p gre -j ACCEPT
ip6tables -A INPUT -p ipv6-icmp -j ACCEPT
ip6tables -A OUTPUT -p ipv6-icmp -j ACCEPT
```

---

## ✅ قسمت سوم: تست تانل

### در سرور ایران:

```bash
# تست ping با IPv6
ping6 fd12:3456:789a::2 -c 5

# تست ping با IPv4 GRE
ping 10.10.10.2 -c 5
```

### در سرور خارج:

```bash
# تست ping با IPv6
ping6 fd12:3456:789a::1 -c 5

# تست ping با IPv4 GRE
ping 10.10.10.1 -c 5
```

**نتیجه موفق:** اگر پینگ‌ها پاسخ دادند، تانل شما با موفقیت برقرار شده!

---

## 🔧 قسمت چهارم: نصب و تنظیم HAProxy (فقط در سرور ایران)

### مرحله 1: نصب HAProxy

```bash
apt update && apt install haproxy -y
```

### مرحله 2: تنظیم کانفیگ HAProxy

فایل کانفیگ را باز کنید:

```bash
nano /etc/haproxy/haproxy.cfg
```

همه محتوای فایل را پاک کنید و این را جایگزین کنید:

```
global
    log /dev/log local0
    chroot /var/lib/haproxy
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    timeout connect 5000
    timeout client  50000
    timeout server  50000

# پورت 443 (HTTPS)
frontend https_front
    bind *:443
    default_backend https_back

backend https_back
    server kharej 10.10.10.2:443 check

# پورت 80 (HTTP)
frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    server kharej 10.10.10.2:80 check

# پورت 8080 (مثال)
frontend port_8080
    bind *:8080
    default_backend back_8080

backend back_8080
    server kharej 10.10.10.2:8080 check
```

برای ذخیره: `Ctrl + X` سپس `Y` سپس `Enter`

### مرحله 3: راه‌اندازی HAProxy

```bash
systemctl restart haproxy
systemctl enable haproxy
systemctl status haproxy
```

اگر وضعیت `active (running)` بود، HAProxy با موفقیت اجرا شده!

---

## 💾 قسمت پنجم: ذخیره‌سازی دائمی تانل‌ها

این قسمت باعث می‌شود تانل‌ها بعد از ریبوت سرور به صورت خودکار بالا بیایند.

### در سرور ایران:

```bash
nano /etc/rc.local
chmod +x /etc/rc.local
```

این محتوا را در فایل قرار دهید (IP ها را جایگزین کنید):

```bash
#!/bin/bash
modprobe sit
modprobe ip6_gre
modprobe ip6_tunnel

ip tunnel add 6to4_To_KH mode sit remote YOUR_KHAREJ_IP local YOUR_IRAN_IP
ip -6 addr add fd12:3456:789a::1/64 dev 6to4_To_KH
ip link set 6to4_To_KH mtu 1280
ip link set 6to4_To_KH up

ip -6 tunnel add GRE6Tun_To_KH mode ip6gre remote fd12:3456:789a::2 local fd12:3456:789a::1
ip addr add 10.10.10.1/30 dev GRE6Tun_To_KH
ip link set GRE6Tun_To_KH mtu 1240
ip link set GRE6Tun_To_KH up

iptables -A INPUT -p gre -j ACCEPT
iptables -A OUTPUT -p gre -j ACCEPT
ip6tables -A INPUT -p ipv6-icmp -j ACCEPT
ip6tables -A OUTPUT -p ipv6-icmp -j ACCEPT

exit 0
```

### در سرور خارج:

```bash
nano /etc/rc.local
chmod +x /etc/rc.local
```

این محتوا را در فایل قرار دهید:

```bash
#!/bin/bash
modprobe sit
modprobe ip6_gre
modprobe ip6_tunnel

ip tunnel add 6to4_To_IR mode sit remote YOUR_IRAN_IP local YOUR_KHAREJ_IP
ip -6 addr add fd12:3456:789a::2/64 dev 6to4_To_IR
ip link set 6to4_To_IR mtu 1280
ip link set 6to4_To_IR up

ip -6 tunnel add GRE6Tun_To_IR mode ip6gre remote fd12:3456:789a::1 local fd12:3456:789a::2
ip addr add 10.10.10.2/30 dev GRE6Tun_To_IR
ip link set GRE6Tun_To_IR mtu 1240
ip link set GRE6Tun_To_IR up

iptables -A INPUT -p gre -j ACCEPT
iptables -A OUTPUT -p gre -j ACCEPT
ip6tables -A INPUT -p ipv6-icmp -j ACCEPT
ip6tables -A OUTPUT -p ipv6-icmp -j ACCEPT

exit 0
```

---

## 🔥 حل مشکلات رایج

### مشکل 1: پینگ نمی‌خورد

**راه حل:**
- MTU را کمتر کنید (مثلاً 1200 و 1160)
- فایروال سرورها را بررسی کنید
- اطمینان حاصل کنید IP ها درست وارد شده‌اند

```bash
# کم کردن MTU
ip link set 6to4_To_KH mtu 1200
ip link set GRE6Tun_To_KH mtu 1160
```

### مشکل 2: سرعت خیلی پایین

**راه حل:**
- MTU را تنظیم کنید
- با این دستور MTU بهینه را پیدا کنید:

```bash
ping 10.10.10.2 -c 5 -M do -s 1200
ping 10.10.10.2 -c 5 -M do -s 1100
ping 10.10.10.2 -c 5 -M do -s 1000
```

اولین مقداری که پینگ خورد + 28 = MTU مناسب

### مشکل 3: بعد از ریبوت تانل بالا نمی‌آید

**راه حل:**
- اطمینان حاصل کنید `/etc/rc.local` ساخته شده
- دستور `chmod +x /etc/rc.local` را اجرا کنید
- یکبار دستی ریبوت کنید و تست کنید

---

## 📊 تست سرعت

### نصب iperf3:

```bash
apt update && apt install iperf3 -y
```

### در سرور خارج:

```bash
iperf3 -s
```

### در سرور ایران:

```bash
# تست دانلود
iperf3 -c 10.10.10.2 -t 20

# تست آپلود
iperf3 -c 10.10.10.2 -t 20 -R

# تست دو طرفه
iperf3 -c 10.10.10.2 -t 20 -d
```

---

## 🗑️ پاک کردن کامل تانل‌ها

اگر می‌خواهید همه چیز را پاک کنید:

### در سرور ایران:

```bash
ip link set GRE6Tun_To_KH down
ip link set 6to4_To_KH down
ip -6 tunnel del GRE6Tun_To_KH
ip tunnel del 6to4_To_KH
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
systemctl stop haproxy
rm -f /etc/rc.local
```

### در سرور خارج:

```bash
ip link set GRE6Tun_To_IR down
ip link set 6to4_To_IR down
ip -6 tunnel del GRE6Tun_To_IR
ip tunnel del 6to4_To_IR
iptables -F
iptables -X
rm -f /etc/rc.local
```

---

## ✨ نکات مهم

1. **همیشه IP های خودتون رو جایگزین کنید** - `YOUR_IRAN_IP` و `YOUR_KHAREJ_IP`
2. **MTU خیلی مهمه** - اگر مشکل سرعت داشتید، اول MTU رو تنظیم کنید
3. **قبل از هر کاری بکاپ بگیرید** - از تنظیمات فعلی سرورتون
4. **پورت‌های HAProxy** - طبق نیاز خودتون اضافه یا کم کنید
5. **فایروال** - اگر فایروال فعال دارید، پورت‌های لازم رو باز کنید

---

## 📞 پشتیبانی

اگر مشکلی پیش اومد:
- لاگ‌ها رو چک کنید: `journalctl -xe`
- HAProxy logs: `tail -f /var/log/haproxy.log`
- دستور `ip tunnel show` و `ip addr show` رو اجرا کنید و خروجی رو بررسی کنید

---

**موفق باشید!** 🚀
