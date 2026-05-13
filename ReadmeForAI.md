# ReadmeForAI — Instruksi untuk AI Assistant

Dokumen ini berisi instruksi untuk AI assistant (GitHub Copilot, Finn, atau AI lainnya) yang bekerja di repo ini.

---

## 📁 Pencatatan Percakapan

> ⚠️ **WAJIB: Setiap percakapan dengan pengguna HARUS selalu dicatat di folder `chats/` tanpa terkecuali.**  
> Ini berlaku untuk semua AI assistant yang bekerja di repo ini — Copilot CLI, Finn, atau AI lainnya.

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

### Aturan pencatatan:
- **Selalu buat atau update file `chats/` di akhir setiap sesi** — jangan tunggu diminta
- Jika topik sama dengan hari yang sama, update file yang sudah ada (jangan buat duplikat)
- Jika sesi berlanjut ke hari berikutnya, buat file baru dengan tanggal baru
- Update tabel **Catatan Teknis Aktif** di bawah setiap kali ada file baru

---

## 🖥️ Konteks Server

- **VPS:** Contabo Singapore (lihat `.credentials`)
- **OS:** Linux (systemd)
- **Service:** `openclaw-gateway.service` (user: `admin`)
- **OpenClaw version:** v2026.5.7
- **Domain:** (lihat `.credentials`)
- **Bot WhatsApp:** (lihat `.credentials`)
- **Pemilik:** (lihat `.credentials`)
- **Log file:** `/tmp/openclaw/openclaw-YYYY-MM-DD.log`
- **Config:** `/home/admin/.openclaw/openclaw.json`

## 🔑 Akses Server

Gunakan SSH native (sudah ada key di `~/.ssh/id_ed25519`):
```bash
ssh root@<SERVER_IP> "perintah"
```
IP server dan semua kredensial lengkap ada di file `.credentials` (jangan commit ke git).  
SSH key sudah terpasang di server — tanpa password.

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
| 2026-05-12 | Finn WhatsApp crash, fix symlink, rename akun, setup SSH key, rencana dual-account | [chats/finn-whatsapp-crash-debug-2026-05-12.md](chats/finn-whatsapp-crash-debug-2026-05-12.md) |
| 2026-05-12 | Custom MD files tidak muncul di web UI OpenClaw — investigasi root cause | [chats/openclaw-custom-md-files-2026-05-12.md](chats/openclaw-custom-md-files-2026-05-12.md) |
| 2026-05-14 | DeepSeek kredit habis + Gemini 2.0 Flash deprecated → fix model ke flash + gemini-2.5-flash | [chats/model-fix-deepseek-gemini-2026-05-14.md](chats/model-fix-deepseek-gemini-2026-05-14.md) |
