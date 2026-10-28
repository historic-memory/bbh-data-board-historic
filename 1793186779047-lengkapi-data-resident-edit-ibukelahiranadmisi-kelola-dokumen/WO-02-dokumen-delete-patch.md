---
title: 'WO-02: Dokumen resident — DELETE & PATCH'
description: H-7 — tambah service/handler/route untuk menghapus dan mengedit dokumen resident (parsial), ter-guard resident.write.
status: planned
created: "2026-10-28"
updated: "2026-10-28"
tags:
    - work-order
    - resident
    - backend
    - documents
---
# WO-02: Dokumen resident — DELETE & PATCH (H-7)

## Referensi
- SPEC: A-7..A-9, AC-5..AC-7, GAP-2.
- `plugins/resident/services/query.go:322` (`ListDocuments`), `:332` (`AddDocument`).
- `plugins/resident/handlers/handlers.go:267-305` (documents handlers + swagger annotations).
- `plugins/resident/plugin.go:74-75` (route).
- Tabel `resident_documents` (migration `..._create_resident_core_tables.up.sql`).

## Tugas
1. **`UpdateDocumentInput`** di `services/service.go` (parsial, semua pointer):
   ```go
   type UpdateDocumentInput struct {
       DocType *string `json:"doc_type"`
       URL     *string `json:"url"`
       Label   *string `json:"label"`
   }
   ```
2. **`Service.DeleteDocument(residentID, docID string) error`**:
   - `Delete` dengan syarat `id = docID AND resident_id = residentID`; `RowsAffected == 0` → `ErrNotFound`.
   - Hard delete (A-8).
3. **`Service.UpdateDocument(residentID, docID string, in UpdateDocumentInput) (*model.ResidentDocument, error)`**:
   - Ambil baris (`id`+`resident_id`); tidak ada → `ErrNotFound`.
   - `doc_type != nil` → `validateEnum` terhadap `ValidDocumentTypes`; `""` → `400`.
   - `url != nil` → trim; blank → `400`. 
   - `label != nil` → `nilIfBlank`.
   - `updated_at = now`; kembalikan baris terbaru.
4. **Handler** (`handlers.go`): `DeleteDocument` (204) & `PatchDocument` (200) + anotasi swagger
   (`@Router /api/admin/residents/{id}/documents/{docId} [delete|patch]`, `@Security BearerAuth`).
5. **Route** (`plugin.go`), urutan setelah `/:id/documents`:
   ```go
   residents.DELETE("/:id/documents/:docId", middleware.RequirePermission(authSvc, "resident.write"), h.DeleteDocument)
   residents.PATCH("/:id/documents/:docId", middleware.RequirePermission(authSvc, "resident.write"), h.PatchDocument)
   ```
6. Pastikan `:docId` tidak bentrok dengan route lain (gin tree) — taruh di grup `/:id/documents`.

## Test
- `service_db_test.go`: delete dokumen milik resident lain → `ErrNotFound`; delete sukses; patch label saja
  (url tetap); patch `url` blank → `ErrInvalid`; patch `doc_type` invalid → `ErrInvalid`.
- Handler test (opsional): status code 204/200/400/404.

## Kriteria Selesai
- AC-5, AC-6, AC-7 terpenuhi.
- `go build ./...` + `go test ./plugins/resident/...` lulus.
- Swagger memuat kedua endpoint setelah WO-04.
