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

### Tambahan owner (2026-11-03): MAPPING EDITOR sebagai UI konfirmasi

Alur diperkaya: upload langsung masuk backend (auto-upload), lalu LLM membalas dengan
**link ke mapping editor** — halaman visual khusus untuk menyunting mapping kolom
(source column → target field). User tidak menuliskan pemetaan lewat teks di chat;
chat menjadi **dispatcher** yang mengarahkan ke editor.

Alur final:

1. User upload Excel/CSV → langsung tersimpan di backend (sesi ingest dibuat).
2. Agent membaca header/sample, menyusun **tebakan mapping awal** (diberi label
   "belum dikonfirmasi" — tidak ada yang lolos tanpa aksi user).
3. Agent membalas di chat dengan **link ke mapping editor**
   (mis. `/ingest/sessions/:id/map`).
4. User mengoreksi mapping di editor (kolom kiri = kolom Excel terdeteksi, kolom kanan
   = target field dari `fields.yaml`, dropdown + preview ±5 baris).
5. Konfirmasi di editor → ekstraksi via pipeline validate → staging → import.

Keputusan teknis mapping editor (analisa Manager, 2026-11-03):

- **TIDAK mengadopsi library eksternal** (react-data-mapper, Handsontable, dll) —
  semuanya berat/keberatan dan sangat konteks-spesifik. Editor dibuat sendiri di React:
  komponen kecil, dan semua sumber kebenaran sudah ada (`schema/fields.yaml`,
  `schema.Template.Columns`).
- Target field di editor = otomatis dari `fields.yaml` per plugin.
- Prinsip strict tetap berlaku di editor: kolom tanpa makna = `skip`; pengecualian
  harus ditulis user; tombol "Ekstrak" terkunci sampai semua gate lolos.
- Chat + editor = satu pasangan: chat untuk sesi belajar (pertanyaan bebas, aturan,
  pengecualian), editor untuk koreksi mapping (terstruktur).

### Sistem Prompt Agent (kontrak perilaku — diputuskan owner)

**Intinya: agent ini DIBERI SYSTEM PROMPT agar mendukung semua solusi di atas.** Strictness,
prinsip learn-together, dan perilaku "chat sebagai dispatcher ke mapping editor" DITEGAKKAN
LEWAT SYSTEM PROMPT agent, bukan hanya kode. Artinya system prompt agent wajib memuat
minimal:

1. Kontrak learn-together: jangan pernah mengekstrak tanpa konsens; selalu mulai dari
   introduksi dokumen.
2. Kontrak strict (5 gerbang di atas): tidak menebak; kolom tak dijelaskan → `skip`;
   pengecualian hanya dari pernyataan user; konfirmasi wajib; fail-closed tanpa mode paksa.
3. Kontrak dispatcher: setelah upload & tebakan mapping siap, agent WAJIB menyertakan
   link mapping editor — bukan memaksa user menuliskan mapping di chat.
4. Kontrak batas wewenang: agent hanya penerjemah/penyusun profil; kebenaran akhir dari
   user; validasi tetap oleh pipeline deterministik.
5. Kontrak output terstruktur: setiap keputusan sesi ditulis ke dokumen profil dalam
   bentuk terstruktur (state machine), bukan hanya narasi percakapan.

Catatan desain: karena perilaku juga ditopang state machine backend, system prompt adalah
lapisan kedua — pelanggaran prompt tetap tertahan oleh gate backend (defense in depth).

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
  backend: `uploaded → introduced → mapping_draft → mapping_confirmed → extracting →
  extracted`), tidak bergantung perkataan model. Lapisan kedua: system prompt agent
  (lihat bagian "Sistem Prompt Agent").
- AI tidak pernah jadi penentu kebenaran: hanya penerjemah yang dikoreksi bersama.
  Validasi tetap deterministik via `fields.yaml` + staging 2-fasa yang sudah ada
  (`*_import_batches`/`*_import_rows`).
- Relevansi topic `1793096602507` (AI-ETL CopilotKit/ACP agent): keputusan D-1..D-9 yang
  masih terbuka di sana akan diadopsi/diputus ulang di topic ini.

## Komponen rencana (owner, 2026-11-03)

Tiga komponen yang disiapkan setelah konsep dasar:

### 1. `deidentify` (deterministik) — github.com/aliengiraffe/deidentify

Library **Go** open-source untuk deidentifikasi PII. Fakta dari repo (diverifikasi
Manager 2026-11-03):

- Replacement **deterministik**: input sama + secret key sama → output sama (HMAC-SHA256,
  `deterministicHash`) → **referential integrity terjaga** antar record/run; join &
  grouping tetap bekerja di data yang sudah dianonimkan.
- Format-preserving per tipe: `TypeName`, `TypeEmail`, `TypePhone`, `TypeSSN`,
  `TypeCreditCard`, `TypeAddress`, `TypeGeneric` (generic = nilai diteruskan apa adanya).
- API: `Text` (teks bebas), `Table` (kolom terstruktur dengan tipe per kolom),
  `Slices` (CSV-like `[][]string`, tipe kolom bisa di-infer).
- Kolom name dipakai sebagai konteks → nilai identik di kolom beda dapat pengganti beda
  (anti-korelasi). Thread-safe.

**Peran di arsitektur kita (usulan Manager)**: lapisan privasi SEBELUM data dikirim ke
LLM — terutama jika O-2 diputuskan LLM API eksternal. Alur: ekstrak kolom PII
(nama bayi/ibu, NIK, phone) → deidentify dgn secret key lokal → payload yang dikirim ke
LLM hanya berisi nilai anonim → hasil tebakan LLM dipetakan balik secara lokal
(deterministik, jadi re-identifikasi bisa dilakukan). Catatan risiko untuk SPEC:
pattern library-nya condong ke format AS (SSN/alamat US), nama Indonesia tidak dikenali
oleh `TypeName` regex → perlu evaluasi kolom mana yang benar-benar di-deidentify dan
tipe yang dipakai.

### 2. Excel mapping editor

Sudah dicatat di bagian "Tambahan owner: MAPPING EDITOR" di atas — dikonfirmasi sebagai
komponen yang disiapkan (dibuat sendiri di React, bukan library eksternal).

### 3. JEV model (TypeSafe AI) — TERKLARIFIKASI (2026-11-03)

Sumber: blog LangChain "Building a Harness with Jev" (link dari owner).

**JEV = model "System One" dari TypeSafe AI** — BUKAN LLM teks. Ia tidak menghasilkan
teks; ia **mengevaluasi keadaan (state) dan mengembalikan jawaban TERTYPE + PROBABILITAS**.
Dilatih dengan RLCD (reinforcement learning for calibrated decisions). Klaim vendor:
inferensi ±200× lebih cepat, biaya ±400× lebih murah dari LLM untuk tugas klasifikasi.

Cara kerja: kirim `state` + satu/lebih `questions` dalam SATU permintaan (dievaluasi
paralel). Tiga jenis pertanyaan:

- **Choice** — pilih dari opsi, kembalikan probabilitas per opsi + skor kepercayaan.
- **Score** — rating kontinu terurut (rendah/sedang/tinggi) + distribusi + confidence.
- **Noul** — ya/tidak, kembalikan probabilitas kebenaran pernyataan.

Integrasi LangChain: `TypeSafeClassifier` (paket `langchain-typesafe`, env
`TYPESAFE_API_KEY`). Use case di blog: model routing (pilih model murah vs kuat) dan
**Auto Mode** — guardrail yang menilai tool call berisiko dan MEMBLOK sebelum dieksekusi.

**Makna untuk arsitektur kita (owner: "bisa di set kondisi")**: JEV = lapisan
**keputusan terstruktur/harness** di jalur insert-via-LLM — bukan penghasil konten.
Pemakaian yang disarankan di SPEC:

1. **Gate tool-call**: sebelum tool insert dieksekusi, tanya JEV (noul): "apakah payload
   ini konsisten dengan mapping yang dikonfirmasi?" / "apakah ini aksi berisiko?" →
   skor/probabilitas jadi kondisi keputusan (set-able oleh owner).
2. **Klasifikasi baris/kolom ambigu** di fase clarify (choice: "kolom ini termasuk
   kategori apa?") — jauh lebih murah dari full-LLM per keputusan.
3. **Routing**: keputusan sepele → Jev; penalaran terbuka (menyusun pertanyaan clarify,
   menghasilkan tebakan mapping) → tetap LLM penuh.

Catatan risiko: produk baru (TypeSafe, dirilis ±2026-09, tergantung API eksternal
`TYPESAFE_API_KEY`) → di SPEC perlu abstraksi antarmuka classifier agar bisa diganti
(atau di-simulasi deterministik saat dev/offline).

### Evaluasi alternatif `deidentify` (riset Manager, 2026-11-03)

Konteks: kekhawatiran owner — pattern `deidentify` condong ke format AS.
Kandidat yang diperiksa:

1. **Presidio** (microsoft → kini data-privacy-stack; Python, 11.7k★, aktif) — framework
   PII paling lengkap: NLP + regex + checksum, multi-bahasa (ada recognizer Jerman dll),
   recognizer custom bisa ditambahkan untuk nama/NIK Indonesia, ada `presidio-structured`.
   **Kons: Python** — container `app` kita Go-only (aturan N-17: image tetap Go-only);
   memakai Presidio berarti sidecar container Python terpisah (kompleksitas deploy naik).
2. **ferret-scan** (awslabs; Go, aktif) — CLI + library Go, deteksi PII/secrets offline,
   confidence context-aware, redaksi format-preserving, "nothing leaves your host".
   Lebih lengkap dari deidentify utk deteksi, namun fokusnya secrets/leak-scan, bukan
   deidentifikasi terstruktur dengan referential-integrity seperti kasus kita.
3. **privacy-filter** (Go, LLM privacy gateway) & **kiji-proxy** (Go) — fokus redaksi
   request LLM secara umum, tidak punya model mapping kolom-typed yang kita butuhkan.
4. **arcjet-js / top_secret (Ruby) / Octopii** — bahasa/pola tidak pas dengan stack Go.

**Kesimpulan Manager (usulan)**: TETAP pakai `deidentify` sebagai basis deterministik
(Go-native, referential integrity, pas pola Table/Slices), DENGAN mitigasi:

- Karena **mapping editor memberi kita tipe kolom secara eksplisit**, kita TIDAK
  tergantung inferensi regex AS-nya — panggil metode per tipe (`Name`/`Phone`) berdasar
  hasil mapping yang dikonfirmasi user.
- Nilai yang tidak dikenal pola tetap bisa dipaksa lewat tipe (mis. seluruh kolom nama
  di-`TypeName` walau regex tak mengenalinya) — replacement tetap deterministik dari
  HMAC secret key.
- Bila butuh NER nyata multi-bahasa (nama Indonesia bebas-teks), barulah pertimbangkan
  **Presidio sebagai sidecar Python opsional** — keputusan diperlukan saat SPEC (O-11).

## Keputusan terkunci

- (belum ada yang formal — owner meminta respons terkini DICATAT DULU (2026-11-03);
  draf SPEC menunggu jawaban pertanyaan di bawah)

### Snapshot status diskusi (2026-11-03, catatan Manager)

Pertanyaan menunggu keputusan owner sebelum draf SPEC:

1. **O-10** — Jev: pakai layanan TypeSafe asli (API eksternal + `TYPESAFE_API_KEY`)
   ATAU cukup adopsi POLA harness-nya (classifier gate diimplement sendiri, abstraksi
   antarmuka, bisa disimulasi deterministik offline)?
2. **O-11** — Presidio sebagai sidecar Python opsional (NER nama Indonesia): default
   usulan Manager **TIDAK dulu** (Go-only terjaga); `deidentify` cukup karena tipe kolom
   datang eksplisit dari mapping editor.
3. **O-1** — pilot modul: usulan Manager `visit`.
4. **O-2** — LLM API eksternal vs lokal (relevan dengan deidentify sebagai lapisan
   privasi pra-LLM).
5. Sisa O-3..O-9 (scope chat, MCP vs tool internal, nama plugin, UI, persistensi sesi,
   RBAC/audit, editor tanpa chat).

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
- O-6: Bentuk UI chat + mapping editor: embed di admin panel (`sub_app/webapp`, grup nav
  baru mis. "Ingest") vs halaman tersendiri.
- O-7: Persistensi sesi: simpan session state chat di DB (bisa lanjut bila tab ditutup)?
- O-8: Keamanan: semua tool tulis wajib melewati RBAC + audit yang sama dengan HTTP handler
  (tidak ada jalur belakang). Ini diasumsikan wajib — perlu konfirmasi owner.
- O-9: Apakah user bisa langsung ke mapping editor TANPA sesi chat (chat opsional),
  atau editor selalu diluncurkan dari chat?
- O-10: **Makna "JEV model LLM"** — TERKLARIFIKASI (lihat bagian 3 di atas):
  Jev = model System-One TypeSafe AI untuk keputusan terstruktur (noul/choice/score).
  Terbuka: apakah kita memang akan memakai layanan TypeSafe (API eksternal + API key)
  atau hanya mengadopsi POLA harness-nya (classifier gate bisa diimplement sendiri)?
- O-11: Presidio sebagai sidecar Python opsional (untuk NER nama Indonesia) — butuh
  atau tidak? Default usulan Manager: TIDAK dulu (Go-only tetap terjaga); deidentify
  dipakai dengan tipe kolom eksplisit dari mapping editor.
