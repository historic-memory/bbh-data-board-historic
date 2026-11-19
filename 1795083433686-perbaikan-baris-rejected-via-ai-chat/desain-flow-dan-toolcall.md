---
title: "Desain — flow perbaikan baris rejected & tool call Node↔Go"
description: "DESAIN FINAL (owner ok 2026-11-19): flow 5 tahap + listing rejected interaktif di editor (checkbox scope AI + edit manual) + loop AI analisa + 3 tool call Node↔Go; keputusan P-1..P-8, T-1..T-4, O-21/O-22/O-25 dikunci"
status: draft
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - gate
    - ai-chat
    - desain
---
# Desain — Flow perbaikan baris rejected & tool call Node↔Go

Topic turunan `1795083433686` (extends `1793711295286`).
**REVISI FINAL 2026-11-19 (owner ok)** — menggantikan rancangan awal
per-baris interaktif dengan **agent-driven loop + listing rejected di
editor (checkbox scope + edit manual)**. Semua rujukan kode terverifikasi
dari workspace.

## 1. Flow final

```
[mapping accepted — semua kolom dikonfirmasi]
  → tombol "AI Analisa"
(1) FULL SCAN — Go: dry-run enforcement atas seluruh berkas
      (penegak existing `EnforceMappingRules`; bukan sample 5 baris)
(2) TABEL BARIS REJECTED muncul di editor (mapping draft) —
      tiap baris/kelompok: checkbox "perbaiki AI" + edit manual inline
(3) SCOPE — user mencentang baris yang mau diserahkan ke AI
      (default cerdas P-8: format ☑, kelengkapan/ambigu ☐)
(4) TRIGGER — tombol "Analisa (N)": simpan scope ke snapshot
      (analyze, WO-01) → tutup mapping editor → chat menerima
      SATU pesan otomatis BASA-BASI (P-9; tanpa data — AI ambil via tool)
(5) AGENT LOOP (mandiri, atas baris terpilih):
      baca (deidentified) → klasifikasi salah-data vs pengecualian →
      tulis proposal batch (row-fix / rule dari katalog) → sisa dilaporkan
      batas: max 3 putaran analisa, max 500 baris/petikan (O-21)
(6) CHECKPOINT — proposal tampil di listing (kolom "usulan AI",
      status menunggu) + ringkasan di chat (blok terstruktur, bukan teks LLM)
(7) APPROVAL — user setujui (per baris / kategori / semua) dari listing
      ATAU kartu di chat → apply_fixes (confirmed_by_user eksplisit)
(8) APPLY — rule → profil append-only (StatedBy=user); row-fix →
      Go bentuk CSV perbaikan → menggantikan berkas sesi (P-1)
(9) RE-SCAN otomatis (dry run) → sisa pelanggaran tampil →
      bisa centang lagi → analisa lagi (loop §5-§9 sampai gate hijau)
(10) CONFIRM ULANG → EXTRACT → extracted (state machine existing;
      tidak ada state baru — revise path dipakai)
```

### Keputusan desain P-1..P-8 (DIKUNCI owner 2026-11-19)

- **P-1 Perbaikan nilai = berkas baru, bukan edit DB.**
  Faktanya `latestUploadData` (`services/extraction.go`) selalu membaca
  berkas ASLI dari storage; staging (`*_import_batches/rows`) hanya jejak
  ekstraksi. Jalur nilai yang jujur: agent/user usulkan baris perbaikan →
  approve → Go menghasilkan **CSV perbaikan** → menggantikan berkas sesi
  (invarian satu-berkas/sesi + mekanisme replace existing).
  Edit baris di staging DB = DITOLAK.
- **P-2 Dua jalur, dua tempat keputusan:**
  (a) rule `allow`/`skip` → profil (rules append-only, O-20);
  (b) nilai baris → berkas perbaikan di storage. Profil tak tersentuh jalur (b).
- **P-3 AI TIDAK pernah menulis — hanya mengusulkan.** Semua tulisan lewat
  action API Go (RBAC + audit, D-11); rule selalu `StatedBy: user`;
  auto-apply oleh sistem DITOLAK.
- **P-4 Dry run wajib** pada evaluasi ulang sebelum extract sungguhan.
- **P-5 Konteks ke chat/listing = deidentified.** Nilai yang tampil di
  listing & dikirim ke LLM lewat jalur `BuildSafePayload`/deidentify —
  PII rejected tidak pernah mentah ke model/browser.
- **P-6 Tanpa state baru, tanpa tabel baru.** Flow hidup di state machine
  existing; snapshot analisa terakhir = jsonb di profil/plugin ingest
  (persisten ringan, O-25).
- **P-7 Scope analisa eksplisit (checkbox).** AI HANYA menganalisa baris
  yang dicentang user; state checkbox milik listing (frontend/proposal
  store), bukan tebakan agent.
- **P-8 Checkbox default cerdas (ditandai Go, bukan UI guess):**
  pelanggaran FORMAT (mis. HP tidak valid) → default ☑ (perlu dibetulkan);
  pelanggaran KELENGKAPAN/KEBIJAKAN (mis. HP kosong) → default ☐ (kandidat
  rule — **AI DILARANG mengarang nilai untuk data yang memang kosong**;
  penegak Go menolak row-fix field-wajib-kosong tanpa rule — fail-closed).
- **P-9 Pesan trigger otomatis = BASA-BASI SAJA (owner 2026-11-19).**
  Tombol "Analisa (N)": simpan scope ke snapshot (analyze) → tutup
  mapping editor → kirim SATU pesan sederhana ke chat (mis. "Tolong
  analisa baris rejected yang barusan disimpan"). Detail TIDAK dikirim
  dalam pesan — session id & snapshot sudah ada di sesi; AI mengambil
  data sendiri via `get_rejected_rows` (T-1). Pesan disusun frontend,
  single-shot (tidak terkirim ulang saat refresh — chat persisten).

## 2. Desain tool call Node ↔ Go

Pola yang diikuti = tool strict existing (`handlers/agent_tools.go` +
`sub_app/ingest_runtime/copilot-runtime.ts`):
- **Go = pemilik fungsi**: endpoint `/admin/ingest/tools/*` ber-guard
  `ingest.write` + audit `provider='ingest-tool'`; semua kebijakan
  (katalog rule, fail-closed, deidentify) dieksekusi di Go.
- **Node = pendaftar skema**: JSON Schema → `convertJsonSchemaToZodSchema`
  (tanpa dependensi baru), `BuiltInAgent` per-request, tool memanggil API
  Go via **Bearer milik user**; Node hanya meneruskan tool-call model —
  tanpa logika bisnis (D-11).

### Tool baru (4)

**T-1 `get_rejected_rows`** — baca; DUA konsumen: agent (loop) & UI editor
(tabel listing). Endpoint sama, sumber sama.
- Endpoint Go: `POST /admin/ingest/tools/get_rejected_rows`
- Input: `session_id`, `max_rows` (dipotong backend).
- Output: snapshot analisa terakhir (skipped+rejected per `RowNo`,
  `Violates`, `Reason`) + nilai baris terkait AMAN (deidentify) +
  katalog `exceptions.yaml` segmen + flag default checkbox (P-8, dari
  metadata `violates`).
- Sumber data: report ekstraksi/snapshot analisa + staging rows (audit).

**T-2 `propose_fixes`** — mengusulkan (batch), BELUM menerapkan.
- Endpoint Go: `POST /admin/ingest/tools/propose_fixes`
- Input: `session_id`, `fixes[]`: `{kind:"rule", violates, action,
  scope, note}` (HARUS ada di katalog → `ErrUnknownRule`) atau
  `{kind:"row", row_no, values:{field: nilai_baru}}`.
- Perilaku: disimpan sebagai PROPOSAL (belum diterapkan) + ter-audit;
  muncul di kolom "usulan AI" listing. Fail-closed P-8: row-fix utk
  field wajib yang kosong TANPA rule → ditolak.
- Output: `proposal_id` + ringkasan.

**T-3 `apply_fixes`** — penerapan, dipicu USER (listing ATAU chat).
- Endpoint Go: `POST /admin/ingest/tools/apply_fixes`
- Input: `session_id`, `proposal_id`, `confirmed_by_user: true`
  (wajib eksplisit — hilang → 400).
- Perilaku per kind:
  - `rule` → validasi ulang katalog (`RuleSet.Validate`) → profil
    append-only, `StatedBy=<admin>`.
  - `row` → Go bentuk CSV perbaikan dari berkas asli + nilai baru →
    menggantikan berkas sesi (replace existing).
- Lanjutan: re-scan otomatis (dry run) → sisa pelanggaran →
  listing ter-update → loop; selesai = `mapping_draft` untuk confirm ulang.

**T-4 `submit_analysis`** — penutup loop agent (baru, iterasi final).
- Endpoint Go: `POST /admin/ingest/tools/submit_analysis`
- Input: `session_id`, `proposal_id`, `summary` (ringkasan pemahaman
  agent atas pola pelanggaran), `remaining` (baris yang menurut agent
  tidak bisa dibereskan otomatis + alasannya).
- Perilaku: mengakhiri putaran analisa — snapshot analisa diperbarui,
  `remaining` ditandai di listing (status "perlu keputusan manusia"),
  ringkasan masuk chat sebagai blok terstruktur + pesan sistem.
- Batas: max 3 putaran analisa per sesi (P-7); putaran ke-4 ditolak —
  agent harus submit.

### Keputusan tool T-1..T-4 + O-21/O-22/O-25 (DIKUNCI owner 2026-11-19)
- T-1..T-4 mengikuti pola tool strict existing; Node hanya skema + forward.
- **O-21 DIKUNCI**: max 500 baris per petikan analisa; lebih dari itu
  dipecah bertahap oleh agent (bukan satu kirim raksasa).
- **O-22 DIKUNCI**: approve cukup `ingest.write` (sama dengan editor;
  tanpa permission baru).
- **O-25 DIKUNCI**: listing/snapshot analisa PERSISTEN ringan — jsonb
  per sesi (plugin ingest), tampil lagi setelah reload; dibersihkan
  bersama retensi sesi (WO-10 topic induk, saat itu jadi kebijakan).

## 3. Listing rejected di editor (baru — hasil iterasi owner)

Panel/tabel **"Baris Rejected"** di `IngestMappingPage` (mapping draft):

- Kolom: `☑` checkbox (default P-8) · RowNo · kolom bermasalah ·
  nilai terdeidentify · `violates` · usulan AI · status
  (menunggu/diusulkan/diedit user/disetujui/terlambat perlu manusia).
- **Edit manual inline** per baris → masuk proposal yang sama dengan
  penanda `edited_by: user` — jalur manual & jalur AI bermuara ke
  SATU proposal store.
- Tombol **"Analisa (N)"** → simpan scope baris tercentang ke snapshot
  (analyze) → tutup editor → auto-submit pesan basa-basi di chat (P-9);
  N di tombol.
- Tombol **"Setujui"** per baris/kategori/semua → `apply_fixes`.
- **Dua arah chat ↔ listing**: tombol per baris/kelompok "perbaiki via
  chat" → buka sesi chat dengan konteks baris (blok terstruktur
  `rejected_listing` dibuat BACKEND Go — pola `editor_link` WO-04,
  bukan teks LLM); dari chat, link kembali menyorot baris di listing.
- Regenerasi: setiap apply/apply-edit → re-scan otomatis → listing
  ter-update dari snapshot terbaru (persisten ringan, O-25).

## 4. Aturan yang TIDAK berubah (paten bawaan)

1. Agent tidak pernah menulis master data/profil langsung — semua lewat
   tool = API Go (RBAC + audit).
2. Rule hanya dari katalog `exceptions.yaml` segmen — `RuleSet.Validate`
   sudah menolak rule sintetis (fail-closed, gratis dari mesin existing).
3. Konteks ke LLM/listing selalu terdeidentify (P-5).
4. Node tidak pernah menyentuh DB — hanya skema + forward.
5. Dry run wajib di evaluasi ulang (P-4).
6. AI dilarang mengarang nilai field wajib yang kosong (P-8) —
   fail-closed di penegak Go.
7. Max 3 putaran analisa; putaran ke-4 wajib `submit_analysis` (T-4).

## 5. Sisa terbuka

- SPEC + pecah WO disusun Manager berikutnya (usulan pembagian:
  WO-01 backend snapshot+tool T-1/T-2/T-4 + fail-closed P-8;
  WO-02 listing UI + checkbox + edit manual + tombol analisa;
  WO-03 apply T-3 + berkas perbaikan + re-scan loop + blok chat).
- Kasus uji pertama: 144 pelanggaran `visit_visitors.phone_raw.lenient_format`
  (isu data nyata dari uji owner 2026-11-19).
