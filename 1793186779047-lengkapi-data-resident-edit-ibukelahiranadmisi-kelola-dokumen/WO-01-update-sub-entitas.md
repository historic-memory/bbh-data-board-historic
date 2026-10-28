---
title: 'WO-01: Update parsial mother/birth/admission'
description: H-6 — perluas UpdateResidentInput dan Service.UpdateResident agar dapat mengubah mother/birth/admission secara parsial dalam satu transaksi.
status: complete
created: "2026-10-28"
updated: "2026-10-28"
tags:
    - work-order
    - resident
    - backend
---
# WO-01: Update parsial mother/birth/admission (H-6)

## Referensi
- SPEC: A-2..A-6, AC-1..AC-4, GAP-1.
- `plugins/resident/services/service.go:159` (`UpdateResidentInput`).
- `plugins/resident/services/query.go:188` (`UpdateResident`), pola di `service.go:261` (`CreateResident`).
- Model: `plugins/resident/model/models.go` (`Mother`, `BirthRecord`, `Admission`).

## Tugas
1. Tambah field ke `UpdateResidentInput`:
   ```go
   Mother    *MotherInput    `json:"mother"`
   Birth     *BirthInput     `json:"birth"`
   Admission *AdmissionInput `json:"admission"`
   ```
2. Ubah `UpdateResident` agar menjalankan update inti + sub-entitas dalam **satu `s.db.Transaction`**:
   - **Mother** (A-5): `mother_id` disuplai → menang. Else `mother != nil`:
     - jika resident sudah punya `mother_id` → `Updates` parsial baris mother (`full_name` jika non-blank,
       `nik`/`phone`/`address` via `nilIfBlank`).
     - jika belum → butuh `full_name` non-blank (else `400`), `Create` mother, lalu set `residents.mother_id`.
   - **Birth** (A-3, upsert 1:1): `Updates` parsial untuk field non-`nil` (int pointer) & `nilIfBlank`
     untuk teks; belum ada baris → `Create`.
   - **Admission** (A-4): pilih baris terbaru (`admitted_at DESC, created_at DESC`); ada → `Updates`
     parsial (`admitted_at` bila non-blank, `exited_at` set/clear, `note` via `nilIfBlank`); belum ada →
     `admitted_at` wajib (else `400`), `Create`.
3. Validasi enum yang sama dengan `CreateResident`: `delivery_method_code`, `attendant_type_code`
   (birth) — tolak dengan `400` bila invalid.
4. `wrapUnique` dipakai agar NIK mother duplikat → `409`.
5. Kembalikan `s.GetResident(id)` setelah transaksi (konsisten handler existing).

## Test
- `service_db_test.go` (butuh DB): 
  - PATCH `birth` parsial → hanya field yang disuplai berubah; create saat belum ada.
  - PATCH `admission` `exited_at` set lalu clear; create butuh `admitted_at`.
  - PATCH `mother` update ter-link; create mother baru + link; NIK duplikat → `ErrConflict`/`409`.
  - Blank teks → `NULL`; field tak disuplai tetap.
  - `mother_id` menang atas `mother`.

## Kriteria Selesai
- AC-1, AC-2, AC-3, AC-4 terpenuhi.
- `go build ./...` & `go test ./plugins/resident/...` (container `app`) lulus.
- Swagger struct ter-update saat `swag init` (dijalankan di WO-04).

## Catatan Implementasi (2026-10-28)

- Payload PATCH memakai tipe pointer khusus `UpdateMotherInput` / `UpdateBirthInput` /
  `UpdateAdmissionInput` (bukan `MotherInput`/`BirthInput`/`AdmissionInput`). Alasan: tipe create
  memakai `string` biasa sehingga `null`/tidak dikirim vs `""` (clear) tidak bisa dibedakan.
  Nama field JSON tetap sama → kontrak **aditif** (A-13).
- Precedence `mother_id` > `mother`; birth = upsert 1:1; admission = baris terbaru
  (`admitted_at DESC, created_at DESC`).
- Semua tulisan berada dalam satu `s.db.Transaction`; validasi enum/teks dijalankan di dalam
  transaksi sehingga kegagalan sub-entitas membatalkan seluruh update (AC-4).
- Verifikasi (container `app`): `go build ./...`, `go vet ./plugins/resident/...`,
  `go test ./plugins/resident/...` lulus. 4 test DB baru PASS (bukan skip):
  birth upsert/partial/clear, admission set-clear-latest, mother update/create/dup NIK/precedence,
  rollback atomik.
- Kode: `c5f7b17`.
