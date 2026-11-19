---
title: "WO-11 — Konfirmasi tebakan agent di editor: aksi eksplisit sekali klik (bulk + per baris)"
description: "Perbaikan UX konfirmasi mapping editor: tebakan agent (guess:true) kini hanya bisa dikonfirmasi dengan mengubah dropdown ulang satu per satu — ditambah tombol Setuju semua tebakan + konfirmasi per baris; TETAP aksi user eksplisit via PUT mapping, TANPA auto-confirm oleh sistem (D-7 gate strict tetap)"
status: complete
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

Status: selesai (2026-11-19). Porsi backend §B ternyata **tidak perlu** —
kontrak mapping v2 sudah punya `guess` + `validated_by`, dan `PUT mapping`
menyimpan apa adanya. Jadi perubahan murni frontend, satu berkas.

- `sub_app/webapp/src/pages/admin/ingest/IngestMappingPage.tsx:188` —
  tambah `guessPendingCount` (predikat `guess && !validated_by && !skip`).
  Sengaja **disamakan** dengan penegak gate di
  `plugins/ingest/services/mapping.go` (`evaluateGates`, gate "mapping") agar
  tombol bulk muncul tepat ketika gate itu tertahan — tidak ada drift UI↔gate.
- `IngestMappingPage.tsx:200` — `persistColumns(next, notice)` mengekstrak jalur
  simpan yang sudah ada (`saveMapping` → `PUT /sessions/:id/mapping`). `save`
  draft, konfirmasi per baris, dan bulk kini lewat jalur API yang sama —
  bukti AC-3 (konfirmasi tetap aksi user `PUT mapping`), bukan endpoint baru
  maupun auto-confirm sistem (D-7 tetap).
- `IngestMappingPage.tsx:219` — `save` lama diubah jadi delegasi
  `persistColumns(payloadColumns, 'Mapping tersimpan sebagai versi baru.')`
  supaya perilaku Simpan draft tidak berubah (regresi nol).
- `IngestMappingPage.tsx:228` — `confirmGuess(index)` (AC-2): hanya set
  `guess:false` + `validated_by:'admin'` pada satu kolom. `target` **tidak
  disentuh** → nilai tebakan dipertahankan (AC-4); `chooseTarget` (jalur
  dropdown manual) tetap seperti semula sehingga editor tanpa tebakan tidak
  berubah.
- `IngestMappingPage.tsx:243` — `agreeAllGuesses()` (AC-1): map semua kolom
  pending → `guess:false` + `validated_by:'admin'`, lalu satu
  `persistColumns`. Nilai kolom dipertahankan (AC-4).
- `IngestMappingPage.tsx:458` — tombol "Setuju semua tebakan (N)" di
  `CardHeader` kartu kolom; hanya render bila `guessPendingCount > 0`, disabled
  saat `busy !== ''`, `title` menjelaskan efek ("nilai kolom tidak berubah"),
  `data-testid="ingest-agree-all-guesses"` untuk uji. Label N = jumlah tebakan
  tersisa, jadi user melihat progres.
- `IngestMappingPage.tsx:~505` — tombol ✓ per baris di sel "Field tujuan",
  hanya render untuk kolom `guess && !validated_by && !skip`; `aria-label`
  menyebut nama kolom (aksestabilitas) dan `data-testid="ingest-confirm-guess-{i}"`.
- Badge "tebakan — belum dikonfirmasi" **tidak diubah** — tetap satu-satunya
  penanda tebakan sampai user menyetujui (§A.3).
- §B backend: **tidak ada perubahan**. Alasan: `saveMapping` mengirim seluruh
  daftar kolom dan `SaveMapping` (`services/mapping.go`) menyimpan dokumen apa
  adanya tanpa men-stempel ulang `guess`; field `validated_by` sudah ada di
  `model.ColumnMapping`. Menambah `confirmed_by_user` akan menduplikasi makna
  (additive tanpa manfaat) → ditolak, sesuai catatan WO ("hanya bila perlu").
- AC-6 jalur RBAC/audit tetap: route `PUT /admin/ingest/sessions/:id/mapping`
  guard `ingest.write` (`plugin.go:118`) + `h.audit(c,"mapping",...)`
  (`handlers/mapping.go:101`); **nol route baru** ditambahkan.

### Revisi UX (temuan uji owner 2026-11-19)

Owner mengira tombol `✓` polos = status "sudah di-ACC" (affordance salah):
diklik sekali lalu tombol+badge hilang, dan disangka bisa di-uncheck. Perbaikan
(murni frontend, tanpa ubah perilaku/gate):

- `IngestMappingPage.tsx:549` — tombol per baris diberi **label teks**
  "✓ Setujui" (bukan ikon telanjang) + `title`/`aria-label` menjelaskan bahwa
  nilai kolom tidak berubah. Menghilangkan kesan "checkbox status".
- `IngestMappingPage.tsx:506-514` — **umpan balik positif**: baris yang sudah
  disetujui menampilkan badge hijau "✓ disetujui" (`data-testid="ingest-guess-accepted-{i}"`)
  alih-alih sekadar menghilang. Badge amber "tebakan — belum disetujui" tetap
  tampil selama masih tebakan.
- `IngestMappingPage.tsx:455` — teks bantuan kartu diperjelas: tebakan WAJIB
  disetujui; pilih field yang benar, atau klik **Setujui** bila sudah tepat.
- `IngestMappingPage.tsx:491,571` — kolom checkbox "Enkripsi" diberi sublabel
  "proteksi PII" di header dan label per baris "wajib (PII)" / "tidak wajib"
  (sebelumnya "wajib"/"opsional" yang mudah disalahpahami sebagai konfirmasi)
  + `title` per checkbox.

Terbukti UI (`:8082`, sesi `visit` 6 kolom tebakan): tombol tampil "✓ Setujui"
×6; klik satu → badge amber baris itu berganti "✓ disetujui", tombol baris
hilang, tombol bulk turun "(6)"→"(5)", tepat 1 `PUT mapping`. `npx tsc -b` +
`npm run build` hijau.

### Bukti uji

- API (curl, sesi `visit` hasil upload, 6 kolom `guess:true`): GET mapping →
  gate mapping `tertahan`; PUT mapping dengan `guess:false`+`validated_by:admin`
  (nilai `target` sama) → gate mapping `lolos`, `target` seluruh kolom
  identik (AC-1/AC-3/AC-4). `ingest_audit` mencatat `PUT .../mapping` 200
  (AC-6).
- UI (Vite `:5173`, Tailscale `100.125.175.34`, root admin): halaman
  `/app/ingest/:id/map` menampilkan badge 6 kolom + tombol "Setuju semua
  tebakan (6)" + 6 tombol ✓.
  - Klik ✓ baris-1 → **satu** `PUT mapping` (AC-2), badge hilang 1, tombol
    jadi "(5)", gate mapping detail menyempit.
  - Klik "Setuju semua tebakan (5)" → **satu** `PUT mapping` (AC-1), badge 0,
    gate mapping `lolos`, tombol bulk hilang.
  - Konfirmasi → gate semua lolos → Ekstrak → `extracted` ("Ekstraksi
    selesai") → pipeline tak berubah (AC-7).
- Build: `npx tsc -b` bersih + `npm run build` hijau (dist 22.95s). Backend
  tidak disentuh sehingga `go build/vet/test` tak terpengaruh.
- Data uji dibersihkan (hard delete 2 sesi uji; tidak ada master data baru —
  laporan ekstraksi `created=0/skipped=2`).

