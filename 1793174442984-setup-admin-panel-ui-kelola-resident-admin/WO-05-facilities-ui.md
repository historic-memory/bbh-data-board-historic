---
title: "WO-05: UI Fasilitas"
description: "Halaman fasilitas: tabel facilities (filter cabang, tipe, status aktif) + form create/edit ringan, serta integrasi dropdown fasilitas pada form resident (WO-04)."
status: complete
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - ui
  - facilities
---

# WO-05: UI Fasilitas

## Referensi
- SPEC: A-5, A-13, AC-4.
- Routes: `GET /api/admin/facilities?branch_key=`, `POST /api/admin/facilities`,
  `PUT /api/admin/facilities/:id` (guard `resident.read` / `resident.write`).
- Model `Facility{id, branch_key?, name, facility_type?, is_active, created_at, updated_at}`
  dan `FacilityAlias{facility_id, alias}`.
- Enum `facility_type`: `hospital`, `clinic`, `midwife_practice`, `foundation`, `other`.

## Langkah
1. `src/features/facilities/types.ts` + api helper (`listFacilities`, `createFacility`,
   `updateFacility`).
2. Halaman `/facilities`: tabel (nama, tipe badge, cabang, aktif), pencarian lokal
   (API tidak punya `q`), filter cabang via `?branch_key=`.
3. Form create/edit: `name` (wajib, unik), `facility_type` (select), `branch_key`
   (kosong = global/lintas cabang), `is_active` (toggle). Catat: API **tidak**
   punya endpoint alias — tampilkan catatan bahwa alias dikelola via importer.
4. Integrasi WO-04: komponen `FacilitySelect` (searchable) dipakai pada field
   `birth.facility_id`; hanya fasilitas `is_active` yang ditampilkan, default
   difilter ke cabang resident bila `branch_key` terisi.
5. Tombol mutasi disembunyikan bila tak punya `resident.write`.

## Kriteria Penerimaan
- [AC-1] Tabel menampilkan fasilitas nyata; filter `branch_key` berfungsi.
- [AC-2] Create/edit memvalidasi `name` wajib; duplikat (bila 409 dari server)
  ditampilkan ramah.
- [AC-3] `is_active=false` tidak muncul di `FacilitySelect` form resident.
- [AC-4] Tidak ada tombol mutasi untuk viewer (`resident.write` absen).
- [AC-5] `npm run build` lulus.

## Bukti
Screenshot tabel + form, uji dropdown fasilitas di form resident.

## Catatan
Endpoint alias tidak tersedia; jangan mengarang UI untuk itu.

## Hasil Implementasi (commit `71c5ac2`)
- Berkas: `src/api/facilities.ts`, `pages/admin/facilities/{FacilityListPage,FacilityFormModal}.tsx`,
  `pages/admin/residents/FacilitySelect.tsx`, integrasi di `ResidentFormPage.tsx`, rute di `App.tsx`.
- Gate lulus: tabel 4 fasilitas; filter `branch_key=local` → 1 baris; pencarian lokal "Bidan" → 1 baris;
  create 201; edit (nonaktifkan) 200; duplikat → 409 pesan Indonesia.
- `FacilitySelect` hanya menampilkan fasilitas `is_active` + global; resident tersimpan dengan
  `birth.facility_id` (terverifikasi).
- Viewer: 0 tombol aksi, 0 `dialog`.
- **Batasan jujur:** (a) API tidak punya `DELETE /facilities/:id` → data uji dibersihkan via SQL (H-9);
  (b) `PUT /facilities/:id` tidak menerima `branch_key` → field cabang **disabled** di mode edit (H-10);
  (c) UI alias tidak dibuat karena endpoint tidak ada.

## Laporan Implementasi (2026-10-28)

### File
| Berkas | Isi |
|---|---|
| `src/api/facilities.ts` (baru) | `Facility`/`FacilityInput`/`FacilityUpdate` + `listFacilities`, `createFacility`, `updateFacility`, `listActiveFacilities` |
| `src/pages/admin/facilities/FacilityListPage.tsx` (baru) | Tabel + filter cabang (API) + pencarian lokal |
| `src/pages/admin/facilities/FacilityFormModal.tsx` (baru) | Form create/edit dalam modal |
| `src/pages/admin/residents/FacilitySelect.tsx` (baru) | Dropdown fasilitas untuk `birth.facility_id` |
| `src/pages/admin/residents/ResidentFormPage.tsx` (ubah) | Pakai `FacilitySelect`; `facilityId` masuk payload `birth` |
| `src/App.tsx` (ubah) | `/facilities` memakai halaman nyata (placeholder dihapus) |

`internal/` & `plugins/` 0 baris.

### Catatan implementasi
1. **Struktur** mengikuti konvensi repo (`src/api/*` + `src/pages/admin/*`), bukan
   `src/features/facilities/` seperti di rencana awal — sama seperti WO-06.
2. **`PUT` tidak menerima `branch_key`.** Backend hanya mengizinkan ubah
   `name`/`facility_type`/`is_active`. Karena itu di mode edit field cabang
   ditampilkan **read-only + disabled** dengan keterangan “Cabang tidak dapat
   diubah” — bukan input yang diam-diam diabaikan.
3. **Fasilitas baru selalu `is_active=true`** (backend meng-hardcode); toggle
   aktif hanya muncul di mode edit.
4. **Filter cabang dikirim ke API**, pencarian nama lokal — endpoint tidak punya `q`.
5. **`FacilitySelect`**: hanya fasilitas aktif; bila cabang resident terisi,
   fasilitas cabang itu + fasilitas **global** yang ditampilkan. Pilihan lama yang
   sudah nonaktif tetap dimunculkan agar data lama tidak “hilang” saat form dibuka.
6. **Alias tidak dibuat** (sesuai catatan WO) — ada teks eksplisit di halaman & form.
7. **Tidak ada endpoint DELETE fasilitas**; data uji dibersihkan langsung via SQL
   (dicatat di bagian gate).

### Gate (browser, `http://100.125.175.34:5173/app/facilities`, data nyata)
Fasilitas uji dibuat lewat UI/API lalu **dihapus via SQL** (tidak ada endpoint
DELETE); resident uji & admin viewer dihapus lewat API. DB kembali bersih.

| # | Uji | Hasil |
|---|---|---|
| 1 | Tabel | 4 baris; badge tipe (`Praktik bidan`, `Rumah sakit`), cabang `Global`/`Instance ini`, status `Aktif`/`Nonaktif` |
| 2 | Filter cabang `local` | 1 baris (hanya `RSUD Uji WO05`) — parameter `?branch_key=local` |
| 3 | Pencarian `Bidan` | 1 dari 4 baris, tanpa request tambahan |
| 4 | Create nama duplikat | **409** → “Nama fasilitas sudah dipakai. Gunakan nama lain.” (modal tetap terbuka) |
| 5 | Create baru | 201 → muncul di tabel, badge `Aktif` |
| 6 | Edit → nonaktifkan | `PUT` 200; baris berubah jadi `Nonaktif` |
| 7 | Edit: field cabang | `disabled`, nilai `Global (semua cabang)` — membuktikan cabang tidak bisa diubah |
| 8 | `FacilitySelect` di form resident | `RSUD Uji WO05` (cabang) + `Praktik Bidan Uji WO05` (global) tampil; fasilitas **nonaktif tidak muncul** (AC-3) |
| 9 | Simpan resident dengan fasilitas | POST body memuat `birth.facility_id`; detail menampilkan `Fasilitas (ID)` yang sama |
| 10 | Viewer tanpa `resident.write` | `main button` = `[]` (kolom Aksi `—`), `dialog` = 0 |
| 11 | `npm run build` | hijau (133 modul) |
| 12 | Stateless | grep host:port di `dist` → kosong |

### Belum dikerjakan
- Pengelolaan **alias** fasilitas (endpoint tidak ada).
- Hapus fasilitas (endpoint tidak ada) — perlu WO backend bila dibutuhkan.
- Memindahkan fasilitas antar cabang (PUT tidak menerima `branch_key`).
