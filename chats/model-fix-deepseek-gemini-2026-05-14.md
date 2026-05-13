# Model Fix: DeepSeek Billing & Gemini Deprecated — 2026-05-14

## Topik
Finn tidak balas pesan WA. Investigasi dan fix model AI yang bermasalah.

## Kronologi Masalah
- Pukul ~02:35 WIB user lapor bot tidak merespons
- Dicek log `/tmp/openclaw/openclaw-2026-05-14.log`

## Root Cause

### 1. DeepSeek V4 Pro — Kehabisan Kredit
```
⚠️ deepseek (deepseek-v4-pro) returned a billing error — your API key has run out of credits or has an insufficient balance.
```
Model utama `deepseek/deepseek-v4-pro` gagal karena saldo API habis.

### 2. Gemini 2.0 Flash — Model Deprecated
```
This model models/gemini-2.0-flash is no longer available to new users. Please update your code to use a newer model.
```
Model fallback juga gagal karena sudah deprecated.

Akibatnya: semua request gagal, bot tidak bisa membalas.

## Fix yang Diterapkan

**File:** `/home/admin/.openclaw/openclaw.json`  
**Backup:** `/home/admin/.openclaw/openclaw.json.bak-2026-05-14`

Perubahan di `agents.defaults`:
```json
// SEBELUM
"models": { "google/gemini-2.0-flash": { "alias": "Gemini Flash" } },
"model": {
  "primary": "deepseek/deepseek-v4-pro",
  "fallbacks": ["google/gemini-2.0-flash"]
}

// SESUDAH
"models": { "google/gemini-2.5-flash": { "alias": "Gemini Flash" } },
"model": {
  "primary": "deepseek/deepseek-v4-flash",
  "fallbacks": ["google/gemini-2.5-flash"]
}
```

Config hot-reload berhasil, lalu service restart bersih via `openclaw gateway restart`.

## Status Akhir
✅ **Berhasil** — Service running normal, config loaded, health-monitor aktif.

## Catatan Penting untuk Sesi Berikutnya
- **DeepSeek kredit masih kosong** — perlu top-up di [platform.deepseek.com](https://platform.deepseek.com) jika ingin pakai model Pro lagi
- Primary sekarang adalah `deepseek-v4-flash` (lebih hemat, ~12x lebih murah dari Pro)
- Fallback sekarang `gemini-2.5-flash` (model terbaru, sudah aktif)
- Jika ingin kembali ke Pro setelah top-up: ubah `primary` ke `deepseek/deepseek-v4-pro`
