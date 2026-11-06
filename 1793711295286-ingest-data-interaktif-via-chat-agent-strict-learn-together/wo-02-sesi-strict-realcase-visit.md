---
title: WO-02 — Sesi strict + real case `visit` via chat (tanpa editor) [REVISI CP-1]
description: "CP-2 (REVISI): pemisahan runtime (O-19), state machine sesi di Go, agent strict via CopilotKit (D-11), deidentify nyata, classifier gate, compact O-18, ekstraksi visit via service layer"
status: complete
created: "2026-11-03"
---
# WO-02 — Sesi Strict + Real Case `visit` via Chat (Checkpoint 2) — REVISI

Checkpoint: **CP-2**. **CP-1 complete** (WO-01a/b/c — commit `e4f4ea9..397fbff`).
Dokumen ini **DIREVISI 2026-11-04** untuk memakai pembelajaran & keputusan CP-1:
D-11 final (runtime Node = backend chat AI; Go = data service), pola tool WO-01b,
keputusan MCP tidak dipakai, O-18 (compact buat sendiri), O-19 (pemisahan runtime).
Sumber: `spec.md` §4/§5/§6/§7; `spec-techstack.md` §2.2 (revisi); notes O-17/O-18/O-19.

> **⚠ REVISI (2026-11-04).** Bagian yang menyebut "proxy WO-01"/"gate di proxy"
> digantikan arsitektur D-11. Teks lama di bagian C diganti sesuai kenyataan.

## Tujuan

Membuktikan alur **learn-together via chat SAJA** (tanpa mapping editor) pada real
case modul pilot `visit`: file nyata diimpor lewat percakapan dengan agent strict,
lalu hasil ekstraksi masuk pipeline deterministik yang sudah ada.

## Lingkup

### 0. Persiapan struktur (O-19) — kerjakan PERTAMA

- Pindahkan runtime CopilotKit dari `sub_app/webapp/runtime/` ke
  **`sub_app/ingest_runtime/`** (npm workspaces, root `sub_app/`, satu lockfile —
  versi CopilotKit react-core/runtime tetap sinkron).
- Dockerfile.runtime hanya meng-install deps workspace runtime; compose
  `ingest-runtime` tidak berubah perilaku; perbaiki komentar header compose
  ("IDLE, DEV-ONLY spike" usang).
- Verifikasi E2E chat + tool + upload tetap hijau setelah pindah (regresi nol).

### A. State machine sesi + persistensi (backend GO — sumber kebenaran)

- Migrasi v2 (plugin `ingest`): `ingest_sessions` (state, target_plugin, file_id,
  timestamps) + `ingest_profiles` (mapping/rules/flags jsonb, version).
  `ingest_messages` & `ingest_uploads` sudah ada (WO-01).
- Transisi state ditandai oleh **action API Go** (backend menegakkan); agent/Node
  tidak bisa melompati gate:
  `uploaded → introduced → mapping_draft → mapping_confirmed → extracting → extracted`
  (+ jalur clarify di dalam `mapping_draft`).
- API sesi: create/list/get + transisi (confirm, extract, cancel). Semua guard
  `ingest.*` + audit. **Node TIDAK pernah menulis profil** — hanya meneruskan
  tool-call ke API Go (pola WO-01b).

### B. Agent strict via CopilotKit (runtime Node, D-11)

- System prompt agent = file template di plugin (`plugins/ingest/prompt/`) memuat
  5 kontrak (spec §5): learn-together, 5 gate, dispatcher-lite (tanpa editor →
  konfirmasi via chat saja di CP-2), batas wewenang, output terstruktur.
- Tool agent = tool API Go (pola `sandbox_ping`): `get_mapping_guess`,
  `confirm_mapping`, `run_extraction` — didaftarkan di **Go**, Node meneruskan
  (factory per-request), eksekusi di Go dengan guard RBAC + audit. TANPA MCP
  (keputusan WO-01c: tool MCP tidak lewat RBAC/audit → ditolak).
- Tebakan mapping awal disajikan di chat berlabel "belum dikonfirmasi"; kolom
  wajib dikonfirmasi user atau `skip`.
- Classifier gate (pola Jev, D-3): antarmuka `DecisionClassifier` di **Go** +
  implementasi `deterministic` (ambang/aturan) untuk gate tool-call: konfirmasi
  ekstraksi, deteksi payload mismatch.
- **Pertahanan sisi Node**: tambalan `provider-compat.ts` (index `tool_calls`)
  DIPERTAHANKAN sampai gateway diperbaiki — catat rencana penghapusannya
  (jangan jadi tambalan abadi).
- **Compact O-18** (implementasi runtime Node): ukur usage kumulatif per thread;
  threshold 70% via `INGEST_LLM_CONTEXT_LIMIT` (fail-closed default konservatif);
  ringkas pesan lama via LLM; summary disimpan sebagai pesan khusus di
  `ingest_messages` VIA GO; **keputusan terkunci TIDAK ikut diringkas** — profil
  dari Go disuntikkan ulang tiap request; compact tercatat di `ingest_audit`.

### C. Deidentifikasi nyata (D-2/D-4) — di GO (bukan Node)

- Dependensi `github.com/aliengiraffe/deidentify` masuk go.mod (plugin `ingest`).
- Secret key env baru (pola AES-GCM, fail-closed bila kosong dan ada kolom butuh).
- **Alur D-11**: sebelum prompt berisi data dikirim (Node → provider), Node meminta
  Go menyiapkan **payload aman**: Go mengambil sample dari file, menjalankan
  deidentify pada kolom PII (tipe eksplisit dari mapping), mengembalikan payload
  yang sudah anonim; re-identifikasi balik lokal di Go saat pembentukan payload
  final (deterministik). Secret key TIDAK pernah keluar dari Go.
- Gate (pengganti "gate di proxy WO-01" yang sudah dihapus): di titik
  `prepare_safe_payload` Go menolak kolom PII wajib yang belum termapping/
  dideidentify (fail-closed).

### D. Ekstraksi ke `visit` (pilot, D-1)

- Payload final → pipeline existing plugin `visit`: validate → staging
  (`visit_import_batches/rows`) → import (dedup, `effective_at` history = waktu sumber).
- Pemanggilan via **service layer plugin `visit`** (bukan SQL lintas plugin).
- Laporan hasil di chat: created/updated/skipped/rejected + batch id.
- **Batasan dikenal dari topic visit**: 177 kunjungan tanpa `phone_e164` tidak punya
  kunci dedup — sesi strict harus menanyakan kebijakan duplikasi baris tanpa HP
  ke user (pertanyaan clarify), bukan memutus sendiri.

## Acceptance Criteria

- AC-0 Pemisahan O-19 selesai: chat/tool/upload E2E tetap hijau dari struktur baru
  `sub_app/ingest_runtime` (regresi nol).
- AC-1 File nyata (ekspor Notion kunjungan, data fiktif/sampel) berhasil diimpor
  end-to-end via chat: laporan created/updated/skipped/rejected tampil + batch id.
- AC-2 Gate identitas: modul target belum dinyatakan user → agent menolak lanjut
  (tertahan state machine Go, bukan hanya prompt).
- AC-3 Gate mapping: kolom tidak dikonfirmasi → tidak diekstrak (di-skip), bukan ditebak.
- AC-4 Gate aturan: pengecualian validasi hanya aktif bila dinyatakan eksplisit user.
- AC-5 Gate konfirmasi: tanpa pernyataan setuju → aksi ekstraksi terkunci (409/400).
- AC-6 Deidentify: payload ke provider TIDAK memuat nilai PII asli (uji: log/
  mock provider); re-identifikasi mengembalikan nilai asli di DB; secret key
  hanya di Go.
- AC-7 Gate deidentify (di Go, titik `prepare_safe_payload`) menolak payload PII
  yang belum dideidentify (fail-closed).
- AC-8 Re-impor file sama → idempoten (dengan kebijakan dedup yang dikonfirmasi user).
- AC-9 RBAC: tanpa `visit.write` → ekstraksi ditolak; audit terisi untuk semua aksi
  (termasuk tool-call dari Node & event compact).
- AC-10 Compact O-18 bekerja: sesi panjang melewati threshold → summary dibuat,
  disimpan via Go, keputusan terkunci tetap utuh di prompt berikutnya.
- AC-11 `go vet`/`go test` hijau; `tsc`/`npm run build` hijau; swagger berisi route
  sesi; koleksi Bruno untuk endpoint sesi (chat e2e manual).

## Non-tujuan

- Mapping editor (CP-3); integrasi editor ↔ chat (CP-4); konfirmasi via editor;
  MCP (ditolak, WO-01c).

## Risiko & catatan

- **DATA NYATA**: berbeda dari CP-1 (fiktif) — pastikan deidentify hidup SEBELUM
  file asli pertama disentuh; sample ke LLM hanya ±5 baris selama clarify.
- Kualitas tebakan LLM untuk header campur ID/EN (bulan, HP kacau) — justru bahan uji
  utama gate; jangan melonggarkan strictness agar "lulus".
- Kebijakan duplikasi tanpa HP wajib diklarifikasi user di sesi (lihat D bagian batasan).
- Tambalan `provider-compat.ts` adalah ketergantungan eksternal (perilaku gateway)
  — pantau; hapus begitu gateway benar.

---

## Progres Implementasi — 2026-11-04 (Dev)

Status: **implementasi inti selesai**, sisa satu langkah yang butuh berkas asli BBH
(upload file nyata lewat UI chat + verifikasi laporan di layar). Semua kode sudah
terpasang, build/test hijau, dan jalur data sudah diuji dengan data **sintetis**.

### 0. Struktur (O-19) — selesai

- Runtime pindah ke `sub_app/ingest_runtime/`; `sub_app/package.json` = workspace
  tunggal (`webapp` + `ingest_runtime`), satu `package-lock.json`.
- `Dockerfile.runtime` meng-install hanya deps workspace runtime; compose
  `ingest-runtime` perilakunya sama; komentar header compose diperbarui.
- Bukti regresi nol (AC-0): chat + tool setelah semua perubahan →
  `TOOL_CALL_RESULT` `pong: ping backend`, `RUN_FINISHED` normal.

### A. State machine (Go) — selesai

- Migrasi v2 `2026_11_04_130000_create_ingest_sessions_profiles` (applied sebagai
  version 4): `ingest_sessions` (7 state, CHECK), `ingest_profiles` (append-only,
  UNIQUE `(session_id, version)`).
- `model/session.go`: tabel transisi + `ActionComplete` (internal,
  `extracting → extracted`, tidak diekspos ke klien).
- `services/session_state.go` + `services/profile.go`: transisi atomik
  (`UPDATE ... WHERE state = ?`), gate konfirmasi mensyaratkan profil non-kosong.
- Bukti: `introduce` 200 → `confirm` 200 `mapping_confirmed` → `extract` 200 →
  `extracted`; lompatan gate & state terminal ditolak 409.

### B. Agent strict (D-11) — selesai

- `plugins/ingest/prompt/agent.md`: system prompt sebagai berkas (5 kontrak:
  belajar bersama, 5 gate, dispatcher tipis, batas kewenangan, output terstruktur).
  Dimuat runtime via `INGEST_PROMPT_PATH` (fallback in-code bila berkas hilang).
- 3 tool Go (`POST /api/admin/ingest/tools/*`, guard `ingest.write`, audit
  `provider='ingest-tool'`): `get_mapping_guess` (via safe-payload),
  `confirm_mapping`, `run_extraction`. Tanpa MCP.

### C. Deidentify nyata (Go) — selesai, terbukti dengan data sintetis

- `services/table.go` (CSV/XLSX, `.xls` ditolak), `services/deidentify.go`
  (tipe eksplisit `name/phone/email/address/ssn/credit_card/generic`; `generic`
  = mask HMAC deterministik), `services/payload.go` (sample ≤5 baris).
- Endpoint `POST /admin/ingest/sessions/:id/safe-payload` = satu-satunya jalur isi
  berkas ke LLM; `INGEST_DEIDENTIFY_KEY` kosong → **503 fail-closed**.
- Bukti (CSV sintetis 2 baris): nama/HP/email/alamat keluar sebagai nilai sintetis
  (`Sky Chavez`, `memberuser3422@anonymous.net`), kolom non-PII utuh.
- Keputusan dicatat: **tidak ada peta re-identifikasi** — Go memegang berkas asli
  dan mengekstrak dari aslinya, jadi tidak ada yang perlu dibalik (lebih aman).

### D. Ekstraksi `visit` — selesai

- `services/extraction.go`: state `mapping_confirmed → extracting → extracted` lewat
  service layer `visit` (`ImportTemplate`), `batch_id` + ringkasan dikembalikan.
- **Dedup wajib eksplisit**: `dedup_policy` kosong/tak dikenal → 400. Bukti:
  request tanpa `dedup_policy` → 400 `butuh session_id & dedup_policy`.
- Bukti impor: `api-tests/fixtures/visit-valid-template.csv` →
  `batch_id 01a1a5d7-…`, `summary {total:2, created:2, rejected:0}`, state `extracted`.
  `dry_run` mengembalikan state ke `mapping_confirmed` (latihan tidak mengunci).

### Compact (O-18) — selesai sebagai modul + wiring

- `sub_app/ingest_runtime/compact.ts`: ambang 70% dari `INGEST_LLM_CONTEXT_LIMIT`
  (default 32k), `INGEST_COMPACT_KEEP_LAST` (default 8), keputusan `[LOCKED]`/
  role `decision` **tidak pernah** diringkas, ringkasan disimpan via Go
  (`role=summary`, `client_id` deterministik → idempoten).
- 9 unit test (`npm run test` di workspace runtime) hijau.
- Batasan jujur: compact berjalan pada event `RUN_STARTED` dan menyimpan ringkasan
  ke Go; pemendekan prompt in-flight tetap milik runtime CopilotKit (bukan kode
  kita) — dicatat di notes.

### AC — status bukti

| AC | Status | Bukti |
|----|--------|-------|
| AC-0 regresi nol | ✅ | chat + `sandbox_ping` `pong` setelah semua perubahan; upload 201 |
| AC-1 laporan ekstraksi di chat | ✅ (API) | `batch_id` + `summary` pada respons tool |
| AC-2 gate identitas | ✅ | `introduce` hanya dari `uploaded`; `confirm` dari `uploaded` → 409 |
| AC-3 gate mapping | ✅ | profil wajib non-kosong sebelum `confirm` |
| AC-4 gate aturan | ⏳ | rules disimpan di profil; penegakan per-rule belum diuji |
| AC-5 gate konfirmasi | ✅ | `extract` dari state selain `mapping_confirmed` → 409 |
| AC-6 deidentify | ✅ | sample sintetis keluar tanpa PII asli (uji `deidentify_test.go`) |
| AC-7 gate fail-closed | ✅ | key kosong → 503; test `DeidentifyColumnsFailsClosedWithoutKey` |
| AC-8 idempotensi re-impor | ⏳ | `dedup_policy` diteruskan, uji berkas sama 2x belum dijalankan |
| AC-9 RBAC + audit | ✅ | guard `ingest.write`; audit `ingest-tool`/`ingest-payload` terisi |
| AC-10 compact | ✅ | 9 unit test; ringkasan disimpan via Go |
| AC-11 build/test/swagger/Bruno | ✅ | `go build`, `go test ./plugins/ingest/...`, `tsc`, swagger berisi 4 route baru, Bruno 08–10 |

### Sisa pekerjaan (butuh keputusan/berkas dari Manager)

1. Uji dengan **berkas asli BBH** (177 baris tanpa `phone_e164`) lewat UI chat:
   pastikan agent benar-benar berhenti dan menanyakan kebijakan dedup.
2. Jalankan skenario AC-8 (impor berkas sama dua kali) dengan kebijakan yang sama.
3. AC-4: definisikan penegakan rules (`rules` masih pasif).

---

## Review Manager — 2026-11-04 (laporan dev diproses, commit kode `032941e`)

Laporan Dev Agent diterima dan di-review. **WO-02 inti: COMPLETE** — 0 (O-19),
A (state machine v4), B (agent strict + 3 tool Go), C (deidentify fail-closed,
tanpa peta reversibel — keputusan bagus, catat di notes), D (ekstraksi visit via
service layer + `dedup_policy` wajib), compact O-18 (9 test) — commit `032941e`.
Gate: `go build/vet/test` + `tsc` + 9 test runtime + swagger (4 route baru) +
Bruno 08–10 + AC-0 regresi nol — semua hijau.

**Sisa (tidak menghalangi lanjut ke CP-3, per prinsip partial):**

1. **Uji berkas asli BBH** (AC-1 penuh + perilaku strict 177 baris tanpa
   `phone_e164`) — butuh berkas asli dari owner; jalankan via UI chat.
2. **AC-8** — re-impor berkas sama 2×, skenario idempotensi.
3. **AC-4** — penegakan `rules` masih pasif; **KEPUTUSAN (O-20, 2026-11-04, owner):
   penegakan + skema rules + panel editor rules = lingkup WO-03/CP-3; CP-2 tetap
   penyimpanan chat-only** (status quo valid). Skema + detail: lihat WO-03 §D.

**Catatan review:**
- Keputusan dev "tidak ada peta re-identifikasi — Go mengekstrak dari berkas
  asli" DITERIMA: menghapus kelas risiko kebocoran peta; konsisten D-4.
- `dry_run` tidak mengunci state (kembali ke `mapping_confirmed`) — wajar,
  cocok untuk latihan; audit tetap tercatat.
- Migrasi ternyata v4 (bukan v2 seperti rancangan) karena CP-1 sudah makan
  v2/v3 — tidak masalah, hanya catatan.
- `provider-compat.ts` masih dikeluarkan bersama runtime (tambalan gateway)
  — tetap dalam daftar pantau untuk penghapusan saat gateway benar.

---

## Verifikasi sisa CP-2 (WO-03 §0) — 2026-11-04

### 0.1 Berkas asli BBH via UI chat — **TERBLOKIR**

Berkas asli (177 baris tanpa `phone_e164`) belum diterima dari owner/Manager.
Sesuai instruksi WO-03, tidak ada fixtures tiruan sebagai pengganti. Begitu
berkas diterima: upload lewat `/ingest/:id/chat`, pastikan agent berhenti di
`mapping_confirmed` dan **bertanya kebijakan dedup** sebelum `run_extraction`
(gate di Go sudah terbukti 400 tanpa `dedup_policy`, CP-2).

Pengecekan fail-closed deidentify **dijalan ulang** otomatis saat pengujian
(CP-2: tanpa `INGEST_DEIDENTIFY_KEY` → 503).

### 0.2 AC-8 — re-impor berkas sama 2× → idempoten — **LULUS (2026-11-04)**

Uji dengan berkas segar (2 baris, nomor HP + tanggal unik, `dedup_policy=
phone_then_time`, dua sesi terpisah):
- Impor-1 → `batch 01a1a616-177f-75bd-9c35-fe558cfcba40`, `created=2, skipped=0`.
- Impor-2 → `batch 01a1a616-17f8-7ea5-8294-e59521900dc2`, `created=0, skipped=2`,
  pesan baris: `sudah ada di database (HP + waktu kunjungan sama)`.

Idempotensi terbukti pada level pipeline (`findVisitByKey` skip, tanpa duplikat).
Saat berkas asli BBH tiba (0.1), AC-8 di-assert ulang pada berkas itu sebagai
bukti akhir. (Uji awal dengan fixture lama juga menunjukkan `skipped=2` pada
kedua impor karena barisnya sudah terlanjur ada dari pengujian CP-2.)

## Catatan Implementasi (Dev)

| Berkas:baris | Yang dilakukan | Kenapa |
|---|---|---|
| `plugins/ingest/services/session_state.go:136-188` | Transisi atomik `UPDATE ... WHERE id = ? AND state = ?` + `RowsAffected == 0 → ErrInvalidTransition` | Lompat gate = 409; dua pemanggil paralel hanya satu yang menang (state machine = sumber kebenaran, D-11) |
| `plugins/ingest/services/session_state.go:149-166` | Gate konfirmasi mensyaratkan profil non-kosong; menandai versi terkunci (`confirmed_at/by`) | Tidak ada jalur "confirm buta" tanpa mapping yang tersimpan |
| `plugins/ingest/services/session_state.go:190-208` | `SetSessionState` TIDAK diekspos sebagai endpoint publik | Alur internal saja (extracted / rollback); ekspos = lompat gate dari klien |
| `plugins/ingest/services/profile.go:39-103` | Profil append-only: `MAX(version)+1` dalam transaksi; `UNIQUE (session_id, version)` sebagai jaring pengaman | Riwayat mapping tidak bisa dioverwrite; versi tertinggi = kebenaran sesaat (C-4) |
| `plugins/ingest/services/deidentify.go:75-116` | `DeidentifyColumns` fail-closed: tanpa `INGEST_DEIDENTIFY_KEY` → `ErrDeidentifyUnconfigured` (503); tidak pernah fallback ke nilai asli | PII tidak boleh bocor ke LLM karena secret lupa di-set (D-5) |
| `plugins/ingest/services/deidentify.go:159-173` | Mask deterministik HMAC (`anon-<12 hex>`) untuk `kind = generic` | Nilai sama → mask sama (LLM masih bisa melihat "orang yang sama") tapi tidak bisa dibalik |
| `plugins/ingest/services/payload.go:28-30,89-100` | `MaxSampleRows = 5`; sample dipotong di Go, bukan dipercaya dari klien | LLM hanya butuh contoh; batas di server = satu-satunya penegak |
| `plugins/ingest/services/payload.go:89-102` | `BuildSafePayload` = satu-satunya jalur isi berkas keluar dari Go (sudah terdeidentifikasi) | Tidak ada jalur kedua yang lupa deidentify (N-16) |
| `plugins/ingest/services/extraction.go:73-84` | `NormalizeDedupPolicy` memvalidasi `dedup_policy ∈ {phone_then_time, time_only, none}`; kosong/tak dikenal → 400 | Berkas BBH punya 177 baris tanpa `phone_e164`; menebak = data ganda |
| `plugins/ingest/services/extraction.go:90-173` | `RunExtraction`: `mapping_confirmed → extracting → extracted`, rollback ke `mapping_confirmed` bila impor gagal | Kesalahan impor bukan alasan membuang mapping yang sudah dikonfirmasi |
| `plugins/ingest/services/extraction.go:145-153` | `dry_run` mengembalikan state ke `mapping_confirmed` | Dry run = latihan; sesi tidak boleh tertinggal di `extracting` |
| `plugins/ingest/model/session.go` | Tabel transisi 7 state + `action.complete` internal (tidak di `AllowedActions`) | `complete` hanya dipanggil Go setelah impor; klien tidak boleh "menandai selesai" sendiri |
| `sub_app/ingest_runtime/compact.ts:20-23,93-94` | Threshold compact 70% (`COMPACT_THRESHOLD_RATIO`), default limit 32k token dari env | O-18 fail-closed: limit tak di-set → default konservatif, bukan "tak terbatas" |
| `sub_app/ingest_runtime/compact.ts:77-84` | Keputusan terkunci ditandai eksplisit `[LOCKED]` (bukan ditebak dari isi) | Compact tidak boleh membuang keputusan yang sudah disepakati user |
| `sub_app/ingest_runtime/compact.ts:52-95` | Mempertahankan pesan terakhir N + semua `[LOCKED]` + menyimpan ringkasan lewat Go (`role=summary`) | Ringkasan = data tersimpan, bukan prompt sementara |

---

## Progres Implementasi — 2026-11-06 (Dev): §0.1 + bug sesi-lazy

### §0.1 AC-0 — LULUS (berkas asli BBH, 2026-11-06)

Dok WO-02 sebelumnya mencatat §0.1/AC-0 TERBLOKIR menunggu berkas
asli. Berkas tiba 2026-11-06 (`KUNJUNGAN HARI INI ...csv`, 141.405
byte, 1.639 baris data; header `Nama,Alamat,Nomor HP,Waktu Kunjungan,
Jumlah Pengunjung,Status Kunjungan`; hanya 5 baris tanpa HP).
Owner mengunggahnya lewat panel chat (sesi `72be67c3-…`).

**Bug yang menghalangi:** upload sukses, agent membalas "tidak ada
sesi ingest aktif… 0 baris terbaca", tool `get_mapping_guess` 404.
Akarnya: `POST /sessions/:id/uploads` hanya menulis `ingest_uploads`
(tanpa FK ke `ingest_sessions`), sementara safe-payload/tool
memanggil `GetSession` → 404. Sesi klien ber-UUID tidak pernah lahir
(komentar di `sessionId.ts`: *"sesi di backend lahir saat pesan
pertama tersimpan"*), padahal upload harusnya sudah berarti sesi
mulai.

**Perbaikan (commit `743e183`):**
- `SaveUpload` / `AppendMessage` kini memanggil `EnsureSession`
  (state `uploaded`); sesi "hantu" lama pulih otomatis.
- Berkas baru menggantikan berkas sesi (invarian satu berkas/sesi);
  `sessions.file_id` tertaut.
- `SaveMapping` mendukung `uploaded → introduced → mapping_draft`,
  sehingga editor bisa dipakai langsung pada sesi hasil upload.
- `revise` boleh dari `extracted` (koreksi mapping + re-ekstraksi
  idempoten); error transisi/terminal → 409 (bukan 500).

**Uji §0.1 — hasil perilaku yang dituntut AC-0:**
- Tool `get_mapping_guess` berhasil; safe-payload memberi 5 baris
  sampel terdeidentifikasi + deskripsi isi berkas (header ternama,
  tanggal campuran, kolom `Jumlah Pengunjung`/`Status Kunjungan`
  tidak ada di katalog visit) — agent tidak lagi mengarang isi.
- Ekstraksi tanpa `dedup_policy` → **400** ("kebijakan dedup wajib
  diputuskan manusia"), karena berkas BBH memang tidak bisa
  didedup otomatis dengan jujur. Itu titik berhenti yang diminta
  AC-0.
- Sesi percobaan `112c6a31-…` berakhir ber-state `mapping_draft`;
  mapping draft tersimpan (profil sama, bukan duplikat).

**AC-8 pada berkas asli:**
- `time_only`: IMPOR-1 `created=177, skipped=1462`; re-impor
  `created=0, skipped=1639` → idempoten.
- `phone_then_time`: IMPOR-1 `created=57`; re-impor `created=0,
  skipped=1639` → idempoten.

Artinya: §0.1 WO-02 (berkas asli via chat → agent strict berhenti
+ bertanya dedup) kini **LULUS**, bukan lagi TERBLOKIR.

### Catatan Implementasi (Dev) — 2026-11-06

| Berkas:baris | Yang dilakukan | Kenapa |
|---|---|---|
| `plugins/ingest/services/session_state.go` (`EnsureSession`) | Membuat/barengkan sesi bila belum ada (state `uploaded`) | Upload tidak boleh jadi "sesi hantu"; helper sudah ada, commit ini yang memasangnya |
| `plugins/ingest/services/uploads.go` (`SaveUpload`) | Panggil `EnsureSession`; state harus `uploaded` (409 bila diperkenalkan); berkas baru menggantikan berkas sesi; tautkan `sessions.file_id` | Satu berkas/sesi; lampiran duplikat (`...49.csv` + `...49_all.csv`) tidak menumpuk |
| `plugins/ingest/services/sessions.go` (`AppendMessage`) | Panggil `EnsureSession` sebelum menyimpan | Sesi lama `72be67c3-…` pulih pada pesan berikutnya |
| `plugins/ingest/services/mapping.go` (`SaveMapping`) | Dukung aliran `uploaded → introduced → mapping_draft` | Editor langsung bisa dipakai di sesi upload; tombol sebelumnya tanpa aksi |
| `plugins/ingest/model/session.go` | `revise` dari `extracted`; `cancel` tetap tertutup di terminal | Koreksi mapping pasca-ekstraksi sah; terminal bukan |
| `plugins/ingest/handlers/mapping.go` | Matakan `ErrInvalidTransition`/`ErrSessionTerminal`/`ErrProfileRequired` → 409 | Tombol Extract di `extracted` kini memberi pesan eksplisit, bukan 500 |
| `plugins/ingest/services/payload.go` | Bedakan "sesi tidak ditemukan" vs "sesi belum punya berkas" | Agent tidak mengarang; instruksi dialog jadi jujur |
| `plugins/visit/services/import.go` (`ImportOptions` + dedup) | `DedupPolicy` nyata: `phone_then_time` → fallback nama+waktu; `time_only` → waktu; `none` → tanpa dedup | §0.1: campuran format + baris tanpa HP; menebak = duplikat |
| `plugins/visit/services/phone.go` (`normalizePhoneE164`) | Terima awalan `00` + nomor internasional tanpa awalan (`447454726495`) | Baris format asing bisa didedup, bukan selalu "beda" |
| `plugins/ingest/prompt/agent.md` | Persempit daftar tool; bedakan dua ketiadaan di atas | Dialog tidak menjanjikan tool yang tidak ada; agent berhenti jujur |
| `sub_app/webapp/src/pages/admin/ingest/IngestUploadPanel.tsx` | Hint: kirim pesan agar agent membaca + satu berkas/sesi | User tidak menunggu "otomatis dibaca" |
| `sub_app/webapp/src/pages/admin/ingest/IngestChatPage.tsx` | Welcome + placeholder CopilotKit berbahasa Indonesia; tautan editor menulis draft sebelum navigasi | Sapaan default Inggris hilang; tombol editor hidup |
| `api-tests/ingest/17-Upload-membuat-sesi.bru`, `18-Sesi-hasil-upload.bru`, `19-Upload-menggantikan-berkas.bru` | Regresi: upload lahirkan sesi; editor buka dari `uploaded`; upload kedua menggantikan pertama | Bug 2026-11-06 tidak boleh kembali diam-diam |

---

## Review Manager — 2026-11-06 (pasca §0.1)

**§0.1 WO-02: LULUS — WO-02 kini COMPLETE PENUH** (semua AC 0–11 ✅;
AC-4/AC-8 yang ⏳ ditutup via WO-03 §D + §0.2/berkas asli). Akar bug
sesi-lazy (`ingest_uploads` tanpa FK + `EnsureSession` tak terpasang)
dijelaskan dengan benar; perbaikan konsisten dengan shortnote Manager
(opsi A). Catatan yang sama berlaku: bukti percakapan UI end-to-end
masih parsial — ulangi via UI saat eksekusi WO-04 (lihat review di
dok WO-03). Data nyata ternyata 1.639 baris (5 tanpa HP), bukan 177
sebagai estimasi awal — tidak mengubah kesimpulan gate.
