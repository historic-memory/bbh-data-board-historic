---
title: notes
description: Konteks diskusi & keputusan konsep ingest data interaktif via chat (agent strict, learn-together)
status: draft
created: "2026-11-03"
updated: "2026-11-23"
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

- **O-14 (USULAN RESMI Manager, 2026-11-04, MENUNGGU KEPUTUSAN owner)** — retensi
  & cleanup sesi/file (dibutuhkan bagian D WO-04):
  1. Sesi non-`extracted` yang idle ≥ **90 hari** → dihapus bersama pesan & file-nya.
  2. File upload (key `ingest/uploads/<session>/...` di `internal/storage`) dihapus
     **30 hari** setelah ekstraksi sukses (`extracted`) — payload aman/deidentify
     ikut (tidak ada kebutuhan pasca-ekstraksi).
  3. Sesi `extracted`: **profil disimpan TAK TERBATAS** — dipakai ulang untuk ekspor
     format sama (inti learn-together; sesi belajar pertama mahal).
  4. Implementasi: job periodik (Go), env konfigurabel
     (`INGEST_RETENTION_SESSION_DAYS`, `INGEST_RETENTION_UPLOAD_DAYS`, 0 = disable),
     setiap penghapusan tercatat di `ingest_audit`.
- **O-13 (terjawab oleh praktik, 2026-11-04):** provider = gateway OpenAI-compatible
  (`donis-agent.rolldev.my.id/v1`, base URL & model via env `INGEST_LLM_*`) — sesuai
  usulan; pilihan provider bebas via env. KUTERIMA-TERJAWAB tanpa dokumen baru.
- **O-15 (MOOT, 2026-11-04):** proxy Go LLM DIHAPUS di WO-01c — pertanyaan cakupan
  proxy tidak relevan lagi.
- **O-16 (terjawab oleh praktik, 2026-11-04):** UI chat = **CopilotKit** (dipakai
  sejak CP-1, runtime Node D-11). KUTERIMA-TERJAWAB tanpa dokumen baru.
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
- **O-19 (BARU, 2026-11-04): pemisahan runtime CopilotKit ke `sub_app/ingest_runtime/`
  — TERPUTUSKAN (owner): PISAH, eksekusi AWAL CP-2 (sebelum WO-02 dieksekusi).**
  Latar: runtime kini menumpang di `sub_app/webapp/` (`runtime/*.ts` + deps
  CopilotKit di package.json yang sama) — menimbulkan kebingungan (owner: "biar
  junior paham juga"). Bentuk yang disetujui: **npm workspaces** di `sub_app/`
  — `sub_app/webapp` (SPA) + `sub_app/ingest_runtime` (service CopilotKit) sebagai
  package terpisah, SATU lockfile di root agar versi CopilotKit react-core/runtime
  tetap sinkron (protokol AG-UI sensitif versi). Konsekuensi: Dockerfile.runtime
  hanya meng-install deps workspace runtime (image lebih ramping); service compose
  `ingest-runtime` tidak berubah perilaku; D-11 tidak berubah (Node tetap tipis,
  kebijakan tetap Go). Termasuk housekeeping kecil: perbarui komentar header
  `docker-compose.yml` yang masih menulis "ingest-runtime IDLE, DEV-ONLY spike"
  (usang sejak D-11 final). Dipindah SEBELUM WO-02 agar CP-2 dimulai dari struktur
  yang final.
- **O-20 (BARU, 2026-11-04): skema `rules` terstruktur + penegakan & editor rules
  — TERPUTUSKAN (owner): SETUJU.** Latar: AC-4 (Gate Aturan) di CP-2 masih pasif —
  `rules` hanya tersimpan di `ingest_profiles` (jsonb) tanpa penegakan; owner juga
  menanyakan "design template rules yang bisa diedit". Klarifikasi Manager (disetujui
  owner): `rules` BUKAN validator CRUD — guard RBAC tetap di atasnya; rules =
  **pengecualian validasi data** untuk jalur ekstraksi ingest, dievaluasi di SATU
  titik sebelum `ImportTemplate` plugin target. Keputusan:
  1. **Skema rule terstruktur** (objek kecil per rule): `rule_id` (id stabil, cocok
     dengan penegak Go), `target` (plugin), `violates` (validasi pipeline yang
     dikecualikan, contoh `phone_e164.required`), `action` (`allow` | `skip`),
     `scope` (`row` | `field`), `stated_by`/`stated_at` (bukti Gate Aturan),
     `version` (ikut profil append-only).
  2. **LLM tidak boleh mengarang rule** — hanya memilih/memodifikasi dari daftar
     pengecualian valid (disajikan agent sebagai kartu di chat; di editor sebagai
     form).
  3. **Penegakan deterministik di Go** (pola classifier D-3): baris ditolak validasi
     X → cocokkan `violates` dari profil; rule cocok + action allow → lanjut; skip
     → skipped; tidak ada → rejected (fail-closed).
  4. **Dua tempat pengeditan**: chat (CP-2, penyimpanan saja — status quo tetap
     valid) + **panel rules di editor WO-03** (tambah/ubah/hapus, tetap harus
     disatakan user). Penegakan + panel = lingkup **WO-03/CP-3**.
  atau tidak? Default usulan Manager: TIDAK dulu (Go-only tetap terjaga); deidentify
  dipakai dengan tipe kolom eksplisit dari mapping editor.
- O-12: Kebijakan flag `is_encrypt`: apakah flag user boleh MENGURANGI proteksi bawaan
  (kolom PII wajib dari fields.yaml)? Usulan Manager: flag hanya boleh MENAMBAH
  proteksi (lebih ketat), tidak bisa lebih longgar — fail-closed.

## Temuan pola tool-calling & upload (WO-01b, 2026-11-04)

Bukti AC-4/AC-5/AC-6 dengan runtime dev `ingest-runtime` + Go data service (D-11).

### Mekanisme tool = API backend (pola definitif CP-2)

1. **Registrasi di Node, eksekusi di Go.** `CopilotRuntime` memakai `agents`
   sebagai *factory* per-request (`agents: ({ request }) => ...`), sehingga
   `BuiltInAgent` bisa dibuat ulang per request dengan `tools:[{ sandbox_ping }]`.
   `execute()` tidak menyimpan state global: ia membaca `Authorization` +
   `X-Ingest-Session-Id` dari `request`, lalu `POST` ke
   `/api/admin/ingest/tools/sandbox_ping`.
2. **Guard + audit tetap di Go.** Route tool memakai `RequirePermission(...,
   "ingest.write")` dan mencatat `ingest_audit` (`provider='ingest-tool'`).
   Node tidak pernah menulis data bisnis — hanya meneruskan tool-call model.
3. **Alur event AG-UI yang terlihat di klien:**
   `TOOL_CALL_START` → `TOOL_CALL_ARGS` → `TOOL_CALL_END` → `TOOL_CALL_RESULT`
   → `TEXT_MESSAGE_*`. Hasil tool tampil di chat sebagai code block + ringkasan
   model (belum ada renderer khusus — muncul warning dev CopilotKit
   "no renderer is registered for tool", boleh ditambah `useRenderTool` di CP-2).
4. **Latency terukur** (sandbox, model gateway `cc-deepseek-v4.1-flash`):
   round-trip tool Node→Go ≈ 3–10 ms; satu run dengan tool + jawaban akhir
   ≈ 2,5–3,5 s (didominasi LLM). Kegagalan yang mungkin: tanpa token → tool
   throw `401` (model melaporkan gagal), izin kurang → `403
   izin ditolak: butuh ingest.write`, `INGEST_GO_API_URL` kosong → tool langsung
   error tanpa request.

### Temuan kompatibilitas provider (WAJIB dibaca sebelum CP-2)

- Gateway `donis-agent.rolldev.my.id` mengirim chunk streaming
  `choices[].delta.tool_calls[]` **tanpa field `index`** (wajib menurut spec
  OpenAI). AI SDK memvalidasi strict → seluruh run menjadi `RUN_ERROR`
  (`invalid_type: expected number, path: choices[0].delta.tool_calls[0].index`).
- Semua alias model di gateway (`copilot/manager/hermes/assistant`) berperilaku
  sama — jadi bukan masalah model.
- Solusi sementara: `runtime/provider-compat.ts` membungkus `fetch` dan hanya
  menyisipkan `index` pada SSE provider (host dari `INGEST_LLM_BASE_URL`).
  Bila gateway diperbaiki → hapus shim ini.
- `COPILOTKIT_OPENAI_API=responses` TIDAK bisa dipakai: jalur Responses API
  dibalas `405 Method Not Allowed` oleh gateway.

### Upload file (WO-01b)

- Endpoint: `POST /api/admin/ingest/sessions/:id/uploads` (guard `ingest.write`)
  → `internal/storage` key acak `ingest/uploads/<session>/<uuid><ext>` +
  metadata `ingest_uploads`. Isi file tidak diproses; tidak ada pembacaan isi.
- Fail-closed: `.csv`/`.xlsx` saja; `.xls`/`.xlsm`/`.xlsb` → **415** dengan
  pesan "simpan ulang sebagai .xlsx" (pola N-17), ekstensi lain → 415,
  > 10 MiB → 413, storage tidak terpasang → 503.
- Idempotency `client_id` (pola WO-01a) → retry mengembalikan
  `{"status":"duplicate"}` tanpa baris/objek baru; objek yatim dihapus.
- Jebakan UI: instance axios proyek default `Content-Type: application/json`.
  Untuk `FormData` wajib override ke `multipart/form-data`, kalau tidak server
  membalas `400 file_missing`.
- Jebakan env: `./storage` di mesin dev bisa dimiliki root (sisa image lama) →
  server (user `go`, uid 1000) gagal `mkdir storage/ingest: permission denied`.
  Perbaikan: pastikan direktori dimiliki user container (atau set
  `STORAGE_ROOT`). Folder `storage/` sudah masuk `.gitignore`.

## Temuan eksperimen MCP + keputusan (WO-01c, 2026-11-04)

Eksperimen dijalankan **di luar jalur utama**: satu server MCP kecil, dinyalakan
manual (bukan di `docker-compose`), dimatikan setelah pengukuran.

### Yang dipakai

- `BuiltInAgent` sudah mendukung MCP native: `mcpServers: [{ type: 'http', url }]`
  (atau `type: 'sse'` + `headers`). Karena agent dibuat per request
  (factory di `CopilotRuntime`), server MCP juga di-resolve per request.
- `@modelcontextprotocol/sdk@1.32.2` **sudah ada** di `node_modules` sebagai
  dependency transitif CopilotKit → eksperimen ini **tidak menambah dependency**.
- Server uji: `sub_app/webapp/runtime/mcp-experiment/server.ts` (±90 LOC,
  `StreamableHTTPServerTransport` stateless, satu tool `mcp_echo`).
- Integrasi di runtime bersifat opt-in: aktif hanya bila
  `INGEST_MCP_EXPERIMENT_URL` diisi (compose mengunci nilainya `""`).
  Default = **MCP tidak dipakai**, jalur tool tetap API Go.

### Hasil

- Tool terbaca dan terpanggil: prompt "pakai tool `exp_mcp_echo`" menghasilkan
  `TOOL_CALL_START/ARGS/END` → `TOOL_CALL_RESULT` berisi `mcp-pong: halo-mcp`
  → `TEXT_MESSAGE_*` → `RUN_FINISHED`. Nama tool otomatis diberi prefix nama
  server (`exp_<tool>`), jadi prompt/model harus tahu nama ber-prefix itu.
- **Tidak** bertabrakan dengan shim provider (`provider-compat.ts`): MCP
  memakai JSON-RPC/HTTP sendiri, bukan SSE format OpenAI. Catatan: shim membungkus
  `globalThis.fetch`, jadi request non-provider WAJIB di-passthrough apa adanya
  (sudah begitu sejak WO-01b).
- Kegagalan MCP bersifat **graceful**: saat server mati, runtime hanya menulis
  `MCP server ... failed to connect — skipping it for this run` dan chat tetap
  jalan (tool MCP saja yang hilang).
- Jebakan implementasi (memakan 1 siklus debug): satu instance `Server` MCP tidak
  boleh di-`connect` ke lebih dari satu transport. Pada mode stateless transport
  dibuat per request → **server juga harus per request**, kalau tidak proses
  crash `Already connected to a transport`. MCP stateful (session id) menambah
  kerumitan lain: lifecycle, cleanup, dan penyimpanan sesi.
- **Celah guard/audit:** panggilan tool MCP **tidak menghasilkan baris
  `ingest_audit`** (yang tercatat hanya run-level `POST /api/copilotkit/...`)
  dan **tidak melewati RBAC Go** — tool dieksekusi penuh di proses Node.
  Bandingkan dengan tool WO-01b: selalu ada baris `provider='ingest-tool'` +
  guard `ingest.write`.

### Effort: MCP vs tool backend biasa

| Aspek | Tool backend (WO-01b) | MCP (WO-01c) |
| --- | --- | --- |
| Kode baru | endpoint Go + service + tests + handler + API client + UI | ±90 LOC server + 3 baris config agent |
| Infrastruktur | tidak ada (HTTP yang sudah ada) | proses/port tambahan + lifecycle + (stateful) session store |
| Guard & audit | otomatis (middleware + `ingest_audit`) | **tidak ada**, harus dibangun manual |
| Validasi input | di service Go (unit-tested) | schema tool saja, tanpa enforcement server-side kita |
| Observability | audit + log Go | log proses terpisah |
| Waktu | 1 WO kecil penuh (dengan bukti AC) | ±1 jam untuk echo tool |
| Nilai jangka panjang | dipakai CP-2+ | hanya jika butuh server MCP pihak ketiga |

Kesimpulan effort: MCP lebih cepat untuk **membungkus tool yang sudah ada di
server lain (ekosistem)**, tetapi untuk backend internal sendiri ia lebih mahal
begitu guard/audit/lifecycle dihitung.

### KEPUTUSAN (terekam, AC-8)

**MCP TIDAK dipakai** untuk CP-2 dan seterusnya. Jalur tool tetap: model →
tool `BuiltInAgent` → endpoint Go (guard + audit). Alasan:

1. D-11 mewajibkan semua tulisan/aksi data lewat Go; tool MCP yang berjalan di
   Node akan **melewati guard RBAC dan audit** — risiko yang tidak bisa
   diterima untuk data residen.
2. Tidak ada kebutuhan nyata: hanya ada satu backend (Go) yang memang kita
   kendalikan sendiri; tidak ada server MCP eksternal yang perlu dikonsumsi.
3. Biaya operasional (proses + lifecycle + session state) tidak sebanding.

Kode eksperimen **dipertahankan** (bukan dihapus) karena kecil, mati secara
default, dan berguna sebagai bukti bila keputusan ditinjau ulang. Pemicu
tinjau-ulang: muncul kebutuhan mengonsumsi server MCP pihak ketiga
(mis. konektor Drive/Sheets) — dan itu pun tetap harus lewat guard Go.

## Rekaman effort adaptasi CopilotKit (masukan CP-4, WO-01c)

Diukur dari percakapan nyata CP-1, bukan estimasi:

1. **Salah pilih arsitektur (WO-01a)**: rencana awal "Go sebagai proxy
   OpenAI-compatible + CopilotKit di frontend" menghabiskan siklus implementasi
   penuh lalu gagal — CopilotKit v2 bicara **AG-UI**, butuh runtime Node.
   Biaya: 1 WO + 1 redesign. Pelajaran CP-4: verifikasi protokol klien
   (`/info`, `/agent/*`) sebelum membangun backend.
2. **Responses API tidak tersedia di gateway** (405) — 1 siklus debug.
3. **Non-compliance provider** (tanpa `index` pada `delta.tool_calls`) → seluruh
   run `RUN_ERROR`; butuh shim `provider-compat.ts`. Ini jenis bug yang harus
   diasumsikan ada di gateway internal: sediakan titik patch + rencana hapus.
4. **Streaming persist**: menulis `String(chunk)` untuk `Uint8Array` membuat
   pesan tersimpan sebagai daftar byte. Butuh `TextDecoder` — 1 siklus debug.
5. **Tool loop** (WO-01b): pola factory-per-request + guard/audit di Go langsung
   stabil; effort kecil karena arsitekturnya sudah benar.
6. Untuk CP-4 (editor/komponen kaya) yang belum ada: renderer hasil tool
   (`useRenderTool`), state editor ↔ model, dan streaming artifact. Belum ada
   bukti apa pun — anggarkan waktu eksplorasi terpisah, jangan menempel ke WO
   fitur.

## Checklist readiness CP-2 (WO-01c)

- [x] **Env aman**: `INGEST_LLM_*` hanya di container `ingest-runtime`; container
  `app` (Go) menimpanya dengan string kosong di `docker-compose.yml`. Go tidak
  lagi membaca env itu sama sekali (proxy dihapus di WO-01c) — lapisan kosong
  dipertahankan sebagai pertahanan berlapis.
- [x] **Permission seed**: `ingest.read`/`ingest.write` hanya di-seed migrasi
  plugin `auth`; route plugin gagal-tertutup (401 tanpa token, 403 tanpa izin,
  403 untuk admin soft-deleted) — sudah diuji di WO-01a/b.
- [x] **Tidak ada data bisnis lewat sandbox**: isi `ingest_messages`/`ingest_audit`
  hanya teks uji ("halo", "pong: …", "halo-mcp"); upload hanya fixture CSV
  contoh. Tidak ada nama/NIK residen.
- [x] **Telemetri mati**: `COPILOTKIT_TELEMETRY_DISABLED=true` di compose.
- [x] **Migrasi**: `ingest_messages`, `ingest_audit`, `ingest_uploads` sudah
  `up`; tidak ada migrasi menggantung.
- [x] **Permukaan endpoint sandbox**: `/api/admin/ingest/{sessions,messages,
  audit-events,tools/sandbox_ping,uploads}` + `/api/copilotkit/*`. Proxy
  `/api/admin/ingest/llm-proxy/*` **sudah dihapus** (404) — kode mati dibersihkan
  sebelum CP-2 menambah logika bisnis.
- [x] **Koleksi Bruno** `api-tests/ingest/` meng-cover semua endpoint sandbox
  (tool, upload sukses/ditolak, sesi, pesan, audit).

## Housekeeping lain (WO-01c)

- Nasib proxy Go `/api/admin/ingest/llm-proxy/*`: **DIHAPUS** (keputusan owner).
  Alasan: D-11 menjadikan runtime Node satu-satunya pemanggil LLM, sehingga
  endpoint itu selalu fail-closed 503 (kode mati) tetapi tetap menambah
  permukaan serangan dan menyiratkan jalur yang tidak ada. Handler
  `LLMProxy` + `injectDefaultModel` + `Config`/`Configured` + konstanta
  `EnvBaseURL/EnvAPIKey/EnvModel` ikut dibuang. Implementasi lama tetap ada di
  riwayat git (commit `a019c26`) bila suatu saat dibutuhkan.
- Verifikasi: `POST /api/admin/ingest/llm-proxy/chat/completions` → **404**,
  `/api/copilotkit/info` tetap 401 (guard hidup), swagger diregenerasi dan tidak
  lagi memuat path proxy.

## Catatan keputusan CP-2 (WO-02, 2026-11-04)

- **Deidentify tidak butuh peta reversibel.** Library `deidentify` v1.0.4 tidak
  punya API balik. Keputusan: Go memegang berkas ASLI dan mengekstrak dari
  aslinya, jadi "re-identifikasi" tidak pernah diperlukan. Ini lebih aman dan
  lebih sederhana daripada menyimpan tabel token→nilai (yang justru menjadi
  sasaran baru). Deidentify hanya berlaku untuk payload yang dilihat LLM.
- **Fail-closed di `prepare_safe_payload`.** `INGEST_DEIDENTIFY_KEY` kosong →
  `/sessions/:id/safe-payload` membalas 503, bukan mengirim sel mentah. Gate ini
  menggantikan gate proxy lama yang hilang bersama proxy di WO-01c.
- **Tipe PII eksplisit, bukan tebakan.** `kind` per kolom berasal dari mapping
  yang dikonfirmasi (`name/phone/email/address/ssn/credit_card/generic/none`).
  `generic` = HMAC deterministik `anon-<12 hex>`: deterministik (LLM masih bisa
  melihat "orang yang sama") tapi tidak bisa dibalik.
- **Sample ≤5 baris** (`MaxSampleRows`); nilai lebih besar dipotong di Go, bukan
  dipercaya dari klien.
- **Dedup wajib eksplisit.** `run_extraction` menolak (`400`) tanpa
  `dedup_policy ∈ {phone_then_time, time_only, none}`. Alasan konkret: berkas
  real case BBH punya 177 baris tanpa `phone_e164`; menebak = data ganda.
- **`complete` internal.** Aksi `extracting → extracted` hanya dipanggil Go
  setelah impor selesai dan tidak muncul di `allowed_actions` klien.
- **Dry run tidak mengunci.** `dry_run=true` mengembalikan sesi ke
  `mapping_confirmed` supaya pengguna bisa memperbaiki mapping/berkas.
- **Compact (O-18) batasnya jujur.** `compact.ts` memicu pada 70% limit,
  mempertahankan N pesan terakhir + semua `[LOCKED]`, lalu menyimpan ringkasan
  lewat Go (`role=summary`). Pemendekan prompt *in-flight* masih milik runtime
  CopilotKit; yang kita kendalikan adalah ringkasan tersimpan + keputusan
  terkunci yang tidak pernah hilang.
- **Prompt sebagai berkas** (`plugins/ingest/prompt/agent.md`, env
  `INGEST_PROMPT_PATH`) supaya aturan agent bisa di-review seperti aturan bisnis.

## CP-3 — keputusan teknis editor mapping (2026-11-04)

- **Satu profil, dua wajah.** Editor menulis dokumen mapping v2 (daftar) ke jsonb
  `ingest_profiles.mapping` yang sama dengan bentuk legacy `{"kolom":"field"}`.
  `ParseMappingDocument` menerima keduanya dan `MergeIntoLegacy()` mengembalikan
  bentuk legacy untuk agent/ekstraksi — tidak ada salinan data kedua (C-4/AC-1).
- **Katalog pengecualian = satu file sumber** (`plugins/visit/schema/exceptions.yaml`,
  `//go:embed`). Penegak Go dan dropdown editor memakai loader yang sama, jadi
  rule "tidak dikenal" tidak mungkin lahir dari UI (O-20/AC-R3).
- **Penegakan mengeluarkan berkas baru, bukan flag.** `EnforceMappingRules`
  memancarkan `<nama>-enforced.csv` yang **diproyeksikan ke header template** dan
  membuang baris skip/reject. Importer hanya pernah melihat CSV itu — tidak ada
  jalur di mana baris tolak tetap masuk staging.
- **Proyeksi header adalah inti editor.** Berkas nyata berheader bebas
  ("Nama Lengkap", "No HP"); importer menuntut header template ("Nama",
  "Nomor HP"). Sebelum proyeksi ini ada, impor menolak semua baris dengan
  "kolom wajib tidak ditemukan" — jadi mapping tanpa proyeksi = fitur setengah.
- **Policy PII hanya boleh diperketat** (D-5). `encrypt: true` di `fields.yaml`
  + validasi `ErrPolicyRelaxed`: editor tidak bisa mematikan deidentifikasi field
  yang sudah dilindungi.
- **`revise` membatalkan persetujuan.** Mengubah mapping setelah `mapping_confirmed`
  memindahkan state ke `mapping_draft`, sehingga pengguna wajib konfirmasi ulang.
  Tanpa ini, "sudah konfirmasi" bisa dipakai untuk menyelundupkan mapping baru.
- **Gate dikembalikan sebagai daftar, bukan boolean.** `409 {gates:{...}}` memberi
  UI alasan konkret (baris ke berapa, kolom apa), bukan sekadar "belum siap".
- **Dedup tetap milik manusia.** Editor menyediakan pilihan `dedup_policy`, tapi
  nilainya tetap wajib dikirim — Go menolak (400) bila kosong. Tombol Ekstrak
  mengunci sampai gate + kebijakan siap.


## Retro-note lintas topik (2026-11-23) — commit e85c0f6

Saat review WO-08 topik **Management Gudang** (`1795415237015`), Manager menerima commit `e85c0f6` — `fix(ingest): resolve model.AnalysisSnapshot untuk swag init (blocker pre-existing)`. Isi: perbaikan blocker `swag init` yang sudah ada sebelumnya di `plugins/ingest/handlers/analyze_tool.go` (import + blank-var, pola identik fix hsh/sec), di-commit terpisah agar scope jelas.

Verifikasi Manager: `build.sh --verify-docs` EXIT=0, `go test ./plugins/ingest/...` hijau. Catatan ini **bukan pembatalan WO ingest mana pun** — hanya agar topik ini tahu ada patch kecil di berkas miliknya yang lahir dari lintas topik. Patch sudah ikut terpush ke origin/master bersama 11 commit gudang.

Jabatan: Manager Agent — Nama Tugas: Retro-note e85c0f6 topic ingest. Sayonara.
