# ReadmeForAI — Instruksi untuk AI Assistant

Dokumen ini berisi instruksi untuk AI assistant (GitHub Copilot, Finn, atau AI lainnya) yang bekerja di repo ini.

---

## 📁 Pencatatan Percakapan

**Setiap pembicaraan teknis yang signifikan harus dicatat di folder `chats/`.**

### Format nama file:
```
chats/{topik-singkat}-{YYYY-MM-DD}.md
```

Contoh:
- `chats/finn-whatsapp-crash-debug-2026-05-12.md`
- `chats/ssl-setup-2026-05-13.md`
- `chats/config-fix-2026-05-14.md`

### Isi yang wajib dicatat:
1. **Topik & tanggal** percakapan
2. **Kronologi masalah** (jika debugging)
3. **Root cause** yang ditemukan
4. **Fix yang diterapkan** (lengkap dengan command)
5. **Status akhir** (berhasil / gagal / ongoing)
6. **Catatan penting** untuk sesi berikutnya

### Kapan membuat file baru:
- Ada debugging/troubleshooting yang diselesaikan
- Ada perubahan konfigurasi server yang signifikan
- Ada informasi teknis penting yang perlu diingat antar sesi

---

## 🖥️ Konteks Server

- **VPS:** Contabo Singapore — `194.233.91.93`
- **OS:** Linux (systemd)
- **Service:** `openclaw-gateway.service` (user: `admin`)
- **OpenClaw version:** v2026.5.7
- **Domain:** `finn.jpu.my.id`
- **Bot WhatsApp:** `+6285163166115`
- **Pemilik:** `+6285920050202`
- **Log file:** `/tmp/openclaw/openclaw-YYYY-MM-DD.log`
- **Config:** `/home/admin/.openclaw/openclaw.json`

## 🔑 Akses Server

Gunakan PuTTY plink (`C:\Program Files\PuTTY\plink.exe`) atau SSH native.  
Kredensial ada di file `.credentials` (jangan commit ke git).

---

## ⚠️ Hal yang JANGAN dilakukan di server

1. **Jangan ubah `openclaw.json`** tanpa konfirmasi eksplisit pemilik
2. **Jangan jalankan `openclaw channels login`** saat ada sesi WhatsApp aktif
3. **Jangan hapus atau pindahkan** folder `credentials/whatsapp/`
4. **Jangan restart service** kecuali diperlukan (health-monitor sudah ada)

---

## 📝 Catatan Teknis Aktif

| Tanggal | Topik | File |
|---------|-------|------|
| 2026-05-12 | Finn WhatsApp crash & fix symlink | [chats/finn-whatsapp-crash-debug-2026-05-12.md](chats/finn-whatsapp-crash-debug-2026-05-12.md) |
