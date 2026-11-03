---
title: WO-03 — Excel mapping editor (berdiri sendiri)
description: "CP-3: UI editor mapping kolom (React, tanpa library), flag is_encrypt, preview, tombol Ekstrak terkunci; API profil mapping"
status: pending
created: "2026-11-03"
---
# WO-03 — Excel Mapping Editor, Berdiri Sendiri (Checkpoint 3)

Checkpoint: **CP-3**. Prasyarat: CP-2 (WO-02) selesai — state machine, deidentify,
dan ekstraksi via chat sudah hidup; editor tinggal menjadi UI alternatif untuk
membentuk/mengubah dokumen profil. Sumber: `spec.md` §6.2 (editor), D-5/D-8;
`spec-techstack.md` §2.5.

## Tujuan

Menyediakan UI visual untuk menyunting mapping kolom — menggantikan penulisan mapping
lewat teks di chat. Pada checkpoint ini editor **berdiri sendiri** (bisa dipakai tanpa
chat, O-9); integrasi dua arah dengan CopilotKit baru di CP-4.

## Lingkup

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

## Acceptance Criteria

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

## Risiko & catatan

- Jangan biarkan editor menjadi "mode cepat" yang melewati gate — kekuatan editor
  justru membuat gate TERA (terlihat) per kolom.
- Duplikat mapping (dua kolom sumber → satu field) wajib dilarang kecuali keduanya
  dideidentifikasi konsisten — catat aturan ini di validator profil.
