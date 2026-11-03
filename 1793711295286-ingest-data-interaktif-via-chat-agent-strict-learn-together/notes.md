---
title: notes
description: "Konteks diskusi & keputusan konsep ingest data interaktif via chat (agent strict, learn-together)"
status: draft
created: "2026-11-03"
---
# Ingest Data Interaktif via Chat — Agent Strict Learn-Together

## Konteks (diskusi owner + Manager, 2026-11-03)

Tujuan: mengurangi penggunaan CRUD admin langsung untuk insert data, dengan dua jalur
yang kemudian **digabung menjadi satu konsep**:

1. ~~Jalur dokumen: upload CSV/Excel dengan template "random tapi terstruktur" → butuh
   lapisan AI mapping dokumen-ke-skema canonical.~~
2. ~~Jalur chat: MCP tool-calling ke service layer.~~

### Konsep final (dipilih owner): FULL CHAT dengan "learn together"

Bukan sekali-upload-langsung-dikerjakan, melainkan:

1. **INTRODUKSI** — user upload dokumen di chat; agent TIDAK langsung mengekstrak.
   Agent hanya membaca header/sample baris dan "memperkenalkan diri" dengan dokumen
   (mis. "ini tampaknya ekspor Notion kunjungan").
2. **TANYA-JAWAB (clarify)** — agent + user menyusun pemahaman bersama: arti kolom,
   aturan data, pengecualian. Agent strict: tidak menebak.
3. **KONSENSUS** — agent menyusun *dokumen profil* (mapping kolom + aturan + pengecualian
   yang dinyatakan user). User menyetujui → titik komit.
4. **EKSTRAKSI** — baru data diambil, masuk pipeline validate → staging → import yang
   SUDAH ADA (gerbang tetap deterministik).

### Keputusan owner: AGENT STRICT

Agent yang diciptakan adalah **strict** — user HARUS menjelaskan dengan baik. Operasional
sebagai gerbang yang tidak bisa dilewati:

1. **Gate Identitas** — user menegaskan dokumen apa, modul mana (`resident`/`visit`/
   `hsh`/`sec`), dan isi datanya.
2. **Gate Mapping** — setiap kolom wajib punya makna dari user ATAU dikonfirmasi lewat
   pertanyaan agent. Kolom yang tidak bisa dijelaskan → ditandai `skip` (tidak diekstrak),
   bukan ditebak.
3. **Gate Aturan** — pengecualian validasi HANYALAH jika dinyatakan eksplisit oleh user;
   agent tidak pernah mengecualikan sendiri (fail-closed).
4. **Gate Konfirmasi** — ringkasan profil + preview ±5 baris hasil mapping; ekstraksi
   tidak jalan tanpa pernyataan setuju user.
5. **Fail-closed** — gate belum terpenuhi → aksi "ambil data" terkunci; tanpa mode "paksa".

## Insight penting yang disepakati

- **Dua jalur awal (dokumen + chat) menjadi SATU jalur** (full chat dengan contract
  learn → confirm → extract).
- Hasil sesi = **dokumen profil yang dapat disimpan & dipakai ulang** untuk ekspor format
  yang sama berikutnya (BBH mengekspor data yang sama berulang — sesi belajar pertama
  mahal/interaktif, selanjutnya otomatis).
- "Strictness" harus **deterministik** (conversation contract + state machine sesi di
  backend: `introduced → clarifying → profile_ready → confirmed → extracted`), tidak
  bergantung perkataan model.
- AI tidak pernah jadi penentu kebenaran: hanya penerjemah yang dikoreksi bersama.
  Validasi tetap deterministik via `fields.yaml` + staging 2-fasa yang sudah ada
  (`*_import_batches`/`*_import_rows`).
- Relevansi topic `1793096602507` (AI-ETL CopilotKit/ACP agent): keputusan D-1..D-9 yang
  masih terbuka di sana akan diadopsi/diputus ulang di topic ini.

## Keputusan terkunci

- (belum ada — topic baru dibuat)

## Pertanyaan terbuka

- O-1: Modul perintis (pilot) untuk sesi belajar pertama? Usulan Manager: `visit`
  (data mentah paling rumit — ujian terbaik agent strict; sumber: ekspor Notion kunjungan).
- O-2: LLM provider — API eksternal (OpenAI/Claude) vs lokal? Catatan: data PII bayi;
  D-10 topic business menyatakan PII plaintext, perlu pertimbangan dari sisi privacy
  LLM API.
- O-3: Cakupan chat: khusus ingest dokumen saja (scope ketat) vs sekalian chat umum ke data.
- O-4: Kebutuhan MCP: apakah tool-calling internal ke service layer cukup untuk pasangan
  ini, atau MCP tetap dipisah (mis. agent provider eksternal)?
- O-5: Penamaan plugin: usulan awal `plugins/ingest` (atau `plugins/ai`). Atau plugin agent
  dipisah dari jalur ekstraksi.
- O-6: Bentuk UI chat: embed di admin panel (`sub_app/webapp`) vs halaman tersendiri.
- O-7: Persistensi sesi: simpan session state chat di DB (bisa lanjut bila tab ditutup)?
- O-8: Keamanan: semua tool tulis wajib melewati RBAC + audit yang sama dengan HTTP handler
  (tidak ada jalur belakang). Ini diasumsikan wajib — perlu konfirmasi owner.
