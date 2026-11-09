---
title: WO-05 — Custom provider LLM via UI admin (pengganti/penyanding env `INGEST_LLM_*`)
description: "Admin menambah/mengubah provider LLM (base URL, API key terenkripsi, model) lewat UI admin; tersimpan di DB terenkripsi; runtime Node mengambil konfigurasi aktif dari Go per request; key tidak pernah tampil ke klien"
status: progress
created: "2026-11-09"
updated: "2026-11-09"
---
# WO-05 — Custom Provider LLM via UI Admin

**Latar:** permintaan owner 2026-11-09 — saat ini provider LLM hanya bisa diganti
via `.env` (`INGEST_LLM_BASE_URL/API_KEY/MODEL` di runtime Node) yang menuntut
edit berkas + restart container. Owner ingin mengelola provider lewat UI admin.
Owner mengonfirmasi bentuk: (1) provider via UI admin, (2) API key di DB
terenkripsi, (3) dibuat WO sekarang.

## Tujuan

Admin dapat menambah/mengubah/menonaktifkan provider LLM (OpenAI-compatible)
lewat UI admin — tersimpan di DB dengan API key terenkripsi — tanpa edit
`.env` dan tanpa restart. Jalur chat tetap mematuhi D-11: key tidak pernah
sampai ke klien.

## Lingkup

### A. Backend (plugin `ingest`)

- Migrasi: tabel `ingest_providers` (name, base_url, model, api_key_ciphertext,
  is_active, timestamps). Satu provider aktif (UNIQUE partial pada
  `is_active=true`) — model sederhana dulu; multi-provider + per-sesi menyusul
  bila diminta.
- **Enkripsi key**: AES-GCM (pola token endpoint / `INGEST_DEIDENTIFY_KEY`
  fail-closed yang sudah ada). Key plaintext TIDAK pernah kembali ke klien —
  UI menampilkan bentuk mask (`sk-…last4`).
- CRUD admin (guard `ingest.write` / halaman pengaturan baru):
  - `GET /admin/ingest/providers` — daftar (key dimask).
  - `POST /admin/ingest/providers` — tambah.
  - `PUT /admin/ingest/providers/:id` — ubah (base_url/model wajib; key opsional
    = hanya ganti bila diisi).
  - `POST /admin/ingest/providers/:id/activate` — set aktif (otomatis menonaktifkan
    lainnya).
  - `POST /admin/ingest/providers/:id/test` — tes koneksi + daftar model dari
    provider (panggilan server-side; hasil sukses/gagal + pesan).
- Audit semua aksi (`provider='ingest-provider'`).
- **Kontrak runtime Node**: endpoint internal `GET /admin/ingest/providers/active`
  (dipakai runtime Node per request atau cache pendek TTL) mengembalikan
  konfigurasi aktif termasuk plaintext key — HANYA diakses runtime Node dengan
  kredensial service (bukan klien admin). Pola sama seperti `safe-payload`:
  jalur server-to-server, guard khusus, ter-audit.

### B. Runtime Node (`ingest_runtime`)

- `createAgent` per-request kini menyelesaikan provider: konfigurasi aktif dari
  Go → fallback env `INGEST_LLM_*` bila TIDAK ada provider aktif di DB
  (fail-safe ke perilaku lama).
- Bridge env `INGEST_LLM_* → OPENAI_*` yang ada di satu titik (log WO-01a)
  diperluas menjadi "resolve provider config" satu titik: env → DB (lewat Go).
- Tidak ada perubahan protokol AG-UI; tambalan `provider-compat.ts` tetap
  berlaku (masih per-request).

### C. UI admin

- Halaman pengaturan baru di grup "Ingest" (`/ingest/providers`): daftar
  provider + bentuk tambah/edit (key ditulis hanya saat set/ubah; tampilan
  mask), tombol "Aktifkan" dan "Tes koneksi" (tampil hasil inline).
- Peringatan keamanan di UI: key tersimpan terenkripsi; tidak bisa dilihat
  kembali — hanya diganti.

### D. Keamanan & aturan

- Key plaintext hanya ada: di DB terenkripsi, di memori Go saat dekripsi, dan
  di memori Node saat pemanggilan provider. Tidak di log, tidak di audit, tidak
  di respons klien (mask saja).
- Fail-closed: key enkripsi kosong → CRUD provider ditolak 503 (jangan menyimpan
  plaintext).
- Rate/kondisi: ganti provider aktif berlaku untuk run BERIKUTNYA; run yang
  sedang berjalan tidak diputus.

## Acceptance Criteria

- AC-1 Admin menambah provider via UI (base URL + key + model) → tersimpan
  terenkripsi; daftar menampilkan mask, bukan plaintext.
- AC-2 Aktivasi provider → chat BERIKUTNYA memakai provider baru (bukti: model/
  header respons berubah) TANPA restart container.
- AC-3 Tanpa provider aktif di DB → fallback env `INGEST_LLM_*` (perilaku lama
  tidak rusak — regresi nol).
- AC-4 "Tes koneksi" menunjukkan sukses/gagal nyata (bukan stub) tanpa
  membocorkan key.
- AC-5 Key plaintext tidak muncul di: respons API, log, audit, DB (hanya
  ciphertext); uji: query DB langsung + grep log.
- AC-6 RBAC: viewer tanpa `ingest.write` → 403 CRUD; tanpa token 401; audit
  terisi semua aksi; endpoint internal `providers/active` menolak kredensial
  admin biasa (khusus runtime).
- AC-7 `go build`/`vet`/`test` hijau; `tsc`/`npm run build` hijau; swagger +
  Bruno (CRUD, activate, test, negatif 401/403/503); Catatan Implementasi (Dev)
  sesuai aturan 2026-11-06.

## Non-tujuan

- Multi-provider aktif bersamaan / pemilihan provider per sesi-chat (kutandai
  sebagai kemungkinan lanjutan, butuh keputusan terpisah).
- Provider non-OpenAI-compatible.
- Mengubah D-11 (key tetap tidak pernah ke klien; kebijakan tetap di Go).

## Keputusan Owner & Verifikasi (2026-11-09 — handoff final)

1. **APPROVED — key env→DB**: owner setuju dengan pergeseran sumber key dari env API key terenkripsi di DB (Go menjaga key; node menerima konfig via endpoint internal server-to-server). Prinsip inti D-11 tetap: key tidak pernah sampai ke klien; hanya berubahnya tempat penyimpanan (DB terenkripsi) dan mekanisme distribusi (fetch per request, bukan env statis).
2. **Verifikasi CopilotKit (Manager, 2026-11-09)**: CopilotKit TIDAK menyediakan management API bawaan untuk provider LLM (tidak ada CRUD base URL/key/model; dashboard Intelligence hanya mengelola memory/learning/threads, bukan kredensial LLM). Pola yang didukung: env, atau config object per-agent. BuiltInAgent menerima `apiKey` langsung di config — pas dengan pola `createAgent` per-request (WO-01a). Kesimpulan: fitur ini WAJIB dibangun mandiri (Go = management API; Node hanya konsumen per request) — sesuai §A/§B di atas; TIDAK menyalahi CopilotKit.
3. **Arsitektur final ditegaskan**: Go menyimpan API key (terenkripsi) & mengechild<|end_of_box|> PLAINTEXT ke Node hanya via endpoint internal; Node tetap satu-satunya pemanggil LLM. Fall-back env dipertahankan (backward compat).
4. WO ini **STATUS: siap eksekusi — handoff ke Dev Agent (2026-11-09)**; manager TIDAK menyentuh source code; semua keputusan desain di dok ini dianggap final.

## Risiko & catatan

- **Key di DB vs env**: ini pergeseran dari D-11 asli ("key hanya di Node env").
  Tetap setia pada PRINSIP D-11 (key tidak pernah ke klien; Go = penjaga
  kebijakan) — key kini transit Go (enkripsi/dekripsi) ke Node per request.
  **Butuh persetujuan owner secara eksplisit di catatan ini** sebelum eksekusi.
- Rotasi key enkripsi DB (kalau `INGEST_DB_KEY_ENC` diganti) = semua provider
  harus di-set ulang — dokumentasikan.
- Runtime Node meng-cache konfigurasi pendek (TTL kecil, mis. 30s) agar tiap
  run tidak memukul Go — sebutkan trade-off: perubahan provider berlaku ≤ TTL.
- Kegagalan provider aktif → error tool/chat jelas ("provider aktif gagal,
  cek pengaturan"), jangan fallback diam-diam ke env (justru membingungkan).

## Catatan Implementasi (Dev)

<!-- WAJIB diisi bertahap per bagian (§A, §B, §C, §D) sesuai aturan 2026-11-06:
     entri = `berkas:baris kunci | yang dilakukan | kenapa (ref D-x/O-x/gotcha)`;
     ±10–20 entri total, hanya keputusan/gate/fail-closed/jebakan — bukan diary. -->

### §A Backend (plugin `ingest`)

_(belum ada — diisi Dev saat §A selesai)_

### §B Runtime Node (`ingest_runtime`)

_(belum ada — diisi Dev saat §B selesai)_

### §C UI admin

_(belum ada — diisi Dev saat §C selesai)_

### §D Keamanan & aturan / lintas bagian

_(belum ada — diisi Dev saat bagian lintas selesai)_
