---
title: WO-10 — Housekeeping & retensi sesi ingest (bagian D WO-04 + keputusan O-14)
description: 'Pemecahan bagian D WO-04 (pola WO-04a): job retensi periodik Go — sesi non-extracted idle >=90 hari dihapus bersama pesan+file, file upload dihapus 30 hari pasca-extracted, profil extracted TAK TERBATAS; env konfigurabel + audit per penghapusan; menunggu keputusan O-14 (D-T1)'
status: pending
created: "2026-11-19"
updated: "2026-11-19"
tags:
    - ingest
    - housekeeping
    - retention
---
# WO-10 — Housekeeping & retensi sesi ingest

Sumber: usulan resmi Manager **O-14** (2026-11-04, masih menunggu keputusan
owner) + bagian D WO-04 (checkpoint CP-4). Pemecahan mengikuti pola WO-04a
(delete sesi dieksekusi lebih dulu dari WO-04 induk): **bagian retensi
dipisah agar bisa diputuskan & dieksekusi sekarang**, sementara integrasi
editor ↔ CopilotKit (WO-04 §A–§C) tetap jalan setelahnya tanpa terblokir.

## Kondisi saat ini

- Daftar sesi bisa penuh dan tidak pernah dibersihkan — delete manual sudah
  ada (WO-04a, complete), tetapi TIDAK ada pembersihan otomatis.
- File upload tersimpan di `internal/storage` dengan key
  `ingest/uploads/<session>/...` — tidak pernah dihapus otomatis.
- Usulan O-14 (tetap 100% relevan, dari diskusi 2026-11-04):
  1. Sesi non-`extracted` yang idle ≥ **90 hari** → dihapus bersama pesan
     & file-nya.
  2. File upload dihapus **30 hari** setelah ekstraksi sukses (`extracted`)
     — payload aman/deidentify ikut; tidak ada kebutuhan pasca-ekstraksi.
  3. Sesi `extracted`: **profil disimpan TAK TERBATAS** — dipakai ulang
     untuk ekspor format yang sama (inti learn-together; sesi belajar
     pertama mahal).
  4. Implementasi: job periodik Go, env konfigurabel
     (`INGEST_RETENTION_SESSION_DAYS`, `INGEST_RETENTION_UPLOAD_DAYS`,
     `0` = disable), setiap penghapusan tercatat di `ingest_audit`.

## Tujuan

Fitur ingest siap dipakai rutin tanpa menumpuk sampah: sesi terbengkalai dan
berkas upload lama dibersihkan otomatis sesuai kebijakan yang jelas dan
ter-audit, tanpa pernah menyentuh profil `extracted` yang dipakai ulang.

## Lingkup

### A. Backend plugin `ingest` — job retensi periodik

1. **Job periodik** (pola cron/ticker Go bawaan app): jalan dengan interval
   konfigurabel (env, default harian).
2. **Kebijakan sesi** — hapus sesi non-`extracted` yang `updated_at` lebih
   tua dari `INGEST_RETENTION_SESSION_DAYS` (default 90): hard delete
   bersama `ingest_messages` + file upload terkait (pola hard delete
   WO-04a; jangan soft-delete untuk pekerjaan background).
3. **Kebijakan file upload** — untuk sesi `extracted` yang umur
   `extracted_at`/`updated_at` lebih tua dari `INGEST_RETENTION_UPLOAD_DAYS`
   (default 30): hapus file di storage, **profil & pesan TETAP disimpan**.
4. **Profil `extracted` = imun** — tidak pernah dihapus oleh job (O-14.3).
5. **Audit** — setiap penghapusan (sesi & file) baris `ingest_audit`
   (kelas `retention_cleanup`: jumlah sesi/file, rentang, policy versi).
6. **Env & disable** — `0` = disable; default env dikunci lewat D-T1;
   job TIDAK jalan bila env tidak terbaca (fail-closed: skip + log).

### B. Frontend (porsi minimal)

1. Tidak ada perubahan UI wajib — daftar sesi otomatis menampilkan hasil
   cleanup. Bila perlu, indikator kecil "terakhir dibersihkan" boleh
   menyusul (opsional, bukan AC).

## Acceptance Criteria

- AC-1 Sesi non-`extracted` idle ≥ ambang → terhapus (sesi + pesan + file);
  di bawah ambang → tidak tersentuh.
- AC-2 File sesi `extracted` lebih tua dari ambang → file hilang, profil &
  pesan tetap utuh; mapping/profil masih bisa dipakai ulang (buka editor
  profil tetap jalan).
- AC-3 Profil `extracted` TIDAK PERNAH dihapus job (uji: sesi extracted
  sangat tua tetap ada).
- AC-4 `INGEST_RETENTION_SESSION_DAYS=0` & `INGEST_RETENTION_UPLOAD_DAYS=0`
  → job no-op; env invalid → job skip + log (fail-closed).
- AC-5 Setiap penghapusan ter-audit di `ingest_audit` (kelas
  `retention_cleanup`).
- AC-6 `go build/vet/test` hijau (unit + test job dengan fixture waktu);
  `tsc --noEmit` + `npm run build` hijau bila ada perubahan webapp;
  swagger regenerasi bila ada endpoint baru.
- AC-7 **Log Implementasi (Dev)** di dok ini, bertahap per bagian.

## Non-tujuan

- Integrasi editor ↔ CopilotKit (WO-04 §A–§C) — tetap di WO-04 induk.
- Perbaikan Bruno pre-existing (folder `ingest/` 05, 08–14) — follow-up
  terpisah, tidak dicampur.
- Retensi data bisnis plugin lain (visit/resident/sec/hsh master data) —
  hanya artefak sesi ingest.
- Perubahan semantik delete manual WO-04a.

## Risiko & catatan

- **Penghapusan permanen otomatis** — ambang default harus longgar (90/30
  hari) dan semua terekam audit; jangan pernah hapus profil `extracted`.
- Waktu acuan idle = `updated_at` (bukan `created_at`) — sesi yang masih
  dipakai terus tidak akan tersapu.
- Job berjalan di process app (compose satu instance) — invarian sama dengan
  WO-07 (single-instance); multi-instance nanti perlu lock.

## Rancangan Manager — 2026-11-19

| Bagian | Isi | Pemilik dominan |
|---|---|---|
| §A | Job retensi + kebijakan + audit + env | Dev backend |
| §B | (opsional) indikator UI | Dev frontend |

Effort kasar: §A ±1 hari; §B ±0,25 hari (opsional).

## Keputusan yang diperlukan owner

- **D-T1 — Approve usulan O-14 sebagai kebijakan final?** (usulan Manager:
  YA dengan angka default 90 hari sesi / 30 hari file / profil tak
  terbatas; env bisa diubah kapan saja tanpa deploy ulang kode)

## Progres Dev — 2026-11-19

(belum ada — menunggu keputusan D-T1 + persetujuan rancangan)
