---
title: work order
description: 'Master index WO plugin ingest — pendekatan checkpoint (partial: CP-1 saja; WO checkpoint berikutnya menyusul)'
status: draft
created: "2026-11-03"
updated: "2026-11-19"
---
# Master Index — Work Order Plugin `ingest` (checkpoint)

Sumber desain: `spec.md` (D-1..D-9 terkunci, pendekatan checkpoint bagian 9) +
`spec-techstack.md` (baseline & dependensi baru).

**Pendekatan (keputusan owner):** WO TIDAK langsung on-target. Empat checkpoint;
setiap checkpoint punya milestone yang bisa dites sebelum lanjut. File WO dibuat
**partial** — hanya checkpoint berikutnya yang dirinci; WO checkpoint selanjutnya
baru dibuat setelah checkpoint sebelumnya selesai (agar pembelajaran dari tahap
sebelumnya masuk ke rancangan). **Pengecualian (keputusan owner, 2026-11-04):**
CP-1 dipecah menjadi tiga sub-WO (WO-01a/b/c) yang semuanya dirinci sekarang,
karena CP-1 adalah checkpoint belajar tanpa data bisnis — risikonya rendah dan
milestone-nya sudah jelas sejak awal. **Pengecualian kedua (2026-11-04):** CP-2
ditutup sebagian — inti (0/A/B/C/D + compact) selesai & teruji dengan data
sintetis; sisa AC-4 (penegakan rules), AC-8 (idempotensi re-impor), dan uji
berkas asli BBH dibawa ke CP-3/WO-03 sebagai prasyarat verifikasi, karena
butuh berkas asli dari owner & keputusan desain rules. **Pengecualian
ketiga (2026-11-06):** delete sesi (bagian housekeeping WO-4 §D) dipindah
jadi WO-04a — patch prioritas owner, dieksekusi SEBELUM CP-4 (kebutuhan nyata
saat uji: daftar sesi penuh); sisa WO-4 tetap CP-4.

**Pembaruan (2026-11-19):** hasil analisa `analisa-mapping-multi-segmen.md`
(opsi **B+** disetujui owner) melahirkan **WO-06 — segmentasi sesi eksplisit**;
temuan owner sesi `extracting` macet melahirkan **WO-07 — penanganan sesi
stale**. Keduanya complete 2026-11-19 dan berada di luar checkpoint. Kandidat
segmen ingest berikutnya: `resident` -> `sec`/`hsh` (katalog field per plugin
menyusul). **WO-08 dibuat & COMPLETE** (2026-11-19) — D-R1 dikunci
owner (dedup `NIK` -> `nama + tanggal_lahir` -> none, mesin dedup dipakai
ulang dari visit); D-R2/D-R3 default Manager aktif.
**WO-09 dibuat** (2026-11-19, pending) — registrasi segmen `sec` + `hsh`
ke dispatcher; pipeline import kedua plugin SUDAH ADA, kerja utama adapter
+ exceptions minimal; D-S1/D-S2 dikunci 2026-11-19 (owner "ok"); SIAP EKSEKUSI.
**WO-10 dibuat** (2026-11-19, pending) — housekeeping & retensi sesi
(bagian D WO-04 dipisah, pola WO-04a); menunggu keputusan O-14 (D-T1).
**WO-11 dibuat** (2026-11-19) dan **COMPLETE (2026-11-19)** — konfirmasi
tebakan agent di editor: tombol "Setuju semua tebakan" + konfirmasi per baris
(dari temuan uji owner pasca WO-09; auto-confirm oleh sistem DITOLAK — D-7
tetap, konfirmasi tetap aksi user eksplisit via PUT mapping). §B backend
ternyata TIDAK perlu (kontrak v2 sudah punya `guess`+`validated_by`) → murni
frontend, satu berkas. Commit `a3a30d1` lokal (belum push).

| Checkpoint | Isi | Status | Bukti |
|---|---|---|---|
| CP-1 | Setup CopilotKit + proxy + sandbox belajar — **dipecah 3 sub-WO**: 01a proxy+chat → 01b tool+upload → 01c MCP+housekeeping | **complete** (2026-11-04) | e4f4ea9, a019c26, a22c03b, f00e542, 397fbff |
| CP-2 | Real case pertama `visit` via chat (tanpa editor) | **complete (2026-11-06)** — §0.1 berkas asli LULUS pasca fix bug sesi-lazy (`743e183`) | 032941e, 743e183 |
| CP-3 | Excel mapping editor + rules O-20 + prasyarat sisa CP-2 | **complete (2026-11-06)** — §0.1 LULUS; bug sesi-lazy difix + 3 test regresi | 2f7cf07, 743e183 |
| CP-4 | Integrasi editor ↔ CopilotKit + housekeeping | TERBUKA — WO-04 rancangan final; **satu-satunya prasyarat tersisa: keputusan O-14**; wajib sertakan verifikasi ulang percakapan UI end-to-end (catatan review §0.1) | — |

## Daftar WO aktif

| WO | Judul | Status | Commit kode | Dok historic |
|---|---|---|---|---|
| WO-01a | CopilotKit chat hidup (CP-1) — **DIREVISI D-11** | complete (2026-11-04) | e4f4ea9, a019c26, a22c03b | — |
| WO-01b | Tool dummy `sandbox_ping` + upload ke storage (CP-1) | complete (2026-11-04) | f00e542 | — |
| WO-01c | Eksperimen MCP + housekeeping (CP-1) | complete (2026-11-04) — CP-1 DITUTUP | 397fbff | a017ba2 |
| WO-02 | Sesi strict + real case `visit` via chat (tanpa editor) — **DIREVISI pasca CP-1** | complete (2026-11-06) | 032941e, 743e183 |
| WO-03 | Excel mapping editor — §0 prasyarat sisa CP-2 + §D rules O-20 | complete (2026-11-06) | 2f7cf07, 743e183 | — |
| WO-04a | Delete sesi ingest (soft/hard/restore/bulk) — **dipindah dari WO-04, PRIORITAS owner** | complete (2026-11-06) | 9040b39 | — |
| WO-05 | Custom provider LLM via UI admin (key terenkripsi di DB, fallback env; di luar checkpoint) — **fitur baru permintaan owner** | **complete (2026-11-09)** — review Manager LULUS; tambahan `/deactivate` disetujui; log implementasi 29 entri; AC-1..AC-7 berbukti | 181e416 | — |
| WO-06 | Segmentasi sesi eksplisit (opsi B+ analisa multi-segmen): picker segmen saat buat sesi + badge + guard kecocokan berkas — **usulan revisi UX TERTUNDA** | **complete (2026-11-19)** — §A→§C selesai; Bruno 44–53 + Playwright smoke hijau | — (belum commit) | wo-06-segmen-sesi-eksplisit.md |
| WO-07 | Penanganan sesi `extracting` stale: penanda run + deteksi stale + cancel & delete (temuan owner sesi `01a1a5d6-…`) | **complete (2026-11-19)** — §A+§B selesai; AC-1..AC-7 berbukti; belum di-push | — (belum commit) | wo-07-penanganan-extracting-stale.md |
| WO-08 | Segmen baru: `resident` — porting pipeline import (staging 2-fasa + gates + dedup + PII) + generalisasi dispatcher ekstraksi; D-R1..R3 dikunci; replikasi pola visit | **complete (2026-11-19)** — approve owner; D-R4: birth_date tanpa encrypt; commit `ac3ca5e` lokal (belum push) | — | wo-08-segmen-resident.md |
| WO-09 | Segmen baru: `sec` + `hsh` (kebutuhan) — registrasi dispatcher (pipeline import kedua plugin sudah ada, semantik upsert) + exceptions minimal; D-S1/D-S2 dikunci | **complete (2026-11-19)** — approve owner; commit `9f78ea1` lokal (belum push); swagger regenerasi menyusul | — | wo-09-segmen-sec-hsh.md |
| WO-10 | Housekeeping & retensi sesi ingest — job periodik (90/30 hari, profil extracted imun, env konfigurabel, audit); **D-T1 (approve O-14) menunggu** | pending (2026-11-19) | — | wo-10-housekeeping-retensi.md |
| WO-11 | Konfirmasi tebakan agent di editor — tombol "Setuju semua tebakan" + konfirmasi per baris; TANPA auto-confirm (D-7 tetap, aksi user via PUT mapping) | **complete (2026-11-19)** — murni frontend (§B backend tidak perlu); AC-1..AC-8 berbukti via API + UI; commit `a3a30d1` lokal (belum push) | a3a30d1 | wo-11-konfirmasi-tebakan-editor.md |
| WO-04 | Integrasi editor ↔ CopilotKit + housekeeping (CP-4 final) — **DIREVISI pasca CP-2/CP-3** | pending (rancangan final; eksekusi setelah keputusan O-14) | — | — | — |

**Aturan lintas WO (pinned):**

- Kode fitur hanya di `plugins/ingest/` + UI di `sub_app/webapp/src/pages/admin/ingest/`;
  migrasi permission di plugin `auth` (fail-closed).
- **(D-11 final, 2026-11-04)** Runtime Node CopilotKit = **backend chat AI penuh**
  (satu-satunya pemanggil provider LLM; key hanya di Node); **Go = data service** —
  semua keputusan bertanggung jawab tetap di Go: tool calls + state machine +
  deidentify (Node meminta payload aman dari Go) + audit (`ingest_audit` pusat).
  Node TIDAK pernah menulis profil. N-17 dilonggarkan khusus service
  `ingest-runtime`. Proxy Go OpenAI-compatible keluar dari jalur chat (nasib akhir
  diputuskan di WO-01c).
- Strictness = state machine backend Go + system prompt agent; CopilotKit hanya
  lapisan UI + transport; tool-calling klien maupun Node tidak pernah menulis profil
  langsung — semua lewat action API Go yang di-guard RBAC.
- Dependensi baru wajib direkam di `spec-techstack.md` (kini: `deidentify` di Go,
  CopilotKit `react-core`+`runtime` di frontend/Node, service `ingest-runtime`).
- **(KEPUTUSAN owner, 2026-11-06) Log Implementasi WAJIB per WO.** Setiap Dev
  Agent menulis **log implementasi codingnya di dok WO terkait** — bukan di chat
  terpisah. Tujuan: junior bisa menelusuri kenapa kode jadi begini tanpa membaca
  seluruh riwayat chat. Ketentuan:
  1. Format: section `## Catatan Implementasi (Dev)` di bagian paling bawah dok WO.
  2. Isi per entri: `berkas:baris` kunci (mis. `services/extraction.go:88-120`) +
     **yang dilakukan** + **kenapa** (referensi keputusan D-x/O-x atau gotcha).
  3. Hanya baris KUNCI (keputusan, gate, fail-closed, jebakan) — ±10–20 entri per
     WO; bukan diary setiap baris.
  4. Diisi bertahap per bagian (§A, §B, ...) saat bagian itu selesai; WO sudah
     selesai (WO-01a/b/c, WO-02, WO-03) di-retrofill dari kode existing.
  5. Masuk laporan dev ke Manager: log terkini ikut disebut, Manager review sebagai
     bagian dari checklist laporan.
  **Status retrofill: COMPLETE (2026-11-06, review Manager lulus, commit dok
  `5dfb4f5`)** — 5 dok WO terisi (01a:11, 01b:11, 01c:8, 02:15, 03:18 entri);
  WO-04 akan diisi bertahap per bagian saat eksekusi.
