# OpenClaw — Custom MD Files Tidak Muncul di Web UI

**Tanggal:** 12 Mei 2026  
**Topik:** File-file custom di workspace tidak tampil di OpenClaw web UI (tab Files)  
**Status:** ✅ Investigasi selesai — tidak ada fix yang diterapkan (terlalu beresiko)

---

## 📋 Kronologi Masalah

User melihat bahwa di web UI OpenClaw (`finn.jpu.my.id`) → Agents → Files, tab yang muncul hanya 7:
`AGENTS`, `SOUL`, `TOOLS`, `IDENTITY`, `USER`, `HEARTBEAT`, `MEMORY`

Padahal di server ada banyak file lagi:
- `AIRBNB.md`
- `BOOT.md`
- `CrashPrevention.md`
- `WHATSAPP.md`
- `USER-PEVITA.md`
- `FILES.md`
- `memory/2026-05-11.md`, `memory/2026-05-12.md`

---

## 🔍 Root Cause

OpenClaw web UI hanya menampilkan **core files yang hardcoded** di source-nya. Ditemukan di:

**File:** `/home/admin/.npm-global/lib/node_modules/openclaw/dist/server-methods-DStUV8Sh.js`

```js
const BOOTSTRAP_FILE_NAMES = [
  DEFAULT_AGENTS_FILENAME,    // AGENTS.md
  DEFAULT_SOUL_FILENAME,      // SOUL.md
  DEFAULT_TOOLS_FILENAME,     // TOOLS.md
  DEFAULT_IDENTITY_FILENAME,  // IDENTITY.md
  DEFAULT_USER_FILENAME,      // USER.md
  DEFAULT_HEARTBEAT_FILENAME, // HEARTBEAT.md
  DEFAULT_BOOTSTRAP_FILENAME  // (internal)
];
const MEMORY_FILE_NAMES = [DEFAULT_MEMORY_FILENAME];
const ALLOWED_FILE_NAMES = new Set([...BOOTSTRAP_FILE_NAMES, ...MEMORY_FILE_NAMES]);
```

File di luar daftar ini tidak akan pernah ditampilkan di web UI.

---

## 💡 Ada Mekanisme Extra Bootstrap

OpenClaw punya mekanisme `bootstrap-extra-files` hook (di `/dist/bundled/bootstrap-extra-files/handler.js`) yang bisa load file tambahan ke **runtime Finn** via config `patterns`/`paths`. Tapi ini hanya untuk Finn membaca file saat startup — bukan untuk menampilkannya di web UI.

---

## ⚠️ Kenapa Tidak Di-Fix

Untuk menampilkan file custom di web UI, perlu **patch minified JS** di node_modules OpenClaw. Resikonya:
1. Hilang saat update `openclaw`
2. Bisa crash `openclaw-gateway.service` kalau salah edit
3. Tidak ada rollback mudah

Kesimpulan: **tidak layak dirisiko**.

---

## ✅ Status File Custom

Semua file custom tetap berfungsi normal — Finn bisa baca dan update saat runtime. Hanya tidak bisa di-edit lewat web UI.

Untuk edit file custom, gunakan:
```bash
ssh root@194.233.91.93 "nano /home/admin/.openclaw/workspace/AIRBNB.md"
```
Atau minta Finn update via WhatsApp.

---

## 📄 Isi File Custom (per 12 Mei 2026)

### File yang ada di workspace tapi tidak tampil di web UI:
| File | Keterangan |
|------|------------|
| `AIRBNB.md` | Rekap hosting Airbnb — 3 properti, reservasi aktif, keuangan |
| `BOOT.md` | Startup checklist Finn — health check, routines, No Shopping Day |
| `CrashPrevention.md` | Lessons learned dari crash 12 Mei — rules setup WhatsApp multi-account |
| `WHATSAPP.md` | Setup & rules multi-account WhatsApp (default + akmal) |
| `USER-PEVITA.md` | Data Pevita (Pinta Uli Sinaga) — whitelisted contact |
| `FILES.md` | Peta semua file workspace |
| `memory/2026-05-11.md` | Memory harian Finn 11 Mei |
| `memory/2026-05-12.md` | Memory harian Finn 12 Mei |

---

## 📝 Catatan untuk Sesi Berikutnya

- Web UI tab Files hanya tampilkan 7 core files — ini by design, bukan bug
- File custom tetap aktif dan dibaca Finn saat runtime
- Kalau perlu edit file custom: SSH langsung atau via Finn di WhatsApp
