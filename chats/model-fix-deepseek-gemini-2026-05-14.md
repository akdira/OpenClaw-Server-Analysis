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

## Fix Tambahan (Sesi yang Sama)

Setelah restart, Finn masih tidak menjawab. Investigasi lanjutan:

### Root Cause #3: Session model override tersimpan lama
File `/home/admin/.openclaw/agents/main/sessions/sessions.json` menyimpan:
```
session: agent:main:whatsapp:direct:+6285920050202 → model: gemini-2.0-flash
```
Session ini ter-set dari percakapan sebelumnya dan TIDAK terhapus saat restart.

**Fix:** Reset model di sessions.json ke `null` (default) untuk semua session yang pakai gemini-2.0-flash.  
Backup: `sessions.json.bak-2026-05-14`

### Root Cause #4: Workspace file instruksikan gemini-2.0-flash
`/home/admin/.openclaw/workspace/AI_AND_SAVING_COST_RULES.md` masih menyebut `gemini-2.0-flash` sebagai fallback model dan instruksi switch model.  
**Fix:** Replace semua referensi ke `gemini-2.5-flash` (4 tempat).

### DeepSeek billing disabled state di-clear
File `/home/admin/.openclaw/agents/main/agent/auth-state.json` menyimpan deepseek dinonaktifkan sampai jam ~07:24 WIB.  
Setelah Akmal top-up DeepSeek, state ini di-clear manual agar OpenClaw langsung coba lagi.

## Test Akhir
- **Gemini 2.5 Flash API**: ✅ response `OK`
- **DeepSeek Reasoner API**: ✅ response `OK` (API alias ke deepseek-v4-flash)
- **Finn balas pesan test**: ✅ "Gue catet dulu... makasih!"

## Update Sesi Lanjutan (~03:00 WIB)

### deepseek-reasoner ditambahkan ke agent models
`openclaw.json` di `agents.defaults.models` ditambahkan entry `deepseek/deepseek-reasoner`.  
Config hot-reload applied. Finn kini bisa pakai DeepSeek Reasoner saat diminta.

### Investigasi Ollama sebagai fallback lokal (gratis)
Finn diminta investigasi opsi AI lokal sebagai fallback terakhir jika semua API berbayar gagal.  
Hasil investigasi Finn:

| Resource | Kapasitas | Status |
|----------|-----------|--------|
| CPU | 4 core AMD EPYC (no GPU) | Pure CPU inference |
| RAM | 6.3GB available | ✅ Cukup model 0.5B–3B |
| Disk | 126GB free | ✅ Lega |
| Ollama | Belum terinstall | Perlu install jika jadi |

- Plugin `@openclaw/ollama-provider` sudah ada di OpenClaw → support native Ollama API
- Model rekomendasi: `deepseek-r1:1.5b` (~1.1GB) atau `qwen2.5:0.5b` (~400MB)
- **Status: DITAHAN** — owner belum memutuskan untuk install. Tunggu instruksi.

Fallback chain yang direncanakan (jika jadi):
```
1️⃣ deepseek/deepseek-v4-flash   ← Primary
2️⃣ google/gemini-2.5-flash      ← Fallback 1
3️⃣ ollama-local/deepseek-r1:1.5b ← Fallback 2 (gratis, lokal)
```

## Catatan Penting untuk Sesi Berikutnya
- Primary: `deepseek/deepseek-v4-flash` (hemat, pake ini untuk chat normal)
- Fallback: `google/gemini-2.5-flash` (deprecated: jangan pakai gemini-2.0-flash lagi)
- Jika mau pakai Pro: ubah `primary` ke `deepseek/deepseek-v4-pro` di openclaw.json
- **Jangan switch model ke gemini-2.0-flash** — sudah deprecated (HTTP 404)
- **Ollama lokal: belum diinstall** — menunggu keputusan owner
