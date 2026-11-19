---
title: "WO-06 — Segmentasi sesi eksplisit (satu sesi per segmen, konteks ke AI)"
description: "Opsi B+ dari analisa multi-segmen: user memilih segmen (modul target) secara eksplisit saat membuat sesi, segmen tampil jelas di editor & chat, AI selalu tahu konteks segmen, plus guard kecocokan berkas vs segmen"
status: draft
created: "2026-11-19"
updated: "2026-11-19"
tags:
  - ingest
  - mapping
  - frontend
---
# WO-06 — Segmentasi sesi eksplisit

Sumber: `analisa-mapping-multi-segmen.md` (keputusan desain 2026-11-19, opsi
**B+** — disetujui owner). Prasyarat: CP-3 (WO-03) complete.

## Tujuan

Membuat pilihan segmen (modul target ingest) menjadi **eksplisit dan
terlihat**: user memilih segmen saat membuat sesi dan mengurusi sesi itu
sampai selesai; map editor dan chat selalu menampilkan konteks segmen;
AI/agent selalu tahu "ini sesi segmen X" dan tidak menebak segmen dari isi
berkas. Backend model mapping **tidak berubah** — `target_plugin` per sesi
(`plugins/ingest/model/session.go:116`) sudah ada; kerja utama di UX dan
prompt agent.

## Lingkup

### A. Backend (plugin `ingest`) — porsi kecil

1. **Daftar segmen untuk picker**: endpoint/kuota yang menyediakan daftar
   modul yang bisa jadi target ingest (pola `MappingOptions(plugin)` —
   tinggal enumerasi plugin yang teregistrasi; minimal `visit`, siapkan
   `resident`/`sec` menyusul tanpa perubahan skema).
2. **Sesi lahir dengan segmen eksplisit**: pembuatan sesi (dan alur
   `EnsureSession` dari upload) menerima/terikat `target_plugin` yang
   dipilih user; **tidak boleh default diam-diam**.
3. **Segmen terkunci setelah sesi berjalan**: ganti `target_plugin`
   ditolak (409) bila sesi sudah punya berkas/profil mapping — ganti
   segmen = buat sesi baru (konsisten invarian satu-berkas/sesi).
4. **Guard kecocokan berkas vs segmen** (fail-open, hanya peringatan):
   saat upload/mapping, bandingkan header berkas dengan katalog field
   segmen; bila tidak ada kecocokan yang layak, lampirkan warning di
   `MappingView`/safe-payload — **tidak menolak upload** (keputusan tetap
   di manusia).

### B. Frontend (webapp)

1. **Picker segmen di awal flow** (`IngestSessionListPage` / alur buat
   sesi): user memilih "Segmentasi" (mis. Kunjungan/Keuangan/Bayi) sebelum
   atau saat membuat sesi; label terlihat di kartu sesi.
2. **Badge segmen di editor & chat**: `IngestMappingPage.tsx` dan
   `IngestChatPage.tsx` menampilkan segmen sesi secara jelas (header/badge),
   termasuk saat warning guard aktif.
3. **Warning guard di editor**: tampilkan pesan "kolom berkas tidak cocok
   dengan segmen X — periksa segmen atau buat sesi terpisah" berbasis data
   dari backend (§A.4), bukan tebakan UI.

### C. Agent/prompt (Node + prompt)

1. **Konteks segmen selalu di prompt**: safe-payload/tool
   (`get_mapping_guess` dll.) menyertakan segmen sesi secara eksplisit;
   katalog field yang dikirim hanya milik segmen itu (sudah begitu —
   dipertahankan dan diuji).
2. **Instruksi agent** (`sub_app/ingest_runtime/` + `plugins/ingest/prompt/agent.md`):
   bila berkas tampak berisi segmen lain/campur, agent **menyarankan
   membuat sesi terpisah per segmen**, tidak mengarang mapping lintas
   modul.

## Acceptance Criteria

- AC-1 Sesi baru selalu punya segmen eksplisit pilihan user; tanpa pilihan
  → pembuatan/upload gagal atau terhalang di UI (tidak ada default diam-diam).
- AC-2 Ganti `target_plugin` pada sesi yang sudah punya berkas/profil → 409.
- AC-3 Badge segmen tampil di halaman daftar sesi, editor, dan chat.
- AC-4 Warning kecocokan berkas vs segmen muncul di editor berbasis data
  backend (upload tidak ditolak); tidak muncul bila cocok.
- AC-5 Agent menerima konteks segmen di payload; ketika berkas campur,
  agent menyarankan sesi terpisah (terbukti di percakapan/dialog).
- AC-6 Katalog field yang terlihat agent & editor tetap hanya milik segmen
  sesi (regresi `ErrUnknownTarget` tetap fail-closed).
- AC-7 RBAC tetap: `ingest.read`/`ingest.write` + audit; tanpa token 401.
- AC-8 `go build/vet/test` hijau; `npx tsc --noEmit` + `npm run build`
  hijau; swagger + Bruno untuk endpoint segmen; Playwright smoke via
  Tailscale (`BASE_URL=http://100.125.175.34:5173`) untuk picker + badge.

## Non-tujuan

- Mapping multi-segmen dalam satu sesi (opsi A/C) — dicadangkan di analisa,
  tidak dieksekusi di WO ini.
- Perubahan model mapping v2, state machine, atau pipeline ekstraksi.
- Penambahan plugin segmen baru di luar yang sudah teregistrasi.

## Risiko & catatan

- **Jangan biarkan picker jadi pintu masuk sesi "tanpa segmen"** —
  fail-closed di pembuatan sesi lebih penting daripada kenyamanan.
- Guard kecocokan hanya peringatan: heuristik header bisa salah; jangan
  pernah menolak upload otomatis (kasus berkas asli BBH pun header bebas).
- Agent tetap tidak boleh mengarang rule/mapping lintas segmen — instruksi
  prompt diperkuat, tapi kebenaran tetap divalidasi Go (C-4 tetap).
- Koordinasi dengan alur `EnsureSession` (bug sesi-lazy 2026-11-06): sesi
  hasil upload WAJIB ikut mendapat segmen eksplisit, bukan hanya sesi yang
  dibuat manual.

---

## Rancangan Manager — 2026-11-19 (disetujui owner "ok kita buat wo nya")

| Bagian | Isi | Pemilik dominan |
|---|---|---|
| §A | Endpoint segmen + sesi bersegmen eksplisit + guard | Dev backend |
| §B | Picker segmen, badge, warning di editor/chat | Dev frontend |
| §C | Konteks segmen di payload + instruksi agent | Dev Node/prompt |

Urutan eksekusi disarankan: §A → §B → §C. Effort kasar: §A ±0,5–1 hari;
§B ±1 hari; §C ±0,5 hari.
