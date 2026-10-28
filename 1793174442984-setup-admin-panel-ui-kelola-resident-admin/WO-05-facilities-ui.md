---
title: "WO-05: UI Fasilitas"
description: "Halaman fasilitas: tabel facilities (filter cabang, tipe, status aktif) + form create/edit ringan, serta integrasi dropdown fasilitas pada form resident (WO-04)."
status: draft
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
