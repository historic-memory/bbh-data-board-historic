---
title: WO-03 — Excel mapping editor (berdiri sendiri)
description: "CP-3: UI editor mapping kolom (React, tanpa library), flag is_encrypt, preview, tombol Ekstrak terkunci; API profil mapping; skema+panel rules (O-20); prasyarat verifikasi sisa CP-2"
status: pending
created: "2026-11-03"
---
# WO-03 — Excel Mapping Editor, Berdiri Sendiri (Checkpoint 3)

Checkpoint: **CP-3**. Prasyarat: CP-2 (WO-02) sebagian complete — inti hidup
(state machine, deidentify, ekstraksi via chat, compact O-18); **sisa verifikasi
CP-2 dibawa ke checkpoint ini** (bagian 0 di bawah). Sumber: `spec.md` §6.2
(editor), D-5/D-8; `spec-techstack.md` §2.5; O-20 (skema rules, 2026-11-04).

## Tujuan

Menyediakan UI visual untuk menyunting mapping kolom — menggantikan penulisan mapping
lewat teks di chat. Pada checkpoint ini editor **berdiri sendiri** (bisa dipakai tanpa
chat, O-9); integrasi dua arah dengan CopilotKit baru di CP-4. Juga menutup Gate
Aturan (AC-4 CP-2) lewat skema rules O-20 + penegakan Go.

## Lingkup

### 0. Prasyarat verifikasi sisa CP-2 (dikerjakan PERTAMA)

Sisa CP-2 yang butuh berkas/keputusan — sekarang sudah lengkap (O-20 menjawab
AC-4), tinggal eksekusi verifikasi:

1. **Uji berkas asli BBH** (177 baris tanpa `phone_e164`) via UI chat: pastikan
   agent strict benar-benar berhenti di konfirmasi ekstraksi dan MENANYAKAN
   kebijakan dedup baris tanpa HP (jangan memutus sendiri). Butuh berkas asli
   dari owner.
2. **AC-8 CP-2** — re-impor berkas sama 2× dengan kebijakan dedup sama → idempoten.
   Sekaligus jadi data uji AC-R2/penegakan rules.
3. Catat hasil (laporan dev + bukti chat/audit) di dok WO-02 sebelum menyentuh
   editor — prasyarat lulus.

> **⚠ Data nyata:** deidentify harus aktif sebelum berkas asli disentuh
> (sudah terbukti CP-2 dengan data sintetis — tetap jalankan ulang pengecekan
> 503 fail-closed saat verifikasi).

### A. API profil mapping (backend plugin `ingest`)

- `GET /api/admin/ingest/sessions/:id/mapping` — kolom terdeteksi + tebakan agent
  + sample ±5 baris + status konfirmasi per kolom.
- `PUT /api/admin/ingest/sessions/:id/mapping` — simpan mapping hasil editan
  (source column → target field | `skip`, tipe, flag `is_encrypt`, aturan/pengecualian).
  Tidak boleh melonggarkan policy bawaan `fields.yaml` (D-5): attempt melonggarkan →
  400; flag is_encrypt hanya menambah proteksi.
- `POST .../mapping/confirm` — transisi state ke `mapping_confirmed` (fail-closed:
  kolom belum mapped/skip, atau aturan belum dijawab → 400 dengan daftar gate yang
  belum lolos).
- Validasi server-side SEMUA (UI tidak dipercaya): field target valid, tipe valid,
  kolom ganda tidak boleh memetakan ke field yang sama (kecuali di-deidentify konsisten).
- Guard `ingest.read` (GET) / `ingest.write` (PUT/confirm) + audit.

### B. UI mapping editor (React, tanpa library eksternal — D-8)

- Rute `/ingest/:id/map` (grup nav "Ingest"); komponen dari `components/ui/*` +
  Tailwind v4 (pola halaman `secondary/`/`visits/`).
- Layout: kiri = kolom Excel/CSV terdeteksi (nama header + contoh nilai), kanan =
  dropdown target field dari `fields.yaml` plugin target + flag `is_encrypt` + opsi
  `skip` + label "tebakan agent — belum dikonfirmasi" untuk mapping otomatis.
- Preview panel: hasil mapping ±5 baris (nilai setelah deidentify di-mask preview).
- Status gate: indikator per gate (identitas/mapping/aturan/konfirmasi); tombol
  **"Ekstrak" terkunci** sampai semua gate lolos (disabled + tooltip alasan).
- Sesi state tetap dibaca dari backend — UI hanya merender (state machine tetap
  penegak tunggal).

### C. Keterhubungan dengan chat (satu arah di CP-3)

- Sesi yang dibuat via chat (CP-2) bisa dibuka di editor untuk mengubah mapping —
  profil yang sama, bukan salinan terpisah.
- Sesi yang dibuat langsung dari halaman daftar `/ingest` (tanpa chat) juga valid —
  chat opsional (O-9); agent tetap menyusun tebakan saat file di-upload (backend yang
  memicu, bukan UI).

### D. Rules (Gate Aturan, O-20) — skema + panel editor + penegakan Go

**Sumber keputusan: O-20 (2026-11-04, owner setuju).** AC-4 (Gate Aturan) dari
CP-2 masih pasif — `rules` hanya tersimpan. WO-03 ini menutupnya: skema, panel
editor, DAN penegakan.

- **Skema rule terstruktur** (per objek, di `ingest_profiles.rules` jsonb, ikut
  versi profil append-only):
  `rule_id` (id stabil, cocok dengan penegak Go, contoh `visit_missing_phone`),
  `target` (plugin target), `violates` (validasi pipeline yang dikecualikan,
  contoh `phone_e164.required`), `action` (`allow` | `skip`), `scope` (`row` |
  `field`), `stated_by`/`stated_at` (bukti Gate Aturan — rule selalu dari
  pernyataan user), `version`.
- **Penegakan deterministik di Go** (pola classifier D-3): evaluasi di SATU titik
  sebelum pemanggilan `ImportTemplate` plugin target. Baris ditolak validasi X →
  cocokkan `violates` rule di profil: cocok + `allow` → lanjut; cocok + `skip` →
  skipped (masuk laporan); tidak ada rule → rejected (fail-closed). `rule_id`
  tidak dikenal penegak → 400 (rule sintetis tidak bisa dieksekusi).
- **Panel rules di editor**: daftar pengecualian berlaku per profil — tambah /
  ubah / hapus via form (bukan JSON mentah); hanya dari daftar pengecualian valid
  (dropdown `violates` yang dikenal penegak); setiap perubahan menulis versi baru
  profil + audit; LLM tidak boleh mengarang rule baru.
- **Daftar pengecualian valid per plugin** = sumber tunggal (file daftar di plugin
  target, pola `fields.yaml`): tiap entri = `violates` id + teks penjelasan +
  opsi action yang diizinkan. Penegak Go dan daftar valid HARUS satu sumber
  (import dari file yang sama) supaya tidak ada rule yang "tidak dikenal" saat
  runtime. Plugin `visit` starter (didasarkan validasi pipeline visit):
  - `phone_e164.required` — baris tanpa HP valid (kasus 177 baris BBH); opsi:
    `allow` (masuk tanpa HP) / `skip` (lewati baris tanpa HP).
  - `phone_format.lenient` — HP format lokal (`0812-...`) diterima (dinormalisasi
    ke E.164 bila bisa).
  - `date_format.custom` — format tanggal file ≠ format pipeline; opsi: deklarasi
    format eksplisit di nilai rule (field `value`).
- **Chat tetap sumber pertama** (CP-2 status quo): agent menyajikan kartu rule,
  jawaban user menyimpan rule ke profil — editor hanya tempat koreksi.

## Acceptance Criteria

- AC-0 Prasyarat CP-3 lulus: berkas asli BBH via chat menghasilkan percakapan
  strict (agent berhenti + bertanya dedup tanpa HP) TERCATAT; re-impor 2×
  idempoten (AC-8 CP-2) — dibuktikan sebelum editor disentuh.
- AC-R1 Rule via chat tampil di editor dan bisa dikoreksi; perubahan menulis
  versi profil baru + audit.
- AC-R2 Baris yang melanggar validasi `violates` yang punya rule `allow` → masuk
  DB; dengan rule `skip` → skipped; tanpa rule → rejected (unit test + Bruno).
- AC-R3 Rule dengan `violates` yang tidak dikenal penegak → 400 (fail-closed).
- AC-R4 Ekstraksi tetap gagal/menolak bila validasi pelanggaran tidak ter-cover
  rule (tidak ada pengecualian diam-diam).
- AC-1 Sesi via chat dibuka di editor: mapping tampil (termasuk yang dikonfirmasi
  di chat); perubahan tersimpan ke profil yang sama (bukan duplikat).
- AC-2 Sesi dibuat langsung dari daftar `/ingest` tanpa chat: tebakan mapping tetap
  dibuat (backend-memicu), editor bisa dipakai sampai ekstraksi penuh.
- AC-3 Flag `is_encrypt` tersimpan, tampil di preview, dan diterapkan saat ekstraksi
  (kolom ber-flag terdeidentifikasi di DB — nilai asli tidak masuk).
- AC-4 Attempt melonggarkan policy PII bawaan → 400; flag hanya menambah proteksi.
- AC-5 Tombol Ekstrak terkunci bila ada kolom ambigu/atruan belum dijawab; pesan
  gate yang belum lolos tampil.
- AC-6 Confirm/extract via editor menghasilkan laporan yang sama dengan jalur chat
  (created/updated/skipped/rejected + batch id) dan idempoten.
- AC-7 RBAC: viewer read-only (PUT/confirm 403); audit terisi; tanpa token 401.
- AC-8 `npm run build` hijau (dist bebas URL backend); `go vet`/`go test` hijau;
  swagger + Bruno untuk endpoint mapping.

## Non-tujuan (CP-4)

- Agent chat menautkan link editor otomatis; sinkronisasi dua arah realtime antara
  percakapan dan editor; konfirmasi dari dalam panel chat.
- Rule baru lintas plugin lain (hsh/sec) — skema disiapkan generik, implementasi
  penegakan per plugin menyusul saat modul jadi target ingest.

## Risiko & catatan

- Jangan biarkan editor menjadi "mode cepat" yang melewati gate — kekuatan editor
  justru membuat gate TERA (terlihat) per kolom.
- Duplikat mapping (dua kolom sumber → satu field) wajib dilarang kecuali keduanya
  dideidentifikasi konsisten — catat aturan ini di validator profil.
- **Daftar `violates` = ketergantungan dua arah** dengan validasi pipeline target —
  kalau pipeline visit berubah, daftar wajib ikut; satu sumber file menurunkan
  risiko desinkron.
- Berkas asli BBH hanya bisa diuji setelah owner mengirim berkas; jangan buat
  fixtures tiruan sebagai pengganti (tujuan AC-0 justru perilaku agent pada data
  kotor nyata).

---

## Rancangan Manager — 2026-11-04 (disetujui owner "ok silahkan")

Rancangan ini difinalisasi mengikuti: (a) pembelajaran CP-2 (integritas state
machine + pola tool API Go terbukti), (b) keputusan O-20 (skema rules),
(c) sisa CP-2 dipindah ke §0. Komposisi kerja per bagian:

| Bagian | Isi | Pemilik dominan |
|---|---|---|
| §0 | Verifikasi berkas asli + AC-8 re-impor | Dev (chat, data owner) |
| §A | API mapping + rules CRUD + penegakan Go | Dev backend |
| §B | UI editor + panel rules + gate indicator | Dev frontend |
| §C | Jembatan sesi chat → editor | Dev backend+frontend |
| §D | Skema rules O-20 + daftar valid + penegakan | Dev backend |

Urutan eksekusi disarankan: §0 → §A+§D (backend) → §B (frontend) → §C (jembatan).
Effort kasar: §0 ±0,5 hari (butuh berkas owner); §A+§D ±2–3 hari; §B ±2–3 hari;
§C ±1 hari.
