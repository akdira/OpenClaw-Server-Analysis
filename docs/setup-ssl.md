# Setup SSL untuk OpenClaw VPS

Server: `194.233.91.93`
Email: `akdira@gmail.com`

> Certbot sudah terinstall di server. Tinggal jalankan setelah domain siap.

---

## Opsi A — DuckDNS (domain gratis)

1. Buka https://www.duckdns.org → login pakai Google
2. Buat subdomain, misal: `akdira-openclaw.duckdns.org`
3. Set IP ke `194.233.91.93`
4. Jalankan di server:

```bash
# Ganti YOUR_DOMAIN dengan domain kamu
DOMAIN="akdira-openclaw.duckdns.org"

# Update nginx config
sed -i "s/194.233.91.93/$DOMAIN/g" /etc/nginx/sites-available/openclaw

# Issue Let's Encrypt cert
certbot --nginx -d $DOMAIN --email akdira@gmail.com --agree-tos --non-interactive

# Reload nginx
systemctl reload nginx
```

5. Update `allowedOrigins` di `/home/admin/.openclaw/openclaw.json`:
```json
"allowedOrigins": [
  "http://localhost:18789",
  "https://akdira-openclaw.duckdns.org"
]
```

6. Restart OpenClaw:
```bash
su - admin -c 'XDG_RUNTIME_DIR=/run/user/1000 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus systemctl --user restart openclaw-gateway.service'
```

---

## Opsi B — Domain sendiri (via Cloudflare)

1. Tambahkan A record di Cloudflare: `openclaw.yourdomain.com` → `194.233.91.93`
2. Set Proxy status: **DNS only** (abu-abu, bukan orange) agar certbot bisa verify
3. Jalankan:
```bash
DOMAIN="openclaw.yourdomain.com"
sed -i "s/194.233.91.93/$DOMAIN/g" /etc/nginx/sites-available/openclaw
certbot --nginx -d $DOMAIN --email akdira@gmail.com --agree-tos --non-interactive
systemctl reload nginx
```
4. Setelah cert aktif, bisa aktifkan Cloudflare Proxy (orange) untuk keamanan tambahan

---

## Status saat ini
- ✅ Self-signed SSL aktif di `https://194.233.91.93` (ada browser warning, tapi berfungsi)
- ✅ Certbot v2.9.0 sudah terinstall, siap dipakai
- ⏳ Menunggu domain untuk Let's Encrypt cert
