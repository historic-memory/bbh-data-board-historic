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
- **Fitur tambahan (ide owner, 2026-11-03): flag `is_encrypt` per kolom mapping.**
  User bisa mencentang kolom untuk dipaksa terenkripsi/dideidentifikasi ketika diproses —
  tidak bergantung deteksi otomatis. Implementasi yang disarankan: flag tersimpan di
  dokumen profil (bagian dari mapping terkonfirmasi), saat ekstraksi kolom ber-flag
  diproses lewat `deidentify` (HMAC deterministik) sebelum payload dibentuk; kolom
  yang TIDAK di-flag tetap mematuhi kebijakan default per plugin (mis. PII kolom
  wajib dari `fields.yaml`, lihat D-10/D-18 topic business: `--no-pii`).
  Manfaat: pilihan privasi per-ekspor, reusable via dokumen profil, dan bisa ditindak
  sebelum payload LLM (O-2) — konsisten dengan prinsip deidentify dulu baru keluar
  data. Perlu keputusan SPEC: hubungannya dengan kebijakan PII wajib per field
  (flag user bisa lebih ketat, TIDAK bisa lebih longgar dari policy bawaan — fail-closed).

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

Owner setuju dengan seluruh saran Manager (2026-11-03: "ok aku ikut saranmu"):

- **D-1 PILOT = `visit`** (O-1) — data mentah paling rumit, ujian terbaik agent strict.
- **D-2 LLM = API eksternal** (O-2) — dengan syarat lapisan privasi: kolom PII
  dideidentifikasi via `deidentify` (secret key lokal) SEBELUM payload dikirim ke
  provider; hasil dipetakan balik lokal (deterministik).
- **D-3 JEV = adopsi POLA harness, bukan layanan TypeSafe asli** (O-10) — classifier
  gate diimplement sendiri dengan antarmuka terabstraksi (bisa disimulasikan
  deterministik saat dev/offline; bila kelak mau, provider TypeSafe bisa ditambah
  sebagai implementasi antarmuka yang sama).
- **D-4 deidentify TETAP dipakai, Presidio TIDAK dulu** (O-11) — tipe kolom eksplisit
  dari mapping editor menghilangkan ketergantungan pada inferensi regex AS;
  Presidio/sidecar Python hanya opsi lanjut bila kelak butuh NER bebas-teks.
- **D-5 flag `is_encrypt` hanya boleh MENAMBAH proteksi** (O-12) — kolom yang wajib
  di-PII oleh `fields.yaml` tidak bisa dilonggarkan oleh user; fail-closed.
- **D-6 RBAC + audit WAJIB** (O-8) — semua tool tulis agent melewati RBAC dan audit
  yang sama dengan HTTP handler; tidak ada jalur belakang.
- **D-10 Penempatan & desain UI chat (owner, 2026-11-04)** — chat di admin panel
  sebagai **halaman khusus (dedicated page)**: grup nav "Ingest" → `/ingest` (daftar
  sesi) + `/ingest/:id/chat` (halaman chat). Design kit STANDAR: session list +
  kotak pesan chat (bubble, input, streaming). CopilotKit di-mount per halaman,
  BUKAN di `AdminLayout` (sandbox terisolasi). Bukan chat global melayang; chat
  ter-embed di halaman domain = kandidat CP-4/lanjutan. Gaya visual: komponen
  `components/ui/*` + Tailwind v4; default CopilotKit tanpa skinning di sandbox
  (polish CP-4). Tercatat di `wo-01a-proxy-chat-hidup.md` (AC-9).

### Snapshot status diskusi (2026-11-03, catatan Manager)

Keputusan O-1/O-2/O-8/O-10/O-11/O-12 telah DIKUNCI (lihat bagian "Keputusan terkunci").
Sisa yang perlu diputuskan saat penyusunan SPEC (akan diusulkan Manager dengan default,
dapat diubah owner saat review):

1. **O-3** — scope chat: usulan = chat khusus ingest dokumen dulu (scope ketat);
   chat umum ke data = fase lanjutan.
2. **O-4** — tool-calling internal ke service layer cukup untuk pilot; MCP ditunda
   sampai kebutuhan agent eksternal nyata muncul.
3. **O-5** — nama plugin: usulan `plugins/ingest` (mencakup sesi chat + mapping editor
   + ekstraksi; komponen classifier di dalamnya).
4. **O-6** — UI: embed di admin panel (`sub_app/webapp`), grup nav baru (mis. "Ingest").
5. **O-7** — persistensi sesi: simpan state sesi di DB (bisa dilanjutkan bila tab
   ditutup) — wajib karena sesi belajar bisa berlangsung lama.
6. **O-9** — mapping editor bisa dibuka langsung tanpa chat (chat opsional) — upload
   manual tetap tersedia; sesi tetap lewat state machine yang sama.

Draf SPEC akan menyertakan usulan keputusan untuk O-3..O-9 di atas.

## Pertanyaan terbuka

- **O-17 (BARU, dari spike WO-01a tim dev, 2026-11-04): strategi adapter CopilotKit
  runtime.** Temuan spike: CopilotKit v2 = protokol AG-UI + runtime Node WAJIB —
  proxy Go OpenAI-compatible saja TIDAK cukup untuk jadi backend CopilotKit. Opsi:
  (A) implement protokol AG-UI di proxy Go (sesuai prinsip: key+gate+audit 1 titik
  di Go; effort = protokol baru di Go); (B) standalone Node runtime `@copilotkit/runtime`
  (dipakai di spike dev: `runtime/copilot-runtime.ts`, `npm run dev:runtime`,
  basePath `/api/copilotkit`, Vite proxy) — cepat tapi service kedua + key/strictness
  bergeser keluar Go (jangan di-deploy produksi); (C) buang CopilotKit, pakai headless
  `openai` client. Temuan lain spike: `/threads` 403 tanpa `identifyUser` (JWT via
  headers, atau `threads.access: "open"` untuk sandbox); telemetri CopilotKit aktif
  (bisa dimatikan via `COPILOTKIT_TELEMETRY_DISABLED=true`). Sesi UI masih stub
  `localStorage` (`lib/ingestSessions.ts`) — WAJIB diganti API `ingest_messages`
  saat backend jadi. Kode spike BELUM di-commit (menunggu review tim). Build hijau,
  chunk chat terpisah (lazy + Suspense), bundle utama 1,4 MB, tanpa URL backend.
  **TERPUTUSKAN & DIREVISI (owner, 2026-11-04 → D-11 FINAL): opsi (B) DIPERLUAS —
  runtime Node CopilotKit pegang SELURUH jalur chat AI (pemanggil provider LLM);
  proxy OpenAI-compatible di Go DIHAPUS dari jalur chat. Go (bbh app) menjadi
  DATA SERVICE murni.** Syarat wajib yang menjaga keputusan lain tetap utuh:
  1. API key hanya di Node runtime — tidak pernah keluar ke klien.
  2. Semua keputusan bertanggung jawab TETAP di Go: tool calls (mapping/confirm/
     extract) → API plugin `ingest` Go (RBAC + audit + state machine); Node TIDAK
     pernah menulis profil.
  3. Gate deidentify tetap di Go — Node meminta Go "menyiapkan payload aman"
     (deidentify + secret key tetap Go), TIDAK direplikasi di Node.
  4. Audit: Node mengirim event audit ke endpoint Go (`ingest_audit` tetap pusat).
  Implikasi: N-17 dilonggarkan permanen (service Node produksi); aturan pinned
  "API key tidak pernah keluar dari backend Go" ditulis ulang menjadi "tidak pernah
  keluar ke klien; dipegang runtime Node". Revisi SPEC/techstack/WO-01a dieksekusi
  2026-11-04 (dok ini).**

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
- **O-18 (BARU, 2026-11-04): compact/summarize context window — TERPUTUSKAN (owner
  ikut rekomendasi Manager): BUAT SENDIRI di runtime Node, TANPA library harness
  eksternal.** Latar: user minta perilaku compact ~70% seperti GitHub Copilot.
  Riset Manager: CopilotKit TIDAK punya fitur compact publik (hanya internal runner
  v2 — test suite menyebut "compaction" thread history, tanpa API publik); library
  npm (`@deepagents/compaction`, `@aparte/plugin-compaction`, dst.) semuanya
  plugin harness/framework asing — tidak pas untuk runtime kita. Desain yang
  disetujui (implementasi di `copilot-runtime.ts`):
  1. Ukur token kumulatif per thread dari `usage` response provider.
  2. Threshold (default 70%) dari env `INGEST_LLM_CONTEXT_LIMIT` → compact
     sebelum request berikutnya; fail-closed bila limit tidak di-set (default
     konservatif).
  3. Ringkas pesan lama (kecuali N pesan terakhir) via LLM (model sama atau
     `INGEST_LLM_SUMMARY_MODEL` opsional).
  4. Summary disimpan sebagai pesan tipe khusus di `ingest_messages` VIA GO
     (Node tidak menulis DB langsung — D-11); compact tercatat di `ingest_audit`.
  5. **Keputusan terkunci TIDAK pernah ikut diringkas**: mapping terkonfirmasi,
     aturan, pengecualian, status gate diambil dari dokumen profil di Go
     (source of truth) dan disuntikkan ulang tiap request — summary hanya konteks
     naratif. Agent strict tetap strict walau riwayat ter-compact.
  6. Prompt berikutnya = system prompt + profil Go terkini + summary + N pesan
     terakhir.
  Sesi (session/thread) juga terekam eksplisit: sumber kebenaran = GO
  (`ingest_sessions`/`ingest_messages` — sudah dibuktikan WO-01a, commit
  `a22c03b`); thread cache di Node runtime hanyalah transport, boleh hilang saat
  restart. **Timing implementasi: CP-2** (setelah CP-1 selesai — sesi real case
  `visit` baru di sana yang bisa panjang); akan masuk rancangan WO-02.
  - O-11: Presidio sebagai sidecar Python opsional (untuk NER nama Indonesia) — butuh
  atau tidak? Default usulan Manager: TIDAK dulu (Go-only tetap terjaga); deidentify
  dipakai dengan tipe kolom eksplisit dari mapping editor.
- O-12: Kebijakan flag `is_encrypt`: apakah flag user boleh MENGURANGI proteksi bawaan
  (kolom PII wajib dari fields.yaml)? Usulan Manager: flag hanya boleh MENAMBAH
  proteksi (lebih ketat), tidak bisa lebih longgar — fail-closed.
