---
title: spec
description: "SPEC draft — plugin ingest: sesi chat learn-together (agent strict) + mapping editor + ekstraksi deterministik"
status: draft
created: "2026-11-03"
---
# SPEC — Plugin `ingest`: Sesi Chat Learn-Together + Mapping Editor + Ekstraksi Deterministik

`status: draft` — MENUNGGU REVIEW OWNER sebelum `work-order.md` dibuat.

## 1. Tujuan

Mengurangi penggunaan CRUD admin langsung untuk insert data. Insert dilakukan lewat
**satu jalur**: sesi chat *learn-together* dengan **agent strict**, dikonfirmasi lewat
**mapping editor**, lalu diekstraksi melalui pipeline deterministik yang sudah ada
(validate → staging → import).

Pilot: modul **`visit`** (D-1).

## 2. Latar & rujukan

- Topic `1793096602507` (AI-ETL CopilotKit/ACP): keputusan D-1..D-9 di sana diadopsi/
  diputus ulang di sini. Topic lama TIDAK melanjutkan secara teknis.
- Aturan project: fitur = plugin di `plugins/<id>/` (internal/ hanya framework).
- Pipeline impor existing per plugin: `fields.yaml` = sumber kebenaran, staging
  2-fasa (`*_import_batches`/`*_import_rows`), `template_version` mismatch ditolak.
- Kebijakan PII topic business: D-10 (PII plaintext, ekspor masking) & D-18
  (`--no-pii`) → flag `is_encrypt` memperluas, tidak mengurangi (D-5).

## 3. Keputusan terkunci (owner)

| ID | Keputusan |
|----|-----------|
| D-1 | Pilot modul = `visit` |
| D-2 | LLM = API eksternal; kolom PII dideidentifikasi via `deidentify` (secret key lokal) SEBELUM payload dikirim; hasil dipetakan balik lokal |
| D-3 | JEV = adopsi POLA harness (classifier gate diimplement sendiri, antarmuka terabstraksi, simulasi deterministik saat dev); layanan TypeSafe = implementasi opsional kelak |
| D-4 | `deidentify` tetap dipakai (tipe kolom eksplisit dari mapping); Presidio TIDAK dulu |
| D-5 | Flag `is_encrypt` hanya boleh MENAMBAH proteksi, tidak bisa lebih longgar dari policy `fields.yaml` — fail-closed |
| D-6 | Semua tool tulis agent melewati RBAC + audit yang sama dengan HTTP handler |
| D-7 | Agent STRICT — 5 gerbang (identitas, mapping, aturan, konfirmasi, fail-closed) ditegakkan via system prompt + state machine backend |
| D-8 | Mapping editor dibuat sendiri di React (tanpa library eksternal) |
| D-9 | `historic save` perilaku baru: auto-commit (catatan proses, bukan keputusan produk) |

## 4. Konsep alur (state machine sesi)

```
uploaded → introduced → mapping_draft → mapping_confirmed → extracting → extracted
                ↑__________|
             (clarify: tanya-jawab, aturan, pengecualian)
```

1. **uploaded** — user upload file (dari chat ATAU langsung) → tersimpan di backend,
   sesi dibuat.
2. **introduced** — agent membaca header/sample, "memperkenalkan diri" dengan dokumen
   (tanpa ekstraksi), mengajukan pertanyaan identitas bila belum jelas.
3. **mapping_draft** — agent menghasilkan tebakan mapping awal (berlabel "belum
   dikonfirmasi") + link mapping editor. Fase clarify: arti kolom, aturan, pengecualian.
4. **mapping_confirmed** — user menyetujui profil di editor (atau di chat). Titik komit.
5. **extracting** — data dimapping kolom-demi-kolom, kolom `is_encrypt` dideidentifikasi,
   lalu masuk validate → staging → import plugin target.
6. **extracted** — laporan hasil (created/updated/skipped/rejected) dilaporkan di chat.

Transisi hanya lewat action API yang bergantung pada state (backend menegakkan; agent
 tidak bisa "melompati" gate).

## 5. Agent strict — kontrak (system prompt + enforcement)

System prompt agent WAJIB memuat:

1. Kontrak learn-together: tidak mengekstrak tanpa konsens; selalu mulai introduksi.
2. Kontrak strict: tidak menebak; kolom tak dijelaskan → `skip`; pengecualian hanya
   dari pernyataan eksplisit user; konfirmasi wajib; tanpa mode paksa.
3. Kontrak dispatcher: setelah tebakan mapping siap, sertakan link mapping editor.
4. Kontrak batas wewenang: agent penerjemah/penyusun profil; kebenaran dari user;
   validasi oleh pipeline deterministik.
5. Kontrak output terstruktur: keputusan sesi ditulis ke dokumen profil (terstruktur),
   bukan hanya narasi.

Defense in depth: pelanggaran prompt tetap tertahan oleh gate state machine backend.

## 6. Komponen

### 6.1 Plugin `plugins/ingest` (usulan O-5 — default)

- Model: `ingest_sessions`, `ingest_messages`, `ingest_profiles` (dokumen profil:
  mapping kolom, tipe, flag `is_encrypt`, aturan/pengecualian user, versi), dan file
  upload tersimpan via `internal/storage`.
- Service: manajemen sesi + state machine; integrasi classifier gate; integrasi
  deidentify; pemanggilan pipeline validate/staging/import plugin target via service
  layer plugin tersebut (bukan SQL lintas plugin — prinsip isolation).
- Classifier gate (pola Jev): antarmuka `DecisionClassifier` (noul/choice/score) +
  implementasi `deterministic` (dev/offline) + `typesafe` (opsional, kelak).
- LLM client: antarmuka terabstraksi; implementasi pertama = provider API eksternal
  (env key); SELALU menerima payload yang sudah dideidentifikasi.

### 6.2 Mapping editor (UI)

- Halaman `sub_app/webapp`: `/ingest` (daftar sesi) + `/ingest/:id/map` (editor).
- Kiri = kolom Excel/CSV terdeteksi (+ sample 5 baris), kanan = target field dari
  `fields.yaml` plugin target, dropdown, flag `is_encrypt`, penanda `skip`.
- Tombol "Ekstrak" terkunci sampai semua gate lolos (kolom mapped/skip, aturan
  dijawab, konfirmasi).
- Embed di admin panel, grup nav baru (O-6) dengan guard `ingest.read/write`.

### 6.3 Deidentifikasi

- `deidentify` (Go) dipanggil per kolom ber-flag `is_encrypt` atau yang policy wajib.
- Tipe kolom datang EKSPLISIT dari mapping (tidak bergantung inferensi regex AS).
- Secret key dari env (AES-GCM style seperti token endpoint; wajib ada, fail-closed
  bila kosong dan ada kolom yang butuh).

### 6.4 Chat UI

- Panel chat di admin panel (embed, O-6); sesi persist di DB (O-7) — bisa dilanjutkan.
- Upload dari chat = auto-upload ke backend + link editor dalam balasan agent.

## 7. Keamanan & batasan

- Semua endpoint & tool guard permission `ingest.*` (seed via migrasi plugin `auth`,
  pola resident/hsh/sec).
- RBAC target modul tetap berlaku: ekstraksi ke `visit` menuntut `visit.write`.
- Audit: setiap aksi agent yang menyentuh state sesi/ekstraksi masuk audit log.
- LLM API eksternal HANYA menerima payload dideidentifikasi; tidak ada PII keluar.
- `.xls` (BIFF8) tetap ditolak (N-17); XLSX/CSV saja.

## 8. Keputusan yang DIUSULKAN Manager (menunggu review owner)

| ID | Usulan | Alasan |
|----|--------|--------|
| O-3 | Chat khusus ingest dulu; chat umum ke data = fase lanjutan | Scope ketat, cepat jadi |
| O-4 | Tool-calling internal ke service layer; MCP ditunda | Cukup untuk pilot |
| O-5 | Nama plugin `ingest` | Satu plugin utk sesi+editor+ekstraksi |
| O-6 | Embed di admin panel, grup nav baru | Konsisten pola UI existing |
| O-7 | Sesi persist di DB | Sesi belajar bisa lama |
| O-9 | Editor bisa dibuka tanpa chat (chat opsional) | Upload manual tetap hidup |

## 9. Rencana Work Order (usulan, draft — final menyusul setelah review)

- **WO-01** Skeleton plugin `ingest` + migrasi tabel sesi/profil + seed permission
  `ingest.*` via plugin auth.
- **WO-02** Sesi & state machine (`uploaded→…→extracted`) + API sesi + audit.
- **WO-03** Upload file + parsing header/sample (CSV/XLSX) + penyimpanan.
- **WO-04** LLM client (abstraksi) + system prompt agent + tebakan mapping awal +
  deidentify pra-payload (D-2/D-4) + secret key env fail-closed.
- **WO-05** Classifier gate (pola Jev, antarmuka + implementasi deterministic) —
  gate tool-call & klasifikasi clarify (D-3).
- **WO-06** Ekstraksi: mapping terkonfirmasi → validate → staging → import plugin
  `visit` (pilot) + laporan hasil.
- **WO-07** Mapping editor UI + daftar sesi + flag `is_encrypt` (D-5/D-8).
- **WO-08** Chat UI embed + persistensi sesi (O-6/O-7).
- **WO-09** Swagger + README + koleksi Bruno + RBAC e2e.

## 10. Pertanyaan terbuka untuk review owner

1. Setujui usulan O-3..O-9 (bagian 8)? Terutama nama plugin `ingest` dan MCP ditunda.
2. Reusabilitas dokumen profil lintas-modul: profil visit yang disimpan boleh
   disarankan untuk ekspor visit berikutnya — cukup, atau perlu versi/modul binding
   lebih ketat?
3. Batas ukuran file upload & retensi file mentah di storage (hapus setelah extracted?).
4. Model LLM pertama yang dipakai (OpenAI/Claude/ lain) & env key-nya — keputusan
   owner (nama layanan di README boleh, tapi implementasi tetap terabstraksi).
