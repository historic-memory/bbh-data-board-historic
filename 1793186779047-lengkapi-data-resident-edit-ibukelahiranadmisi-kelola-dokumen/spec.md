---
title: 'SPEC: Lengkapi Data Resident — edit ibu/kelahiran/admisi & kelola dokumen'
description: 'Melengkapi kontrak admin resident: jalur update parsial untuk mother/birth/admission di PATCH /api/admin/residents/:id, plus kelola dokumen (DELETE + PATCH) — beserta penyesuaian UI form/detail. Backend tipis + frontend sub_app/webapp.'
status: complete
created: "2026-10-28"
updated: "2026-10-28"
tags:
    - resident
    - backend
    - contract
    - documents
    - ui
    - spa
---
# SPEC: Lengkapi Data Resident (ibu/kelahiran/admisi & dokumen)

## Ringkasan
Topik sebelumnya (`1793174442984`) adalah **frontend-only**: form resident hanya bisa mengirim
`mother`/`birth`/`admission` saat **create**, karena `UpdateResidentInput` backend **belum punya
field** untuk itu (H-6). Demikian pula dokumen resident hanya bisa **dibaca & ditambah**
(`GET`/`POST /:id/documents`) — belum bisa **dihapus/diedit** (H-7).

Topik ini menutup dua gap fungsional itu:
1. **H-6** — `UpdateResidentInput` diperluas dengan `mother`, `birth`, `admission` (parsial), dan
   `Service.UpdateResident` menanganinya dalam satu transaksi (upsert).
2. **H-7** — endpoint baru `DELETE` dan `PATCH` untuk dokumen resident, plus UI aksi hapus/edit.

Cakupan: **perubahan backend tipis** di `plugins/resident/` + **UI** di `sub_app/webapp`.
Tidak ada fitur baru di `internal/`.

## Kondisi Awal (baseline terverifikasi)
- `plugins/resident/services/service.go:159` — `UpdateResidentInput` **hanya** field inti:
  `full_name`, `gender`, `nik`, `bpjs_no`, `birth_date`, `birth_place`, `religion_code`,
  `blood_type_code`, `mother_id`, `branch_key`. **Tidak** ada `mother`/`birth`/`admission`.
- `Service.UpdateResident` (`query.go:188`) membangun `map[string]any` partial + `nilIfBlank` untuk
  field teks; `CreateResident` (`service.go:261`) **sudah** punya logika membuat baris
  mother/birth_record/admission di satu transaksi — jadi pola upsert bisa mengacu padanya.
- `PATCH /api/admin/residents/:id` (guard `resident.write`) memakai `Patch` handler
  (`handlers.go:185`) yang memanggil `UpdateResident(..., requireCore=false)`; `PUT` `requireCore=true`.
- `resident_documents` (`migrations/postgres/...up.sql:100`): kolom `doc_type`
  (`drive_folder|media|other`, NOT NULL, CHECK), `url` (NOT NULL), `label` (nullable). **Tanpa**
  unique index pada `(resident_id, doc_type)` — boleh >1 dokumen per tipe.
- `mothers` punya `uq_mothers_nik` (partial unique, `WHERE nik IS NOT NULL`).
- `birth_records` **1:1** dengan resident (`resident_id ... UNIQUE`).
- `admissions` **1:N** (index `(resident_id, admitted_at DESC)`).
- Route dokumen saat ini (`plugin.go:74-75`): hanya `GET` & `POST /:id/documents`.
- Model `ResidentDocument` (`models.go:106`): `ID, ResidentID, DocType, URL, Label, CreatedAt, UpdatedAt`.
- UI: `sub_app/webapp/src/api/residents.ts:239` mencatat eksplisit bahwa `resident_code`/`mother`/
  `birth`/`admission` **tidak** ada di payload `PATCHResidentPayload` → form edit tidak menampilkannya.
  `ResidentFormPage.tsx` mengirim `mother`/`birth`/`admission` hanya di jalur create.

## Keputusan Desain
| # | Keputusan | Nilai |
|---|---|---|
| A-1 | Lokasi perubahan | `plugins/resident/` (tips: input/service/handler/route/swagger/Bruno) + `sub_app/webapp/src/`. **Tidak** menyentuh `internal/`. |
| A-2 | Semantik parsial | Mengikuti konvensi existing: pointer `nil` = **tidak diubah**; field teks `""`/blank → **clear** (set NULL) via `nilIfBlank`; field int pointer `nil` = tidak diubah. |
| A-3 | `birth` (1:1) | Bila `birth != nil`: **upsert**. Baris ada → `Updates` hanya field yang disuplai; belum ada → `Create`. Kompatibel CHECK enum `delivery_method_code`/`attendant_type_code` (divalidasi service). |
| A-4 | `admission` (1:N) | Bila `admission != nil`: target baris = admission **terbaru** resident (`ORDER BY admitted_at DESC, created_at DESC`). Ada → update parsial; belum ada → create (`admitted_at` **wajib** untuk create). `exited_at` non-blank → set; `""` → kosongkan (`NULL`). |
| A-5 | `mother` | Bila `mother_id` disuplai → **menang** (attach existing, tanpa menyentuh baris mother). Else bila `mother != nil` & `full_name` non-blank → update baris mother yang ter-link; belum ter-link → create mother + link. `mother` tanpa `full_name` saat belum ada mother → `400`. `nik` duplikat → `409` (via `wrapUnique`). |
| A-6 | Transaksi | Update inti + sub-entitas dijalankan dalam **satu `s.db.Transaction`** (seperti `CreateResident`), agar tidak ada partial-write. |
| A-7 | Permission dokumen | `DELETE`/`PATCH` dokumen memakai **`resident.write`** (konsisten dengan `POST /:id/documents`; belum ada permission dokumen tersendiri). |
| A-8 | Hapus dokumen | **Hard delete** baris `resident_documents` (tidak ada kolom soft-delete). `404` bila `docId` bukan milik `resident_id` tsb. |
| A-9 | Edit dokumen | `PATCH` terima `doc_type`/`url`/`label` parsial. `url` di-set blank → `400` (NOT NULL + wajib). `doc_type` invalid → `400`. |
| A-10 | Kontrak UI | Sumber kebenaran endpoint/field = **swagger** + koleksi Bruno `api-tests/residents/` — UI tidak mengarang kontrak. |
| A-11 | Fail-closed UI | Tombol hapus/edit dokumen & tombol simpan edit mengikuti `resident.write`; tanpa permission → sembunyikan/disable. Server tetap penentu akhir. |
| A-12 | Bahasa | UI Bahasa Indonesia; identifier kode Inggris (warisan topik sebelumnya). |
| A-13 | Breaking change | Payload `PATCH` diperluas **aditif** (field baru opsional) → **tidak** memecah klien lama. Wajib regen swagger + Bruno baru. |
| A-14 | Status & dokumen create | `status` dan `documents` (list) **tetap** hanya jalur create/route terpisah (status append-only AC-4). Tidak disatukan ke PATCH. |

## Perubahan Backend (rincian, batas AC)
1. **`plugins/resident/services/service.go`** — `UpdateResidentInput` + `Mother *MotherInput`,
   `Birth *BirthInput`, `Admission *AdmissionInput`.
2. **`plugins/resident/services/query.go`** — `UpdateResident` memproses ketiganya dalam transaksi
   (helper kecil: `upsertBirth`, `upsertAdmission`, `upsertMother` atau inline sesuai pola existing).
3. **`plugins/resident/services/query.go`** — `DeleteDocument(residentID, docID) error` &
   `UpdateDocument(residentID, docID, in UpdateDocumentInput) (*model.ResidentDocument, error)`.
4. **`plugins/resident/services/service.go`** — `UpdateDocumentInput` (pointer/string parsial).
5. **`plugins/resident/handlers/handlers.go`** — handler `DeleteDocument`, `PatchDocument` + swagger.
6. **`plugins/resident/plugin.go`** — route
   `DELETE /:id/documents/:docId` & `PATCH /:id/documents/:docId` (guard `resident.write`).
7. **`docs/swagger/*`** — regen (dijalankan di container `app` via `swag init`).
8. **`api-tests/residents/`** — Bruno request baru (patch sub-entitas, delete/patch dokumen).

## Cakupan Fungsional
- Edit resident: memperbarui `mother`, `birth`, `admission` setelah resident dibuat.
- Mengosongkan field teks sub-entitas (mis. `birth_place` mother) lewat nilai blank.
- Hapus dokumen resident; edit `label`/`url`/`doc_type` dokumen.
- UI form edit menampilkan seksi Ibu/Kelahiran/Admisi (terisi dari data existing) & mengirim delta.
- UI detail: tombol hapus/edit pada tabel dokumen.

## Non-Goals
- Tidak menambah soft-delete / audit trail untuk dokumen.
- Tidak mengubah semantik status (**tetap append-only**, tanpa edit/hapus).
- Tidak menyatukan `status`/`documents` ke dalam `PATCH /:id`.
- Tidak menyentuh importer (`1793109556782`) atau plugin lain.
- Tidak menambah permission baru (`resident.document.*`) — pakai `resident.write`.
- Tidak mengerjakan housekeeping lain (H-2/H-3/H-4/H-8/H-9/H-10) di topik ini.

## Gap yang Diketahui
| GAP | Deskripsi | Rencana |
|---|---|---|
| GAP-1 | `UpdateResidentInput` tanpa `mother`/`birth`/`admission` → tidak bisa diubah setelah create (H-6) | Diselesaikan **WO-01** |
| GAP-2 | Dokumen hanya `GET`/`POST`; tidak ada `DELETE`/`PATCH` (H-7) | Diselesaikan **WO-02** |
| GAP-3 | UI form edit tidak menampilkan seksi ibu/kelahiran/admisi & tanpa aksi dokumen | Diselesaikan **WO-03** |
| GAP-4 | Swagger/Bruno belum memuat kontrak baru | Diselesaikan **WO-04** |
| GAP-5 | `admissions` 1:N — pilihan "baris terbaru" adalah konvensi UI (belum ada penanda "aktif"); diterima sebagai keputusan A-4, didokumentasikan |
| GAP-6 | Verifikasi cutoff permission memakai root (`*`) → jalur 403 sub-entitas tidak teruji end-to-end (H-8) | Di luar lingkup; dicatat |

## Kriteria Penerimaan (SPEC-level)
- **AC-1** — `PATCH /api/admin/residents/:id` menerima `mother`/`birth`/`admission` dan memperbarui
  baris terkait (mother ter-link / birth_record / admission terbaru) tanpa menyentuh field yang tak disuplai.
- **AC-2** — Field teks sub-entitas yang di-set blank menjadi `NULL` (bukan diabaikan); `nil` = tidak berubah.
- **AC-3** — `birth`/`admission` belum ada → dibuat; `admitted_at` wajib saat create admission (else `400`).
- **AC-4** — Update berjalan atomik (satu transaksi); gagal → tidak ada partial-write.
- **AC-5** — `DELETE /api/admin/residents/:id/documents/:docId` → `204`; doc bukan milik resident → `404`.
- **AC-6** — `PATCH .../documents/:docId` update parsial; `url` blank → `400`; `doc_type` invalid → `400`; sukses → `200` dokumen.
- **AC-7** — Kedua route dokumen ter-guard `resident.write`; tanpa token → `401`, tanpa permission → `403`.
- **AC-8** — UI form edit menampilkan & mengirim seksi Ibu/Kelahiran/Admisi; aksi dokumen (hapus/edit)
  tersedia bila `resident.write`; sukses menampilkan toast & refresh data.
- **AC-9** — Swagger (`docs/swagger/*`) & Bruno (`api-tests/residents/`) memuat kontrak baru.
- **AC-10** — `go test ./plugins/resident/...` lulus (termasuk test baru untuk upsert & dokumen);
  `npm run build` di HOST lulus tanpa error TS.
- **AC-11** — Tanpa mengubah `internal/`; tanpa push ke remote.

## Risiko
- **Admission "terbaru"** bisa salah sasaran bila ada >1 admission; dimitigasi konvensi A-4 + dokumentasi.
- **Mother NIK duplikat** → `409`; UI harus menampilkan pesan yang jelas (bukan crash).
- Perubahan kontrak menuntut regen swagger & Bruno agar tidak drift (A-10).
