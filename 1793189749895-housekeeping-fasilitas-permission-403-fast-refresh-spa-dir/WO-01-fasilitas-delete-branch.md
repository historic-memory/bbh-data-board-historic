---
title: "WO-01: Fasilitas — DELETE & branch_key"
description: "Tambah DELETE /api/admin/facilities/:id (guard resident.delete) dan branch_key pada UpdateFacility/PUT; plus UI tombol hapus & field cabang aktif. Menutup H-9 & H-10."
status: pending
created: "2026-10-28"
updated: "2026-10-28"
tags:
  - work-order
  - resident
  - facilities
  - backend
---

# WO-01: Fasilitas — DELETE & branch_key (H-9, H-10)

## Referensi
- SPEC: A-1..A-3, A-9, AC-1..AC-4, AC-8, GAP-1, GAP-2.
- `plugins/resident/plugin.go` (grup `facilities`), `services/query.go:725` (`UpdateFacility`),
  `services/facility.go`, `handlers/handlers.go` (facility handlers).
- Skema: `facility_aliases (ON DELETE CASCADE)`, `birth_records.facility_id (ON DELETE SET NULL)`.
- UI: `sub_app/webapp/src/api/facilities.ts`, `pages/admin/facilities/*`.

## Tugas (backend)
1. **`Service.DeleteFacility(id string) error`**: hard delete; `RowsAffected == 0` → `ErrNotFound`.
   Sebelum hapus, hitung `birth_records` yang menunjuk fasilitas (untuk info dampak).
2. **Handler `DeleteFacility`** → `204`; anotasi swagger (`@Router /api/admin/facilities/{id} [delete]`,
   `@Security BearerAuth`).
3. **Route**: `facilities.DELETE("/:id", middleware.RequirePermission(authSvc, "resident.delete"), h.DeleteFacility)`.
4. **`UpdateFacility`** += parameter `branchKey *string`:
   - `nil` → tidak diubah.
   - `""`/blank → set `branch_key = NULL` (cabang dikosongkan).
   - nilai lain → `resolveBranchKey`; tak dikenal → `400`.
5. **Handler `UpdateFacility`** membaca `branch_key` dari body (pointer) dan meneruskannya.

## Tugas (test)
- DB test: hapus fasilitas → `404` untuk id tak ada; alias hilang; `birth_records.facility_id` → `NULL`.
- DB test: `UpdateFacility` `branch_key` (`nil`/`""`/valid/invalid).

## Tugas (UI, `sub_app/webapp`)
- `api/facilities.ts`: tambah `deleteFacility(id)` + `branch_key` pada payload update.
- `FacilityListPage.tsx`: tombol **Hapus** (permission-aware, modal konfirmasi, tampilkan jumlah
  `birth_records` terdampak bila tersedia) + toast.
- `FacilityFormModal.tsx`: field **Cabang** aktif (tidak disabled) di mode edit; kirim `""` untuk
  mengosongkan.

## Kriteria Selesai
- AC-1, AC-2, AC-3, AC-4, AC-8.
- `go build ./...` + `go test ./plugins/resident/...` lulus; `npm run build` lulus.
