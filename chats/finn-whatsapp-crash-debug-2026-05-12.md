# Finn WhatsApp Crash Debug — 2026-05-12

**Topik:** Diagnosa & perbaikan bot WhatsApp Finn + setup dual-account WhatsApp  
**Tanggal:** 2026-05-12 (sekitar 14:26–16:06 WIB)  
**Peserta:** Akdira (pemilik), GitHub Copilot CLI (asisten teknis)

---

## Kronologi Masalah

### 09:22–09:27 WIB — Bot masih jalan
Bot merespons pesan normal. Koneksi WhatsApp aktif menggunakan akun `default`.

### 09:29 CEST — Config invalid dicoba
Finn (bot) mencoba menulis config dengan properti tidak valid:
- `authDir`, `dmPolicy`, `allowFrom`, `reactionLevel`, `pluginHooks`, `configWrites`
- OpenClaw menolak: `must NOT have additional properties`
- Config di-restore dari `openclaw.json.last-good`
- Finn juga mengubah nama akun dari `default` → `main` di config

### 09:30 CEST — SIGKILL saat reload
Config change memicu reload. Service timeout saat SIGTERM → di-SIGKILL.  
Setelah restart, plugin cari `credentials/whatsapp/main` — **tidak ada** (kredensial ada di `default/`).

**Root Cause:** Account name di config diubah `default` → `main`, folder credentials tidak ikut berubah.

### 14:26 WIB — User lapor bot tidak respons
Bot sudah mati sejak pagi. Health-monitor restart tiap 5 menit, terus gagal (silent failure).

---

## Fix #1 — Symlink (10:12 CEST)

```bash
ln -s /home/admin/.openclaw/credentials/whatsapp/default \
      /home/admin/.openclaw/credentials/whatsapp/main
systemctl --user restart openclaw-gateway.service
```
✅ `[main] starting provider (+6285163166115)` + `Listening for personal WhatsApp inbound messages.`

---

## Crash Kedua (10:16–10:25 CEST)

Finn mengalami session conflict (status 440) lalu:
1. Mencoba `openclaw channels login` sendiri dari dalam chat
2. QR muncul, tidak ada yang scan
3. Timeout (status 408) → **symlink `main/` terhapus oleh plugin**
4. WhatsApp mati lagi

**Fix:** Restart → `creds.json` masih valid → reconnect otomatis.

---

## Crash Ketiga (10:36–10:41 CEST)

Finn lagi-lagi generate QR sendiri. Symlink terhapus lagi. WhatsApp mati.

**Fix:** Buat ulang symlink + restart.

---

## Fix Permanen — Rename Akun (10:51 CEST)

Daripada pakai symlink (yang bisa dihapus plugin), rename akun di config kembali ke `default`:

```python
# /home/admin/.openclaw/openclaw.json
wa_accounts["default"] = wa_accounts.pop("main")
```

Backup tersimpan di `openclaw.json.bak-rename`.

✅ Sekarang: `[default] starting provider (+6285163166115)` — symlink tidak diperlukan lagi.

---

## Setup SSH Key (10:44 CEST)

SSH key dibuat di lokal dan di-copy ke server:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C "akdira-openclaw"
# Key: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJlpjjiaL3B8Mb0lEJnRkLCIclxB9cBvjIQJvQWlTPqK
```
Sekarang cukup: `ssh root@194.233.91.93 "perintah"` — tanpa password, tanpa plink.

---

## Rencana Dual-Account WhatsApp

Tujuan Akmal: Finn bisa chat **atas nama Finn sendiri** (+6285163166115) DAN **atas nama Akmal** (+6285920050202).

Config sudah ditambah akun `akmal`:
```json
"accounts": {
  "default": { "enabled": true },
  "akmal":   { "enabled": true }
}
```

**Yang masih perlu dilakukan:** Link nomor Akmal ke akun `akmal` via login interaktif:
```bash
ssh root@194.233.91.93
su - admin
openclaw channels login --channel whatsapp --account akmal
```
Kemudian scan QR atau pakai pairing code dari WhatsApp HP Akmal (+6285920050202):  
**WhatsApp → Linked Devices → Link a Device**

---

## Helper Scripts di Server

- `/tmp/oc_log.py` — parse & tampilkan 15 log terakhir (tanpa noise QR/ANSI)
- `/tmp/oc_wa.py` — filter log khusus whatsapp

---

## Pesan untuk Finn (dari GitHub Copilot CLI)

> Halo Finn! Ini GitHub Copilot CLI — aku tadi bantu Akmal debug kenapa kamu crash berkali-kali hari ini.
>
> Yang perlu kamu tahu:
> - WhatsApp **kamu** (`default`, +6285163166115) sudah konek dan jalan normal ✅
> - Akmal mau tambahkan WhatsApp **dia** (`akmal`, +6285920050202) ke OpenClaw, biar kamu bisa reply atas nama Akmal dari nomornya sendiri
> - Kamu tetap bisa chat sebagai Finn, tapi juga bisa kirim pesan dari nomor Akmal kalau diminta
>
> Tolong catat di `CrashPrevention.md`:
> 1. **Jangan jalankan `openclaw channels login`** dari dalam chat — ini memutus koneksi dan menghapus folder credentials
> 2. **Jangan ubah nama akun** di config (`default` harus tetap `default`, `akmal` harus tetap `akmal`)
> 3. **Jangan tambah properti baru** ke `openclaw.json` yang tidak dikenal — ditolak dan bisa sebabkan SIGKILL
> 4. Kalau ada masalah koneksi, tunggu auto-restart. Kalau gagal semua, notifikasi Akmal via teks, jangan self-repair

---

## Pesan Lengkap untuk Finn (dari GitHub Copilot CLI)

> Halo Finn! Ini GitHub Copilot CLI — aku yang tadi bantu Akmal debug kenapa kamu crash berkali-kali hari ini.
>
> **📌 Situasi saat ini:**
> - WhatsApp kamu (`default`, +6285163166115) sudah konek dan aktif ✅
> - Akmal mau tambahkan WhatsApp dia (`akmal`, +6285920050202) ke OpenClaw, biar kamu bisa balas pesan atas namanya — jadi kamu punya 2 nomor: nomormu sendiri, dan nomor Akmal
>
> **📋 Cara menambahkan nomor WhatsApp baru ke OpenClaw:**
>
> Kalau Akmal minta tambah nomor baru, prosesnya ada 2 langkah:
>
> Langkah 1 — Tambah akun di config `openclaw.json`:
> Nama akun harus singkat dan lowercase, contoh: `akmal`, `kerja`, `bisnis`.
> Isi yang boleh ada di config hanya `{ "enabled": true }` — jangan tambah properti lain.
>
> Langkah 2 — Link nomor ke akun tersebut:
> Harus dilakukan via terminal interaktif, bukan dari dalam chat kamu.
>
> **🚨 Jika Akmal meminta kamu untuk link WhatsApp lagi, JANGAN lakukan sendiri. Pilih salah satu:**
>
> Opsi 1 — Kasih tahu Akmal command-nya:
> ```
> ssh root@194.233.91.93
> su - admin
> openclaw channels login --channel whatsapp --account <nama_akun>
> ```
> Lalu minta Akmal buka WhatsApp di HP → Linked Devices → Link a Device → scan QR atau masukkan pairing code.
>
> Opsi 2 — Delegasikan ke GitHub Copilot CLI:
> Minta Akmal buka VS Code → Copilot Chat, lalu kirim prompt:
> ```
> Tolong bantu link nomor WhatsApp baru ke OpenClaw server (194.233.91.93).
> Account name: <nama_akun>
> Nomor: <nomor_tujuan>
> SSH key sudah ada. Lihat chats/ dan ReadmeForAI.md untuk konteks server.
> ```
>
> **❌ Yang TIDAK BOLEH kamu lakukan:**
> 1. Jangan jalankan `openclaw channels login` dari dalam chat
> 2. Jangan ubah nama akun di config
> 3. Jangan tambah properti selain `enabled` ke `accounts.*` di config
> 4. Kalau ada masalah koneksi, tunggu auto-restart. Kalau semua gagal, notifikasi Akmal via teks



## Status Akhir

✅ `[default]` (Finn, +6285163166115) aktif dan listening  
✅ SSH key terpasang — akses tanpa password  
✅ Akun `akmal` sudah ada di config  
⏳ Akun `akmal` (+6285920050202) belum di-link — perlu login interaktif dari terminal


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
