# FinnWA — Setup & WhatsApp Integration Session
**Tanggal:** 2026-05-13 s/d 2026-05-14  
**Peserta:** Akmal (pemilik), GitHub Copilot CLI (asisten teknis)

---

## Latar Belakang

Integrasi WhatsApp existing (OpenClaw di VPS) bermasalah. Akmal minta buatkan sistem baru yang bisa:
- Baca pesan WhatsApp
- Finn (AI) nilai kepentingan pesan
- Kasih tahu Akmal kalau ada yang penting
- Balas **atas izin dulu** — Finn tidak balas sendiri tanpa konfirmasi
- Goal jangka panjang: multi-akun WA + PostgreSQL

---

## Apa yang Dibangun

**Folder:** `D:\SC\Akdira\FinnWA\`  
**Nama proyek:** FinnWA (Finn WhatsApp Assistant)

### Stack
- **Node.js** + `@whiskeysockets/baileys` → WhatsApp client (WebSocket, tanpa Chrome/Puppeteer)
- **Python** → Finn AI logic, permission system, terminal UI
- **HTTP API lokal** di `:3001` → bridge antara Node.js dan Python

### Kenapa Baileys, bukan whatsapp-web.js?
Pertama dicoba `whatsapp-web.js` (Puppeteer-based). Crash terus dengan `ProtocolError: Runtime.callFunctionOn timed out` dan `ready` event tidak pernah firing. Diganti ke Baileys — WebSocket langsung, tidak butuh browser headless.

---

## Kronologi Session

### Attempt 1 — whatsapp-web.js (gagal)
- Install Node.js + Python via `winget`
- Buat struktur folder: `whatsapp/`, `finn/`, `data/`, `sessions/`
- Tulis `whatsapp/index.js` pakai `whatsapp-web.js` + Puppeteer
- QR muncul di background terminal Copilot, tidak terlihat user
- **Fix:** Tambah endpoint `/qr` → render QR sebagai HTML di browser, auto-open
- User scan → `ProtocolError` crash saat `getChats()` dipanggil di startup
- Timeout dinaikkan ke 300 detik → masih crash, `ready` event tidak pernah firing

### Switch ke Baileys
- Uninstall `whatsapp-web.js`, install `@whiskeysockets/baileys` + `pino`
- Rewrite `whatsapp/index.js` dari nol untuk Baileys 6.x API
- Error pertama: `makeInMemoryStore is not a function` — dihapus di Baileys 6.x
- Fix: hapus `makeInMemoryStore`, buat `chatMap` + `contactMap` manual pakai Map
- QR muncul, user scan → **✅ WhatsApp terhubung!**

### Problem: Chat dump = 0
Setelah login, `GET /chats` selalu return array kosong.

**Root cause:** 
- `chats.set` hanya firing saat first-ever login dengan `syncFullHistory: true`
- Pada reconnect dengan session yang sudah ada, event ini tidak firing sama sekali
- `messaging-history.set` juga tidak firing pada reconnect

**Fix yang diterapkan:**
1. Set `syncFullHistory: true` di config Baileys
2. Tambah fallback: setelah `connection: 'open'`, tunggu 5 detik lalu call `sock.groupFetchAllParticipating()` untuk ambil semua grup
3. Simpan contacts ke disk (`data/contacts.json`) untuk cross-session DM discovery
4. Auto-register chat ke `chatMap` setiap ada pesan masuk (DM auto-discover)
5. Persist `chatMap` ke `data/chats.json` dan load ulang saat startup

**Hasil:** 79 grup berhasil ter-dump. DM akan muncul otomatis saat ada pesan masuk.

### Filter protocolMessage
Saat reconnect, banyak `protocolMessage` flood masuk ke `messages.json` sebagai "pesan baru".  
**Fix:** Filter `protocolMessage` dan `senderKeyDistributionMessage` di handler `messages.upsert`.

---

## Status Akhir

| | Status |
|---|---|
| WhatsApp login via QR (browser) | ✅ |
| Session persist (tidak perlu scan ulang) | ✅ |
| 79 grup ter-dump ke `data/chats.json` | ✅ |
| HTTP API di `localhost:3001` | ✅ |
| Monitor pesan masuk → `messages.json` | ✅ |
| Filter spam protocolMessage | ✅ |
| Finn Python app (menu + monitor mode) | ✅ |
| DM chat list | ⚠️ Muncul otomatis saat ada pesan masuk |
| Integrasi LLM untuk Finn | ⏳ Belum |
| Multi-akun WhatsApp | ⏳ Belum |
| PostgreSQL | ⏳ Belum |
| Docker | ⏳ Belum |

---

## File Penting

| File | Keterangan |
|---|---|
| `FinnWA/whatsapp/index.js` | WhatsApp client + HTTP API |
| `FinnWA/finn/main.py` | Finn AI — menu, analisis, permission |
| `FinnWA/data/chats.json` | Snapshot 79 grup |
| `FinnWA/data/messages.json` | Log pesan masuk |
| `FinnWA/sessions/baileys/` | Session WA — jangan dihapus! |
| `FinnWA/GOALS.md` | Roadmap lengkap + schema PostgreSQL |
| `FinnWA/HowTo.md` | Panduan operasional |

---

## Catatan Teknis (untuk Finn / AI berikutnya)

- **Baileys 6.x**: `makeInMemoryStore` sudah dihapus, harus buat store manual
- **Chat sync on reconnect**: WhatsApp tidak kirim ulang DM list. Hanya grup yang bisa di-fetch via `groupFetchAllParticipating()`. DM harus dari pesan masuk atau first-login fresh.
- **`chats.set` event**: Hanya firing saat first-ever login dengan `syncFullHistory: true`. Jangan andalkan ini untuk reconnect.
- **QR code 515**: Setelah scan QR, WhatsApp kirim disconnect code 515 (normal — upgrade connection). Harus auto-reconnect, jangan anggap logout.
- **Session path**: `sessions/baileys/` — multifile auth state. Jika dihapus, harus scan QR baru.

---

## Cara Jalankan Ulang

```bash
cd D:\SC\Akdira\FinnWA

# Terminal 1 — WhatsApp server
npm start

# Terminal 2 — Finn AI
python finn/main.py
```

Lihat `FinnWA/HowTo.md` untuk panduan lengkap.
