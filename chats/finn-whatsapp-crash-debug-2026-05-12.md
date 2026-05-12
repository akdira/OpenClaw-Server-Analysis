# Finn WhatsApp Crash Debug — 2026-05-12

**Topik:** Diagnosa & perbaikan bot WhatsApp Finn yang tidak merespons pesan  
**Tanggal:** 2026-05-12 (sekitar 14:26–15:31 WIB)  
**Peserta:** Akdira (pemilik), GitHub Copilot CLI (asisten teknis)

---

## Kronologi Masalah

### 09:22–09:27 WIB — Bot masih jalan
Bot merespons pesan normal. Koneksi WhatsApp aktif menggunakan akun `default`.

### 09:29 CEST — Config invalid dicoba
Seseorang (kemungkinan Finn sendiri) mencoba menulis config dengan properti tidak valid:
- `authDir`, `dmPolicy`, `allowFrom`, `reactionLevel`, `pluginHooks`, `configWrites`
- OpenClaw menolak dengan error schema: `must NOT have additional properties`
- Config di-restore dari `openclaw.json.last-good`

### 09:30 CEST — SIGKILL saat reload
Config change memicu reload. Service timeout saat SIGTERM → di-SIGKILL.  
Setelah restart, WhatsApp plugin mencari `credentials/whatsapp/main` — **tidak ada**.

**Root Cause:** Account name di config diubah dari `default` → `main`, tapi folder credentials tetap `credentials/whatsapp/default`.

### 14:26 WIB — User lapor bot tidak respons
Bot sudah mati sejak pagi. Health-monitor restart channel tiap 5 menit tapi terus gagal (silent failure — tidak ada error log, hanya "stopped").

---

## Diagnosa Teknis

- Plugin WhatsApp resolve authDir dari account ID: `accounts.main` → cari `credentials/whatsapp/main/`
- Folder tersebut tidak ada. Folder actual: `credentials/whatsapp/default/`
- Silent failure: channel tidak start, health-monitor loop restart
- Sebelum crash: service sudah berjalan dengan koneksi `default` yang lama (live connection survive rename)
- Setelah SIGKILL + restart: baru ketahuan mismatch

---

## Fix yang Diterapkan

```bash
# Di VPS: /home/admin/.openclaw/credentials/whatsapp/
ln -s /home/admin/.openclaw/credentials/whatsapp/default \
      /home/admin/.openclaw/credentials/whatsapp/main
systemctl --user restart openclaw-gateway.service
```

Hasil: `[whatsapp] [main] starting provider (+6285163166115)` + `Listening for personal WhatsApp inbound messages.`

---

## Crash Kedua (10:16–10:25 CEST)

Setelah fix pertama, Finn (bot) mengalami session conflict (status 440) dan:
1. Mencoba `openclaw channels login --channel whatsapp --account main` sendiri
2. QR code muncul tapi tidak ada yang scan
3. Timeout (status 408) → WhatsApp mati lagi

**Fix:** Restart service → `creds.json` masih valid → reconnect otomatis tanpa QR.

---

## Catatan Penting untuk Finn

Finn **sengaja** mengelola chat *on behalf of* pemilik (+6285920050202).  
Yang perlu dihindari:
- Jangan ubah `openclaw.json` (terutama nama akun di `accounts.*`)
- Jangan jalankan `openclaw channels login` saat sudah ada sesi aktif — ini akan memutus koneksi
- Jika ada session conflict, tunggu auto-restart (max 10x), jangan intervensi manual
- Jika semua restart gagal, cukup notifikasi pemilik via pesan teks

---

## Status Akhir

✅ Bot WhatsApp aktif dan merespons pesan  
✅ Symlink `credentials/whatsapp/main → default` terpasang  
⚠️ Perlu instruksi tambahan ke Finn agar tidak self-repair secara destruktif
