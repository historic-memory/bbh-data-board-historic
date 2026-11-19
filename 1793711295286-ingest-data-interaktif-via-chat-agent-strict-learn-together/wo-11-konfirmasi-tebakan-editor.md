---
title: "WO-11 — Konfirmasi tebakan agent di editor: aksi eksplisit sekali klik (bulk + per baris)"
description: "Perbaikan UX konfirmasi mapping editor: tebakan agent (guess:true) kini hanya bisa dikonfirmasi dengan mengubah dropdown ulang satu per satu — ditambah tombol Setuju semua tebakan + konfirmasi per baris; TETAP aksi user eksplisit via PUT mapping, TANPA auto-confirm oleh sistem (D-7 gate strict tetap)"
status: pending
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - mapping-editor
    - frontend
---
# WO-11 — Konfirmasi tebakan agent di editor (aksi eksplisit sekali klik)

Sumber: temuan uji owner 2026-11-19 (uji UI pasca WO-09 complete + rebuild
server): semua tebakan agent sudah benar, tapi user HARUS mengubah dropdown
ulang satu per satu untuk mengubah `guess: true → false` — gate Konfirmasi
& Mapping kolom tertahan tanpa cara yang ringkas. Analisa visual Manager
(2026-11-19): **perilaku by design, UX-nya buruk** — ada gap antara
"tebakan benar" dan "aksi konfirmasi".

## Kondisi saat ini

- Tebakan agent disimpan dengan flag `guess: true` (WO-03 §A, profil v2).
- Gate `evaluateGates` menghitung gate "Mapping kolom" tertahan selama ada
  kolom ber-flag `guess: true` — fail-closed, benar.
- Satu-satunya jalur UI untuk meng-unflag = user memilih ulang nilai
  dropdown kolom tersebut (PUT `/sessions/:id/mapping` memperlakukan
  pilihan user sebagai keputusan manusia).
- Real case: 6 kolom semuanya `guess: true` padahal isinya sudah benar →
  6 interaksi manual yang tidak menambah informasi apa pun.
- **Auto-confirm oleh sistem DILARANG** (D-7, kontrak strict 5 gate):
  kebenaran tebakan harus berasal dari KEPUTUSAN manusia, bukan
  heuristik "data match".

## Tujuan

Memperpendek langkah konfirmasi dari N interaksi menjadi 1, TANPA
melemahkan gate strict: konfirmasi tetap aksi user eksplisit yang ditulis
lewat API mapping yang sama.

## Lingkup

### A. Frontend (`IngestMappingPage.tsx`) — porsi utama

1. **Tombol "Setuju semua tebakan"** (bulk): muncul bila ada ≥1 kolom
   `guess: true`; satu klik = semua kolom tebakan dikirim ulang ke
   `PUT /sessions/:id/mapping` dengan nilai yang sama + flag tebakan
   dilaporkan sebagai keputusan user (tetap SATU request `PUT` mapping,
   bukan endpoint baru).
2. **Konfirmasi per baris**: indikator/kontrol kecil (mis. ✓) di baris
   yang `guess: true` untuk menyetujui satu kolom saja tanpa menyentuh
   dropdown.
3. **Label jelas**: badge "tebakan — belum dikonfirmasi" tetap tampil
   sampai user mengonfirmasi; tombol bulk menjelaskan efeknya ("menandai
   semua tebakan agent sebagai keputusan Anda").
4. **Gate & confirm tidak diubah** — setelah semua tebakan dikonfirmasi,
   gate Mapping kolom lolos seperti sekarang; tombol konfirmasi ekstraksi
   tetap tampil seperti biasa.

### B. Backend (porsi kecil, hanya bila perlu)

1. `SaveMapping` saat ini menerima seluruh daftar kolom — bulk confirm
   cukup kirim ulang daftar sama dengan penanda keputusan user. Bila
   kontrak v2 butuh field eksplisit (mis. `confirmed_by_user` per kolom),
   tambahkan secara additive (append-only profil tetap berlaku).
2. TIDAK ada endpoint confirm otomatis, TIDAK ada perubahan state machine,
   TIDAK ada perubahan gate.

### C. Uji

1. Playwright/Tailscale: sesi dengan tebakan agent → tombol bulk satu
   klik → gate Mapping kolom lolos → confirm → extract tetap jalan.
2. Regresi: editor tanpa tebakan (user mapping manual) tidak berubah.
3. `go build/vet/test` hijau bila backend tersentuh; `tsc --noEmit` +
   `npm run build` hijau.

## Acceptance Criteria

- AC-1 Tombol "Setuju semua tebakan" mengonfirmasi SEMUA kolom
  `guess: true` dalam satu aksi; gate Mapping kolom langsung lolos.
- AC-2 Konfirmasi per baris tersedia untuk kolom tunggal.
- AC-3 Konfirmasi TETAP menghasilkan `PUT mapping` oleh user (aksi
  eksplisit) — tidak ada jalur auto-confirm oleh sistem (D-7 tetap).
- AC-4 Nilai kolom TIDAK berubah oleh aksi konfirmasi (nilai tebakan
  yang benar dipertahankan).
- AC-5 Profil append-only tetap berlaku; riwayat versi tidak hilang.
- AC-6 RBAC tetap (`ingest.write`); audit tetap.
- AC-7 Build hijau + uji UI terbukti.
- AC-8 **Log Implementasi (Dev)** di dok ini, bertahap per bagian.

## Non-tujuan

- Auto-confirm otomatis oleh sistem saat "data match" — DITOLAK di
  analisa 2026-11-19 (bertentangan D-7); jika owner kelak berubah pikiran,
  butuh keputusan eksplisit yang merevisi kontrak strict, bukan diam-diam
  di WO ini.
- Penanganan 144 pelanggaran `phone_raw.lenient_format` (gate Aturan) —
  masalah DATA terpisah, bukan UX konfirmasi.
- Perubahan state machine, gate `evaluateGates`, atau pipeline ekstraksi.

## Risiko & catatan

- Karena UI adalah satu-satunya penanda "keputusan user", konsistensikan
  juga jalur chat: setelah user konfirmasi di editor, konteks agent
  tidak boleh menebak ulang (C-4 — profil satu, sumber kebenaran Go).
- Jangan ubah makna `guess` di backend untuk kepentingan UI — flag tetap
  "tebakan belum disetujui manusia".

## Rancangan Manager — 2026-11-19

| Bagian | Isi | Pemilik dominan |
|---|---|---|
| §A | Bulk confirm + per baris + label | Dev frontend |
| §B | Penanda keputusan user (additive bila perlu) | Dev backend |
| §C | Uji UI + regresi | Dev |

Effort kasar: §A ±0,5 hari; §B ±0,25 hari; §C ±0,5 hari.

## Catatan Implementasi (Dev)

(belum ada — isi bertahap per bagian sesuai aturan pinned)
